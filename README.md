`BUILDKITE_BADGE_PLACEHOLDER`

[![View in Campground](https://admin.wayfair.com/d/applife-system-health/v1/badges/health/fa)](https://admin.wayfair.com/d/application-health-campaign-management-interface/repository/name/spring-boot-test-r)

[![Grafana](docs/images/grafana.png) SDE Grafana k8s dashboard](https://devgrafana.csnzoo.com/d/ClRlCgKGk/kubernetes-deployment-status?orgId=2&var-cluster_name=gke-sdeprod-c3-dsm1&var-namespace=fa)

[![Grafana](docs/images/grafana.png) Production Grafana k8s dashboard](https://grafana.csnzoo.com/d/rfCUVOXik/kubernetes-deployment-status?orgId=2&var-cluster_name=gke-sdeprod-c3-dsm1&var-cluster_name=gke-prod-c2-iad1&var-namespace=fa)

# spring-boot-test-r

Thank you for using Mamba and choosing the Spring Boot application template to start your new project.

This application should have everything you need to start work straight away, and out of the box it can deploy to both SDE and production (you may need to prepare your application first).

This application comes ready with:

- REST service
- Metrics and logging integration using existing Wayfair infrastructure
- Our current best-practices for deploying to GCP using Kubernetes
- PostgreSQL server support via [JDBI 3](https://jdbi.org/)

### Important Note for new developers:

Before this project can communicate with the company's artifactory instance. Follow the instructions [here](https://docs.csnzoo.com/java/docs/artifactory/#maven) to set up a global proxy config.

#### Add profile to ~/.m2/settings.xml to allow snapshots

```xml
<profile>
    <id>allow-snapshots</id>
    <activation>
        <activeByDefault>false</activeByDefault>
    </activation>
    <repositories>
        <repository>
            <id>mvn-all</id>
            <url>https://artifactorybase.service.csnzoo.com/artifactory/mvn-all</url>
            <releases>
                <enabled>true</enabled>
            </releases>
            <snapshots>
                <enabled>true</enabled>
            </snapshots>
        </repository>
    </repositories>
</profile>
```

## Getting Started

You have a couple of options when running your application: either directly in IntelliJ or as a Docker container.

Docker is useful for setting up an environment easily (if necessary). It can provide dependency services like Redis or SQL server. It can also be slower to work with in some workflows.

IntelliJ can be convenient, a bit more streamlined and useful when debugging.

Once it’s running, test your application using cURL:
```shell
$ curl localhost:8080
Hello world!
```

### Running through Docker
To get started locally for development, first build the project and docker image `mvn clean package -DskipTests && docker-compose build service`. When the build is done, run `docker-compose up service` to run the service locally.

The local `target` directory is mounted on the service docker container, avoiding the need to rebuild your docker image regularly. When iterating on your application you can build locally using `mvn package -Dmaven.test.skip=true -P allow-snapshots` and then run via `docker-compose up service`.

You can force a rebuild of the docker images by doing this first because it's not automatically rebuilt when you change your code: `docker build service`.
Before you can run your applications, you'll need to perform migrations otherwise your database will be empty. See below for more details.

#### Debugging
See [this section of the Java Platform documentation](https://docs.csnzoo.com/java/docs/tutorials/running-an-application-inside-a-docker-container-with-intellij/).

If you are using a mac, pay attention to [step 0](https://docs.csnzoo.com/java/docs/tutorials/running-an-application-inside-a-docker-container-with-intellij/#0-pre-requirement-for-mac-users).

### Running through Intellij
In your IDE, you should also be able to run the SpringboottestrApplication directly because it has a `main()` method. **If IntelliJ cannot install dependencies correctly, ensure you have [configured Maven to pull from our Artifactory](https://docs.csnzoo.com/java/docs/artifactory/#maven)**.

As your application uses PostgreSQL, you'll need to start this by running `docker-compose up psql`.

#### Debugging
See [this section of the IntelliJ documentation](https://www.jetbrains.com/help/idea/debugging-code.html).

## Get Credentials for running locally

To use credentials for local development, please see our [guide here](https://docs.csnzoo.com/java/docs/tutorials/using-secrets/).


## Connecting a database client
By default, once your database is running you can connect a local database client by using the following settings:

- Username: `postgres`
- Password: `Password123!`
- Host: `localhost`
- Port: `5432`

If you don't have one already, some popular database clients for Postgres include:

- [DBeaver](https://dbeaver.io) (macOS, Linux and Windows; free)
- [IntelliJ](https://www.jetbrains.com/help/idea/database-tool-window.html) (macOS, Linux and Windows; Ultimate edition only)
- [TablePlus](https://tableplus.com) (macOS, Linux and Windows)
- [Postico](https://eggerapps.at/postico2/) (macOS)

### Connection timeouts
#### HikariCP
The following property has been added to the application.yaml file to replace the default timeout of 30s for how long this
app will wait trying to request connections from the pool.

```yaml
spring:
  datasource:
    hikari:
      connection-timeout: 30000 # value in ms
```

#### JDBI
To give JDBI a timeout when establishing a connection to the JDBC source after the driver has been identified:

```java
    DriverManager.setLoginTimeout(QUERY_TIMEOUT_SECONDS);
```


#### Query Timeout
Query timeouts can be established a couple different ways.

Setting a timeout in seconds on one query by using the annotation `@QueryTimeOut`
```java
    @SqlQuery("select * from sku")
    @RegisterRowMapper(SkuMapper.class)
    @QueryTimeOut(QUERY_TIMEOUT_SECONDS)
    List<Sku> findAll();
```

Setting a timeout in seconds on one query by using `setQueryTimeout()` on a `SqlStatement`

```java
    jdbi.useHandle(handle -> {
      handle.createUpdate("INSERT INTO sku (id, name) VALUES (:id, :name)")
          .setQueryTimeout(QUERY_TIMEOUT_SECONDS)
          // ...
          .execute();
    });
```

Setting a timeout in seconds globally across all queries by using `setQueryTimeout()` on `SqlStatements`

```java
    jdbi.configure(SqlStatements.class, stmt -> stmt.setQueryTimeout(QUERY_TIMEOUT_SECONDS));
```

### Database migrations
#### Running migrations locally
Before you can execute any queries in your application, your schema needs to be created. 

If your DB server isn't running in Docker, then start it first:

    docker-compose up -d psql

Then, run the `migrate` container to begin performing migrations and allow it to finish:

    docker-compose run migrate apply
### Liquibase

Some features may require you to modify the database schema. This will be done via Liquibase.

Liquibase are used to sequentially apply schema changes to our database. They are SQL scripts that live in migrations/.
They are version controlled alongside the rest of our application, and are applied when our buildkite pipeline runs.
The liquibase init container must be used to create these migration files.

[Liquibase general docs](https://docs.liquibase.com/home.html)
[Liquibase at WF docs](https://docs.csnzoo.com/shared/liquibase-docker/)

### ChangeLog and Making Changes

We are using an XML format for the changelog file, to which you can add change sets directly or import
changelog files.
Changes can be run through sql or xml files, but Datatech recommends using an XML format for the changelog files.

### Using XML Files

Add the XML file containing desired changes to the changelogs directory inside migrations changelogs.
The main changelog file is already set up to apply changes using any new updates in the folder.
Database changes are recorded into the changelog sequentially as change sets. Liquibase uses the changelog to execute
changes to the database.

Standard XML ChangeLog:

```bash
<?xml version="1.1" encoding="UTF-8" standalone="no"?>
<databaseChangeLog xmlns="http://www.liquibase.org/xml/ns/dbchangelog"
                   xmlns:ext="http://www.liquibase.org/xml/ns/dbchangelog-ext"
                   xmlns:pro="http://www.liquibase.org/xml/ns/pro"
                   xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                   xsi:schemaLocation="http://www.liquibase.org/xml/ns/dbchangelog-ext
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-ext.xsd
                   http://www.liquibase.org/xml/ns/pro
                   http://www.liquibase.org/xml/ns/pro/liquibase-pro-4.6.xsd
                   http://www.liquibase.org/xml/ns/dbchangelog
                   http://www.liquibase.org/xml/ns/dbchangelog/dbchangelog-4.6.xsd">
    <includeAll path="migrations/" relativeToChangelogFile="true" />
</databaseChangeLog>
```

### Using sql Files
SQL files need to be added in migrations/ folder and need to have the following formatting lines at the top of the file.
Please make sure to also include the files in the Main Changelog directory.

```bash
--liquibase formatted sql
--changeset <username>:<changeset id>
```

### Changeset ID
Every Change Set must include an author and id. The format our team uses is the Datatech
recommended username + PH ticket number + changeset (basically for every ticket just start counting at 1).

```bash 
e.g- <changeSet author="aa714q" id="SEOSO-1111-1">
```

Changes should always roll forward to ensure consistancy across database instances. Liquibase does offer flexibility
for changing changelogs for troubleshooting bad database states.


### Applying Change Sets
The liquibase update command executes all changes not run on the database instance and records if they have been
executed in DATABASECHANGELOG.

To apply changes locally, run:

```bash
docker-compose run liquibase update --log-level INFO
```

Use DBeaver or psql to connect with the database:

Host: localhost
Port: 5432
Database: Database name
Username: app
Password: SA password

Deploy changes will be handled by the liquibase command in the K8S.yaml. The line is command: *liquibase-command-update

Note: Make sure to distribute your database password vault key in the application k8s namespace.

### Cloud SQL Auth Proxy¶

All connections to your database should be through the Cloud SQL Authentication Proxy.

See [Logging In](https://docs.csnzoo.com/shared/gcp-docs/decoupled-services-catalog/cloudsql-postgresql/logging-in/)
and
[Connect to Cloud SQL for PostgreSQL using the Cloud SQL Auth proxy]
(https://cloud.google.com/sql/docs/postgres/connect-instance-auth-proxy) for details.


## Deploying your app
This app is Continuous deployment enabled. What that means is that you have the capability of deploying directly to the dev kubernetes cluster on any push for any branch.

**To do this your commit message must contain one of two things: a `:rocket:` emoji or the word `DEPLOY`.**

**For production pushes and push to main will deploy to production automatically without further configuration.**

There are several options to control your BuildKite pipeline detailed below.

**_Options_**
* SKIP_LINT
  * will skip your helm linting
* SKIP_TEST
  * will skip your tests
* SKIP_IMAGE
  * this will skip creation of your container
  * **WARNING** if you skip this and have a deploy step it will break your pipeline
* SKIP_DOCS
  * this will skip the creation of your mkdocs
* `:rocket:` or `DEPLOY`
  * this will deploy your pr to the dev branch

### Backstage
[Backstage](https://infohub.corp.wayfair.com/display/ENG/Backstage) is a “single pane of glass” for managing infrastructure at Wayfair. The intent is to provide service owners with simple self-service tools for managing the lifecycle of their service.

### Mamba Hub
[Mamba Hub](https://backstage.service.csnzoo.com/mamba-hub) will show you all the services generated using Mamba for your team, including this one.

The _Kubernetes_ section of the Mamba Hub has a wealth of information about the health of your pods.

### Preparing your application
To perform your first deploy, click the "Prepare" button in Mamba Hub. The effects this will have are displayed on that page.

**Only do so when you are happy that your project has been generated properly, with the correct configuration values**, as changing them later involves more toil than doing so before preparing your application.

Once the preparation has occured, there should be a ["BuildKite"](https://buildkite.com/docs) button on the page, which takes you to the first CI build of your application. The final step of this is that your application will be deployed to both [SDE](https://docs.csnzoo.com/shared/sde-docs/) and production.

After the deploy step has occured, you can access your application in SDE:

```shell
$ curl kube-fa.service.intradsm1.sdeconsul.csnzoo.com
Hello world!
```

And also production:

```shell
$ curl kube-fa.service.intraiad1.consul.csnzoo.com
Hello world!
```

These hostnames are defined in [`k8s.yaml`](k8s.yaml).

### Kubernetes resource quota
Your service includes a default [`k8s.yaml`](k8s.yaml) configuration that uses [resource automation](https://docs.csnzoo.com/docker/kubernetes/cluster-reference/resource-automation/) for your application. In short, this means that you don't need to specify CPU and memory requests or limits - instead, the cluster will determine appropriate values for you. For more information, check the documentation.

## wf-config files

Wayfair applications typically make use of Key-Value information found in `wfconfig` files. These files include various infrastructure information (e.g. data center locations). Refer to the docs [here](https://docs.csnzoo.com/shared/k8s-docs/kubernetes/cluster-reference/secrets-and-config-files-management/index.html#config-files-management) and [here](https://docs.csnzoo.com/shared/k8s-docs/kubernetes/development-resources/config-files/index.html) to learn more about these config files and how they are generated.

These files are injected into application pods under `/wayfair/etc` through [annotations](https://github.csnzoo.com/shared/k8s-helm/blob/4c96177bcda6c686993b0d94a9b8a611f6dad702/charts/base-java/templates/http_deployment.yaml#L34). By default, only `wf-config.yaml` is [made available](https://github.csnzoo.com/shared/java-project-templates/blob/a48a0767414cbc698b31ca10abd763ba3f86dfac/templates/general-purpose/%7B%7Bcookiecutter.project_name_canonical%7D%7D/k8s.yaml#L53) for Java applications.

The Key-Value pairs are accessible [as environment variables](https://github.csnzoo.com/shared/java-project-templates/blob/a48a0767414cbc698b31ca10abd763ba3f86dfac/templates/general-purpose/%7B%7Bcookiecutter.project_name_canonical%7D%7D/docker/files/entrypoint.sh#L5). In addition, the config is [optionally imported into Spring Boot](https://github.csnzoo.com/shared/java-project-templates/blob/a48a0767414cbc698b31ca10abd763ba3f86dfac/templates/general-purpose/%7B%7Bcookiecutter.project_name_canonical%7D%7D/src/main/resources/application.yaml#L7).

The local secret distributor can also [pull wf-config files](https://github.csnzoo.com/shared/docker-localdev-secret-distributor#environment-variables). You just have to make sure to adapt the volume mount paths for your service.

## Included Spring Libraries/Features
### OpenAPI (formerly known as Swagger)
A good microservice has a well documented API and that's where OpenAPI/Swagger is useful. If you change your classes used in API calls or responses, they'll automatically be documented by OpenAPI; the different response codes that can be returned by an endpoint can also be documented. You can try it by testing your service locally at the [OpenAPI endpoint](http://localhost:8080/swagger-ui.html). And for further information, there's a [great getting started article on Baeldung](https://www.baeldung.com/spring-rest-openapi-documentation).

## IntelliJ plugins
This project includes recommended IntelliJ plugins in the `.idea/externalDependencies.xml` file that will make your development easier. IntelliJ will prompt you to install these whenever the project is loaded so feel free to add/remove as needed.

### CheckStyle
[Checkstyle](https://docs.csnzoo.com/java/docs/code-checkstyles/) makes your Java code conform to Wayfair's Java code styling. The plugin lets IntelliJ show you issues before you put your code in a PR.

### SonarLint
[SonarLint](https://docs.csnzoo.com/java/docs/code-sonarlint/) shows in-editor warnings about[Sonarqube](https://sonarqube-enterprise.service.csnzoo.com/) code quality issues in your code. This does require some [IntelliJ setup](https://docs.csnzoo.com/java/docs/code-sonarlint/).

### Docker
The [Docker](https://www.jetbrains.com/help/idea/docker.html) plugin lets you manage Docker containers from within IntelliJ.

## DataDog
Performance monitoring is set up using [Datadog APM Tracing](https://app.datadoghq.com/apm/traces) when deployed to production. All your endpoints and the Java runtime are automatically monitored.

[Go to Service Traces](https://app.datadoghq.com/apm/traces?query=service%3A$spring-boot-test-r)

## Renovate & Dependency Management
A very basic Renovate configuration is included with your project. It is highly recommended that you create a
[team-preset](https://github.csnzoo.com/shared/renovate-config/tree/main/team-presets) that minimizes toil experienced as a result of dependency maintenance.

In the event that your project has robust pipelines and testing, your team may be able to utilize
[Renovate's automerging functionality](https://docs.csnzoo.com/shared/dev-accel-docs/dependencies/renovate/automerging/)
to automate away most "work" associated with dependency updates, intervening only when a new version introduces a breaking change.

## API Automation Test
### Karate Test
[Karate](https://github.csnzoo.com/shared/karate/wiki) is an open-source API test automation framework with the following key features.
* The Gherkin BDD (Behaviour Driven Development) syntax is straightforward.
* API calls to both of REST and GraphQL HTTP endpoints are supported.
* Can be integrated with CI/CD tools such as BuildKite.
* HTML test report is built-in.
* [And more](https://github.com/karatelabs/karate#features)

On the local machine, the Karate test is executed by a docker image "wayfair/karate-base" with the command `docker-compose up karate`. In the BuildKite pipeline, the test is executed by [karate-buildkite-plugin](https://github.csnzoo.com/shared/karate-buildkite-plugin).

The [karate-config.js](https://github.com/karatelabs/karate#karate-configjs) which includes the global variables is under the directory "src/test/java/karate/resources". The test scripts (*.feature files) are under the directory "src/test/java/karate/features" and written in the Gherkin BDD [syntax](https://github.com/karatelabs/karate#syntax-guide). Here are some [best practices](https://github.csnzoo.com/shared/karate/wiki/Best-Practices) recommended to follow when writing test scripts.

## Helpful Links
* [Developing in a container](https://docs.csnzoo.com/docker/containers/container-development/)
* [Continuous Deployment](https://docs.csnzoo.com/docker/kubernetes/app-debug/)
* [Trouble Shooting your App](https://docs.csnzoo.com/docker/kubernetes/app-debug/)
* [Istio @ Wayfair](https://docs.csnzoo.com/docker/istio/istio/)
* [Istio Rate Limiting](https://docs.csnzoo.com/docker/istio/getting-started/service-mesh-features/#rate-limiting)

## Final Notes
This is your code and your repo, you should change this read me to allow for easier onboarding of people who are making changes to your app.

Thanks for using this template! Please provide feedback to the Java Platform Team. `@javaplats` in
`#java-forum`
