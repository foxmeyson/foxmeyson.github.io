---
layout: post
title: Injecting secrets directly into Pods and Gitlab from Hashicorp Vault in EKS/K8s.
excerpt_image: /assets/images/posts/vault-in-EKS/vault-loader-dark.gif
categories: Kubernetes
tags: [Kubernetes, Vault]
---

![banner](/assets/images/posts/vault-in-EKS/vault-loader-dark.gif)

In this post, I'll show you how to deploy Vault in EKS/K8s (there are some minor differences, but the workflow is very similar) and use DynamoDB as a backend, as well as how to inject secrets directly into a pod without using K8s Secrets (more details: [Vault Agent Injector](https://developer.hashicorp.com/vault/docs/platform/k8s/injector)). And then I'll tell you how to use it to inject secrets into the Gitlab pipeline.

So, the moment has come, you've decided on a secret storage solution and chosen Hashicorp Vault. This is a good choice (at the very least, it's cheaper than AWS Secrets Manager 😊). The next step is to determine which backend to use for Hashicorp Vault.
Whichever backend you choose, keep in mind that it may or may not have HA properties (don't confuse this with Vault's own HA - a separate feature that allows it to create a cluster across multiple nodes).
Each backend has its strengths and weaknesses, such as: support from Hashicorp or the community, HA capability, cost, vendor lock-in, access timeout, backups, and so on. More details here: [Vault backend options](https://developer.hashicorp.com/vault/docs/configuration/storage/aerospike). There's a lot of information that's beyond the scope of this post, so let's move on.

> Note that DynamoDB has fairly low rate limits, so it won't be suitable for everyone, but the deployment for a different backend will only differ by a few lines of configuration.

## Prepare for deploy in K8s

```bash
kubectl create namespace hashicorp-vault-prod
helm repo add hashicorp https://helm.releases.hashicorp.com
helm search repo hashicorp/vault
```

Create an `override-values.yaml` file. This will include all our parameters for creating the Vault release - this is one of the most important steps:

```bash
# this values file is prepared for deployment with TLS disabled for internal traffic (within the K8s cluster)
# I'll explain below how to enable TLS and what you need for that

global:
  enabled: true
  tlsDisable: true 
injector:
  enabled: true
  metrics:
    enabled: true
  nodeSelector:
    nodegroup: hashicorp-vault-nodes
  port: 8080
  agentDefaults:
    cpuLimit: 500m
    cpuRequest: 250m
    memLimit: 128Mi
    memRequest: 64Mi
server:
  enabled: '-'
  standalone:
    enabled: false
  auditStorage:
    enabled: true
    accessMode: ReadWriteOnce
    mountPath: /vault/audit
    size: 10Gi
  dataStorage:
    enabled: false
  nodeSelector:
    nodegroup: hashicorp-vault-nodes
  extraEnvironmentVars:
    VAULT_CACERT: ""
  # extraEnvironmentVars:
  #   VAULT_CACERT: /vault/userconfig/vault-server-tls/vault.ca
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::$ACCOUNT_ID:role/hashicorp-vault-role # for EKS + IAM only
    create: true
  ha:
    enabled: true
    replicas: 3
    config: |
      ui = true

      listener "tcp" {
        tls_disable = 1
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        # if tls is enabled
        # tls_cert_file = "/vault/userconfig/vault-server-tls/vault.crt"
        # tls_key_file = "/vault/userconfig/vault-server-tls/vault.key"
        # tls_ca_cert_file = "/vault/userconfig/vault-server-tls/vault.ca"
      }

      # For internal ssl vault <-> injector:
      # listener "tcp" {
      #   tls_disable = 0
      #   address = "[::]:8202"
      #   cluster_address = "[::]:8201"
      # }

      storage "dynamodb" {
        ha_enabled = "true"
        region = "$REGION"
        table = "$DYNAMODB_TABLE"
      }

      seal "awskms" {
        region     = "eu-west-1"
        kms_key_id = "$KMS_KEY_ID"
        # no need now: endpoint   = "https://vpce-xxxxxxxxxxxxxxx.kms.eu-west-1.vpce.amazonaws.com"
      }

      service_registration "kubernetes" {}
    disruptionBudget:
      enabled: true
      maxUnavailable: null
  ingress:
    enabled: true
    activeService: true
    annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/cluster-issuer: "letsencrypt"
      nginx.ingress.kubernetes.io/rewrite-target: "/"  
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    ingressClassName: nginx
    labels: {}
    pathType: Prefix
    tls:
      - hosts:
          - vault.example.com
        secretName: $TLS_SECRET_NAME
    hosts: 
      - host: vault.example.com
ui:
  enabled: true
  serviceType: "ClusterIP"
  externalPort: 8202
  targetPort: 8202
```

## Setup on the AWS side

### Create DynamoDB

You can do this with Terragrunt:

```bash
locals {
  environment = "production"
}

terraform {
  source = "tfr:///terraform-aws-modules/dynamodb-table/aws?version=4.2.0"
}

remote_state {
  backend = "s3"
  generate = {
    path      = "backend.tf"
    if_exists = "overwrite_terragrunt"
  }
  config = {
    bucket = "$BUCKET_FOR_BACKEND"

    key = "${local.environment}/dynamodb/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "terraform-locks"
  }
}

inputs = {
  name          = "${local.environment}-vault-hashicorp-backend"
  hash_key      = "Path"
  billing_mode   = "PAY_PER_REQUEST"

  attribute = [
    {
      name = "Path"
      type = "S"
    }
  ]

  tags = {
      ...
    }
}
```

### Setup AWS IAMs + KMS

Next, we'll configure:

- KMS key for [auto unseal vault](https://developer.hashicorp.com/vault/tutorials/auto-unseal),
- IAM role trust policy,
- Permission for KMS key

```bash
#!/bin/bash

export CLUSTER_NAME=...
export ACCOUNT_ID=$(aws sts get-caller-identity --query Account --output text)
export OIDC_ID=$(aws eks describe-cluster --name $CLUSTER_NAME --query "cluster.identity.oidc.issuer" --output text | sed 's|https://||')
export SERVICEACCOUNT_NAME=hashicorp-vault
export KMS_KEY_ID=$(aws kms create-key --description "Hashicorp Vault Encryption Key" --region eu-west-1 --query "KeyMetadata.KeyId" --output text)


# Creating IAM role trust policy
cat <<EOF | envsubst | aws iam create-role --role-name hashicorp-vault-role --assume-role-policy-document file://-
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Principal": {
        "Federated": "arn:aws:iam::$ACCOUNT_ID:oidc-provider/$OIDC_ID"
      },
      "Action": "sts:AssumeRoleWithWebIdentity",
      "Condition": {
        "StringEquals": {
          "$OIDC_ID:sub": [
            "system:serviceaccount:$SERVICEACCOUNT_NAME-prod:$SERVICEACCOUNT_NAME-prod"
          ]
        }
      }
    }
  ]
}
EOF

cat <<EOF | envsubst | aws iam create-policy --policy-name VaultKMSDynamoDBPolicy --policy-document  file://-
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Sid": "AllowKMS",
            "Effect": "Allow",
            "Action": [
                "kms:Decrypt",
                "kms:Encrypt",
                "kms:DescribeKey"
            ],
            "Resource": "arn:aws:kms:eu-west-1:$ACCOUNT_ID:key/$KMS_KEY_ID"
        },
        {
            "Sid": "AllowDynamoDB",
            "Effect": "Allow",
            "Action": [
                "dynamodb:DescribeLimits",
                "dynamodb:DescribeTimeToLive",
                "dynamodb:ListTagsOfResource",
                "dynamodb:DescribeReservedCapacityOfferings",
                "dynamodb:DescribeReservedCapacity",
                "dynamodb:ListTables",
                "dynamodb:BatchGetItem",
                "dynamodb:BatchWriteItem",
                "dynamodb:CreateTable",
                "dynamodb:DeleteItem",
                "dynamodb:GetItem",
                "dynamodb:GetRecords",
                "dynamodb:PutItem",
                "dynamodb:Query",
                "dynamodb:UpdateItem",
                "dynamodb:Scan",
                "dynamodb:DescribeTable"
            ],
            "Resource": [
                "arn:aws:dynamodb:eu-west-1:$ACCOUNT_ID:table/production-vault-hashicorp-backend"
            ]
        }
    ]
}
EOF

aws iam attach-role-policy --role-name hashicorp-vault-role --policy-arn arn:aws:iam::$ACCOUNT_ID:policy/VaultKMSDynamoDBPolicy
```

## Deploy Vault in cluster

```bash
helm install prod-hashicorp-vault hashicorp/vault --namespace prod-hashicorp-vault --create-namespace -f ./override-values.yaml

# and IRSA in EKS for access to DynamoDB:
kubectl annotate serviceaccount prod-hashicorp-vault -n prod-hashicorp-vault \
  eks.amazonaws.com/role-arn=arn:aws:iam::$ACCOUNT_ID:role/hashicorp-vault-role
```

At this point, you should see a StatefulSet in your cluster with the number of pods you defined in the `override-values.yaml` manifest under `ha: replicas:` and also a ReplicaSet with the number of pods for the Injector, like this:

```bash
k get pods -n prod-hashicorp-vault
NAME                                                   READY   STATUS    RESTARTS   AGE
prod-hashicorp-vault-0                                 1/1     Running   0          2m
prod-hashicorp-vault-1                                 1/1     Running   0          2m
prod-hashicorp-vault-2                                 1/1     Running   0          2m
prod-hashicorp-vault-agent-injector-7c6c7f7fc4-2hkph   1/1     Running   0          2m
```

## Init Vault

```bash
kubectl exec -n prod-hashicorp-vault -it prod-hashicorp-vault-0 -- /bin/sh
vault status

# Make sure the output shows:
# Initialized = false
# Sealed = true
# Storage Type = dynamodb
# HA Enabled = true
vault operator init

# Save all tokens from the initialization output in a secure place. Losing them will render the vault inoperable!
# You can use AWS Secrets Manager for this by storing the entire output in a single secret.

vault status
# Make sure the output shows:
# Recovery Seal Type = shamir
# Initialized = true
# Sealed = false
# Storage Type = dynamodb
# HA Enabled = true
# and that the network address of the Vault cluster is from your K8s
```

At this point, you have a working Vault cluster and an injector agent for it.

## Injection secrets into Pods

Let's look at injecting secrets into Pods. For this, we'll create a test secret in Vault. Don't use [cubbyhole](https://developer.hashicorp.com/vault/docs/secrets/cubbyhole)!

```bash
kubectl exec -n prod-hashicorp-vault -it prod-hashicorp-vault-0 -- /bin/sh
vault login $MAIN_TOKEN
vault audit enable

vault kv put my-kv/my-secret token=my-token password=my-password
vault kv get my-kv/my-secret
```

Let's create an integration with K8s

```bash
vault auth enable kubernetes
vault write auth/kubernetes/config \
  kubernetes_host="https://$KUBERNETES_PORT_443_TCP_ADDR:443" \
  token_reviewer_jwt="$(cat /var/run/secrets/kubernetes.io/serviceaccount/token)" \
  kubernetes_ca_cert=@/var/run/secrets/kubernetes.io/serviceaccount/ca.crt \
  issuer="https://kubernetes.default.svc.cluster.local"
```

Create a read policy for this secret (for Pod-consumer)

```bash
echo 'path "my-kv/data/my-secret" { capabilities = ["read"] }' > /tmp/policy.hcl
vault policy write devweb-policy /tmp/policy.hcl
rm /tmp/policy.hcl
```

Match the policy (for Pod-consumer) with the Vault role (for the container, it will match by JWT token + SA + manifest for SA)

```bash
vault write auth/kubernetes/role/devweb-app \
  bound_service_account_names=internal-app \
  bound_service_account_namespaces=default \
  policies=devweb-policy \
  ttl=24h
```

Create a Pod-consumer for the `my-kv/my-secret` secret and a ServiceAccount for it, through which it can get the secret from the injector

```bash
apiVersion: v1
kind: ServiceAccount
metadata:
  name: internal-app
  namespace: default
---

apiVersion: apps/v1
kind: Deployment
metadata:
  name: devwebapp
  namespace: default
  labels:
    app: devwebapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: devwebapp
  template:
    metadata:
      labels:
        app: devwebapp
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/ca-cert: "/run/secrets/kubernetes.io/serviceaccount/ca.crt"
        vault.hashicorp.com/role: "devweb-app"
        vault.hashicorp.com/agent-inject-secret-config: "my-kv/data/my-secret"
        vault.hashicorp.com/agent-inject-template-config: |
          {{ with secret "my-kv/data/my-secret" -}}
          TOKEN={{ .Data.data.token }}
          PASSWORD={{ .Data.data.password }}
          {{- end }}
    spec:
      serviceAccountName: internal-app
      containers:
        - name: test-container
          image: busybox
          command: ["/bin/sh", "-c"]
          args:
            - |
              while true; do
                if [ -f /vault/secrets/config ]; then
                  source /vault/secrets/config
                  echo "Token: $TOKEN"
                  echo "Password: $PASSWORD"
                fi
                sleep 5
              done
```

Done, you're awesome!

In the pod's log output, you should see `"Token: $TOKEN"` and `"Password: $PASSWORD"` with the secret values from Vault.

In this step, you should pay attention to the manifest section:

```bash
      annotations:
        vault.hashicorp.com/agent-inject: "true"
        vault.hashicorp.com/ca-cert: "/run/secrets/kubernetes.io/serviceaccount/ca.crt"
        vault.hashicorp.com/role: "devweb-app"
        vault.hashicorp.com/agent-inject-secret-config: "my-kv/data/my-secret"
        vault.hashicorp.com/agent-inject-template-config: |
          {{ with secret "my-kv/data/my-secret" -}}
          TOKEN={{ .Data.data.token }}
          PASSWORD={{ .Data.data.password }}
          {{- end }}
```

These are instructions for the injector, which it uses to work with secrets. All possible annotations for the injector: [Vault Agent Injector annotations](https://developer.hashicorp.com/vault/docs/platform/k8s/injector/annotations), they are quite extensive and allow you to perform various tasks.

## Injection secrets into Gitlab

Let's look at injecting secrets into Gitlab. This approach will help you store secrets in Vault and use them in Gitlab CI pipelines.

```bash
kubectl exec -n prod-hashicorp-vault -it prod-hashicorp-vault-0 -- /bin/sh
vault login $MAIN_TOKEN

vault kv put my-kv/my-secret token=my-token password=my-password
vault kv get my-kv/my-secret
```

Create an integration with Gitlab

```bash
vault auth enable -path jwt_v2 jwt

vault write auth/jwt/role/gitlab-role \
    role_type="jwt" \
    bound_audiences="https://mygitlab.example" \
    user_claim="sub" \
    policies="gitlab-policy" \
    ttl="1h"
```

Create a read policy for GitLab with access to read secrets

```bash
cat <<EOF > /tmp/gitlab-policy.hcl
path "my-kv/data/*" {
  capabilities = ["read"]
}
path "my-kv/metadata/*" {
  capabilities = ["list", "read"]
}
EOF

vault policy write gitlab-policy /tmp/gitlab-policy.hcl

vault policy read gitlab-policy
```

Create a CI pipeline in Gitlab to retrieve the secret

```bash
stages:
  - vault

fetch_secret:
  variables:
    VAULT_AUTH_ROLE: "gitlab-role"
    VAULT_AUTH_PATH: "jwt_v2"
    VAULT_SERVER_URL: "https://prod-hashicorp-vault.prod-hashicorp-vault.svc:8200"
    # or if your runner not in K8s:
    # VAULT_SERVER_URL: "https://mygitlab.example:8200"
    # and if you use ssl transit
    # VAULT_CACERT: /var/run/secrets/kubernetes.io/serviceaccount/ca.crt
  image: hashicorp/vault
  stage: vault
  script:
    - export VAULT_TOKEN=$(vault write -field=token auth/jwt/login role="gitlab" jwt="$CI_JOB_JWT")
    - vault kv get -field=username my-kv/my-secret
    - TOKEN=$(vault kv get -field=token my-kv/my-secret)
    - PASSWORD=$(vault kv get -field=password my-kv/my-secret)
    - echo TOKEN=$TOKEN
    - echo PASSWORD=$PASSWORD
    - echo "$TOKEN" >> vault_secrets.env
    - echo "$PASSWORD" >> vault_secrets.env
  artifacts:
    paths:
      - vault_secrets.env
    expire_in: 1 hour

deploy_app:
  stage: deploy
  image: alpine
  dependencies:
    - fetch_secret
  script:
    - source vault_secrets.env
    - echo  "$TOKEN"
    - rm -f vault_secrets.env
```

Done, you're awesome!

## Enable SSL everywhere for Vault (transit)

The standard approach is that your EKS/K8s cluster is a trusted site, and traffic within it can travel unencrypted, aggregating SSL only at the ingress/LB. But you may encounter a situation where you want to use SSL between the injector and Vault everywhere inside your cluster. The downsides of this solution are the cluster-signed certificate valid for 1 year (meaning it will need to be renewed) and the fact that this certificate will need to be "distributed" to applications that will use it (in this case, only the injector).

Below I'll explain how to do this:

### Generate SSL certificates

```bash
#!/bin/bash

NAMESPACE="prod-hashicorp-vault"
SECRET_NAME="vault-server-tls"
TMPDIR="."
SERVICE="prod-hashicorp-vault"
CSR_NAME="vault-csr"

openssl genrsa -out ${TMPDIR}/vault.key 2048

cat <<EOF > ${TMPDIR}/csr.conf
[req]
req_extensions = v3_req
distinguished_name = req_distinguished_name

[req_distinguished_name]

[v3_req]
basicConstraints = CA:FALSE
keyUsage = nonRepudiation, digitalSignature, keyEncipherment
extendedKeyUsage = serverAuth
subjectAltName = @alt_names

[alt_names]
DNS.1 = ${SERVICE}
DNS.2 = ${SERVICE}.${NAMESPACE}
DNS.3 = ${SERVICE}.${NAMESPACE}.svc
DNS.4 = ${SERVICE}.${NAMESPACE}.svc.cluster.local
IP.1 = 127.0.0.1
EOF

openssl req -new -key ${TMPDIR}/vault.key -subj "/CN=${SERVICE}.${NAMESPACE}.svc" -out ${TMPDIR}/server.csr -config ${TMPDIR}/csr.conf

cat <<EOF > ${TMPDIR}/csr.yaml
apiVersion: certificates.k8s.io/v1
kind: CertificateSigningRequest
metadata:
  name: ${CSR_NAME}
spec:
  signerName: beta.eks.amazonaws.com/app-serving
  request: $(cat ${TMPDIR}/server.csr | base64 | tr -d '\n')
  usages:
    - digital signature
    - key encipherment
    - server auth
  groups:
    - system:authenticated
EOF

kubectl create -f ${TMPDIR}/csr.yaml

kubectl get csr
# CONDITION = Pending

kubectl certificate approve ${CSR_NAME}
kubectl get csr
# CONDITION =  Approved,Issued

serverCert=$(kubectl get csr ${CSR_NAME} -o jsonpath='{.status.certificate}')
echo "${serverCert}" | openssl base64 -d -A -out ${TMPDIR}/vault.crt

kubectl config view --raw --minify --flatten -o jsonpath='{.clusters[].cluster.certificate-authority-data}' | base64 -d > ${TMPDIR}/vault.ca

kubectl create secret generic ${SECRET_NAME} \
    --namespace ${NAMESPACE} \
    --from-file=vault.key=${TMPDIR}/vault.key \
    --from-file=vault.crt=${TMPDIR}/vault.crt \
    --from-file=vault.ca=${TMPDIR}/vault.ca
```

### Override-values.yaml for SSL certificates

```bash
global:
  enabled: true
  tlsDisable: false 
injector:
  enabled: true
  metrics:
    enabled: true
  nodeSelector:
    nodegroup: hashicorp-vault-nodes
  port: 8080
  agentDefaults:
    cpuLimit: 500m
    cpuRequest: 250m
    memLimit: 128Mi
    memRequest: 64Mi
server:
  enabled: '-'
  standalone:
    enabled: false
  auditStorage:
    enabled: true
    accessMode: ReadWriteOnce
    mountPath: /vault/audit
    size: 10Gi
  dataStorage:
    enabled: false
  nodeSelector:
    nodegroup: hashicorp-vault-nodes
  extraEnvironmentVars:
    VAULT_CACERT: /vault/userconfig/vault-server-tls/vault.ca
  serviceAccount:
    annotations:
      eks.amazonaws.com/role-arn: arn:aws:iam::$ACCOUNT_ID:role/hashicorp-vault-role # for EKS + IAM only
    create: true
  ha:
    enabled: true
    replicas: 3
    config: |
      ui = true

      listener "tcp" {
        tls_disable = 0
        address = "[::]:8200"
        cluster_address = "[::]:8201"
        # if tls is enabled
        tls_cert_file = "/vault/userconfig/vault-server-tls/vault.crt"
        tls_key_file = "/vault/userconfig/vault-server-tls/vault.key"
        tls_ca_cert_file = "/vault/userconfig/vault-server-tls/vault.ca"
      }

      # For internal ssl vault <-> injector:
      listener "tcp" {
        tls_disable = 0
        address = "[::]:8202"
        cluster_address = "[::]:8201"
      }

      storage "dynamodb" {
        ha_enabled = "true"
        region = "$REGION"
        table = "$DYNAMODB_TABLE"
      }

      seal "awskms" {
        region     = "eu-west-1"
        kms_key_id = "$KMS_KEY_ID"
        # no need now: endpoint   = "https://vpce-xxxxxxxxxxxxxxx.kms.eu-west-1.vpce.amazonaws.com"
      }

      service_registration "kubernetes" {}
    disruptionBudget:
      enabled: true
      maxUnavailable: null
  ingress:
    enabled: true
    activeService: true
    annotations:
      kubernetes.io/ingress.class: "nginx"
      cert-manager.io/cluster-issuer: "letsencrypt"
      nginx.ingress.kubernetes.io/rewrite-target: "/"  
      nginx.ingress.kubernetes.io/ssl-redirect: "true"
      nginx.ingress.kubernetes.io/proxy-body-size: "100m"
    ingressClassName: nginx
    labels: {}
    pathType: Prefix
    tls:
      - hosts:
          - vault.example.com
        secretName: $TLS_SECRET_NAME
    hosts: 
      - host: vault.example.com
ui:
  enabled: true
  serviceType: "ClusterIP"
  externalPort: 8202
  targetPort: 8202
```
