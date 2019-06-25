## Lab3 - Breaking the monolith apart - I

In the previous labs you learned how to take an existing monolithic Java EE application to the cloud
with JBoss EAP and OpenShift, and you got a glimpse into the power of OpenShift for existing applications.

You will now begin the process of modernizing the application by breaking the application into multiple
microservices using different technologies, with the eventual goal of re-architecting the entire application as a set of
distributed microservices. Later on we'll explore how you can better manage and monitor the application after
it is re-architected.

In this lab you will learn more about **Supersonic Subatomic Java** [Qurakus](https://Quarkus.io/), has been designed with `Container first & Cloud Native World` 
in mind, and provides first-class support for these different paradigms. Quarkus development model morphs to adapt itself to the type of application you are developing.

Quarkus is a `Kubernetes Native Java` stack tailored for `GraalVM & OpenJDK HotSpot`, crafted from the best of breed Java libraries and standards. 
Amazingly fast boot time, incredibly low RSS memory (not just heap size!) offering near instant scale up and high density memory utilization 
in container orchestration platforms like Kubernetes. Quarkus uses a technique called compile time boot. [Learn more](https://quarkus.io/vision/container-first).

This will be one of the runtimes included in [Red Hat OpenShift Application Runtimes](https://developers.redhat.com/products/rhoar) in 2020. 

You will implement one component of the monolith as a Quarkus microservice and modify it to address
microservice concerns, understand its structure, deploy it to OpenShift and exercise the interfaces between
Quarkus apps, microservices, and OpenShift/Kubernetes.

#### Goals of this lab

---

The goal is to deploy this new microservice alongside the existing monolith, and then later on we'll tie them together.
But after this lab, you should end up with something like:

![lab3_goal]({% image_path goal.png %}){:width="700px"}

#### What is Quarkus? 

---

![quarkus-logo]({% image_path quarkus-logo.png %})

For years, the client-server architecture has been the de-facto standard to build applications. 
But a major shift happened. The one model rules them all age is over. A new range of applications 
and architecture styles has emerged and impacts how code is written and how applications are deployed and executed. 
HTTP microservices, reactive applications, message-driven microservices and serverless are now central players in modern systems.

[Qurakus](https://Quarkus.io/) offers 4 major benefits to build cloud-native, microservices, and serverless Java applicaitons:

* `Developer Joy` - Cohesive platform for optimized developer joy through unified configuration, Zero config with live reload in the blink of an eye,
   streamlined code for the 80% common usages with flexible for the 20%, and no hassle native executable generation.
* `Unifies Imperative and Reactive` - Inject the EventBus or the Vertx context for both Reactive and imperative development in the same application.
* `Functions as a Service and Serverless` - Superfast startup and low memory utilization. With Quarkus, you can embrace this new world without having 
  to change your programming language.
* `Best of Breed Frameworks & Standards` - CodeReady Workspaces Vert.x, Hibernate, RESTEasy, Apache Camel, CodeReady Workspaces MicroProfile, Netty, Kubernetes, OpenShift,
  Jaeger, Prometheus, Apacke Kafka, Infinispan, and more.


####1. Setup an Inventory proejct

---

Run the following commands to set up your environment for this lab and start in the right directory:

In the project explorer, right-click on **inventory** and then change a directory to inventory path on **Terminal**.

![inventory_setup]({% image_path codeready-workspace-inventory-project.png %}){:width="500px"}

####2. Examine the Maven project structure

---

The sample Quarkus project shows a minimal CRUD service exposing a couple of endpoints over REST, 
with a front-end based on Angular so you can play with it from your browser.

> Click on the `inventory` folder in the project explorer and navigate below folders and files.

While the code is surprisingly simple, under the hood this is using:

 * RESTEasy to expose the REST endpoints
 * Hibernate ORM with Panache to perform the CRUD operations on the database
 * A PostgreSQL database; see below to run one via Linux Container

`Hibernate ORM` is the de facto JPA implementation and offers you the full breadth of an Object Relational Mapper. 
It makes complex mappings possible, but it does not make simple and common mappings trivial. Hibernate ORM with 
Panache focuses on making your entities trivial and fun to write in Quarkus.

This project currently contains no code other than web resources in `src/main/resources` and Dockerfile to build a container that 
runs the Quarkus application in JVM mode as well as native (no JVM) mode in `src/main/docker`:

* Dockerfile in JVM mode

~~~java
####
# This Dockerfile is used in order to build a container that runs the Quarkus application in JVM mode
#
# Before building the docker image run:
#
# mvn package
#
# Then, build the image with:
#
# docker build -f src/main/docker/Dockerfile.jvm -t quarkus/hibernate-orm-panache-resteasy-jvm .
#
# Then run the container using:
#
# docker run -i --rm -p 8080:8080 quarkus/hibernate-orm-panache-resteasy-jvm
#
###
> FROM fabric8/java-alpine-openjdk8-jre
> ENV JAVA_OPTIONS="-Dquarkus.http.host=0.0.0.0 -Djava.util.logging.manager=org.jboss.logmanager.LogManager"
> ENV AB_ENABLED=jmx_exporter
> COPY target/lib/* /deployments/lib/
> COPY target/*-runner.jar /deployments/app.jar
> ENTRYPOINT [ "/deployments/run-java.sh" ]
~~~

* Dockerfile in Native mode

~~~java
####
# This Dockerfile is used in order to build a container that runs the Quarkus application in native (no JVM) mode
#
# Before building the docker image run:
#
# mvn package -Pnative -Dnative-image.docker-build=true
#
# Then, build the image with:
#
# docker build -f src/main/docker/Dockerfile.native -t quarkus/hibernate-orm-panache-resteasy .
#
# Then run the container using:
#
# docker run -i --rm -p 8080:8080 quarkus/hibernate-orm-panache-resteasy
#
###
> FROM registry.fedoraproject.org/fedora-minimal
> WORKDIR /work/
> COPY target/*-runner /work/application
> RUN chmod 775 /work
> EXPOSE 8080
> CMD ["./application", "-Dquarkus.http.host=0.0.0.0"]
~~~

Now let's write some code and create a domain model, service interface and a RESTful endpoint to access inventory:

![Inventory RESTful Service]({% image_path inventory-arch.png %}){:width="700px"}

####3. Add Qurkus Extensions

---

We will add Qurakus extensions to the Inventory application for using `Panache`, `Postgres` and we'll use the Quarkus Maven Plugin.
Copy the following commands to add the Hibernate ORM with Panache extension via CodeReady Workspaces **Terminal**:

Go to `inventory' directory:

`cd cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-hibernate-orm-panache"`

And then for local H2 database:

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-jdbc-h2"`

>**NOTE:** There are many [more extensions](https://quarkus.io/extensions/) for Quarkus for popular frameworks 
like [CodeReady Workspaces Vert.x](https://vertx.io/), [Apache Camel](http://camel.apache.org/), [Infinispan](http://infinispan.org/), 
Spring DI compatibility (e.g. @Autowired), and more.

####4. Create Inventory Entity

---

With our skeleton project in place, let's get to work defining the business logic.

The first step is to define the model (entity) of an Inventory object. Since Quarkus uses Hibernate ORM Panache,
we can re-use the same model definition from our monolithic application - no need to re-write or re-architect!

Create a new Java class named `Inventory.java` in
`com.redhat.coolstore` package with the following code, identical logics to the monolith code:

~~~java
package com.redhat.coolstore;

import javax.persistence.Cacheable;
import javax.persistence.Column;
import javax.persistence.Entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;

@Entity
@Cacheable
public class Inventory extends PanacheEntity {

    public String itemId;
    public String location;
    public int quantity;
    public String link;

    public Inventory() {

    }

}
~~~

By extending **PanacheEntity** in your entities, you will get an ID field that is auto-generated. If you require a custom ID strategy, 
you can extend **PanacheEntityBase** instead and handle the ID yourself.

By using Use public fields, there is no need for functionless getters and setters (those that simply get or set the field). You simply refer to fields like Inventory.location without the need to write a Inventory.geLocation() implementation. Panache will auto-generate any getters and setters you do not write, or you can develop your own getters/setters that do more than get/set, which will be called when the field is accessed directly.

The `PanacheEntity` superclass comes with lots of super useful static methods and you can add your own in your derived entity class, and much like traditional object-oriented programming it's natural and recommended to place custom queries as close to the entity as possible, ideally within the entity definition itself. 
Users can just start using your entity Inventory by typing Inventory, and getting completion for all the operations in a single place.

When an entity is annotated with `@Cacheable`, all its field values are cached except for collections and relations to other entities.
This means the entity can be loaded without querying the database, but be careful as it implies the loaded entity might not reflect recent changes in the database.

####5. Define the RESTful endpoint of Inventory

---

In this step we will mirror the abstraction of a _service_ so that we can inject the Inventory _service_ into
various places (like a RESTful resource endpoint) in the future. This is the same approach that our monolith
uses, so we can re-use this idea again. Create an **InventoryResource** class in the `com.redhat.coolstore` package:

~~~java
package com.redhat.coolstore;

import java.util.List;
import java.util.stream.Collectors;

import javax.enterprise.context.ApplicationScoped;
import javax.json.Json;
import javax.ws.rs.Consumes;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.Produces;
import javax.ws.rs.WebApplicationException;
import javax.ws.rs.core.Response;
import javax.ws.rs.ext.ExceptionMapper;
import javax.ws.rs.ext.Provider;

import org.jboss.resteasy.annotations.jaxrs.PathParam;

@Path("/services/inventory")
@ApplicationScoped
@Produces("application/json")
@Consumes("application/json")
public class InventoryResource {

    @GET
    public List<Inventory> getAll() {
        return Inventory.listAll();
    }

    @GET
    @Path("{itemId}")
    public List<Inventory> getAvailability(@PathParam String itemId) {
        return Inventory.<Inventory>streamAll()
        .filter(p -> p.itemId.equals(itemId))
        .collect(Collectors.toList());
    }

    @Provider
    public static class ErrorMapper implements ExceptionMapper<Exception> {

        @Override
        public Response toResponse(Exception exception) {
            int code = 500;
            if (exception instanceof WebApplicationException) {
                code = ((WebApplicationException) exception).getResponse().getStatus();
            }
            return Response.status(code)
                    .entity(Json.createObjectBuilder().add("error", exception.getMessage()).add("code", code).build())
                    .build();
        }

    }
}
~~~

The above REST services defines two endpoints:

* `/inventory` that is accessible via **HTTP GET** which will return all known product Inventory entities as JSON
* `/inventory/<location>` that is accessible via **HTTP GET** at
for example **/inventory/Boston** with
the last path parameter being the location which we want to check its inventory status.

####6. Add inventory data

---

Let's add inventory data to the database so we can test things out. Open up the `src/main/resources/import.sql` file and 
copy the following SQL statements to `import.sql`:

~~~java
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '329299', 'http://maps.google.com/?q=Raleigh', 'Raleigh', 736);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '329199', 'http://maps.google.com/?q=Boston', 'Boston', 512);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165613', 'http://maps.google.com/?q=Seoul', 'Seoul', 256);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165614', 'http://maps.google.com/?q=Singapore', 'Singapore', 54);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165954', 'http://maps.google.com/?q=London', 'London', 87);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444434', 'http://maps.google.com/?q=NewYork', 'NewYork', 443);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444435', 'http://maps.google.com/?q=Paris', 'Paris', 600);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444437', 'http://maps.google.com/?q=Tokyo', 'Tokyo', 230);
~~~

In Development, we will configure to use local in-memory H2 database for local testing, as defined in `src/main/resources/application.properties`:

~~~java
quarkus.datasource.url=jdbc:h2:file://projects/database.db
quarkus.datasource.driver=org.h2.Driver
quarkus.datasource.username=inventory
quarkus.datasource.password=mysecretpassword
quarkus.datasource.max-size=8
quarkus.datasource.min-size=2
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.sql=false
~~~

####7. Run Quarkus Inventory application

---

Now we are ready to run the inventory application. Click on **Commands Palette** then select `Build and Run Locally` in Run menu:

![codeready-workspace-maven]({% image_path quarkus-dev-run-paletter.png %})

You can also use a `maven plugin command` to run the Quarkus application locally via CodeReady Workspaces **Terminal**:

`mvn compile quarkus:dev`

You should see a bunch of log output that ends with:

~~~java
12:56:43,106 INFO  [io.quarkus] Quarkus 0.15.0 started in 9.429s. Listening on: http://[::]:8080
12:56:43,106 INFO  [io.quarkus] Installed features: [agroal, cdi, hibernate-orm, jdbc-h2, narayana-jta, resteasy, resteasy-jsonb]
~~~

Open a new CodeReady Workspaces **Terminal** and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`

~~~java
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com
/?q=Boston","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"item
Id":"165614","link":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London"
,"location":"London","quantity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444
435","link":"http://maps.google.com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tok
yo","quantity":230}]
~~~


`curl http://localhost:8080/services/inventory/329199 ; echo`

~~~java
[{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Boston","location":"Boston","quantity":512}]
~~~~

> **NOTE**: Make sure to stop Quarkus development mode via `Close` the `Build and Run Locally` terminal.

####8. Add Test Codes and Make a package

---

In this step, we will add Quarkus test codes so that we can inject Unit test during `mvn package`. 
Open up the `src/test/java/com/redhat/coolstore/InventoryEndpointTest.java` file and copy the following codes:

~~~java
package com.redhat.coolstore;

import static io.restassured.RestAssured.given;
import static org.hamcrest.CoreMatchers.containsString;

import io.quarkus.test.junit.QuarkusTest;
import org.junit.jupiter.api.Test;

@QuarkusTest
public class InventoryEndpointTest {

    @Test
    public void testListAllInventory() {
        //List all, should have all 8 cities inventory the database has initially:
        given()
              .when().get("/services/inventory")
              .then()
              .statusCode(200)
              .body(
                    containsString("Raleigh"),
                    containsString("Boston"),
                    containsString("Seoul"),
                    containsString("Singapore"),
                    containsString("London"),
                    containsString("NewYork"),
                    containsString("Paris"),
                    containsString("Tokyo")
                    );
     
        //List a certain item(ID:329299), Raleigh should be returned:
        given()
        .when().get("/services/inventory/329299")
        .then()
        .statusCode(200)
        .body(                   
              containsString("Raleigh")
        );
    }

}
~~~

Make a package to create a uber.jar then we will deploy it to `OpenShift cluster` soon. Use the following mav plugin command via CodeReady Workspaces **Terminal**:

`mvn clean package`

> **NOTE**: Make sure to build this mvn command at working directory(i.e /projects/cloud-native-workshop-v2m1-labs/inventory/).

If builds successfully (you will see `BUILD SUCCESS`), continue to the next step to deploy the application to OpenShift.

You can also run the Uber.jar to make sure if the inventory works. Use the following `Java command` then you see a similar output:

`java -jar target/inventory-1.0-SNAPSHOT-runner.jar`

~~~java
2019-05-29 06:54:16,123 INFO  [io.quarkus] (main) Quarkus 0.15.0 started in 7.826s. Listening on: http://[::]:8080
2019-05-29 06:54:16,201 INFO  [io.quarkus] (main) Installed features: [agroal, cdi, hibernate-orm, jdbc-h2, narayana-jta, resteasy, resteasy-jsonb]
~~~

Open a new CodeReady Workspaces **Terminal** and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`

~~~java
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com
/?q=Boston","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"item
Id":"165614","link":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London"
,"location":"London","quantity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444
435","link":"http://maps.google.com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tok
yo","quantity":230}]
~~~


`curl http://localhost:8080/services/inventory/329199 ; echo`

~~~java
[{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Boston","location":"Boston","quantity":512}]
~~~~

> **NOTE**: Make sure to stop Quarkus runtimes via `Close` the terminal.

You have now successfully created your first microservice using `Quarkus` and implemented a basic RESTful
API on top of the Inventory database. Most of the code look simpler than the monolith, demonstrating how
easy it is to migrate existing monolithic Java EE applications to microservices using `Quarkus`.

In next steps of this lab we will deploy our application to OpenShift Container Platform and then start
adding additional features to take care of various aspects of cloud native microservice development.

####9. Create OpenShift Project

---

We have already deployed our coolstore monolith to OpenShift, but now we are working on re-architecting it to be
microservices-based.

In this step, we will deploy our new Inventory microservice for our CoolStore application,
so create a separate project to house it and keep it separate from our monolith and our other microservices we will
create later on.

Before going to OpenShift console, we will repackage the Quarkus application for adding a PostgreSQL extension 
because Inventory service will connect to PostgeSQL database in production on OpenShift cluster.

Add a `quarkus-jdbc-postgresql` extendsion via CodeReady Workspaces **Terminal**:

`cd cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-jdbc-postgresql"`

Comment the **quarkus.datasource.url, quarkus.datasource.drive** configuration and add the following variables in `src/main/resources/application.properties`:

~~~shell
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
~~~

Repackage the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})


Or you can run a maven plugin command directly in **Terminal**:

`mvn clean package -DskipTests`

> **NOTE**: You should **SKIP** the Unit test because you don't have PostgreSQL database in local environment.

Create a new project for the _inventory_ service:

Click **Create Project**, fill in the fields, and click **Create**:

* Name: `userXX-inventory`
* Display Name: `USERXX CoolStore Inventory Microservice Application`
* Description: _leave this field empty_

![create_dialog]({% image_path create_inventory_dialog.png %}){:width="500"}

Click on the name of the newly-created project:

![create_new]({% image_path create_new_inventory.png %}){:width="500"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

####10. Deploy to OpenShift

---

Let's deploy our new inventory microservice to OpenShift!

Our production inventory microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing the following commands via CodeReady Workspaces **Terminal**:

`oc project userXX-inventory`

~~~shell
oc new-app -e POSTGRESQL_USER=inventory \
  -e POSTGRESQL_PASSWORD=mysecretpassword \
  -e POSTGRESQL_DATABASE=inventory openshift/postgresql:latest \
  --name=inventory-database
~~~

> **NOTE:** If you change the username and password you also need to update `src/main/resources/application.properties` which contains
the credentials used when deploying to OpenShift.

This will deploy the database to our new project. 

![inventory_db_deployments]({% image_path inventory-database-deployment.png %})

####11. Build and Deploy

---

Red Hat OpenShift Application Runtimes includes a powerful maven plugin that can take an
existing Quarkus application and generate the necessary Kubernetes configuration.

Build and deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces **Terminal**:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=inventory-quarkus -l app=inventory-quarkus`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index), providing foundational software needed to run Java applications, while staying at a reasonable size.

> **NOTE**: Make sure if you log in OpenShift via `oc login command` at Terminal.

Next, create a temp directory to store only previously-built application with necessary lib directory via CodeReady Workspaces **Terminal**:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

> **NOTE**: You can also use a true source-based S2I build, but we're using binaries here to save time.

And then start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-dir=target/binary --follow`

Once the build is done, we'll deploy it as an OpenShift application and override the Postgres URL to specify our production Postgres credentials:

`oc new-app inventory-quarkus -e QUARKUS_DATASOURCE_URL=jdbc:postgresql://inventory-database:5432/inventory`

and expose your service to the world:

`oc expose service inventory-quarkus`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus`

Wait for that command to report replication controller "inventory-quarkus-1" successfully rolled out before continuing.

>**NOTE:** Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

`oc get routes`

Replace your own route URL in the above command output: 

`curl http://inventory-quarkus-userxx-inventory.apps.seoul-7b68.openshiftworkshop.com/services/inventory ; echo`

So now `Inventory` service is deployed to OpenShift. You can also see it in the Overview in the OpenShift Console 
with its single replica running in 1 pod (the blue circle), along with the Postgres database pod:

####12. Access the application running on OpenShift

---

This sample project includes a simple UI that allows you to access the Inventory API. This is the same
UI that you previously accessed outside of OpenShift which shows the CoolStore inventory. Click on the
route URL at `OpenShift Web Console` to access the sample UI.

> You can also access the application through the link on the OpenShift Web Console Overview page.

![Overview link]({% image_path inventory-route-link.png %})

> **NOTE**: If you get a '404 Not Found' error, just reload the page a few times until the Inventory UI appears. This
is due to a lack of health check which you are about to fix!

The UI will refresh the inventory table every 2 seconds, as before.

Back on the OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory-quarkus` and then click on
the top-most `(latest)` deployment in the listing (most likely `#1` or `#2`):

![Overview link]({% image_path deployment-list.png %})

Notice OpenShift is warning you that the inventory application has no health checks:

![Health Check Warning]({% image_path inventory-healthcheck-warning.png %}){:width="800px"}

In the next steps you will enhance OpenShift's ability to manage the application lifecycle by implementing
a _health check pattern_. By default, without health checks (or health _probes_) OpenShift considers services
to be ready to accept service requests even before the application is truly ready or if the application is hung
or otherwise unable to service requests. OpenShift must be _taught_ how to recognize that our app is alive and ready
to accept requests. 

##### What is MicroProfile Health?

`MicroProfile Health` allows applications to provide information about their state to external viewers 
which is typically useful in cloud environments where automated processes must be able to determine whether 
the application should be discarded or restarted. `Quarkus application` can utilize the MicroProfile Health specification through the `SmallRye Health extension`.

##### What is a Health Check?

A key requirement in any managed application container environment is the ability to determine when the application is in a ready state. Only when an
application has reported as ready can the manager (in this case OpenShift) act on the next step of the deployment process. OpenShift
makes use of various _probes_ to determine the health of an application during its lifespan. A _readiness_
probe is one of these mechanisms for validating application health and determines when an
application has reached a point where it can begin to accept incoming traffic. At that point, the IP
address for the pod is added to the list of endpoints backing the service and it can begin to receive
requests. Otherwise traffic destined for the application could reach the application before it was fully
operational resulting in error from the client perspective.

Once an application is running, there are no guarantees that it will continue to operate with full
functionality. Numerous factors including out of memory errors or a hanging process can cause the
application to enter an invalid state. While a _readiness_ probe is only responsible for determining
whether an application is in a state where it should begin to receive incoming traffic, a _liveness_ probe
is used to determine whether an application is still in an acceptable state. If the liveness probe fails,
OpenShift will destroy the pod and replace it with a new one.

In our case we will implement the health check logic in a REST endpoint and let Quarkus publish
that logic on the `/health` endpoint for use with OpenShift.

####13. Add Health Check Extension

---

We will add a Qurakus extension to the Inventory application for using `mallrye-health` and we'll use the Quarkus Maven Plugin.
Copy the following commands to import the smallrye-health extension that implements the MicroProfile Health specification 
via CodeReady Workspaces **Terminal**:

Go to `inventory' directory:

`cd cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="health"`

####14. Run the health check

---

Once you imported the `smallrye-health extension`, the **/health** endpoint is exposed directly that can be used to run the health check procedures.
Be sure to rollback H2 database configuration as defined in `src/main/resources/application.properties`:

~~~java 
quarkus.datasource.url=jdbc:h2:file://projects/database.db
quarkus.datasource.driver=org.h2.Driver
~~~

 * Run the Inventory application via `mvn compile quarkus:dev` or click on `Build and Run Locally` in **Commands Palette**:

![codeready-workspace-maven]({% image_path quarkus-dev-run-paletter.png %})

 * Access the `health check` endpoint using `curl http://localhost:8080/health` and the result looks like:

~~~java
{
      "outcome": "UP",
     "checks": [
     ]
}
~~~

The health REST enpoint returns a simple JSON object with two fields:

 * `outcome` - the overall result of all the health check procedures
 * `checks` - an array of individual checks

The general `outcome` of the health check is computed as a logical AND of all the declared health check procedures. 
The `checks` array is empty as we have not specified any health check procedure yet so let’s define some.

####15. Create your first health check

---

Next, let's fill in the class by creating a new RESTful endpoint which will be used by OpenShift to probe our services.
Open empty Java class: `src/main/java/com/redhat/coolstore/InventoryHealthCheck.java` and the follwing logic will be put into a new Java class.

~~~java
package com.redhat.coolstore;

import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;

import org.eclipse.microprofile.health.Health;
import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;

@Health
@ApplicationScoped
public class InventoryHealthCheck implements HealthCheck {

    @Inject
    private InventoryResource inventoryResource;

    @Override
    public HealthCheckResponse call() {

        if (inventoryResource.getAll() != null) {
            return HealthCheckResponse.named("Success of Inventory Health Check!!!").up().build();
        } else {
            return HealthCheckResponse.named("Failure of Inventory Health Check!!!").down().build();
        }
    }
}
~~~

The `call()` method exposes an HTTP GET endpoint which will return the status of the service. The logic of
this check does a simple query to the underlying database to ensure the connection to it is stable and available.
The method is also annotated with Quarkus's `@Health` annotation, which directs Quarkus to expose
this endpoint as a health check at `/health`.

>**NOTE:** If you don't terminate Quarkus Development mode, you don't need to re-run the Inventory application because Quarkus will **reload the changes automatically**.

Re-run the Inventory application via `mvn compile quarkus:dev` or click on `Build and Run Locally` in **Commands Palette**
and access the `health check` endpoint using `curl http://localhost:8080/health` and the result looks like:

`curl http://localhost:8080/health`

~~~java
{
   "outcome": "UP",
    "checks": [
        {
            "name": "Success of Inventory Health Check!!!",
            "state": "UP"
        }
    ]
}
~~~

With our new health check in place, we'll need to build and deploy the updated application in the next step.

####16. Re-Deploy to OpenShift

---

> **NOTE**: Be sure to rollback Postgres database configuration as defined in `src/main/resources/application.properties`:
 
~~~java
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
~~~

Repackage the inventory application via clicking on `Package for OpenShift` in `Commands Palette`:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Next, update a temp directory to store only previously-built application with necessary lib directory via CodeReady Workspaces **Terminal**:

`rm -rf target/binary && mkdir -p target/binary && cp -r target/*runner.jar target/lib target/binary`

And then start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-dir=target/binary --follow`

You should see a **Push successful** at the end of the build output and it. To verify that deployment is started and completed automatically, 
run the following command via CodeReady Workspaces **Terminal** :

`oc rollout status -w dc/inventory-quarkus`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

> **NOTE**: The # of deployment(i.e. `inventory-quarkus-2`) might be different in your project. Be sure if the sequence is increased(i.e. #1 -> #2).

Back on the OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory-quarkus` and then click on
the `Edit Health Checks` in `Actions`:

![inventory-healthcheck-redeploy]({% image_path inventory-healthcheck-redeploy.png %})

You should input the following variables in Readiness Path Probe and Liveness Probe:

 * Path: _/health_
 * Port: _8080_
 * Initial Delay: _10_
 * Timeout: _1_

![inventory-healthcheck-webconsole]({% image_path inventory-healthcheck-webconsole.png %})

You should also be able to access the health check logic
at the `inventory` endpoint via a web browser:

You should see a JSON response like:

~~~java
{
    "outcome": "UP",
    "checks": [
        {
            "name": "Success of Inventory Health Check!!!",
          "state": "UP"
        }
    ]
}
~~~

You can see the definition of the health check from the perspective of OpenShift via CodeReady Workspaces **Terminal**:

`oc describe dc/inventory-quarkus | egrep 'Readiness|Liveness'`

You should see:


>     Liveness:   http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
>     Readiness:  http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3

####17. Adjust probe timeout

---

The various timeout values for the probes can be configured in many ways. Let's tune the _liveness probe_ initial delay so that
we don't have to wait 3 minutes for it to be activated. Use OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory-quarkus` 
and then click on the `Edit Health Checks` in `Actions`:

![inventory-change-deplaytime]({% image_path inventory-change-deplaytime.png %})

You can also use the **oc** command to tune the
probe to wait 30 seconds before starting to poll the probe:

`oc set probe dc/inventory-quarkus --liveness --initial-delay-seconds=30`

And verify it's been changed (look at the `delay=` value for the Liveness probe) via CodeReady Workspaces **Terminal**:

`oc describe dc/inventory-quarkus | egrep 'Readiness|Liveness'`


>     Liveness:   http-get http://:8080/health delay=30s timeout=1s period=10s #success=1 #failure=3
>     Readiness:  http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3

In the next step, we'll exercise the probe and watch as it fails and OpenShift recovers the application.

####18. Exercise Health Check

---

From the OpenShift Web Console overview page, click on the route link to open the sample application UI:

![Route Link]({% image_path inventory-routelink.png %})

This will open up the sample application UI in a new browser tab:

![App UI]({% image_path app.png %})

The app will begin polling the inventory as before and report success:

![Greeting]({% image_path inventory.png %})

Now you will corrupt the service and cause its health check to start failing.
To simulate the app crasing, let's kill the underlying service so it stops responding. Execute via CodeReady Workspaces **Terminal**:

`oc rsh dc/inventory-quarkus pkill java`

This will execute the Linux `pkill` command to stop the running Java process in the container.

Check out the application sample UI page and notice it is now failing to access the inventory data, and the
`Last Successful Fetch` counter starts increasing, indicating that the UI cannot access inventory. This could have
been caused by an overloaded server, a bug in the code, or any other reason that could make the application
unhealthy.

![Greeting]({% image_path inventory-fail.png %})

At this point, return to the OpenShift web console and click on the _Overview_ tab for the project. Notice that the
dark blue circle has now gone light blue, indicating the application is failing its _liveness probe_:

![Not Ready]({% image_path notready.png %})

After too many liveness probe failures, OpenShift will forcibly kill the pod and container running the service, and spin up a new one to take
its place. Once this occurs, the light blue circle should return to dark blue. This should take about 30 seconds.

Return to the same sample app UI (without reloading the page) and notice that the UI has automatically
re-connected to the new service and successfully accessed the inventory once again:

![Greeting]({% image_path inventory.png %})

####19. Managing Application Configuration

---

In this step, you will learn how to manage application configuration and how to provide environment 
specific configuration to the services.

Applications require configuration in order to tweak the application behavior 
or adapt it to a certain environment without the need to write code and repackage 
the application for every change. These configurations are sometimes specific to 
the application itself such as the number of products to be displayed on a product 
page and some other times they are dependent on the environment they are deployed in 
such as the database coordinates for the application.

The most common way to provide configurations to applications is using environment 
variables and external configuration files such as properties, JSON or YAML files.

configuration files and command line arguments. These configuration artifacts
should be externalized from the application and the docker image content in
order to keep the image portable across environments.

OpenShift provides a mechanism called [ConfigMaps]({{OPENSHIFT_DOCS_BASE}}/dev_guide/configmaps.html) 
in order to externalize configurations 
from the applications deployed within containers and provide them to the containers 
in a unified way as files and environment variables. OpenShift also offers a way to 
provide sensitive configuration data such as certificates, credentials, etc to the 
application containers in a secure and encrypted mechanism called Secrets.

This allows developers to build the container image for their application only once, 
and reuse that image to deploy the application across various environments with 
different configurations that are provided to the application at runtime.

So far Catalog and Inventory services have been using each PostgreSQL database. 

####20. Externalize Inventory Configuration

---

Quarkus supports multiple mechanisms for externalizing configurations such as environment variables, 
Maven properties, command-line arguments and more. The recommend approach for the long-term for externalizing 
configuration is however using a **application.properties** which you have already packaged within the Inventory Maven project.

Quarkus also uses [MicroProfile Config](https://microprofile.io/project/eclipse/microprofile-config) to inject the configuration in the application. 
The injection uses the `@ConfigProperty` annotation.

> Check out `inventory/src/main/resources/application.properties` which contains the local H2 database configuration.

Create a new properties file with the PostgreSQL database credentials. Note that you can give an arbitrary 
name to this configuration (e.g. `prod`) in order to tell Quarkus which one to use. Open `src/main/resources/application-prod.properties` file and copy the following contents:

~~~java
quarkus.datasource.url=jdbc:postgresql:inventory
quarkus.datasource.driver=org.postgresql.Driver
quarkus.datasource.username=inventory
quarkus.datasource.password=mysecretpassword
quarkus.datasource.max-size=8
quarkus.datasource.min-size=2
quarkus.hibernate-orm.database.generation=drop-and-create
quarkus.hibernate-orm.log.sql=false
~~~

The hostname defined for the PostgreSQL connection-url corresponds to the PostgreSQL 
service name published on OpenShift. This name will be resolved by the internal DNS server 
exposed by OpenShift and accessible to containers running on OpenShift.

And then create a config map that you will use to overlay on the default `application-prod.properties` which is 
packaged in the Inventory JAR archive:

`oc create configmap inventory-quarkus --from-file=src/main/resources/application-prod.properties`

If you don't like bash commands, Go to the **USERXX CoolStore Inventory Microservice Application** 
project in OpenShift Web Console and then on the left sidebar, **Resources >> Config Maps**. Click 
on **Create Config Maps** button to create a config map with the following info:

 * Name: `inventory-quarkus`
 * Key: `application-prod.properties`
 * Value: *copy-paste the content of the above application-prod.properties*

Config maps hold key-value pairs and in the above command an `inventory-quarkus` config map 
is created with `application-prod.properties` as the key and the content of the `application-prod.properties` as the 
value. Whenever a config map is injected into a container, it would appear as a file with the same 
name as the key, at specified path on the filesystem.

You can see the content of the config map in the OpenShift Web Console or by 
using `oc describe cm inventory-quarkus` command.

Modify the Inventory deployment config so that it injects the `application-prod.properties` configuration you just created as 
a config map into the Inventory container:

`oc set volume dc/inventory-quarkus --add --configmap-name=inventory-quarkus --mount-path=/app/config`

The above command mounts the content of the `inventory-quarkus` config map as a file inside the Inventory container 
at `/app/config/application-prod.properties`

You can also connect to Inventory PostgreSQL database and check if the seed data is 
loaded into the database.

In OpenShift Web Console, navigate the left sidebar, **Applications >>Pods >>inventory-database-xxxxx**. 

Click on **Terminal** tab menu to run the following info:

`sh-4.2$ psql -U inventory -c "select * from inventory"`

![inventory-posgresql-terminal]({% image_path inventory-posgresql-terminal.png %})

You have now created a config map that holds the configuration content for Inventory and can be updated 
at anytime for example when promoting the container image between environments without needing to 
modify the Inventory container image itself. 

#### Summary

---

You learned a bit more about what Quarkus is, and how it can be used to create
modern Java microservice-oriented applications.

You created a new Inventory microservice representing functionality previously implmented in the monolithic
CoolStore application. For now this new microservice is completely disconnected from our monolith and is
not very useful on its own. In future steps you will link this and other microservices into the monolith to
begin the process of [strangling the monolith](https://www.martinfowler.com/bliki/StranglerApplication.html).

Quarkus brings in a number of concepts and APIs from the Java EE community, so your existing
Java EE skills can be re-used to bring your applications into the modern world of containers,
microservices and cloud deployments.

Quarkus will be one of many components of Red Hat OpenShift Application Runtimes soon. **Stay tuned!!**
In the next lab, you'll use Spring Boot, another popular framework, to implement additional microservices. Let's go!
