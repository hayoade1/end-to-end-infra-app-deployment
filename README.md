# Overview

Here we trace the workflow of a developer deploying infrastructure and applications to Azure using Packer, GitHub, Jenkins, Terraform, Vault, Ansible, and Consul. We use our webblog application to demo this.

There are 4 blog posts associated with this repo and talk about how everything works:
- [Part 1: HashiCorp Vault Azure Secrets Engine](https://tekanaid.com/posts/hashicorp-vault-azure-secrets-engine-secure-your-azure-resources/)
- [Part 2: HashiCorp Packer, Terraform, and Ansible to Set Up Jenkins](https://tekanaid.com/posts/hashicorp-packer-terraform-and-ansible-to-set-up-jenkins/)
- [Part 3: The Secret Zero Problem Solved for HashiCorp Vault](https://tekanaid.com/posts/secret-zero-problem-solved-for-hashicorp-vault/)
- [Part 4: Jenkins, Vault, Terraform, Ansible, and Consul End-to-End CI/CD Pipeline](https://tekanaid.com/posts/jenkins-vault-terraform-ansible-and-consul-end-to-end-ci-cd-pipeline/)

## Topics to Learn
1. Vault Azure Secrets Engine
2. Packer Images in Azure
3. Terraform Building VMs in Azure based on Packer Images
4. Ansible to Configure an Azure VM
5. Vault Secure Introduction
6. Vault App Role
7. Vault Dynamic Database Secrets for MongoDB
8. Vault Transit Secrets Engine
9. Advanced CI/CD Pipeline Workflow using GitHub(VCS), Jenkins(CI/CD), Terraform(IaC), Ansible(Config Mgmt), Vault(Secrets Mgmt)
10. Consul Service Mesh

## Vault Azure Secrets Engine
Let's take a look at how we can build this. You can find details in the [docs](https://www.vaultproject.io/docs/secrets/azure). You can also follow the [step-by-step guide](https://learn.hashicorp.com/tutorials/vault/azure-secrets).

Below is a diagram of the Vault Azure Secrets Engine Workflow

![Vault Azure Secrets Engine Workflow Diagram](https://learn.hashicorp.com/img/vault-azure-secrets-0.png)

### Vault Configuration

The configuration setup below needs to be done by a Vault Admin. This [Vault policy](https://learn.hashicorp.com/tutorials/vault/azure-secrets#policy-requirements) is used with a token to run the configuration commands. We use the root token in this demo for simplicity, however, in a production setting it's not recommended to use the root token.

We are re-using our existing Vault cluster. The Vault admin configuration is located in the [infrastructure-gcp GitLab repo](https://gitlab.com/public-projects3/infrastructure-gcp/-/tree/master/terraform-vault-configuration)

#### Setup

Below an admin uses the Vault Terraform Provider. This is found in the [main.tf file](https://gitlab.com/public-projects3/infrastructure-gcp/-/blob/master/terraform-vault-configuration/main.tf)

```shell
resource "azurerm_resource_group" "myresourcegroup" {
  name     = "${var.prefix}-jenkins"
  location = var.location
}

resource "vault_azure_secret_backend" "azure" {
  subscription_id = var.subscription_id
  tenant_id = var.tenant_id
  client_secret = var.client_secret
  client_id = var.client_id
}

resource "vault_azure_secret_backend_role" "jenkins" {
  backend                     = vault_azure_secret_backend.azure.path
  role                        = "jenkins"
  ttl                         = 300
  max_ttl                     = 600

  azure_roles {
    role_name = "Contributor"
    scope =  "/subscriptions/${var.subscription_id}/resourceGroups/${azurerm_resource_group.myresourcegroup.name}"
  }
}
```

### Request Azure Creds Manually

```shell
vault policy write jenkins Vault/policies/jenkins_azure_policy.hcl
vault token create -policy=jenkins
VAULT_TOKEN=xxxxxx vault read azure/creds/jenkins
```

### Packer to Build a Jenkins Image in Azure

#### Steps
1. Create a Packer image in Azure with Docker installed
2. Build a Docker image that has Jenkins, Terraform, and Ansible installed

***Note***
When using the Azure creds, I couldn't use the ones generated by Vault because they are specific to the `samg-jenkins` resource group. Packer for some reason uses a random Azure resource group when building therefore it needs creds that have a scope for any resource group. I used the regular service principal creds.

### Terraform to Build a Jenkins VM in Azure and Ansible to Configure it

We use Terraform to build an Azure VM based on the Packer image we previously created. 

**Note**
Here we can use the Vault generated creds to build a VM in Azure since the creds are tied to the `samg-jenkins` resource group.

### Secure Introduction

Below are some resources that talk about Secure Introduction and Secret Zero
[HashiTalk on Vault Response Wrapping and Secret Zero](https://www.hashicorp.com/resources/vault-response-wrapping-makes-the-secret-zero-challenge-a-piece-of-cake)
[GitHub Repo for above HashiTalk](https://github.com/misurellig/hashitalks-demo)

#### Secure Introduction Workflow for Pipelines

1. A Vault Admin does the following
   a. Create AppRoles for Jenkins node and the pipeline with policies in Vault
   b. Insert AppRole auth creds into Jenkins node's Vault plugin
   c. Deliver the Role ID for the pipeline into the Jenkinsfile
2. The Jenkins node creates a wrapped secret ID for the pipeline
3. The pipeline unwraps the secret ID and logs into Vault via AppRole for pipeline
4. The pipeline retrieves the Terraform Cloud token
5. The pipeline calls TFC to build the App VMs and generate dynamic Azure creds
6. Terraform Builds the App VMs

#### Create an Approle for the Jenkins Node

This is done via Terraform using the following configuration:

```shell
resource "vault_policy" "jenkins_policy" {
  name = "jenkins-policy"
  policy = file("policies/jenkins_policy.hcl")
}

resource "vault_auth_backend" "jenkins_access" {
  type = "approle"
  path = "jenkins"
}

resource "vault_approle_auth_backend_role" "jenkins_approle" {
  backend            = vault_auth_backend.jenkins_access.path
  role_name          = "jenkins-approle"
  secret_id_num_uses = "5"
  secret_id_ttl      = "300"
  token_ttl          = "1800"
  token_policies     = ["default", "jenkins-policy"]
}
```

The `jenkins_policy.hcl` file mentioned here contains the following policy:

```shell
path "auth/pipeline/role/pipeline-approle/secret-id" {
  policy = "write"
  min_wrapping_ttl   = "100s"
  max_wrapping_ttl   = "300s"
}
```

Once you configure Vault via Terraform, you can then run the two commands below to get the `role-id` and the `secret-id`. You can see more [instructions in the documentation.](https://www.vaultproject.io/docs/auth/approle)

```shell
vault read auth/jenkins/role/jenkins-approle/role-id
vault write -field=secret_id -f auth/jenkins/role/jenkins-approle/secret-id
```

You can now take the `role-id` and the `secret-id` and insert them into the Jenkins Vault plugin for authentication. Please make sure you have the correct path for the AppRole.

You can run a test to login below:
```shell
vault write auth/jenkins/login \
    role_id=a79bdd3a-81e3-e356-4c9e-46d22ff3fdc5 \
    secret_id=8b635683-82d1-2fc5-7028-682566137e74
```
#### Create an Approle for the Jenkins Pipeline

Once again we use Terraform for configuration as shown below:

```shell
resource "vault_policy" "pipeline_policy" {
  name = "pipeline-policy"
  policy = file("policies/jenkins_pipeline_policy.hcl")
}

resource "vault_auth_backend" "pipeline_access" {
  type = "approle"
  path = "pipeline"
}

resource "vault_approle_auth_backend_role" "pipeline_approle" {
  backend            = vault_auth_backend.pipeline_access.path
  role_name          = "pipeline-approle"
  secret_id_num_uses = "1"
  secret_id_ttl      = "300"
  token_ttl          = "1800"
  token_policies     = ["default", "pipeline-policy"]
}
```

The `jenkins_pipeline_policy.hcl` file mentioned here contains a policy to allow the pipeline to retrieve Azure credentials so that Terraform can provision Azure VMs. Here is the policy configuration:

```shell
path "azure/*" {
  capabilities = [ "read" ]
}
```

You then need to read the role-id for the Jenkins policy and insert that into the jenkinsfile for the pipeline. The Jenkins node will create a wrapped secret ID for the pipeline and in fact, that's the only capability it has as defined in the Jenkins policy mentioned above. The pipeline then unwraps the secret-id and retrieves a VAULT_TOKEN that will get used for the remainder of the pipeline. Below is the command used to generate the role-id for the pipeline.

```shell
vault read auth/pipeline/role/pipeline-approle/role-id
```

#### Create an Approle for the Vault Agent

Below is the Terraform configuration for Vault:

```shell
resource "vault_policy" "webblog" {
  name   = "webblog"
  policy = file("policies/webblog_policy.hcl")
}

resource "vault_auth_backend" "apps_access" {
  type = "approle"
  path = "approle"
}

resource "vault_approle_auth_backend_role" "webblog_approle" {
  backend            = vault_auth_backend.apps_access.path
  role_name          = "webblog-approle"
  secret_id_num_uses = "1"
  secret_id_ttl      = "600"
  token_ttl          = "1800"
  token_policies     = ["default", "webblog"]
}
```

The `webblog_policy.hcl` file mentioned here contains a policy to allow Vault agent to create a token for the webblog app to use. The policy allows the webblog app to read dynamic MongoDB secrets as well as utilize the Vault Transit secrets engine to encrypt the content of the blog posts. Here is the policy configuration:

```shell
path "internal/data/webblog/mongodb" {
  capabilities = ["read"]
}
path "mongodb/creds/mongodb-role" {
  capabilities = [ "read" ]
}
path "mongodb_nomad/creds/mongodb-nomad-role" {
  capabilities = [ "read" ]
}
path "mongodb_azure/creds/mongodb-azure-role" {
  capabilities = [ "read" ]
}
path "transit/*" {
  capabilities = ["list","read","update"]
}
```

The pipeline will need to run the following commands to create a `role-id` and a `wrapped-secret-id` for the Vault agent:

```shell
vault read -field=role_id auth/approle/role/webblog-approle/role-id > /tmp/webblog_role_id
vault write -field=wrapping_token -wrap-ttl=200s -f auth/approle/role/webblog-approle/secret-id > /tmp/webblog_wrapped_secret_id
```

#### Run the Vault Agent

Below is the Vault agent configuration:

```shell
pid_file = "./pidfile"

vault {
  address = "http://vault.hashidemos.tekanaid.com:8200"
}

auto_auth {
  method "approle" {
    mount_path = "auth/approle"
      config = {
        role_id_file_path = "/tmp/webblog_role_id"
        secret_id_file_path = "/tmp/webblog_wrapped_secret_id"
        remove_secret_id_file_after_reading = true
        secret_id_response_wrapping_path = "auth/approle/role/webblog-approle/secret-id"
    }
  }

  sink "file" {
    config = {
      path = "/tmp/vault_token"
      }
    }
}
```

Then you can run the Vault agent using the command below:

```shell
vault agent -config=vault_agent_config.hcl
```

**Note:** The token that the Vault agent generates is a token that has access to the policy defined in the role used for the Vault agent. So this generated token has the necessary Vault privileges that our Webblog app needs.


## Create the Webblog App VMs

### Jenkins to Retrieve Azure Creds from Vault

```shell
vault read azure/creds/jenkins
```

Jenkins will retrieve the Azure creds from Vault and then use those in the command below:

```shell
vault read -format=json azure/creds/jenkins > /tmp/azure_creds.json
cat /tmp/azure_creds.json | jq .data.client_id && cat /tmp/azure_creds.json | jq .data.client_secret
echo client_id=$(cat /tmp/azure_creds.json | jq .data.client_id) > client_id.auto.tfvars
echo client_secret=$(cat /tmp/azure_creds.json | jq .data.client_secret) > client_secret.auto.tfvars
```

```shell
terraform apply --auto-approve
```


## Update Vault Dynamic Secrets Configuration

A Vault admin will need to run the configuration below to allow our App to retrieve dynamic mongoDB secrets from Vault.

```shell
resource "vault_mount" "db_azure" {
  path = "mongodb_azure"
  type = "database"
  description = "Dynamic Secrets Engine for WebBlog MongoDB on Azure."
}

resource "vault_database_secret_backend_connection" "mongodb_azure" {
  backend       = vault_mount.db_azure.path
  name          = "mongodb_azure"
  allowed_roles = ["mongodb-azure-role"]

  mongodb {
    connection_url = "mongodb://${var.DB_USER}:${var.DB_PASSWORD}@${var.DB_URL_AZURE}/admin"
    
  }
}

resource "vault_database_secret_backend_role" "mongodb-azure-role" {
  backend             = vault_mount.db_azure.path
  name                = "mongodb-azure-role"
  db_name             = vault_database_secret_backend_connection.mongodb_azure.name
  default_ttl         = "10"
  max_ttl             = "86400"
  creation_statements = ["{ \"db\": \"admin\", \"roles\": [{ \"role\": \"readWriteAnyDatabase\" }, {\"role\": \"read\", \"db\": \"foo\"}] }"]
}
```
I edited this file
