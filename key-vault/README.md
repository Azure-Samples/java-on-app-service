# Using KeyVault References

This tutorial will walk through how to to use Key Vault references in our Java application without modifying our [data source example app](../data-sources) or adding any dependencies. Using Key Vault to store our secrets (such as database connection strings) give us a centralized storage location with management tools like role-based access control, secrets rotation, and encryption.

## Prerequisites

This tutorial continues from the [data source configuration](../data-sources) example. You will need to complete that tutorial before doing this one.

## Setting up a Managed Identity

A system-assigned identity is tied to your application and is deleted if your app is deleted. An app can only have one system-assigned identity. System-assigned identity support is generally available for Windows apps.

```bash
az webapp identity assign --name <app_name> --resource-group <resource_group_of_app>
```

Save the `principalId`

## Provisioning the Key Vault

1. Let's spin up a Key Vault named `java-app-key-vault`. For information about the parameters specified below, run `az keyvault create --help`.

    ```bash
    az keyvault create --name java-app-key-vault              \
                       --resource-group <your_resource_group> \
                       --location <location>                  \
                       --enabled-for-deployment true          \
                       --enabled-for-disk-encryption true     \
                       --enabled-for-template-deployment true \
                       --sku standard
    ```

1. Give the System assigned identity `get` and `list` access to the Key Vault

    ```bash
    az keyvault set-policy --name java-app-key-vault     \
                           --secret-permission get list  \
                           --object-id <the principal ID from earlier>
    ```

1. Now we will add the Postgres username, password, and URL to the Key Vault. If you followed the tutorial on data sources, you should still have the secrets saved as environment variables on your machine. (If you are using Powershell, use the `$env:ENV_VAR` syntax to inject the environment variable into the command).

    ```bash
    az keyvault secret set --name POSTGRES-USERNAME      \
                       --value $POSTGRES_USERNAME        \
                       --vault-name java-app-key-vault
    az keyvault secret set --name POSTGRES-PASSWORD      \
                       --value $POSTGRES_PASSWORD        \
                       --vault-name java-app-key-vault
    az keyvault secret set --name POSTGRES-URL           \
                       --value $POSTGRES_URL             \
                       --vault-name java-app-key-vault
    ```

## Configuring our App

Our Spring app can access these secrets when deployed on App Service through [Application Settings](https://docs.microsoft.com/en-us/azure/app-service/web-sites-configure#app-settings). We will now reconfigure our Maven "production" profile and the App Service plugin. 

1. First, we need the URI's of our three secrets. Run the commands below and in each output copy the `id` field.

    ```bash
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-USERNAME
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-PASSWORD
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-URL
    ```

1. Add the following to the Azure App Service plugin section of the `pom.xml`. Adding this configuration will create Application Settings with the given name and value. The value of each setting should be a Key Vault reference with the corresponding `id` from the previous step.

    ```xml
    <appSettings>
        <property>
            <name>POSTGRES_USERNAME</name>
            <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
        </property>
        <property>
            <name>POSTGRES_PASSWORD</name>
            <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
        </property>
        <property>
            <name>POSTGRES-URL</name>
            <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
        </property>
    </appSettings>
    ```

    A Key Vault reference is of the form `@Microsoft.KeyVault(SecretURI)`, where `SecretURI` is data-plane URI of a secret in Key Vault, including a version. There is an alternate syntax [documented here](https://docs.microsoft.com/en-us/azure/app-service/app-service-key-vault-references#reference-syntax).

1. Now we will edit the "production" profile so that our `application.properties` file will be hydrated with the environment variables at runtime. In the "production" profile of `pom.xml`, make the following change:

    ```xml
    
    ```

## Deploy and Test

- The connection should be made
- SSH in and print the values, give command to do so

## Next Steps

- Azure SDK

## Helpful Links

- [You can use the Azure Spring Boot Stater to do this differently](https://github.com/microsoft/azure-spring-boot/tree/master/azure-spring-boot-samples/azure-keyvault-secrets-spring-boot-sample)