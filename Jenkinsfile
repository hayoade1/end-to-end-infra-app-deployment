#!/usr/bin/env groovy
def ROLE_ID = "00d28979-e55d-b862-c860-ee1f9d0f3ded" //this is the Role-id for the Pipeline and needs to be inserted manually by a Vault Admins
def VAULT_ADDR = "http://13.92.96.202:8200"
env.VAULT_ADDR = VAULT_ADDR

node() {
  timestamps{
  // Below are the Jenkins AppRole credentials provided by the Vault Admin. These are Jenkins wide credentials and are not to be confused with the Jenkins Pipeline AppRole credentials that are specific to this pipeline for this application
  withCredentials([
        [
            $class: 'VaultTokenCredentialBinding',
            credentialsId: 'vault_token',
            vaultAddr: 'http://13.92.96.202:8200'
        ]
    ]){
        // The Jenkins Node is only allowed to create the wrapped secret ID and with a wrap-ttl between 100s and 300s
        stage ('Jenkins Node Creates a Wrapped Secret ID for the Pipelines') {
          
            def WRAPPED_SID = ""
            env.WRAPPED_SID = sh(
              returnStdout: true,
              script: "vault write -field=wrapping_token -wrap-ttl=200s -f auth/pipeline/role/pipeline-approle/secret-id"
            )
        }
        // the stage below doesn't work with the Vault Jenkins policy to write wrapped secrets only, an admin needs to manually provide the ROLE_ID as defined at the top of this file
        // stage ("Get Role ID for the pipeline AppRole") {
        //   def ROLE_ID = ""
        //   env.ROLE_ID = sh(
        //     returnStdout: true,
        //     script: "vault read -field=role_id auth/pipeline/role/pipeline-approle/role-id"
        //   )
        // }
        stage("Unwrap Secret ID"){
          def UNWRAPPED_SID = ""
          env.UNWRAPPED_SID = sh(
            returnStdout: true,
            script: "vault unwrap -field=secret_id ${WRAPPED_SID}"
          )
        }
      }
        stage("Pipeline gets login token with Role ID and unwrapped Secret ID"){
          def VAULT_LOGIN_TOKEN = ""
          env.VAULT_LOGIN_TOKEN = sh(
            returnStdout: true,
            script: "vault write -field=token auth/pipeline/login role_id=${ROLE_ID} secret_id=${UNWRAPPED_SID}"
          )
        }
        stage("Log into Vault with Pipeline AppRole") {
          def VAULT_TOKEN = ""
          env.VAULT_TOKEN = sh(
            returnStdout: true,
            script: "vault login -field=token ${VAULT_LOGIN_TOKEN}"
          )
        }
        stage("Create Azure Creds for Terraform to Provision App VMs") {
          sh(returnStdout:false, script: "vault read -format=json azure/creds/jenkins > /tmp/azure_creds.json")
          sh '''
          cat /tmp/azure_creds.json | jq .data.client_id && cat /tmp/azure_creds.json | jq .data.client_secret
          echo client_id=$(cat /tmp/azure_creds.json | jq .data.client_id) > /var/jenkins_home/workspace/Webblog_App@script/Terraform/ProvisionAppVMs/client_id.auto.tfvars
          echo client_secret=$(cat /tmp/azure_creds.json | jq .data.client_secret) > /var/jenkins_home/workspace/Webblog_App@script/Terraform/ProvisionAppVMs/client_secret.auto.tfvars
          '''
        }
    
       
        stage("Retrieve TFC Token from Vault") {
         sh 'echo token = "$(vault kv get -field=terraform kv/terraform)"'  
         sh 'echo "Lunch and Learn April 2020"'   
        } 
    

      //  stage("Terraform to Provision the 3 App VMs + Consul Server VM in Azure") {
          // Search for the output FQDN from Terraform using jq and feed it into the inventory file of Ansible
         
      //    sh '''
       //       cp /var/jenkins_home/.terraformrc /var/jenkins_home/workspace/Webblog_App@script/Terraform/ProvisionAppVMs
        //      cd /var/jenkins_home/workspace/Webblog_App@script/Terraform/ProvisionAppVMs
        //     #terraform destroy --auto-approve
         //     terraform init
          //    terraform fmt
          //    terraform validate
          //    terraform apply --auto-approve
         //     sed -i "s/<placeholder_app>/$(terraform output -json webblog_public_dns | jq -r '.["samg-webblog-01-ip"]')/g" ../../Ansible/WebblogApp/inventory
          //    sed -i "s/<placeholder_db>/$(terraform output -json webblog_public_dns | jq -r '.["samg-webblog-02-ip"]')/g" ../../Ansible/WebblogApp/inventory
           //   sed -i "s/<placeholder_consul_server>/$(terraform output -json webblog_public_dns | jq -r '.["samg-webblog-03-ip"]')/g" ../../Ansible/WebblogApp/inventory
      //    '''
      //  }  
       // stage("Create Role-id and Wrapped Secret-id for the Vault Agent on App VM") {
          // Ansible to send the Role-id and the Wrapped Secret-id give 15 minutes for the wrap ttl to give enough time for ansible to prep the machines.
  //        sh '''
      //        vault read -field=role_id auth/approle/role/webblog-approle/role-id > /tmp/app_role_id
       //       vault write -field=wrapping_token -wrap-ttl=900s -f auth/approle/role/webblog-approle/secret-id > /tmp/app_wrap_secret_id
     //     '''
      //  }
      //  stage("Ansible to Configure Webblog App VM Machines") {
          // We will need to install Vault to use the Vault agent and Consul for service mesh
          // Grab MongoDB Root Credentials from Vault and pass it to Ansible to create the MongoDB Container
          // set +x below is to hide the response from vault with the mongo creds from displaying in Jenkins' logs
    //      sh '''
      //        set +x
       //       cd /var/jenkins_home/workspace/Webblog_App@script/Ansible/WebblogApp
         //     ansible-playbook -i inventory --extra-vars "mongo_root_user=$(vault kv get -field=username internal/webblog/mongodb) mongo_root_password=$(vault kv get -field=password internal/webblog/mongodb)" appPlaybook.yaml
     //     '''
 //       } 
  }          
}
//
