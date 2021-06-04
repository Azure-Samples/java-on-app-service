# Deploying with Maven

These instructions will walk through deploying a Spring Boot JAR application to App Service. We will start at the `initial/` directory. If you run into problems, see the `complete/` directory. There are supplemental instructions to convert the application to a WAR and deploy that onto App Service.

## Run Locally

First, test that the application builds and runs successfully.

1. Build the application with Maven. This project does not have any test classes.

    ```shell
    mvn clean package
    ```

1. Run the application with the following command:

    ```shell
    java -jar ./target/app.jar
    ```

1. Open your browser and navigate to `http://localhost/`. (Try `http://127.0.0.1/` if the first link does not work.) You should see a simple web page with green text displaying, "Hello App Service!"

## Deploying to App Service with Maven

Now that we have confirmed the JAR runs locally, we will deploy and run this app on App Service Linux.

1. First, insert the [App Service Maven plugin](https://docs.microsoft.com/en-us/java/api/overview/azure/maven/azure-webapp-maven-plugin/readme?view=azure-java-stable) into the `<plugins>` sections of the `pom.xml`.

    ```xml
    <plugin>
        <groupId>com.microsoft.azure</groupId>
        <artifactId>azure-webapp-maven-plugin</artifactId>
        <!-- Check Maven Central (https://search.maven.org/) for the latest version -->
        <version>1.15.0</version>
    </plugin>
    ```

1. Next, we will use the plugin to generate the configuration for our webapp. Run the `mvn azure-webapp:config` command and select the default options in the following prompts. The command will generate a configuration similar to the one below. Feel free to change the resource group and app name to something more memorable.

    ```xml
    <configuration>
        <schemaVersion>V2</schemaVersion>
        <resourceGroup>maven-deployment-1555354589298-rg</resourceGroup>
        <appName>maven-deployment-1555354589298</appName>
        <region>westeurope</region>
        <pricingTier>P1V2</pricingTier>
        <runtime>
        <os>linux</os>
        <javaVersion>jre8</javaVersion>
        <webContainer>jre8</webContainer>
        </runtime>
        <deployment>
        <resources>
            <resource>
            <directory>${project.basedir}/target</directory>
            <includes>
                <include>*.jar</include>
            </includes>
            </resource>
        </resources>
        </deployment>
    </configuration>
    ```

1. Now we will deploy to App Service! In the terminal, run `mvn clean package azure-webapp:deploy`. We are cleaning the target directory and repackaging in case there were any changes. You can also run `mvn azure-webapp:deploy` if you already packaged the webapp.

    > Note that the Maven plugin uses the [Azure CLI](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli?view=azure-cli-latest), so you must be authenticated in the CLI for the deployment to succeed.

    When the command finishes, there will be a URL to your newly created web app. Copy/paste this URL into your browser and you will see the same simple web page and green text that we saw locally.

## Deploying a WAR application

In its current state, the Spring Boot application uses an embedded Tomcat servlet and JSP compiler. To run this application on App Service's Tomcat 8.5 or 9.0, we will remove these dependencies from the project and package the application as a WAR.

1. In the `pom.xml`, change the packaging type to a JAR by replacing "war" with "jar" in the `<packaging>` tags.

1. Next, we will mark the spring-boot-starter-tomcat and tomcat-embed-jasper dependencies as "provided". Adding these xml snippets will instruct the compiler that these dependencies will be provided at runtime, but we will not package them into our WAR.

    ```xml
    <dependency>
      <groupId>org.springframework.boot</groupId>  
      <artifactId>spring-boot-starter-tomcat</artifactId>
      <scope>provided</scope>
    </dependency>  
    <dependency>
      <groupId>org.apache.tomcat.embed</groupId>  
      <artifactId>tomcat-embed-jasper</artifactId>
      <scope>provided</scope>  
    </dependency>
    ```

1. Lastly, update the App Service plugin configuration to deploy the WAR onto Tomcat. Change the webContainer from "jre-8" to "tomcat 8.5" (or "tomcat 9.0") and the include tag's content from ".jar" to ".war".

    ```xml
    <webContainer>tomcat 8.5</webContainer>
    ...
    <include>*.war</include>
    ```

1. Now we can deploy to App Service by running `mvn clean package azure-webapp:deploy`.

**If you ran into issues in this tutorial, please open an issue on the [GitHub repository](https://github.com/Azure-Samples/java-on-app-service).**
