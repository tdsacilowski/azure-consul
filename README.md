# Consul on Azure
This guide provides a basic deployment of a Consul cluster in Azure. This is a high-level overview of the environment that is created:

* Creates a Resource Group to contain all resources created by this guide
* Creates a virtual network, one public subnet, and three private subnets
* Creates a publically-accessible jumphost for SSH access in the public subnet
* Creates one virtual machine in each private subnet using a custom managed image with Consul installed and configured
* Uses Consul's cloud auto-join to connect the three instances into one Consul cluster

## Deployment Prerequisites

1. In order to perform the steps in this guide, you will need to have an Azure subscription for which you can create Service Principals as well as network and compute resources. You can create a free Azure account [here](https://azure.microsoft.com/en-us/free/).

2. Certain steps will require entering commands through the Azure CLI. You can find out more about installing it [here](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli).

3. Create Azure API Credentials: set up the main Service Principal that will be used for Packer and Terraform:
    * [https://www.terraform.io/docs/providers/azurerm/index.html]()
    * The above steps will create a Service Principal with the [Contributor](https://docs.microsoft.com/en-us/azure/active-directory/role-based-access-built-in-roles#contributor) role in your Azure subscription

4. `export` environment variables for the main (Packer/Terraform) Service Principal. For example, create a `.sh` file with the following values (obtained from step `1` above):

    ```
    export ARM_SUBSCRIPTION_ID="xxxxxxxx-yyyy-zzzz-xxxx-yyyyyyyyyyyy"
    export ARM_CLIENT_ID="xxxxxxxx-yyyy-zzzz-xxxx-yyyyyyyyyyyy"
    export ARM_CLIENT_SECRET="xxxxxxxx-yyyy-zzzz-xxxx-yyyyyyyyyyyy"
    export ARM_TENANT_ID="xxxxxxxx-yyyy-zzzz-xxxx-yyyyyyyyyyyy"
    ```

5. Finally, create a read-only Azure Service Principal (using the Azure CLI) that will be used to perform the Consul auto-join (make note of these values as you will use them later in this guide):

    ```
    $ az ad sp create-for-rbac --role="Reader" --scopes="/subscriptions/[YOUR_SUBSCRIPTION_ID]"
    ```

## Deployment Steps

### Create Packer Image(s) In your Azure Subscription

Once you've created the appropriate API credentials for Packer/Terraform, you will now create the Azure managed images that will be used to deploy our cluster(s).

First, you will need to create a Resource Group in your Azure subscription in which to store the managed images. In the example below, we've used `"PackerImages"`. If you use a different name, make sure to use the new resource group when running the Packer template (e.g. `AZURE_RESOURCE_GROUP="PackerImages"`)

On your client machine:

1. Install Packer. See [https://www.packer.io/docs/install/index.html]() for more details.

2. `git clone` the `tdsacilowski/azure-consul` repository

3. Run the `azure-consul/packer/consul/consul.json` template:

    ```
    $ cd packer/consul

    # Validate the Packer template
    $ AZURE_RESOURCE_GROUP="PackerImages" AZURE_LOCATION="West US" PACKER_ENVIRONMENT="dev" CONSUL_VERSION="0.9.2" packer validate consul.json

    # Build the templates
    $ AZURE_RESOURCE_GROUP="PackerImages" AZURE_LOCATION="West US" PACKER_ENVIRONMENT="dev" CONSUL_VERSION="0.9.2" packer build consul.json
    ```

4. Once the Packer build process is complete, you can output the resource ID for the new image using the Azure CLI:

    ```
    $ az image list --query "[?tags.Product=='Consul'].id"

    [
      "/subscriptions/xxxxxxxx-yyyy-zzzz-0000-123456789xyz/resourceGroups/PACKERIMAGES/providers/Microsoft.Compute/images/dev-consul-server-0.9.2-Ubuntu_16.04"
    ]
    ```

    Make note of this resource ID as you will use this in your Terraform template as described below.

### Deploy the Consul Cluster

Once the Packer image creation process is complete, you can begin the process of deploying the remainder of this guide to create a Consul cluster in your Azure subscription.

1. `cd` into the `azure-consul/terraform` directory

2. At this point, you will need to customize the `terraform.tfvars` with your specific values. There's a `terraform.tfvars.example` file provided. Update the appropriate values:

    * `custom_image_id` will be the Azure managed image resource ID that you queried for in the previous section (e.g. `"/subscriptions/xxxxxxxx-yyyy-zzzz-0000-123456789xyz/resourceGroups/PACKERIMAGES/providers/Microsoft.Compute/images/dev-consul-server-0.9.2-Ubuntu_16.04"`)

    * `auto_join_subscription_id`, `auto_join_client_id`, `auto_join_client_secret`, `auto_join_tenant_id` will use the values obtained from creating the read-only auto-join Service Principal created in step #5 of the Deployment Prerequisites earlier.

3. Run `terraform init` to initialize the working directory and download appropriate providers

4. Run `terraform plan` to verify deployment steps and validate all modules

5. Finally, run `terraform apply` to deploy the Consul cluster

### Verify Deployment

* SSH into jumphost, then SSH into Consul servers:
```
jumphost_ssh_connection_strings = [
    ssh-add private_key.pem && ssh -A -i private_key.pem azure-user@13.64.0.0
]
consul_private_ips = [
    ssh azure-user@172.31.48.4,
    ssh azure-user@172.31.64.4,
    ssh azure-user@172.31.80.4
]
```

* Once logged into any one of your Consul servers, run `consul members` to view the status of the cluster:

```
$ consul members

Node             Address           Status  Type    Build  Protocol  DC
consul-westus-0  172.31.48.4:8301  alive   server  0.9.2  2         dc1
consul-westus-1  172.31.64.4:8301  alive   server  0.9.2  2         dc1
consul-westus-2  172.31.80.4:8301  alive   server  0.9.2  2         dc1
```
