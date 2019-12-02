## Lab3 - Breaking the monolith apart - I

In the previous labs you learned how to take an existing monolithic Java EE application to the cloud
with JBoss EAP and OpenShift, and you got a glimpse into the power of OpenShift for existing applications.

You will now begin the process of modernizing the application by breaking the application into multiple
microservices using different technologies, with the eventual goal of re-architecting the entire application as a set of
distributed microservices. Later on we'll explore how you can better manage and monitor the application after
it is re-architected.

In this lab you will learn more about `Supersonic Subatomic Java` [Quarkus](https://quarkus.io/){:target="_blank"}, which is designed to be container and developer friendly.

Quarkus is a _Kubernetes Native_ Java stack, crafted from the best of breed Java libraries and standards.
Amazingly fast boot time, incredibly low RSS memory (not just heap size!) offering near instant scale up and high density memory utilization
in container orchestration platforms like Kubernetes. Quarkus uses a technique called compile time boot. [Learn more](https://quarkus.io/vision/container-first){:target="_blank"}.

This will be one of the runtimes included in [Red Hat Runtimes](https://www.redhat.com/en/products/runtimes){:target="_blank"}.


#### Goals of this lab

---

You will implement one component of the monolith as a Quarkus microservice and modify it to address
microservice concerns, understand its structure, deploy it to OpenShift and exercise the interfaces between
Quarkus apps, microservices, and OpenShift/Kubernetes.

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

[Quarkus](https://Quarkus.io/){:target="_blank"} offers 4 major benefits to build cloud-native, microservices, and serverless Java applicaitons:

* **Developer Joy** - Cohesive platform for optimized developer joy through unified configuration, Zero config with live reload in the blink of an eye,
   streamlined code for the 80% common usages with flexible for the 20%, and no hassle native executable generation.

* **Unifies Imperative and Reactive** - Inject the EventBus or the Vertx context for both Reactive and imperative development in the same application.

* **Functions as a Service and Serverless** - Superfast startup and low memory utilization. With Quarkus, you can embrace this new world without having
  to change your programming language.

* **Best of Breed Frameworks & Standards** - CodeReady Workspaces Vert.x, Hibernate, RESTEasy, Apache Camel, CodeReady Workspaces MicroProfile, Netty, Kubernetes, OpenShift, Jaeger, Prometheus, Apacke Kafka, Infinispan, and more.


####1. Setup an Inventory proejct

---

Run the following commands to set up your environment for this lab and start in the right directory:

In the project explorer, expand the **inventory**  project.

![inventory_setup]({% image_path codeready-workspace-inventory-project.png %}){:width="500px"}

####2. Examine the Maven project structure

---

The sample Quarkus project shows a minimal CRUD service exposing a couple of endpoints over REST,
with a front-end based on Angular so you can play with it from your browser.

While the code is surprisingly simple, under the hood this is using:

 * RESTEasy to expose the REST endpoints
 * Hibernate ORM with Panache to perform CRUD operations on the database
 * A PostgreSQL database; see below to run one via Linux Container
 * Some example `Dockerfile`s to generate new images for JVM and Native mode compilation

`Hibernate ORM` is the de facto JPA implementation and offers you the full breadth of an Object Relational Mapper.
It makes complex mappings possible, but it does not make simple and common mappings trivial. Hibernate ORM with
Panache focuses on making your entities trivial and fun to write in Quarkus.

Now let's write some code and create a domain model, service interface and a RESTful endpoint to access inventory:

![Inventory RESTful Service]({% image_path inventory-arch.png %}){:width="700px"}

####3. Add Qurkus Extensions

---

We will add Quarkus extensions to the Inventory application for using `Panache` (a simplified way to access data via Hibernate ORM), a database with `Postgres` (in production) and `H2` (for testing) and we'll use the Quarkus Maven Plugin. Copy the following commands to add the _Hibernate ORM with Panache_ extension via CodeReady Workspaces Terminal:

Go to `inventory' directory:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="hibernate-orm-panache"`

And then for local H2 database:

`mvn quarkus:add-extension -Dextensions="jdbc-h2"`

> NOTE: There are many [more extensions](https://quarkus.io/extensions/){:target="_blank"} for Quarkus for popular frameworks like [CodeReady Workspaces Vert.x](https://vertx.io/){:target="_blank"}, [Apache Camel](http://camel.apache.org/){:target="_blank"}, [Infinispan](http://infinispan.org/){:target="_blank"}, Spring DI compatibility (e.g. `@Autowired`), and more.

####4. Create Inventory Entity

---

With our skeleton project in place, let's get to work defining the business logic.

The first step is to define the model (entity) of an Inventory object. Since Quarkus uses Hibernate ORM Panache, we can re-use the same model definition from our monolithic application - no need to re-write or re-architect!

Open up the empty **Inventory.java** file in _com.redhat.coolstore_ package and paste the following code into it (identical to the monolith code):

~~~java
package com.redhat.coolstore;

import javax.persistence.Cacheable;
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

By extending `PanacheEntity` in your entities, you will get an ID field that is auto-generated. If you require a custom ID strategy, you can extend `PanacheEntityBase` instead and handle the ID yourself.

By using Use public fields, there is no need for functionless getters and setters (those that simply get or set the field). You simply refer to fields like Inventory.location without the need to write a Inventory.geLocation() implementation. Panache will auto-generate any getters and setters you do not write, or you can develop your own getters/setters that do more than get/set, which will be called when the field is accessed directly.

The `PanacheEntity` superclass comes with lots of super useful static methods and you can add your own in your derived entity class, and much like traditional object-oriented programming it's natural and recommended to place custom queries as close to the entity as possible, ideally within the entity definition itself.
Users can just start using your entity Inventory by typing Inventory, and getting completion for all the operations in a single place.

When an entity is annotated with `@Cacheable`, all its field values are cached except for collections and relations to other entities.
This means the entity can be loaded quicker without querying the database for frequently-accessed, but rarely-changing data.

####5. Define the RESTful endpoint of Inventory

---

In this step we will mirror the abstraction of a _service_ so that we can inject the Inventory _service_ into various places (like a RESTful resource endpoint) in the future. This is the same approach that our monolith uses, so we can re-use this idea again. Open up the empty **InventoryResource.java** class in the _com.redhat.coolstore_ package.

Add this code to it:

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

* `/inventory` that is accessible via _HTTP GET_ which will return all known product Inventory entities as JSON

* `/inventory/<itemId>` that is accessible via _HTTP GET_ at for example `/inventory/329199` with the last path parameter being the ID for which we want inventory status.

####6. Add inventory data

---

Let's add inventory data to the database so we can test things out. Open up the `src/main/resources/import.sql` file and
copy the following SQL statements to **import.sql**:

~~~sql
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '329299', 'http://maps.google.com/?q=Raleigh', 'Raleigh', 736);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '329199', 'http://maps.google.com/?q=Boston', 'Boston', 512);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165613', 'http://maps.google.com/?q=Seoul', 'Seoul', 256);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165614', 'http://maps.google.com/?q=Singapore', 'Singapore', 54);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '165954', 'http://maps.google.com/?q=London', 'London', 87);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444434', 'http://maps.google.com/?q=NewYork', 'NewYork', 443);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444435', 'http://maps.google.com/?q=Paris', 'Paris', 600);
INSERT INTO INVENTORY (id, itemId, link, location, quantity) values (nextval('hibernate_sequence'), '444437', 'http://maps.google.com/?q=Tokyo', 'Tokyo', 230);
~~~

In Development, we will configure to use local in-memory H2 database for local testing. Add these lines to `src/main/resources/application.properties`:

~~~
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

Now we are ready to run the inventory application. Click on **Commands Palette** then select **Build and Run Locally** in Run menu:

![codeready-workspace-maven]({% image_path quarkus-dev-run-paletter.png %})

> This simply runs `mvn compile quarkus:dev` for you

You should see a bunch of log output that ends with:

~~~
2019-09-26 15:55:38,447 INFO  [io.quarkus] (main) Quarkus 0.22.0 started in 2.905s. Listening on: http://0.0.0.0:8080
2019-09-26 15:55:38,447 INFO  [io.quarkus] (main) Profile dev activated. Live Coding activated.
2019-09-26 15:55:38,447 INFO  [io.quarkus] (main) Installed features: [agroal, cdi, hibernate-orm, jdbc-h2, narayana-jta, resteasy, resteasy-jsonb]
~~~

Open a **new** CodeReady Workspaces Terminal and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`

~~~json
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com
/?q=Boston","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"item
Id":"165614","link":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London"
,"location":"London","quantity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444
435","link":"http://maps.google.com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tok
yo","quantity":230}]
~~~

`curl http://localhost:8080/services/inventory/329199 ; echo`

~~~json
[{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Boston","location":"Boston","quantity":512}]
~~~

##### Stop the application

Stop Quarkus development mode by closing the _Build and Run Locally_ Terminal window.

####8. Add Test Code and create a package

---

In this step, we will add unit tests so that we can test during `mvn package`.
Open up the `src/test/java/com/redhat/coolstore/InventoryEndpointTest.java` file and replace the following code with the below code:

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

Next we'll build an executable jar then deploy it to **OpenShift** soon. Use the following maven command via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory`

`mvn clean package`

If builds successfully (you will see **BUILD SUCCESS**), continue to the next step to deploy the application to OpenShift.

You can also run the Uber.jar to make sure if the inventory works. Use the following **Java command** then you see a similar output:

`java -jar target/inventory-1.0-SNAPSHOT-runner.jar`

~~~
2019-09-26 16:16:08,977 INFO  [io.quarkus] (main) Quarkus 0.23.1 started in 1.806s. Listening on: http://0.0.0.0:8080
2019-09-26 16:16:08,989 INFO  [io.quarkus] (main) Profile prod activated.
2019-09-26 16:16:08,989 INFO  [io.quarkus] (main) Installed features: [agroal, cdi, hibernate-orm, jdbc-h2, narayana-jta, resteasy, resteasy-jsonb]
~~~

Open a new CodeReady Workspaces Terminal and invoke the RESTful endpoint using the following CURL commands. The output looks like here:

`curl http://localhost:8080/services/inventory ; echo`

~~~json
[{"id":1,"itemId":"329299","link":"http://maps.google.com/?q=Raleigh","location":"Raleigh","quantity":736},{"id":2,"itemId":"329199","link":"http://maps.google.com
/?q=Boston","location":"Boston","quantity":512},{"id":3,"itemId":"165613","link":"http://maps.google.com/?q=Seoul","location":"Seoul","quantity":256},{"id":4,"item
Id":"165614","link":"http://maps.google.com/?q=Singapore","location":"Singapore","quantity":54},{"id":5,"itemId":"165954","link":"http://maps.google.com/?q=London"
,"location":"London","quantity":87},{"id":6,"itemId":"444434","link":"http://maps.google.com/?q=NewYork","location":"NewYork","quantity":443},{"id":7,"itemId":"444
435","link":"http://maps.google.com/?q=Paris","location":"Paris","quantity":600},{"id":8,"itemId":"444437","link":"http://maps.google.com/?q=Tokyo","location":"Tok
yo","quantity":230}]
~~~

`curl http://localhost:8080/services/inventory/329199 ; echo`

~~~json
[{"id":2,"itemId":"329199","link":"http://maps.google.com/?q=Boston","location":"Boston","quantity":512}]
~~~

##### Stop the application

Stop Quarkus development mode by closing the Terminal window in which you ran `java -jar`

You have now successfully created your first microservice using Quarkus and implemented a basic RESTful
API on top of the Inventory database. Most of the code looks simpler than the monolith, demonstrating how
easy it is to migrate existing monolithic Java EE application to microservices using `Quarkus`.

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
because our Inventory service will connect to PostgeSQL database in production on OpenShift.

Add a _quarkus-jdbc-postgresql_ extension via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="jdbc-postgresql"`

Quarkus supports the notion of _configuration profiles_. These allows you to have multiple configurations in the same file and select between
then via a _profile name_.

By default Quarkus has three profiles, although it is possible to use as many as you like. The default profiles are:

 * **dev** - Activated when in development mode (i.e. **quarkus:dev**)

 * **test** - Activated when running tests

 * **prod** - The default profile when not running in development or test mode

There are two ways to set a custom profile, either via the `quarkus.profile` system property or the `QUARKUS_PROFILE` environment variable.
If both are set the system property takes precedence. Note that it is not necessary to define the names of these profiles anywhere,
all that is necessary is to create a config property with the profile name, and then set the current profile to that name.

Let's add the following variables in _src/main/resources/application.properties_:

~~~shell
%prod.quarkus.datasource.url=jdbc:postgresql://inventory-database:5432/inventory
%prod.quarkus.datasource.driver=org.postgresql.Driver
%prod.quarkus.datasource.username=inventory
%prod.quarkus.datasource.password=mysecretpassword
%prod.quarkus.datasource.max-size=8
%prod.quarkus.datasource.min-size=2
%prod.quarkus.hibernate-orm.database.generation=drop-and-create
%prod.quarkus.hibernate-orm.sql-load-script=import.sql
%prod.quarkus.hibernate-orm.log.sql=true
~~~

Repackage the inventory application via clicking on **Package for OpenShift** in Commands Palette:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

In OpenShift, click on the name of the **userXX-inventory** project:

![create_new]({% image_path create_new_inventory.png %})

This will take you to the project overview. There's nothing there yet, but that's about to change.

####10. Deploy to OpenShift

---

Let's deploy our new inventory microservice to OpenShift!

Our production inventory microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing the following commands via CodeReady Workspaces Terminal:

`oc project userXX-inventory`

Then run:

~~~shell
oc new-app -e POSTGRESQL_USER=inventory \
  -e POSTGRESQL_PASSWORD=mysecretpassword \
  -e POSTGRESQL_DATABASE=inventory openshift/postgresql:10 \
  --name=inventory-database
~~~

> NOTE: If you change the username and password you also need to update `src/main/resources/application.properties` which contains
the credentials used when deploying to OpenShift.

This will deploy the database to our new project.

![inventory_db_deployments]({% image_path inventory-database-deployment.png %})

####11. Build and Deploy

---

Red Hat OpenShift Application Runtimes includes a powerful maven plugin that can take an
existing Quarkus application and generate the necessary Kubernetes configuration.

Build and deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces Terminal:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=inventory-quarkus -l app=inventory-quarkus`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index){:target="_blank"}, providing foundational software needed to run Java applications, while staying at a reasonable size.

> NOTE: After a while, you may get logged out of OpenShift. If the above command fails due to permissions, you can repeat the OpenShift login process from earlier.

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow`

Once the build is done, we'll deploy it as an OpenShift application and override the Postgres URL to specify our production Postgres credentials:

`oc new-app inventory-quarkus -e QUARKUS_PROFILE=prod`

and expose your service to the world:

`oc expose service inventory-quarkus`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/inventory-quarkus`

Wait for that command to report replication controller "inventory-quarkus-1" successfully rolled out before continuing.

> NOTE: Even if the rollout command reports success the application may not be ready yet and the reason for that is that we currently don't have any liveness/readiness check configured, but we will add that in the next steps.

And now we can access using curl once again to find all inventories:

`export URL="http://$(oc get route | grep inventory | awk '{print $2}')"`

`curl $URL/services/inventory ; echo`

So now `Inventory` service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console
with its single replica running in 1 pod, along with the Postgres database pod.

####12. Access the application running on OpenShift

---

This sample project includes a simple UI that allows you to access the Inventory API. This is the same
UI that you previously accessed outside of OpenShift which shows the CoolStore inventory. Click on the
route URL at `Networking > Routes` to access the sample UI.

![Overview link]({% image_path inventory-route-link.png %})

> NOTE: If you get a '404 Not Found' error, just reload the page a few times until the Inventory UI appears. This is due to a lack of health check which you are about to fix!

You can also access the application through the link on the `Project Status` page.

![Overview link]({% image_path inventory-route-link-status.png %})

The UI will refresh the inventory table every 2 seconds, as before.

In the next steps you will enhance OpenShift's ability to manage the application lifecycle by implementing
a _health check pattern_. By default, without health checks (or health _probes_) OpenShift considers services
to be ready to accept service requests even before the application is truly ready or if the application is hung
or otherwise unable to service requests. OpenShift must be _taught_ how to recognize that our app is alive and ready
to accept requests.

##### What is MicroProfile Health?

**MicroProfile Health** allows applications to provide information about their state to external viewers which is typically useful in cloud environments where automated processes must be able to determine whether the application should be discarded or restarted. **Quarkus application** can utilize the MicroProfile Health specification through the _SmallRye Health extension_.

##### What is a Health Check?

A key requirement in any managed application container environment is the ability to determine when the application is in a ready state. Only when an application has reported as ready can the manager (in this case OpenShift) act on the next step of the deployment process. OpenShift makes use of various _probes_ to determine the health of an application during its lifespan. A _readiness_ probe is one of these mechanisms for validating application health and determines when an application has reached a point where it can begin to accept incoming traffic. At that point, the IP address for the pod is added to the list of endpoints backing the service and it can begin to receive requests. Otherwise traffic destined for the application could reach the application before it was fully operational resulting in error from the client perspective.

Once an application is running, there are no guarantees that it will continue to operate with full functionality. Numerous factors including out of memory errors or a hanging process can cause the application to enter an invalid state. While a _readiness_ probe is only responsible for determining whether an application is in a state where it should begin to receive incoming traffic, a _liveness_ probe is used to determine whether an application is still in an acceptable state. If the liveness probe fails, OpenShift will destroy the pod and replace it with a new one.

In our case we will implement the health check logic in a REST endpoint and let Quarkus publish that logic on the _/health_ endpoint for use with OpenShift.

####13. Add Health Check Extension

---

We will add a Qurakus extension to the Inventory application for using **smallrye-health** and we'll use the Quarkus Maven Plugin.
Copy the following commands to import the smallrye-health extension that implements the MicroProfile Health specification
via CodeReady Workspaces Terminal:

Go to `inventory` directory and add the extension:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory/`

`mvn quarkus:add-extension -Dextensions="health"`

####14. Run the health check

---

When you import the _smallrye-health extension_, the `/health` endpoint is automatically exposed directly that can be used to run the health check procedures.

 * Run the Inventory application via `mvn compile quarkus:dev` or click on **Build and Run Locally** in Commands Palette:

![codeready-workspace-maven]({% image_path quarkus-dev-run-paletter.png %})

 * In a separate Terminal, access the `health check` endpoint using `curl http://localhost:8080/health` and the result should look like:

~~~json
{
      "status": "UP",
     "checks": [
     ]
}
~~~

The health REST enpoint returns a simple JSON object with two fields:

 * **status** - the overall result of all the health check procedures
 * **checks** - an array of individual checks

The general _status_ of the health check is computed as a logical AND of all the declared health check procedures.
The _checks_ array is empty as we have not specified any health check procedure yet so let’s define some.

####15. Create your first health check

---

Next, let's fill in the class by creating a new RESTful endpoint which will be used by OpenShift to probe our services.
Open empty Java class: **src/main/java/com/redhat/coolstore/InventoryHealthCheck.java** and the following logic will be put into a new Java class.

Replace the following codes with the exsiting entire codes:

~~~java
package com.redhat.coolstore;

import javax.enterprise.context.ApplicationScoped;
import javax.inject.Inject;

import org.eclipse.microprofile.health.HealthCheck;
import org.eclipse.microprofile.health.HealthCheckResponse;
import org.eclipse.microprofile.health.Readiness;

@Readiness
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

The **call()** method exposes an HTTP GET endpoint which will return the status of the service. The logic of this check does a simple query to the underlying database to ensure the connection to it is stable and available. The method is also annotated with Quarkus's **@Readiness** annotation, which directs Quarkus to expose this endpoint as a health check at _/health/ready_.

> NOTE: You don't need to stop and re-run re-run the Inventory application because Quarkus will **reload the changes automatically**.

Access the _Readiness health check_ endpoint again using _curl_ and the result looks like:

`curl http://localhost:8080/health/ready`

~~~json
{
   "status": "UP",
    "checks": [
        {
            "name": "Success of Inventory Health Check!!!",
            "state": "UP"
        }
    ]
}
~~~

With our new health check in place, we'll need to build and deploy the updated application in the next step. Before move to the next step, be sure to close the termial where you're running Quarkus development mode.

`Tip`: You can define liveness probe using **@Liveness** annotation and the liveness check can be accessible at **/health/live** endpoint.

####16. Re-Deploy to OpenShift

---

Repackage the inventory application via clicking on **Package for OpenShift** in Commands Palette:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Start and watch the build, which will take about a minute to complete:

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow`

You should see a **Push successful** at the end of the build output and it. To verify that deployment is started and completed automatically,
run the following command via CodeReady Workspaces Terminal:

`oc rollout status -w dc/inventory-quarkus`

And wait for the result as below:

`replication controller "inventory-quarkus-XX" successfully rolled out`

Let's set _readiness probe_ in OpenShift using _InventoryHealthCheck_. Run the follwing _oc set probe_ command in CodeReady Workspaces:

`oc set probe dc/inventory-quarkus --readiness --get-url=http://:8080/health/ready`

This will instruct OpenShift to use the `/health/ready` endpoint to continually check the health of the app.

Back on the OpenShift console, Navigate to _Deployment Configs_ on the left menu then click on _inventory-quarkus_:

![inventory-dc]({% image_path inventory-dc.png %})

 Click on _YAML_ tab then you will see the following variables in _template.spec.containers.resources_ path:

~~~yaml
        readinessProbe:
            httpGet:
              path: /health/ready
              port: 8080
              scheme: HTTP
            timeoutSeconds: 1
            periodSeconds: 10
            successThreshold: 1
            failureThreshold: 3
~~~

![inventory-healthcheck-webconsole]({% image_path inventory-healthcheck-webconsole.png %})

You should also be able to access the health check logic at the _inventory_ endpoint via a web browser:

`export URL="http://$(oc get route | grep inventory | awk '{print $2}')"`

`curl $URL/health/ready ; echo`

You should see a JSON response like:

~~~java
{
    "status": "UP",
    "checks": [
        {
            "name": "Success of Inventory Health Check!!!",
          "state": "UP"
        }
    ]
}
~~~

You can see the definition of the health check from the perspective of OpenShift via CodeReady Workspaces Terminal:

`oc describe dc/inventory-quarkus | egrep 'Readiness|Liveness'`

You should see:

>     Readiness:  http-get http://:8080/health/ready delay=0s timeout=1s period=10s #success=1 #failure=3

####17. Adjust probe timeout

---

The various timeout values for the probes can be configured in many ways. Let's tune the _readiness probe_ initial delay so that we have to wait 3o seconds for it to be activated. Use the _oc_ command to tune the probe to wait 30 seconds before starting to poll the probe:

`oc set probe dc/inventory-quarkus --readiness --initial-delay-seconds=30`

And verify it's been changed (look at the _delay=_ value for the Readiness probe) via CodeReady Workspaces Terminal:

`oc describe dc/inventory-quarkus | egrep 'Readiness|Liveness'`


>     Readiness:  http-get http://:8080/health/ready delay=30s timeout=1s period=10s #success=1 #failure=3

In the next step, we'll exercise the probe and watch as it fails and OpenShift recovers the application.

####18. Exercise Health Check

---

From the project status page, click on the route link to open the sample application UI:

![Route Link]({% image_path inventory-route-link.png %})

This will open up the sample application UI in a new browser tab:

![App UI]({% image_path app.png %})

The app will begin polling the inventory as before and report success:

![Greeting]({% image_path inventory.png %})

Now you will corrupt the service and cause its health check to start failing.
To simulate the app crasing, let's kill the underlying service so it stops responding. Execute via CodeReady Workspaces Terminal:

`oc rsh dc/inventory-quarkus pkill java`

This will execute the Linux **pkill** command to stop the running Java process in the container.

Check out the application sample UI page and notice it is now failing to access the inventory data, and the
_Last Successful Fetch_ counter starts increasing, indicating that the UI cannot access inventory. This could have
been caused by an overloaded server, a bug in the code, or any other reason that could make the application
unhealthy.

![Greeting]({% image_path inventory-fail.png %})

At this point, return to the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} and click on the _Pods_ on the left menu. Notice that the
**ContainersNotReady** indicates the application is failing its _readiness probe_:

![Not Ready]({% image_path notready.png %})

After too many healthcheck probe failures, OpenShift will forcibly kill the pod and container running the service, and spin up a new one to take its place. Once this occurs, the light blue circle should return to dark blue. This should take about 30 seconds.

Return to the same sample app UI (without reloading the page) and notice that the UI has automatically re-connected to the new service and successfully accessed the inventory once again:

![Greeting]({% image_path inventory.png %})

#### Summary

---

You learned a bit more about what Quarkus is, and how it can be used to create modern Java microservice-oriented applications.

You created a new Inventory microservice representing functionality previously implmented in the monolithic CoolStore application. For now this new microservice is completely disconnected from our monolith and is not very useful on its own. In future steps you will link this and other microservices into the monolith to
begin the process of [strangling the monolith](https://www.martinfowler.com/bliki/StranglerApplication.html){:target="_blank"}.

Quarkus brings in a number of concepts and APIs from the Java EE community, so your existing Java EE skills can be re-used to bring your applications into the modern world of containers, microservices and cloud deployments.

Quarkus will be one of many components of Red Hat OpenShift Application Runtimes soon. **Stay tuned!!**
In the next lab, you'll use Spring Boot, another popular framework, to implement additional microservices. Let's go!
