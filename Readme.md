# NPD Find & Explore Azure Resource Manager templates

## To do

The following have not yet been added to these templates, so should be configured as required for your service:

- [ ] Monitoring
- [ ] Aggregated logging
- [ ] Backups for PostgreSQL
- [ ] Azure Blob Store for ActiveStorage
- [ ] Azure Cache for Redis for background jobs/caching
- [ ] Bonus points: Azure Search integration
- [ ] SSH access to Web App Service (see the end of this Readme)

## Prerequisites

- [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest)
- [jq](https://stedolan.github.io/jq/) for the helper scripts in this readme.

These scripts have been tested on MacOS. The ARM templates should work fine on any OS, but you may need to adapt the commands shown here to your operating system.

## Resources these templates create

- 1x Resource Group
- 1x Container Registry
- 1x App Service plan
- 1x App Service (production)
- 1x Web App (staging)
- 2x Azure Database for PostgreSQL server


## How to run the templates

The ARM templates are idempotent, so are safe to be run repeatedly. Re-running a template when the resources already exist will result in a no-op.

> Note: these templates do not destroy existing resources that don't match the definitions provided, so (for example) renaming a service in the template and re-running it would simply add a second copy of the resource with the new name.

The following instructions and all the files in this branch are set up to be ran in development environment.

To set up the app in the production subscription instead, replace 'development' with 'production' and 's112d01' with 's112p01'.

First, you'll need to login:

```bash
az login
az account set --subscription "s112-findandexploredatainnpd-development"
```

Then you can run each template in order. For each command you will want to pass:

1. The resource group
2. The ARM template file
3. The common parameters file (optional)
4. The parameters required by the template

The following script runs through each ARM template, using environment variables to save typing common strings repeatedly:

> Important: make sure to change the two PostgreSQL passwords and Rails master key at the top of this script before running.

```bash
POSTGRES_ADMIN_PASSWORD=CHANGE_THIS_TO_A_VERY_SECURE_PASSWORD_^100%
POSTGRES_ADMIN_PASSWORD_STAGING=CHANGE_THIS_TO_A_DIFFERENT_VERY_SECURE_PASSWORD_^100%
RAILS_MASTER_KEY=TODO_COPY_FROM_RAILS

# Create the resource group to contain everything
az deployment create \
  --location westeurope \
  --template-file 0_resource_groups.json \
  --parameters @_common.parameters.json

# This is the resource group assigned by DfE Platform – the typo is necessary.
RESOURCE_GROUP=s112d01-find-npd-data-persistent-resources

# Create the container registry
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 1_container_registry.json \
  --parameters @_common.parameters.json

AZ_CR_NAME=s112d01-find-npd-data

# Retrieve Azure Container Registry credentials:
AZ_CR_CRED_JSON=`az acr credential show --name $AZ_CR_NAME`
AZ_CR_USERNAME=`echo $AZ_CR_CRED_JSON | jq -r '.["username"]'`
AZ_CR_PASSWORD=`echo $AZ_CR_CRED_JSON | jq -r '.["passwords"][0]["value"]'`

# Note: now is a good time to deploy an image to your container registry

# Create the PostgreSQL database, Web App instance
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 5_web_app_service.json \
  --parameters @_common.parameters.json \
  --parameters administratorLogin=npd_admin \
  --parameters administratorLoginPassword=$POSTGRES_ADMIN_PASSWORD \
  --parameters stagingAdministratorLoginPassword=$POSTGRES_ADMIN_PASSWORD_STAGING \
  --parameters dockerRegistryUrl=https://${AZ_CR_NAME}.azurecr.io \
  --parameters dockerRegistryUsername=$AZ_CR_USERNAME \
  --parameters dockerRegistryPassword=$AZ_CR_PASSWORD \
  --parameters dockerImageName=${AZ_CR_NAME}.azurecr.io/dfedigital_find_npd_data_web:latest \
  --parameters railsMasterKey=$RAILS_MASTER_KEY \
  --parameters customHostname=find-npd-data.education.gov.uk

# Retrieve WebApp outbound IP addresses
AZ_WEBAPP_IPS=`az webapp show --resource-group $RESOURCE_GROUP --name s112p01-find-npd-data | jq -r '.["possibleOutboundIpAddresses"]'`

# Lock down the PostgreSQL firewall
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 6_postgres_firewall.json \
  --parameters @_common.parameters.json \
  --parameters webAppIPAddresses=$AZ_WEBAPP_IPS
```

Don't be worried when your app fails to start. This is expected. You've created a container registry and pointed the Web App to use it, however there aren't any images in there yet. At this point, you will want to configure your CI service to deploy the production container it builds to the container registry, and trigger a deploymnt to the Web App instance.

## Important steps after running the templates

1. Enable diagnostic logging on both Web App slots. The templates aren't honoured for application logs, so these need to be set to retain 100Mb (the maximum) and 365 days.

2. Update the Postgres backup retention to 31 days. This is under the "Pricing tier" section in the Azure portal.

## Connecting to the production system

> This hung for me in testing... it needs an extra process in the Docker container :/

```bash
az webapp ssh --resource-group s112p01-find-npd-data-persistent-resources --name s112p01-find-npd-data
```

## Logs

The logs are available through the Azure Portal, or using the following CLI command:

```bash
az webapp log tail --resource-group $RESOURCE_GROUP --name s112p01-find-npd-data
```

## Contributing

Bug reports and pull requests are welcome on GitHub at
https://github.com/DfE-Digital/npd-find-and-explore-infra. This project is
intended to be a safe, welcoming space for collaboration, and contributors are
expected to adhere to the [Contributor Covenant](CODE_OF_CONDUCT.md)
code of conduct.

### Standards & principles

There are a handful of government-specific principles and standards we strive to adhere to:

1. [The DfE Coding Principles](https://dfe-digital.github.io/technology-guidance/principles/coding-principles/#coding-principles)
2. [The GDS Service Standard](https://www.gov.uk/service-manual/service-standard), points 8, 9, and 10 are particularly pertinent. It's also worth referring to the [GDS Service Manual](https://www.gov.uk/service-manual/technology).

### Other DfE environments

This set of templates has been crafted with naming schemes for the DfE CIP Azure tenancy. The `non-cip-azure` branch has an example of the changes you might want to make to fit another naming scheme.

## License

This project is made available under the terms of the [MIT License](LICENCE.md).
