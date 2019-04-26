# NPD Find & Explore Azure Resource Manager templates

### TODO

- [ ] Explain resource groups
- [ ] Store/pull DB credentials from Key Vault

### 1_find-npd-data-persistent

This ARM template and resource group contains the long-lived resources for Find & Explore:

1. The `find-npd-data-persistent` Resource Group
2. TODO The production PostgreSQL database
3. TODO The production Redis instance
4. TODO The production blob store

### 2_find-npd-data-transient

This ARM template and resource group contains the stateless and short-lived resources for Find & Explore:

1. TODO The `find-npd-data-persistent` Resource Group
2. TODO The production and staging Web App Service
3. TODO The staging PostgreSQL database
4. TODO The staging Redis instance
5. TODO The staging blob store

### Running the scripts

First, you'll need to login:

```bash
az login
```

```bash
POSTGRES_ADMIN_PASSWORD=CHANGE_THIS_TO_A_VERY_SECURE_PASSWORD_^100%
POSTGRES_ADMIN_PASSWORD_STAGING=CHANGE_THIS_TO_A_DIFFERENT_VERY_SECURE_PASSWORD_^100%
RAILS_MASTER_KEY=TODO_COPY_FROM_RAILS 

az deployment create \
  --location westeurope \
  --template-file 0_resource_groups.json \
  --parameters @_common.parameters.json

RESOURCE_GROUP=s112p01-find-npd-data-persistent-resources

az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 1_container_registry.json \
  --parameters @_common.parameters.json

# Retrieve Azure Container Registry credentials:
AZ_CR_CRED_JSON=`az acr credential show --name s112p01findnpddata`
AZ_CR_USERNAME=`echo $AZ_CR_CRED_JSON | jq -r '.["username"]'`
AZ_CR_PASSWORD=`echo $AZ_CR_CRED_JSON | jq -r '.["passwords"][0]["value"]'`

# TODO: existingKeyVaultId
# TODO: existingKeyVaultSecretName
az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 5_web_app_service.json \
  --parameters @_common.parameters.json \
  --parameters administratorLogin=npd_admin \
  --parameters administratorLoginPassword=$POSTGRES_ADMIN_PASSWORD \
  --parameters stagingAdministratorLoginPassword=$POSTGRES_ADMIN_PASSWORD_STAGING \
  --parameters dockerRegistryUsername=$AZ_CR_USERNAME \
  --parameters dockerRegistryPassword=$AZ_CR_PASSWORD \
  --parameters railsMasterKey=$RAILS_MASTER_KEY \
  --parameters customHostname=find-npd-data.education.gov.uk

# Retrieve WebApp outbound IP addresses 
AZ_WEBAPP_IPS=`az webapp show --resource-group $RESOURCE_GROUP --name s112p01-find-npd-data | jq -r '.["possibleOutboundIpAddresses"]'`

az group deployment create \
  --resource-group $RESOURCE_GROUP \
  --template-file 6_postgres_firewall.json \
  --parameters @_common.parameters.json \
  --parameters webAppIPAddresses=$AZ_WEBAPP_IPS
```
# Logs

```bash
az webapp log tail --resource-group find-npd-data-persistent-resources --name find-npd-data
```
# SSH into production

> TODO This hung for me in testing... it needs an extra process in the Docker container :/

```bash
az webapp ssh --resource-group find-npd-data-persistent-resources --name find-npd-data
```

# TODO!
> TODO steps -
- [ ] deploy key vault 
- [ ] deploy ARM template
- [ ] configure pipeline to deploy to web app service