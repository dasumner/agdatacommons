# Ag Data Commons Azure Deployment

## Provisioning
The infrastructure will be provisioned in a customer-provided Azure account.  The starting point for the build will require the Azure resources listed below.  CivicActions can provide the Terraform code to the client to run.  After successful provisioning the client can provide the required credentials and/or access keys to CivicActions.  Alternatively, the customer will provide a privileged account which CivicActions will use to provision the infrastructure.  The required Azure resources are:

1. Microsoft Azure Cloud account Resource Group with service account.
2. Storage Account for Terraform plan and resource files as well as storage for Azure Cloud Shell
3. Key Vault to store account secrets.
4. Application Gateway
5. Virtual Network 
6. Public IP
7. Network Profile
8. Network Security Group
9. Azure Database for MariaDB Servers
10. Container Group
11. Service Principal

Terraform will provision the following infrastructure resources: 

1. Azure Database for MariaDB relational database with MariaDB Server engine
2. Azure Storage Shares for persistent storage of sanitized database backups and uploaded datasets
3. Azure Container Group with Docker Containers for Apache, Drupal, SOLR and client/support
4. Network and firewall configuration rules for MariaDB and Container Groups
5. Virtual network with public and private subnets
6. Application Gateway to manage incoming traffic and application of security rules (firewall)

## Executing the Terraform Plan
It is assumed that the resource group has been created.  The administrator should perform the following tasks via the Azure Portal Command Console:

1. get a list of subscription ID and tenant ID values:
> az account list --query "[].{name:name, subscriptionId:id, tenantId:tenantId}"

2. Set the SUBSCRIPTION_ID environment variable with the value for the subscription where the infrastructure will be deployed:
> az account set --subscription="${SUBSCRIPTION_ID}"

3. Create a service principal for use by Terraform and Jenkins:
> az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/${SUBSCRIPTION_ID}"
Your appId, password, sp_name, and tenant are returned. Make a note of the appId and password.

4. Save these values in a shell script in the console storage:
#!/bin/sh
echo "Setting environment variables for Terraform"
export ARM_SUBSCRIPTION_ID=your_subscription_id
export ARM_CLIENT_ID=your_appId
export ARM_CLIENT_SECRET=your_password
export ARM_TENANT_ID=your_tenant_id

5. Execute the shell script to set the required environment variables.
Save the Terraform file to the console storage.

6. Initialize the Terraform deployment:
> terraform init
The system should return a message indicating successful initialization.

7. Update the resource group name in the Terraform file.

8. Import the resource group into the state file.
> terraform import azurerm_resource_group.<resource_group_name>

9. Review the actions to be completed by the Terraform script with terraform plan command. When ready to create the resource group, apply the Terraform plan as follows:
> terraform apply
Terraform will output the execution plan and prompt to proceed.  Answer yes to proceed.

After provisioning via Terraform the  client will have the option to administer the infrastructure via Terraform, the Azure Portal or Azure Client Interface.  If either of the latter two options are used, it will be critical not to return to Terraform without accurately updating the known state of the infrastructure using the import function in Terraform.  Failure to do so can lead to unwanted deletion and/or recreation of resources under management.  Always examine the plan before agreeing to any changes proposed by Terraform.
 
The Terraform plan, state files and access credentials can be stored on Azure cloud storage and accessed via the Cloud Shell that is part of the Azure portal.  The storage is encrypted and only accessible to pre-defined resources and accounts that have been granted access.  

The resource group will contain the production infrastructure and one or more copies of the stage, QA and test infrastructures.  Each copy of the infrastructure will be composed of a unique Public IP address, MariaDB database service instance, and container group with the Web and SOLR docker containers.

## Migration Tasks
After the infrastructure has been provisioned, the following tasks must be performed to complete the migration from the OnPrem environment to the Azure cloud environment:
1. A full backup of the production database in the existing deployment will be taken.  The database dump files will be migrated to Azure and then loaded into the cloud database via import.
2. To implement HTTPS, an SSL certificate must be obtained by USDA and then loaded into the Azure application gateway.
USDA will have to update DNS to associate the existing data.nal.usda.gov domain with the new public IP in Azure.
3. A service principal account must be created in Azure.  Credentials for the service principal must be loaded into the Jenkins server where pipeline jobs will be executed.
4. Pipelines must be loaded into Jenkins server (manually or via import).

## Database
The database service has integrated backup functionality.  Using a combination of full, incremental and transaction log backups the database can be restored to any point-in-time within the backup retention period.  The integrated firewall only permits connection to the database from the Web and Solr services running in the Docker containers.  All other traffic is blocked.  The restoration process will create a new database server and instance.

The database has options to scale the number of virtual cores (logical CPUs) and RAM assigned to the server.  Storage can be set to a fixed amount or to automatically grow as necessary.  Configuration values can be adjusted via the Azure portal or updating and applying new Terraform code.  Increases in resources will result in service price increases.  Review the estimated price summary in the Azure portal to estimate costs associated with proposed changes.

## Docker Containers
The resources assigned to the Docker containers in the container group can be adjusted up to the cumulative limits of the underlying host.  Azure publishes information on resource availability in each region https://docs.microsoft.com/en-us/azure/container-instances/container-instances-region-availability here.  The Docker container configurations can be adjusted using the Azure portal or via the Terraform code.  Any changes will require that the containers be rebuilt.  This will happen automatically once the changes are committed in the Azure portal or in Terraform.

## Network Configuration
The virtual network has subnets: one public and one private.  The containers exist in the private one, where nothing outside the vnet can access them.  This policy is enforced via the network security group.  On the public subnet, the application gateway resides.  It routes traffic to the containers thus making it reachable while still protected from direct external access.  An external computer will make a request to the public IP on port 80. The application gateway will take the request, forward it to the web container, get the response and send it back to the requestor.

## Operations
The web container is configured with the mysql client and drush.  System maintenance and administration tasks will be performed via this container as it has access to all the underlying components.  Console access to the OS is available via the Azure portal.  Commands can also be run using the Azure CLI.  Only properly credentialed accounts can access the container.  CI/CD Pipeline operations via Jenkins will spawn commands on the web container via the CLI.

## Terraform
Great care should be exercised when making changes with Terraform.  Before agreeing to apply a revised plan, review the details of what will be created, destroyed or changed and consider whether data loss or extended system outages may occur as a result.

## Persistent Storage
There are two persistent storage resources for each deployment.  One stores uploaded files.  The other stores sanitized database backups.  These file shares are separated from the Docker containers so that they remain in place and available in the event that the containers are rebuilt or replaced by upgraded versions.  Automatically-generated nightly snapshots are made of both file shares for disaster recovery.

## CI/CD
QA and Test environments will be provisioned via Jenkins.  The pipeline will provision duplicate copies of the private subnet, container group, Docker containers and database.  A new public IP for the environment will be created and the application gateway will be updated with routing information to direct traffic accordingly.  Although the environments will reside within the same resource group, they will be segregated from each other via network rules.

Jenkins can run pipelines against the environment using the Azure CLI Jenkins plugin.  In order to use this plugin the following steps must be performed:

1. Install the Jenkins plugin from the plugin manager.
2. Install the Azure CLI on the Jenkins server.  https://docs.microsoft.com/en-us/cli/azure/install-azure-cli-apt?view=azure-cli-latest
3. Create a service principal for use with the plugin.  https://docs.microsoft.com/en-us/cli/azure/create-an-azure-service-principal-azure-cli?view=azure-cli-latest
4. Load the service principal credentials into Jenkins from the Credentials submenu.
