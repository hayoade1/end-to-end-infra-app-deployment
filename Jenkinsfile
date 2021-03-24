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
            credentialsId: 'Jenkins_Node_Vault_AppRole_1',
            vaultAddr: 'http://13.92.96.202:8200'
        ]
    ])
        // the stage below doesn't work with the Vault Jenkins policy to write wrapped secrets only, an admin needs to manually provide the ROLE_ID as defined at the top of this file
        // stage ("Get Role ID for the pipeline AppRole") {
        //   def ROLE_ID = ""
        //   env.ROLE_ID = sh(
        //     returnStdout: true,
        //     script: "vault read -field=role_id auth/pipeline/role/pipeline-approle/role-id"
        //   )
        // }
 
    
       
       
      
  }          
}

