---
layout: post
title: How to import all existing AWS resources into Terraform
subtitle: Creating all terraform files and structures for AWS. Step-by-step guide. 
excerpt_image: /assets/images/posts/move-resources-into-terraform/terraform-gif.gif
categories: Terraform
tags: [Terraform, AWS, Cloud, S3]
top: 1
---

![banner](/assets/images/posts/move-resources-into-terraform/terraform-gif.gif)

### Setup your aws config file

```bash
~/.aws/config
```

### Install terraformer

In current directory.
You can find out about all the installation methods from [GitHub repo Terraformer](https://github.com/GoogleCloudPlatform/terraformer)

```bash
brew install terraformer
```

### Create version.tf file

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ue-west-1"
}
```

### Init terraform

And check that you providers is correct

```bash
terraform init
terraform providers
```

### Start import

Set specific resources or all resources in the account

```bash
terraformer import aws --resources=ec2_instance,ebs --regions=eu-west-1 # for some resources
terraformer import aws --resources="*" --regions=eu-west-1 # for all resources
```

At this stage you may encounter errors like this. This means that in your AWS account does not have these entities, or Terraformer cannot connect to it with account that was transferred to the local AWS config:

```bash
panic: runtime error: index out of range [0] with length 0
...
...
...
aws error initializing resources in service cloud9, err: operation error Cloud9: ListEnvironments, https response error StatusCode: 400
```

Then you need use `--excludes` parameter, passing resources that cause errors:

```bash
terraformer import aws --resources="*" --excludes="cloud9,identitystore" --regions=eu-west-1
```

Terraformer by default separates each resource into a file, which is put into a current service directory. The default path for resource files is `{generated}/{provider}/{service}/{resource}.tf` and can vary for each provider.

### After import

#### Clean uo non-existing resources

Inside `{generated}/{provider}/{service}/` may be file `terraform.tfstate ` with empty resource. It is possible that you don't need such resources. Then delete the directories with it:

```hcl
{
    "version": 3,
    "terraform_version": "0.12.31",
    "serial": 1,
    "lineage": "*****-fe52-*****-6e0f-51f858*****",
    "modules": [
        {
            "path": [
                "root"
            ],
            "outputs": {},
            "resources": {},
            "depends_on": []
        }
    ]
}
```

#### Check provider.tf file

Check that the directories have `provider.tf file`. It may look like this, the data in file must match your version of terraform and your current providers:

```hcl
provider "aws" {
  region = "eu-west-1"
}

terraform {
	required_providers {
		aws = {
	    version = "~> 5.88.0"
		}
  }
}
```

#### Replace tfstates versions

Now, you have all you resources that you created in AWS both manually in directory `{generated}/{provider}/`! Super!

![screenshot transfer catalog](/assets/images/posts/move-resources-into-terraform/scren-clean-transfer-catalog.png)

Please note that all files `{generated}/{provider}/{service}/terraform.tfstate` containsuch a header:

```hcl
{
    "version": 3,
    "terraform_version": "0.12.31",
    "serial": 1,
    "lineage": "******-e4dd-0626-*****-9ceccb5a928a",
    "modules": [
```

It means that Terraformer based on **Terraform version** `0.12.+` for which file `terraform.tfstate` created with `"version": 3`. This is a problem, because you current and next versions terraform is newer. For fix it do next command in every directory `{generated}/{provider}/{service}/`:

```bash
terraform state replace-provider -auto-approve "registry.terraform.io/-/aws" "hashicorp/aws"
```

Read more about it here [Terraform state replace-provider](https://developer.hashicorp.com/terraform/cli/commands/state/replace-provider)

#### Create backend.tf for remote state

In each directory create backend.tf files for remote state
 
```hcl
terraform {
  backend "s3" {
    bucket         = "$BACKET_NAME"
    key            = "$KEYNAME/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "$DYNAMO_DB_LOCKS_TABLE" # if you need it
    encrypt        = true
  }
} 
```

#### Start init terraform

In each directory start init terraform

```bash
terraform init -force-copy
```

#### Script for all the previous steps

Fix it for yourself, but the logic is correct

```bash
#!/bin/bash

# Checking for required environment variables
if [ -z "$AWS_REGION" ] || [ -z "$TF_STATE_BUCKET" ]; then
    echo "Please set the following environment variables:"
    echo "export AWS_REGION='your-region'"
    echo "export TF_STATE_BUCKET='your-bucket-name'"
    exit 1
fi

# Save current directory
SCRIPT_DIR=$(pwd)
BASE_DIR="terraformer/generated/aws"

# Get list of all services
for SERVICE_DIR in $BASE_DIR/*; do
    if [ ! -d "$SERVICE_DIR" ]; then
        continue
    fi

    SERVICE_NAME=$(basename "$SERVICE_DIR")
    echo "========================================="
    echo "Testing migration for service $SERVICE_NAME..."
    echo "========================================="

    cd "$SERVICE_DIR"

    echo "2. Replacing provider..."
    terraform state replace-provider -auto-approve "registry.terraform.io/-/aws" "hashicorp/aws"

    echo "3. Creating backend.tf from template..."
    TEMPLATE_PATH="$SCRIPT_DIR/backend.tf.tmpl"

    if [ ! -f "$TEMPLATE_PATH" ]; then
        echo "Error: Template file $TEMPLATE_PATH not found!"
        exit 1
    fi

    echo "Using template: $TEMPLATE_PATH"
    echo "Creating backend.tf in: $(pwd)"

    cat "$TEMPLATE_PATH" | \
    sed "s/__SERVICE__/$SERVICE_NAME/g" | \
    sed "s/your-aws-region/$AWS_REGION/g" | \
    sed "s/your-terraform-state-bucket/$TF_STATE_BUCKET/g" > backend.tf

    if [ ! -s backend.tf ]; then
        echo "Error: backend.tf is empty after creation!"
        exit 1
    fi

    echo "Contents of created backend.tf:"
    cat backend.tf

    echo "4. Migrating state to S3..."
    terraform init \
        -force-copy \
        -backend=true \
        -backend-config="bucket=$TF_STATE_BUCKET" \
        -backend-config="key=$YOURS_BACKEND_S3/$SERVICE_NAME/terraform.tfstate" \
        -backend-config="region=$AWS_REGION" \
        -backend-config="dynamodb_table=$NAME" \
        -backend-config="encrypt=true"

    # Status check
    if [ $? -eq 0 ]; then
        echo "5. Checking state..."
        terraform state list
    else
        echo "Error during state migration for $SERVICE_NAME!"
        # Continue with next service instead of exiting
        cd "$SCRIPT_DIR"
        continue
    fi

    # Return to original directory for next iteration
    cd "$SCRIPT_DIR"
    echo "----------------------------------------"
    echo "Migration of $SERVICE_NAME completed!"
    echo "----------------------------------------"
done

echo "Migration of all services completed!" 
```

### After init

At this stage you will have all resources states in bucket. For example, you S3 may look like this:

```bash
aws s3 ls s3://$BACKET_NAME/$KEYNAME --recursive | awk '{print $4}'
$BACKET_NAME/acm/terraform.tfstate
$BACKET_NAME/alb/terraform.tfstate
$BACKET_NAME/auto_scaling/terraform.tfstate
$BACKET_NAME/cloudformation/terraform.tfstate
$BACKET_NAME/cloudfront/terraform.tfstate
$BACKET_NAME/cloudwatch/terraform.tfstate
$BACKET_NAME/config/terraform.tfstate
$BACKET_NAME/docdb/terraform.tfstate
$BACKET_NAME/dynamodb/terraform.tfstate
$BACKET_NAME/ebs/terraform.tfstate
$BACKET_NAME/ec2_instance/terraform.tfstate
$BACKET_NAME/ecr/terraform.tfstate
$BACKET_NAME/efs/terraform.tfstate
$BACKET_NAME/eip/terraform.tfstate
$BACKET_NAME/eks/terraform.tfstate
...
```

#### Rewrite variables

Now you files variables.tf contain local links for outputs to other resources. For example:

```hcl
data "terraform_remote_state" "sg" {
  backend = "local"

  config = {
    path = "../../../generated/aws/sg/terraform.tfstate"
  }
}

data "terraform_remote_state" "subnet" {
  backend = "local"

  config = {
    path = "../../../generated/aws/subnet/terraform.tfstate"
  }
}
```

You need rewrite all files to this:

```hcl
data "terraform_remote_state" "sg" {
  backend = "s3"

  config = {
    bucket = "$BACKET_NAME"
    key    = "$KEYNAME"
    region = "eu-west-1"
  }
}

data "terraform_remote_state" "subnet" {
  backend = "s3"

  config = {
    bucket = "$BACKET_NAME"
    key    = "$KEYNAME"
    region = "eu-west-1"
  }
}
```

This script can help you:

```bash
#!/bin/bash

# Script for replacing local backend with remote S3 backend in variables.tf files

# S3 bucket settings
S3_BUCKET="$TF_STATE_BUCKET"
REGION="$AWS_REGION"
KEY_PREFIX="$TF_STATE_KEY_PREFIX"

# Infrastructure directory (for macOS)
INFRA_DIR="$(cd "$(dirname "$0")" && pwd)"
echo "Working directory: $INFRA_DIR"

# Function to replace backend in file
replace_backend() {
    local file="$1"
    local service_name="$2"
    
    echo "Processing file: $file for service: $service_name"
    
    # Create temporary file
    local temp_file=$(mktemp)
    
    # Completely rewrite the file, fixing all terraform_remote_state blocks
    # Use sed to extract names of all remote_state blocks
    remote_states=$(grep -o 'data "terraform_remote_state" "[^"]*"' "$file" | awk -F'"' '{print $4}')
    
    # If there are no remote_state blocks, skip the file
    if [ -z "$remote_states" ]; then
        echo "No terraform_remote_state blocks found in $file"
        rm "$temp_file"
        return
    fi
    
    # Create new file with correct blocks
    > "$temp_file"
    
    for rs in $remote_states; do
        echo "Processing remote_state block: $rs"
        
        # Add block with correct S3 configuration
        cat >> "$temp_file" << EOF
data "terraform_remote_state" "$rs" {
  backend = "s3"

  config = {
    bucket = "$S3_BUCKET"
    key    = "$KEY_PREFIX/$rs/terraform.tfstate"
    region = "$REGION"
  }
}

EOF
    done
    
    # Check if file was changed
    if ! cmp -s "$file" "$temp_file"; then
        # Create backup copy
        cp "$file" "${file}.bak"
        echo "Backup created: ${file}.bak"
        
        # Replace file
        mv "$temp_file" "$file"
        echo "File updated: $file"
    else
        rm "$temp_file"
        echo "File already up to date: $file"
    fi
}

# Main loop
echo "Starting variables.tf files update..."

# Process each subdirectory
for dir in "$INFRA_DIR"/*; do
    if [ -d "$dir" ] && [ "$(basename "$dir")" != ".git" ]; then
        service_name=$(basename "$dir")
        variables_file="$dir/variables.tf"
        
        echo "Checking directory: $dir"
        echo "Service name: $service_name"
        
        if [ -f "$variables_file" ]; then
            echo "Found variables.tf file in $dir"
            replace_backend "$variables_file" "$service_name"
        else
            echo "variables.tf file not found in $dir"
        fi
    fi
done

echo "Update completed!" 
```

### Done! Push your IaC in git

You may run `terraform plan` for checks. And prepare .gitignore file, and do `git init`, `git remote add origin...`, `git push`:

```bash
# Local .terraform directories
**/.terraform/*

# .terraform.lock.hcl contains the exact versions of the providers and their hashes
# It is recommended to save this file in the repository to ensure
# Reproducibility of infrastructure and security
# Uncomment the following line if you want to exclude this file (not recommended)
# **/.terraform.lock.hcl

# .tfstate files
*.tfstate
*.tfstate.*

*.tfplan
**/*.bak
**/backup.tfstate
*.sh
**/*.sh

# Crash log files
crash.log
crash.*.log

# Exclude all .tfvars files, which are likely to contain sensitive data, such as
# password, private keys, and other secrets. These should not be part of version 
# control as they are data points which are potentially sensitive and subject 
# to change depending on the environment.
*.tfvars
*.tfvars.json

# Ignore override files as they are usually used to override resources locally and so
# are not checked in
override.tf
override.tf.json
*_override.tf
*_override.tf.json

# Ignore transient lock info files created by terraform apply
.terraform.tfstate.lock.info

# Include override files you do wish to add to version control using negated pattern
# !example_override.tf

# Ignore CLI configuration files
.terraformrc
terraform.rc

.DS_Store
*.swp
*.swo 
```

### Breaking down large .tf files into modules

Here's an example approach:

Let's say we need to create a new NodeGroup for the `cluster1` (not the most relevant example since node groups are generally one logical piece, but we need to "cut out" the `cluster.cluster1` node groups from the general codebase this all clusters).

#### Important Considerations

- **Always create backups before making changes**

- **Remove resources from the state before deleting code**, otherwise Terraform might try to delete actual resources in AWS

- **Check the plan after changes** to ensure Terraform isn't trying to create or delete resources

- **Be careful with dependencies**:
  
  - If other resources depend on the node groups being removed, errors may occur
  
  - In this case, you'll also need to update dependent resources
  
  - Update references to outputs if other modules reference outputs from this module

After completing these steps, your main module will no longer manage node groups for the `cluster1`, and they will be fully controlled by the new module.

#### Step-by-Step Process

We're moving node groups for cluster1 from:
`aws/eks/eks_node_group.tf`
To:
`aws/eks/cluster1/eks_node_group.tf`

And "cutting out" only the cluster1 nodes from the common file.
Note: This code is not a perfect example! It's just to illustrate the principle.
Create a backup of the current state `terraform state pull > backup.tfstate`
Create the directory structure for the new module `mkdir -p aws/eks/cluster1`

Create `aws/eks/cluster1/backend.tf`:
```hcl
terraform {
  backend "s3" {
    bucket         = "$BUCKET_NAME"
    key            = "cluster1/terraform.tfstate"
    region         = "eu-west-1"
    dynamodb_table = "$NAME"
    encrypt        = true
  }
}
```

Create `provider.tf` in the new directory:
```hcl
provider "aws" {
  region = "eu-west-1"
}

terraform {
  required_providers {
    aws = {
      version = "~> 5.88.0"
    }
  }
}
```

As needed, create `variables.tf` and `outputs.tf` in the new directory, and copy `.terraform.lock.hcl`.

Create `eks_node_group.tf` (copy only the nodes for the cluster1 cluster):
```hcl
resource "aws_eks_node_group" "common" {
  ami_type       = "xxxxx"
  capacity_type  = "ON_DEMAND"
  cluster_name   = "${aws_eks_cluster1.name}"
  disk_size      = "100"
  instance_types = ["xxxxx"]
  # ...
}

# Add your new node group
resource "aws_eks_node_group" "MY-SUPER-NODE-GROUP" {
  ami_type       = "xxxxx"
  capacity_type  = "ON_DEMAND"
  cluster_name   = "${aws_eks_cluster1.name}"
  disk_size      = "100"
  instance_types = ["xxxxx"]
  # ...
}
```

Initialize and move state:
```bash
# Initialize the working directory, download required providers, 
# configure backend for state storage, and install modules if used
terraform init

# Very important step - this will change the name in the remote state 
# to a new one without the "tfer--" prefix that was generated during export.
# Do this for all resources you've cut from the common file to the new one,
# but not for resources you've added (aws_eks_node_group.MY-SUPER-NODE-GROUP)
terraform state mv 'aws_eks_node_group.tfer--common' 'aws_eks_node_group.common'

# Check that everything looks good
terraform plan
```

If everything looks good, deploy through CI: `terraform plan -> terraform apply`

After separating the node groups for the cluster1 into a separate module, you need to remove them from the main module. This is a two-step process:

#### 1. Remove the Extracted cluster1 Node Groups from the State

For the common file:
```bash
terraform state rm aws_eks_node_group.tfer--common
terraform state rm aws_eks_node_group.tfer--common1
terraform state rm aws_eks_node_group.tfer--common2
# ...
```

#### 2. Delete Resource Code from .tf Files

After removing resources from the state, you need to delete their definitions from the .tf files. In your case, this is the file `aws/eks/cluster1/eks_node_group.tf`.
You need to remove code blocks for all node groups related to the `cluster1` cluster that you've moved to a separate file.
Verify the changes:
```bash
terraform plan
```

The plan should not show any changes for the removed resources since they've already been removed from the state.


#### Migrate future manuals

It may happen that even after migrating to terraform, you will continue to create resources manually. Which is undesirable behavior, but it's still life 😊. In this case, you can use this logic to periodically scan your AWS account for resources, compare them with terraform, and then migrate them:

```bash
#!/bin/bash
aws_instances=$(aws ec2 describe-instances --query 'Reservations[*].Instances[*].[InstanceId]' --output text)

# Getting a list of all instances in AWS
tf_instances=$(terraform show -json | jq -r '.values.root_module.resources[] | select(.type == "aws_instance") | .values.id')

# Compare and show the difference
for instance in $aws_instances; do
    if ! echo "$tf_instances" | grep -q "$instance"; then
        echo "A new instance has been found: $instance"
        # Automatic import can be added
        # terraform import aws_instance.new_$instance $instance
    fi
done
```
