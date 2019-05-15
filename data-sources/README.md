# Configuring Data Sources

These instructions will walk through the process of configuring a PostgreSQL database for a Spring Boot app on App Service Linux. Our dependencies are managed using Spring Starters, which abstract the necessary dependencies for common development scenarios (such as cache, data access, or messaging). 

## Prerequisites

You will need the following tools installed locally.

- Azure CLI
- Maven 3
- psql command line tool (optional)

## Provision the Server

You can provision a Postgres database through the Portal by going to Create a Resource and searching for "Azure Database for PostgreSQL". In the following screen enter your desired server name, admin username, and password.

Alternatively, you can use the following Azure CLI command to create the database by replacing the placeholder values. To get a full list of all the Azure locations (aka "regions"), run `az account list-locations`. Don't forget your username and password!

```bash
az postgres server create -n <desired-name> -g <resource-group> --sku-name B_Gen5_1 -u <username> -p <password> -l centralus
```

## Create a Database

We have created a PostgreSQL Sever, and now we need to create a database for our application to connect to. If your client computer has PostgreSQL installed, you can use a local instance of psql, or the Azure Cloud Console to connect to an Azure PostgreSQL server. Let's now use the psql command-line utility to connect to the Azure Database for PostgreSQL server.

1. Run the following psql command to connect to an Azure Database for PostgreSQL database:

    ```bash
    psql --host=<servername> --port=<port> --username=<user@servername> --dbname=<dbname>
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

Our Spring Boot 

## Helpful Links

- [Getting Started with Azure PostgreSQL](https://docs.microsoft.com/en-us/azure/postgresql/tutorial-design-database-using-azure-cli)
- [Full list of Spring Boot Starters](https://github.com/spring-projects/spring-boot/tree/master/spring-boot-project/spring-boot-starters)