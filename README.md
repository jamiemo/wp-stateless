# Stateless WordPress in Azure
This project is to deploy WordPress on Azure in a cost effective and secure deployment. Not really stateless, but the state is contained within a MySQL flexible database and a storage account file share, allowing WordPress to be redeployed without concern for the installation.

The typical deployment of the database and web server on a single instance can be noteably cheaper, but is fraught with issues:

- How do you protect the instance or recover in the event of a hack?  
- How can you verify the integrity of the WordPress install after an intrusion?

In this case it is easy to redeploy from a known good source.

The aims of the project:

- Use only Azure services, not third party domain registration or SSL, where possible
- Deploy the latest version of WordPress in an automated manner (not waiting for third parties to package)
- Have some form of backup (30 days MySQL snapshots and 30 days soft delete on the storage account file share)
- Use a custom domain with SSL/TLS
- Scalable web and database tiers
- Be a cost effective deployment
- Automated patching of [MySQL](https://docs.microsoft.com/azure/mysql/flexible-server/concepts-maintenance), and the [App Service OS and runtime](https://docs.microsoft.com/azure/app-service/overview-patch-os-runtime)
- Locks are applied to all stateful data (storage account file share and MySQL database) to avoid accidental deletion

## Costs
The major costs are the App Service and the MySQL database. These are the most scaled down versions practical while allowing for the minimum required features.

Estimates from the Azure console:

- MySQL flexible server (Standard_B1s): AUD$16.83/month
- App Service (B1): AUD$19.04/month

There are additional costs for the storage account, DNS, custom domain and traffic egress.

## Prerequisites
The following prerequisites are required:

- A Resource Group
- A custom domain via App Services Domains
- An Azure DNS zone hosting the App Services Domain (created by default when registering an App Services Domain)

Due to the difficulty experienced when trying to register an App Service Domain which requires a USD$1000 purchase history on the subscription or fails with the *"Resource cannot be created because the subscription doesn't have a sufficient payment history"* error, the alternative is:

- A Resource Group
- A third party domain registration of the custom domain
- An Azure DNS zone hosting the custom domain [(All Services | DNS Zones)](https://portal.azure.com/#blade/HubsExtension/BrowseResource/resourceType/Microsoft.Network%2FdnsZones)
- Public delegation of the custom domain to the Azure DNS name servers (nsX-XX.azure-dns.XXX) from the domain registrar

Another alternative is to set Deploy DNS Zone to `true`, and this will deploy the Azure DNS, but fail to verify the custom domain and certificate until the public delegation of the custom domain to the Azure DNS name servers. The template can be redeployed once this is done to complete the deployment incrementally and bind the custom domain and App Service Managed Certificate.

Using third party DNS will prevent the automated verification of the custom domain and App Service Managed Certificate.

## Deployment
### Deploy the Template

[![Deploy To Azure](https://aka.ms/deploytoazurebutton)](https://portal.azure.com/#create/Microsoft.Template/uri/https%3A%2F%2Fraw.githubusercontent.com%2Fjamiemo%2Fwp-stateless%2Fmaster%2Fazuredeploy.json)

### Deploy WordPress to the App Service

Once all components have successfully deployed, deploy WordPress using a bash [Azure Cloud Shell](https://portal.azure.com/#cloudshell/).

If using a specific subscription switch to that:

`az account set --subscription <subscription>`

Deploy the latest WordPress from wordpress.org:

`az webapp deploy --resource-group <resource group> --name <app service> --clean true --type zip --src-url https://wordpress.org/latest.zip`

You can now navigate to the website and complete the WordPress install process.

This process can also be used to update WordPress to the latest version or recover from a compromise.

Once the install is complete, navigate to Settings in the WordPress Dashboard and set the following URLs to be https:
- WordPress Address (URL)
- Site Address (URL)

## Compromises
### MySQL
MySQL is a public instance which will allow public access from any Azure service within Azure to this server (AllowAllWindowsAzureIps). Ideally this would be a [private virtual network](https://docs.microsoft.com/en-us/azure/mysql/flexible-server/concepts-networking-vnet) but requires the App Service to use [regional virtual network integration](https://docs.microsoft.com/azure/app-service/overview-vnet-integration#regional-virtual-network-integration). For further details on the architecture check out [Web app private connectivity to Azure SQL Database](https://azure/architecture/example-scenario/private-web-app/private-web-app).

### MySQL User
MySQL is using the admin account for database access. Ideally this would be using a separate account with access only to the database, which was a built and tested solution, but there was no easy automated way to roll this out. As there are unlikely to be any other databases on the MySQL instance this approach was selected to make deployment as seamless as possible.

### Storage Account
Ideally the Storage Account would use a [private endpoint](https://docs.microsoft.com/en-us/azure/storage/common/storage-private-endpoints) and be limited to a private virtual network, but this comes at an additional hourly cost.

### App Service
Ideally this would use [regional virtual network integration](https://docs.microsoft.com/azure/app-service/overview-vnet-integration#regional-virtual-network-integration) to access other resources over a private virtual network, but this is not available in anything but the [Standard and Premium](https://docs.microsoft.com/en-au/azure/app-service/overview-vnet-integration#limitations) App Service plans.

## Additional Features
Additional features that could be added at additional cost:

- [Azure Backup](https://docs.microsoft.com/azure/backup/backup-afs) for the storage account file share
- [Azure Front Door ](https://docs.microsoft.com/azure/frontdoor/front-door-waf) to scale and protect the App Service
- Connect the App Service to [Redis Cache](https://github.com/uglide/azure-content/blob/master/articles/app-service-web/web-sites-connect-to-redis-using-memcache-protocol.md)
- Connect resources via a private virtual network as above

## Demo Site
A demo site deployed using this template is available here: [https://statelesswordpress.com](https://statelesswordpress.com)