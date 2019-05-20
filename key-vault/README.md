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

### Key Vault References

Our Spring app will be able to access these secrets when deployed on App Service through [Application Settings](https://docs.microsoft.com/en-us/azure/app-service/web-sites-configure#app-settings). We will now configure the App Service plugin to set the Application Settings when we deploy the app.

1. First, we need the URI's of our three secrets. Run the commands below and copy the `id` value in each.

    ```bash
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-USERNAME
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-PASSWORD
    az keyvault secret show --vault-name java-app-key-vault --name POSTGRES-URL
    ```

1. Add the following to the Azure App Service plugin section of the `pom.xml`. Adding this configuration will create Application Settings with the given name and value. The value of each setting should be a Key Vault reference with the corresponding `id` from the previous step.

    ```xml
    <appSettings>
      <property>
        <name>SPRING_DATASOURCE_URL</name>
        <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
      </property>
      <property>
        <name>SPRING_DATASOURCE_USERNAME</name>
        <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
      </property>
      <property>
        <name>SPRING_DATASOURCE_PASSWORD</name>
        <value>@Microsoft.KeyVault(INSERT_ID_HERE)</value>
      </property>
    </appSettings>
    ```

    A Key Vault reference is of the form `@Microsoft.KeyVault(SecretURI)`, where `SecretURI` is data-plane URI of a secret in Key Vault, including a version. There is an alternate syntax [documented here](https://docs.microsoft.com/en-us/azure/app-service/app-service-key-vault-references#reference-syntax).

### Environment Configuration

The Key Vault references will be replaced with the actual secrets when our App Service boots up. This means our Spring Application needs to resolve the connection strings at runtime, but currently gets these strings at build time. We also want to be able to use our H2 database for development, and optionally connect to the production DB from our local machine to run tests. To fill all these requirements, we will create two new configuration files: `application-dev.properties`, and `application-prod.properties`.

1. Create a file under `src/main/resources` named `application-dev.properties`. Copy/paste the following into the file:

    ```txt
    # ===============================
    # = DATA SOURCE
    # ===============================
    # Set here configurations for the database connection
    spring.datasource.url=jdbc:h2:mem:testdb
    spring.datasource.username=sa
    spring.datasource.password=
    spring.datasource.driver-class-name=org.h2.Driver

    # ===============================
    # = JPA / HIBERNATE
    # ===============================

    # Allows Hibernate to generate SQL optimized for a particular DBMS
    spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.H2Dialect

    # App Service
    server.port=8080
    ```

1. Create a file under `src/main/resources` named `application-dev.properties`. Copy/paste the following into the file. We do not set the connection strings here. Instead, Spring will resolve them at runtime by looking for the uppercase and underscored versions of `spring.datasource.url`, `spring.datasource.username`, and `spring.datasource.password`.

    ```txt
    # ===============================
    # = DATA SOURCE
    # ===============================
    
    # The connection URL, username, and password will be sourced from environment variables
    # on App Service
    
    # Set here configurations for the database connection
    spring.datasource.driver-class-name=org.postgresql.Driver
    
    # ===============================
    # = JPA / HIBERNATE
    # ===============================
    
    # Allows Hibernate to generate SQL optimized for a particular DBMS
    spring.jpa.properties.hibernate.dialect=org.hibernate.dialect.PostgreSQLDialect
    
    # App Service
    server.port=80
    ```

1. Now we can slim-down our original `application.properties` file. Replace the contents of `application.properties` with the following.

    ```txt
    # Active profile is set by Maven
    spring.profiles.active=@spring.profiles.active@
    
    # ===============================
    # = DATA SOURCE
    # ===============================
    
    # Keep the connection alive if idle for a long time (needed in production)
    spring.datasource.testWhileIdle=true
    spring.datasource.validationQuery=SELECT 1
    
    # ===============================
    # = JPA / HIBERNATE
    # ===============================
    # Show or not log for each sql query
    spring.jpa.show-sql=true
    
    # Hibernate ddl auto (create, create-drop, update): with "create-drop" the database
    # schema will be automatically created afresh for every start of application
    spring.jpa.hibernate.ddl-auto=create
    
    # Naming strategy
    spring.jpa.hibernate.naming.implicit-strategy=org.hibernate.boot.model.naming.ImplicitNamingStrategyLegacyHbmImpl
    spring.jpa.hibernate.naming.physical-strategy=org.springframework.boot.orm.jpa.hibernate.SpringPhysicalNamingStrategy
    ```

1. Finally, we can also slim down our Maven profiles because we have moved th information to the new properties files. The profile section of your `pom.xml` should now be the following:

    ```xml
    <profiles>
      <profile>
        <!-- This profile will configure Spring to use an in-memory database for local development and testing. -->  
        <id>dev</id>  
        <activation>
            <activeByDefault>true</activeByDefault>
        </activation>  
        <properties>
            <spring.profiles.active>dev</spring.profiles.active>
        </properties>
      </profile>  
      <profile>
        <!-- This profile will configure the application to use our Azure PostgreSQL server. -->  
        <id>prod</id>  
        <properties>  
            <spring.profiles.active>prod</spring.profiles.active>
        </properties>
      </profile>
    </profiles>
    ```

## Deploy and Test

Check that the development profile works as expected by running the following commands and opening a browser to `http://localhost:8080/`.

    ```bash
    mvn clean package -Pdev
    java -jar target/app.jar
    ``` 

Before deploying to App Service, you can build your application with the production profile and test against your PostgreSQL DB from your local machine. To do so, simply rename the three environment variables beginning with `POSTGRES_` and rename them to `SPRING_DATASOURCE_URL`, `SPRING_DATASOURCE_USERNAME`, and `SPRING_DATASOURCE_PASSWORD` respectively. Run the following commands to build and start your app. Spring will resolve the connection strings in the environment variables at runtime.

    ```BASH
    mvn clean package -Pprod
    java -jar target/app.jar
    ```  

Finally, deploy the production app to App Service with `mvn azure-webb:deploy`. Browse to the application and test that the application works properly.

## Next Steps

- Azure SDK

## Helpful Links

- [You can use the Azure Spring Boot Stater to accomplish this scenario as well](https://github.com/microsoft/azure-spring-boot/tree/master/azure-spring-boot-samples/azure-keyvault-secrets-spring-boot-sample)