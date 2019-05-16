# Configuring Data Sources

These instructions will walk through the process of configuring a PostgreSQL database for a Spring Boot app on App Service Linux. Our dependencies are managed using Spring Starters, which abstract the necessary dependencies for common development scenarios (such as cache, data access, or messaging). This tutorial starts at the `initial/` directory and works towards the `complete/` directory.

## Run Locally

Our Spring Boot application uses Spring Data JPA to map our Java objects to relational objects in the PostgreSQL database. Under the hood, Spring Data JPA uses Hibernate, an implementation of the JPA specification. This means we use Spring's annotations and interfaces instead of  explicitly writing SQL commands in our application. This project is configured with an in-memory H2 database for local development and testing. To run the app locally, run the following commands from the `initial/` directory.

```bash
mvn clean package
java -jar target/app.jar
```

Open a browser to `http://localhost:8080/` and you will see a simple link to create a new product. Try creating a new product with an example description, price, and image URL. You can see the SQL commands in the terminal. If you kill the app and restart it, the contents will be lost because H2 is an embedded, in-memory database. You can see all the items in the database by browsing to `http://localhost:8080/product/list`

> When we deploy to App Service, the application will instead listen on port 80.

## Provision the Server

You can provision a Postgres database through the Portal by going to Create a Resource and searching for "Azure Database for PostgreSQL". In the following screen enter your desired server name, admin username, and password.

Alternatively, you can use the following Azure CLI command to create the database by replacing the placeholder values. To get a full list of all the Azure locations (aka "regions"), run `az account list-locations`. Don't forget your username and password!

```bash
az postgres server create -n <desired-name> -g <resource-group> --sku-name B_Gen5_1 -u <username> -p <password> -l centralus
```

### Configure the Server

Right now the server requires SSL to connect, does not allow access to Azure services, and does not allow access from our client machine. In the portal, navigate to the "Connection Security" portion of your PostgreSQL Server and update the following:

- "Allow access to Azure services" should be set to **on**.
- Click "Add client IP" (near the Save and Discard buttons).
- Finally set "Enforce SSL connection" to **disabled**.

Don't forget to click the save button!

### Create a Database

We have created a PostgreSQL Sever, and now we need to create a database for our application to connect to. If your client computer has psql installed, you can use a local instance of psql, or the Azure Cloud Console to connect to an Azure PostgreSQL server. Let's now use the psql command-line utility to connect to the Azure Database for PostgreSQL server.

1. Run the following psql command to connect to an Azure Database for PostgreSQL database:

    ```bash
    psql --host=<servername> --port=<port> --username=<username>@<servername> --dbname=<dbname>
    ```

    For example, the following command connects to the default database called postgres on your PostgreSQL server mydemoserver.postgres.database.azure.com using access credentials. Enter the `<server_admin_password>` you chose when prompted for password.

    ```bash
    psql --host=mydemoserver.postgres.database.azure.com --port=5432 --username=myadmin@mydemoserver --dbname=postgres
    ```

2. Once you are connected to the server, create a blank database at the prompt:

    ```sql
    CREATE DATABASE postgres_demo;
    ```

If you do **not** have psql installed locally, you can run the same commands from the Azure Cloudshell in the Portal.

## Configure the App

To get this application working with Azure PostgreSQL, we just need to configure Spring Data JPA via the `application.properties` file. This project already has a second Maven profile configured for use with Postgres. This allows us to build our app with a different configuration for when we deploy to App Service.

### Connection Strings

It is bad practice to put connection strings in source control, so we will put this information in environment variables and inject them into our application at build time.

1. First, create the following environment variables with the corresponding values.
    - `POSTGRES_URL`: The full URL of the PostgreSQL database: `<servername>.postgres.database.azure.com:5432/postgres_demo?sslmode=require`
    - `POSTGRES_USERNAME`: The username you used appended with `@<servername>`. For example: `username@mydemoserver`
    - `POSTGRES_PASSWORD`: The password you used when provisioning the PostgreSQL server

    > See these instructions to set environment variables on [Windows](https://www.techjunkie.com/environment-variables-windows-10/) and [Mac](http://osxdaily.com/2015/07/28/set-enviornment-variables-mac-os-x/).

1. Paste the following snippet into the `production` profile of the `pom.xml`. We will use this profile to build our application before deploying to App Service.

    ```xml
    <spring.datasource.url>${POSTGRES_URL}</spring.datasource.url>
    <spring.datasource.username>${POSTGRES_USERNAME}</spring.datasource.username>
    <spring.datasource.password>${POSTGRES_PASSWORD}</spring.datasource.password>
    ```

## Build and Deploy

We will now build our app and deploy it to App Service Linux.

1. Replace the resource group and application name in the App Service Maven plugin with your own values. (You can also change the location field to a region closer to you.)

    ```xml
    ...
    <resourceGroup>YOUR RESOURCE GROUP</resourceGroup>
    <appName>YOUR APP NAME</appName>
    ...
    ```

1. Build your app with the "production" profile and deploy it with `mvn clean package -Pproduction`. If you navigate to `target/classes/application.properties` you should see the Postgres username, password, and URL.

1. Finally, deploy the application with `mvn azure-webapp:deploy`. In the future, you can chain these commands by running, `mvn clean package -Pproduction azure-webapp:deploy`.

Browse to your application and you will see the same app now running on App Service using Azure PostgreSQL! You can confirm the application is using Postgres by querying the database with psql or pgadmin.

## Next steps

Follow [this guide](../key-vault) to learn how to store your connection strings in Key Vault, thus providing easy secret management and rotation!

## Helpful Links

- [Instructions for installing the Postgres certificate on your local machine](https://docs.microsoft.com/en-us/azure/postgresql/concepts-ssl-connection-security#applications-that-require-certificate-verification-for-ssl-connectivity)
- [Getting Started with Azure PostgreSQL](https://docs.microsoft.com/en-us/azure/postgresql/tutorial-design-database-using-azure-cli)
- [Full list of Spring Boot Starters](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-project/spring-boot-starters)
- [Relationship between Spring Data JPA, JPA, and Hibernate](https://thoughts-on-java.org/what-is-spring-data-jpa-and-why-should-you-use-it/)
