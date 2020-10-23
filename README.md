# springkeyvaultexample

#################
# This guide will walk you through the end to end process of: 
#       1. Building a springboot application which pulls secret values from Azure KeyVault 
#           (Secrets are made avaialble to the application via environment variables after they are pulled from KeyVault)  -- This happens automagically 
#       2. Containerizing/Building docker Image of that application
#       3. Pushing the docker image to your image Repository
#       4. Deploying the image to a kubernetes cluster with rolling updates enabled.   
#################

#################
# Pre-reqs: 
#   1.  Azure Subscription with appropriate permissions to create resources
#   2.  Kubernetes Cluster 
#   3.  JAVA 
#################


# #Create Azure Resource Group if you don't have one already created.   
az group create --name <InsertNameHere> -l <InsertLocationHere>     #replace the name and location variables   

# #Create Azure Key Vault  
az keyvault create --name <InsertNameHere> -g <InsertResourceGroupName>    #replace the name and ResourceGroup name variables

# # replace the variable below with the vault name to be used in next step
# # —> https://<InsertVaultNameHere>.vault.azure.net/
# # Example: https://myExampleKeyVaultName.vault.azure.net/  #Please make a note of this URL.   

# Set your secrets in Vault 
 az keyvault secret set --vault-name tkgi \
     --name "<NameOfSecret>" \
     --value "<VauleOfSecret>"

# #Grant your app access to KeyVault 
az keyvault set-policy --name tkgi --object-id <InsertObjectIDofServicePrincipal>  --secret-permissions set get list

# #Optional Step - Generate a sample project using start.spring.io    #Only if your starting from scratch 
curl https://start.spring.io/starter.tgz -d dependencies=web,azure-keyvault-secrets -d baseDir=springapp -d bootVersion=2.3.1.RELEASE -d javaVersion=1.8 | tar -xzvf -


# # Specify your key vault in your app properties 
# Edit —>  src/main/resources/application.properties
# Add the following to the properties file
    ###############################################################
    #               azure.keyvault.enabled=true
    #               azure.keyvault.uri=${vaulturl}
    #               azure.keyvault.client-id=${clientid}
    #               azure.keyvault.client-key=${clientkey}
    #               azure.keyvault.tenant-id=${tenantid}
    ################################################################

During Build process of your Code --> 
                                    # Export DEVELOPMENT-ONLY K8s Secrets to environment 
                                    # This will ensure you code builds however it will not save the environment variables with the JAR.   
                                    # When running the JAR, it will require these values to be set via host env or kubenetes secrets depending on your target environment.  
                                ############################################################    
                                #    export clientid=############################
                                #    export clientkey=############################
                                #    export tenantid=############################
                                #    export vaulturl=https://<InsertVaultNameHere>.vault.azure.net
                                ############################################################    

#Build the code
./mvnw clean package 

# Create K8s Secret 
kubectl create secret generic dev-kv-secret --from-literal=azure_keyvault_client-id=$clientid --from-literal=azure_keyvault_client-key=$clientkey --from-literal=azure_keyvault_tenant-id=$tenantid --from-literal=azure_keyvault_uri=$vaulturl

#Build docker image from target JAR produced by build
docker build -t <YourImageRepoLocationNameAndTag> .

#Push docker image to Repository 
docker push <YourImageRepoLocationNameAndTag>:<TagName>

#Deploy new image to K8s Cluster 
kubectl apply -f deployment.yaml

# Initiate Rolling Update 
kubectl rollout restart -f deployment.yaml