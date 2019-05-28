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

You will implement one component of the monolith as a Thorntail microservice and modify it to address
microservice concerns, understand its structure, deploy it to OpenShift and exercise the interfaces between
Thorntail apps, microservices, and OpenShift/Kubernetes.

#### Goals of this lab

The goal is to deploy this new microservice alongside the existing monolith, and then later on we'll tie them together.
But after this lab, you should end up with something like:

![lab3_goal]({% image_path goal.png %}){:width="700px"}

---

#### What is Quarkus? 

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
* `Best of Breed Frameworks & Standards` - Eclipse Vert.x, Hibernate, RESTEasy, Apache Camel, Eclipse MicroProfile, Netty, Kubernetes, OpenShift,
  Jaeger, Prometheus, Apacke Kafka, Infinispan, and more.


**1. Setup an Inventory proejct**

Run the following commands to set up your environment for this lab and start in the right directory:

In the project explorer, right-click on **inventory** and then change a directory to inventory path on **Terminal**.

![inventory_setup]({% image_path bootstrap-che-inventory-project.png %}){:width="500px"}

**2. Examine the Maven project structure**

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

This project currently contains no code other than web resources such as index.html in `src/main/resources`.

Build and package the app using Maven to make sure the changed code still compiles via Eclipse Che **BUILD** window:

![inventory_build]({% image_path bootstrap-che-build-inventory.png %})

> **NOTE**: Make sure to build this mvn command at working directory(i.e inventory).

If builds successfully (you will see `BUILD SUCCESS`), then let's move on to the next issue! If it does not compile,
verify you made all the changes correctly and try the build again.

Once built, the resulting *jar* is located in the **target** directory via Eclipse Che **Terminal** window:

`ll target/*-runner.jar`

![inventory_build_success]({% image_path inventory-build-success.png %})

The listed jar archive, **inventory-1.0.0-SNAPSHOT-runner.jar** , is an uber-jar with
all the dependencies required packaged in the *jar* to enable running the
application with **java -jar**. Quarkus also creates a native executable image which improves the startup time of 
the application, and produces a minimal disk footprint. The native image will be running on [GraalVM](https://www.graalvm.org/).

Now let's write some code and create a domain model, service interface and a RESTful endpoint to access inventory:

![Inventory RESTful Service]({% image_path inventory-arch.png %})

**3. Add Qurkus Extensions**

We will add Qurakus extensions to the Inventory application for using `Panache` and `Postgres` and We'll use the Quarkus Maven Plugin.
Copy the following commands to add the Hibernate ORM with Panache extension via **Terminal**:

~~~shell
mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-hibernate-orm-panache"
~~~

And then for Postgres:

~~~shell
mvn quarkus:add-extension -Dextensions="io.quarkus:quarkus-jdbc-postgresql"
~~~

>**NOTE:** There are many [more extensions](https://quarkus.io/extensions/) for Quarkus for popular frameworks 
like [Eclipse Vert.x](https://vertx.io/), [Apache Camel](http://camel.apache.org/), [Infinispan](http://infinispan.org/), 
Spring DI compatibility (e.g. @Autowired), and more.

**4. Create Inventory Entity**

With our skeleton project in place, let's get to work defining the business logic.

The first step is to define the model (entity) of an Inventory object. Since Quarkus uses Hibernate ORM Panache,
we can re-use the same model definition from our monolithic application - no need to re-write or re-architect!

Create a new Java class named `Inventory.java` in
`com.redhat.coolstore` package with the following code, identical logics to the monolith code:

~~~java
ackage com.redhat.coolstore;

import javax.persistence.Cacheable;
import javax.persistence.Column;
import javax.persistence.Entity;

import io.quarkus.hibernate.orm.panache.PanacheEntity;

@Entity
@Cacheable
public class Inventory extends PanacheEntity {

	@Column
    public String location;

	@Column
    public int quantity;

	@Column
    public String link;

    public Inventory() {

    }

    public Inventory(Long itemId, int quantity, String location, String link) {
        super();
        this.quantity = quantity;
        this.location = location;
        this.link = link;
    }

}
~~~

By extending **PanacheEntity** in your entities, you will get an ID field that is auto-generated. If you require a custom ID strategy, 
you can extend **PanacheEntityBase** instead and handle the ID yourself.

By using Use public fields, there is no need for functionless getters and setters (those that simply get or set the field). You simply refer to fields like Inventory.location without the need to write a Inventory.geLocation() implementation. Panache will auto-generate any getters and setters you do not write, or you can develop your own getters/setters that do more than get/set, which will be called when the field is accessed directly.

The `PanacheEntity` superclass comes with lots of super useful static methods and you can add your own in your derived entity class, and much like traditional object-oriented programming it's natural and recommended to place custom queries as close to the entity as possible, ideally within the entity definition itself. 
Users can just start using your entity Inventory by typing Inventory, and getting completion for all the operations in a single place.

Also note that the configurations uses `src/main/resources/META-INF/load.sql` to import
initial data into the database.

Examine `src/main/resources/project-stages.yml` to see the database connection details.
An in-memory H2 database is used in this lab for local development and in the following
steps will be replaced with a PostgreSQL database with credentials coming from an OpenShift _secret_. Be patient! More on that later.

Build and package the app using Maven to make sure the changed code still compiles via Eclipse Che **BUILD** window:

![inventory_build]({% image_path bootstrap-che-build-inventory.png %})

> **NOTE**: Make sure to build this mvn command at working directory(i.e inventory).

If builds successfully (you will see `BUILD SUCCESS`), continue to the next step to create a new service.

**5. Define the RESTful endpoint of Inventory**

In this step we will mirror the abstraction of a _service_ so that we can inject the Inventory _service_ into
various places (like a RESTful resource endpoint) in the future. This is the same approach that our monolith
uses, so we can re-use this idea again. Create an **InventoryService** class in the `com.redhat.coolstore.service` package:

~~~java
package com.redhat.coolstore.service;


import javax.ejb.Stateless;
import javax.persistence.EntityManager;
import javax.persistence.PersistenceContext;
import javax.persistence.Query;

import com.redhat.coolstore.model.Inventory;

import java.util.Collection;
import java.util.List;

@Stateless
public class InventoryService {

    @PersistenceContext
    private EntityManager em;

    public InventoryService() {

    }

    public boolean isAlive() {
        return em.createQuery("select 1 from Inventory i")
                .setMaxResults(1)
                .getResultList().size() == 1;
    }
    public Inventory getInventory(String itemId) {
        return em.find(Inventory.class, itemId);
    }

    public List<Inventory> getAllInventory() {
        Query query = em.createQuery("SELECT i FROM Inventory i");
        return query.getResultList();
    }
}
~~~

Review the **InventoryService** class and note the EJB and JPA annotations on this class:

* **@Stateless** marks
the class as a _Stateless EJB_, and its name suggests, means that instances of the class do not maintain state,
which means they can be created and destroyed at will by the management system, and be re-used by multiple clients
without instantiating multiple copies of the bean. Because they can support multiple
clients, stateless EJBs can offer better scalability for applications that require large numbers of
clients.

* **@PersistenceContext** objects are created by the Java EE server based on the JPA definition in `persistence.xml` that
we examined earlier, so to use it at runtime it is injected by this annotation and can be used to issue queries against
the underlying database backing the **Inventory** entities.

This service class exposes a few APIs that we'll use later:

* **isAlive()** - A simple health check to determine if this service class is ready to accept requests. We will use
this later on when defining OpenShift health checks.

* **getInventory()** and **getAllInventory()** are APIs used to query for one or all of the stored **Inventory* entities. We'll use this
later on when implementing a RESTful endpoint.

Build and package the app using Maven to make sure the changed code still compiles via Eclipse Che **BUILD** window:

![inventory_build]({% image_path bootstrap-che-build-inventory.png %})

> **NOTE**: Make sure to build this mvn command at working directory(i.e inventory).

If builds successfully (you will see `BUILD SUCCESS`), continue to the next step to
create a new RESTful endpoint that uses this service.

**5. Create RESTful Endpoints**

Thorntail uses JAX-RS standard for building REST services. Create a new Java class named
`InventoryEndpoint.java` in `com.redhat.coolstore.rest` package with the following
content by clicking on *Copy to Editor*:

~~~java
package com.redhat.coolstore.rest;

import java.io.Serializable;
import java.util.List;

import javax.enterprise.context.RequestScoped;
import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;
import javax.ws.rs.PathParam;
import javax.ws.rs.Produces;
import javax.ws.rs.core.MediaType;

import com.redhat.coolstore.model.Inventory;
import com.redhat.coolstore.service.InventoryService;

@RequestScoped
@Path("/inventory")
public class InventoryEndpoint implements Serializable {

    private static final long serialVersionUID = -7227732980791688773L;

    @Inject
    private InventoryService inventoryService;

    @GET
    @Produces(MediaType.APPLICATION_JSON)
    public List<Inventory> getAll() {
        return inventoryService.getAllInventory();
    }

    @GET
    @Path("{itemId}")
    @Produces(MediaType.APPLICATION_JSON)
    public Inventory getAvailability(@PathParam("itemId") String itemId) {
        return inventoryService.getInventory(itemId);
    }

}
~~~

The above REST services defines two endpoints:

* `/services/inventory` that is accessible via **HTTP GET** which will return all known product Inventory entities as JSON
* `/services/inventory/<id>` that is accessible via **HTTP GET** at
for example **/services/inventory/329299** with
the last path parameter being the product id which we want to check its inventory status.

The code also injects our new **InventoryService** using the [CDI @Inject](https://docs.oracle.com/javaee/7/tutorial/partcdi.htm) annotation, which gives
us a runtime handle to the service we defined in the previous steps that we can use to query
the database when the RESTful APIs are invoked.

Build and package the app using Maven to make sure the changed code still compiles via Eclipse Che **BUILD** window:

![inventory_build]({% image_path bootstrap-che-build-inventory.png %})

> **NOTE**: Make sure to build this mvn command at working directory(i.e inventory).

If builds successfully (you will see `BUILD SUCCESS`), continue to the next step to deploy the application to OpenShift.

You have now successfully created your first microservice using Thorntail and implemented a basic RESTful
API on top of the Inventory database. Most of the code is the same as was found in the monolith, demonstrating how
easy it is to migrate existing monolithic Java EE applications to microservices using Thorntail.

In next steps of this lab we will deploy our application to OpenShift Container Platform and then start
adding additional features to take care of various aspects of cloud native microservice development.


**6. Create OpenShift Project**

We have already deployed our coolstore monolith to OpenShift, but now we are working on re-architecting it to be
microservices-based.

In this step we will deploy our new Inventory microservice for our CoolStore application,
so let's create a separate project to house it and keep it separate from our monolith and our other microservices we will
create later on.

Create a new project for the _inventory_ service:

Click **Create Project**, fill in the fields, and click **Create**:

* Name: `inventory`
* Display Name: `CoolStore Inventory Microservice Application`
* Description: _leave this field empty_

![create_dialog]({% image_path create_inventory_dialog.png %}){:width="500"}

Click on the name of the newly-created project:

![create_new]({% image_path create_new_inventory.png %}){:width="500"}

This will take you to the project overview. There's nothing there yet, but that's about to change.

**7. Deploy to OpenShift**

Let's deploy our new inventory microservice to OpenShift!

Our production inventory microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing the following commands via Eclipse Che **Terminal**:

`oc project inventory`

`oc new-app -e POSTGRESQL_USER=inventory \
             -e POSTGRESQL_PASSWORD=mysecretpassword \
             -e POSTGRESQL_DATABASE=inventory \
             openshift/postgresql:latest \
             --name=inventory-database`

> **NOTE:** If you change the username and password you also need to update `src/main/fabric8/credential-secret.yml` which contains
the credentials used when deploying to OpenShift.

This will deploy the database to our new project. 

![inventory_db_deployments]({% image_path inventory-database-deployment.png %})

**8. Build and Deploy**

Red Hat OpenShift Application Runtimes includes a powerful maven plugin that can take an
existing Thorntail application and generate the necessary Kubernetes configuration.
You can also add additional config, like ``src/main/fabric8/inventory-deployment.yml`` which defines
the deployment characteristics of the app (in this case we declare a few environment variables which map our credentials
stored in the secrets file to the application), but OpenShift supports a wide range of [Deployment configuration options](https://docs.openshift.org/latest/architecture/core_concepts/deployments.html) for apps).

Build and deploy the project using the following command, which will use the maven plugin to deploy via Eclipse Che **Terminal**:

`mvn clean fabric8:deploy -Popenshift`

The build and deploy may take a minute or two. Wait for it to complete. You should see a **BUILD SUCCESS** at the
end of the build output.

> **NOTE**: If you see messages like `Current reconnect backoff is 2000 milliseconds (T1)` you can safely
ignore them, it is a known issue and is harmless.

After the maven build finishes it will take less than a minute for the application to become available.
To verify that everything is started, run the following command and wait for it complete successfully:

`oc rollout status -w dc/inventory`

>**NOTE:** Even if the rollout command reports success the application may not be ready yet and the reason for
that is that we currently don't have any liveness check configured, but we will add that in the next steps.

**9. Access the application running on OpenShift**

This sample project includes a simple UI that allows you to access the Inventory API. This is the same
UI that you previously accessed outside of OpenShift which shows the CoolStore inventory. Click on the
route URL at `OpenShift Web Console` to access the sample UI.

> You can also access the application through the link on the OpenShift Web Console Overview page.

![Overview link]({% image_path inventory-route-link.png %})

> **NOTE**: If you get a '404 Not Found' error, just reload the page a few times until the Inventory UI appears. This
is due to a lack of health check which you are about to fix!

The UI will refresh the inventory table every 2 seconds, as before.

Back on the OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory` and then click on
the top-most `(latest)` deployment in the listing (most likely `#1` or `#2`):

![Overview link]({% image_path deployment-list.png %})

Notice OpenShift is warning you that the inventory application has no health checks:

![Health Check Warning]({% image_path inventory-healthcheck-warning.png %}){:width="800px"}

In the next steps you will enhance OpenShift's ability to manage the application lifecycle by implementing
a _health check pattern_. By default, without health checks (or health _probes_) OpenShift considers services
to be ready to accept service requests even before the application is truly ready or if the application is hung
or otherwise unable to service requests. OpenShift must be _taught_ how to recognize that our app is alive and ready
to accept requests. 

**10. Add Health Check Fraction**

#### What is a Fraction?

Thorntail is defined by an unbounded set of capabilities. Each piece of functionality is called a fraction.
Some fractions provide only access to APIs, such as JAX-RS or CDI; other fractions provide higher-level capabilities,
such as integration with RHSSO (Keycloak).

The typical method for consuming Thorntail fractions is through Maven coordinates, which you add to the pom.xml
file in your application. The functionality the fraction provides is then packaged with your application into an
_Uberjar_.  An uberjar is a single Java .jar file that includes everything you need to execute your application.
This includes both the runtime components you have selected, along with the application logic.

#### What is a Health Check?

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

In our case we will implement the health check logic in a REST endpoint and let Thorntail publish
that logic on the `/health` endpoint for use with OpenShift.

**11. Add `monitor` fraction**

First, open the `pom.xml` file.

Thorntail includes the `monitor` fraction which automatically adds health check infrastructure to your
application when it is included as a fraction in the project. Open the file to insert the new dependencies
into the `pom.xml` file at the `<!-- Add monitor fraction-->` marker:

~~~java
<dependency>
    <groupId>org.wildfly.swarm</groupId>
    <artifactId>monitor</artifactId>
</dependency>
~~~

By adding the `monitor` fraction, Fabric8 will automatically add a _readinessProbe_ and _livenessProbe_ to the OpenShift
_DeploymentConfig_, published at `/health`, once deployed to OpenShift. But you still need to implement the logic behind
the health check, which you'll do next.

**12. Define Health Check Endpoint**

We are now ready to define the logic of our health check endpoint.

Create empty Java class: `src/main/java/com/redhat/coolstore/rest/HealthChecks.java` and the logic will be put into a new Java class.

Methods in this new class will be annotated with both the JAX-RS annotations as well as
[Thorntail's `@Health` annotation](https://wildfly-swarm.gitbooks.io/wildfly-swarm-users-guide/content/advanced/monitoring.html), indicating it should be used as a health check endpoint.

**13. Add health check logics**

Next, let's fill in the class by creating a new RESTful endpoint which will be used by OpenShift to probe our services.

~~~java
package com.redhat.coolstore.rest;

import javax.inject.Inject;
import javax.ws.rs.GET;
import javax.ws.rs.Path;

import com.redhat.coolstore.service.InventoryService;
import org.wildfly.swarm.health.Health;
import org.wildfly.swarm.health.HealthStatus;

@Path("/infra")
public class HealthChecks {

    @Inject
    private InventoryService inventoryService;

    @GET
    @Health
    @Path("/health")
    public HealthStatus check() {

        if (inventoryService.isAlive()) {
            return HealthStatus.named("service-state").up();
        } else {
            return HealthStatus.named("service-state").down();
        }
    }
}
~~~

The `check()` method exposes an HTTP GET endpoint which will return the status of the service. The logic of
this check does a simple query to the underlying database to ensure the connection to it is stable and available.
The method is also annotated with Thorntail's `@Health` annotation, which directs Thorntail to expose
this endpoint as a health check at `/health`.

With our new health check in place, we'll need to build and deploy the updated application in the next step.

**14. Re-Deploy to OpenShift**

With our health check in place, lets rebuild and redeploy using the same command as before via Eclipse Che **Terminal**:

`mvn fabric8:undeploy clean fabric8:deploy -Popenshift`

You should see a **BUILD SUCCESS** at the end of the build output.

To verify that everything is started, run the following command via Eclipse Che **Terminal** and wait for it report
`replication controller "inventory-xxxx" successfully rolled out`

`oc rollout status -w dc/inventory`

Back on the OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory` and then click on
the `Edit Health Checks` in `Actions`:

![inventory-healthcheck-redeploy]({% image_path inventory-healthcheck-redeploy.png %})

You should see `/health` endpoint in Readiness Path Probe and Liveness Probe:

![inventory-healthcheck-webconsole]({% image_path inventory-healthcheck-webconsole.png %})

You should also be able to access the health check logic
at the `inventory` endpoint via a web browser:

You should see a JSON response like:

~~~
{"checks": [
{"id":"service-state","result":"UP"}],
"outcome": "UP"
}
~~~

You can see the definition of the health check from the perspective of OpenShift via Eclipse Che **Terminal**:

`oc describe dc/inventory | egrep 'Readiness|Liveness'`

You should see:

~~~
    Liveness:	http-get http://:8080/health delay=180s timeout=1s period=10s #success=1 #failure=3
    Readiness:	http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
~~~

**15. Adjust probe timeout**

The various timeout values for the probes can be configured in many ways. Let's tune the _liveness probe_ initial delay so that
we don't have to wait 3 minutes for it to be activated. Use OpenShift console, Navigate to _Applications_ -> _Deployments_ -> `inventory` 
and then click on the `Edit Health Checks` in `Actions`:

![inventory-change-deplaytime]({% image_path inventory-change-deplaytime.png %})

You can also Use the **oc** command to tune the
probe to wait 30 seconds before starting to poll the probe:

`oc set probe dc/inventory --liveness --initial-delay-seconds=30`

And verify it's been changed (look at the `delay=` value for the Liveness probe) via Eclipse Che **Terminal**:

`oc describe dc/inventory | egrep 'Readiness|Liveness'`

~~~
    Liveness:	http-get http://:8080/health delay=30s timeout=1s period=10s #success=1 #failure=3
    Readiness:	http-get http://:8080/health delay=10s timeout=1s period=10s #success=1 #failure=3
~~~

In the next step we'll exercise the probe and watch as it fails and OpenShift recovers the application.

**16. Exercise Health Check**

From the OpenShift Web Console overview page, click on the route link to open the sample application UI:

![Route Link]({% image_path inventory-routelink.png %})

This will open up the sample application UI in a new browser tab:

![App UI]({% image_path app.png %})

The app will begin polling the inventory as before and report success:

![Greeting]({% image_path inventory.png %})

Now you will corrupt the service and cause its health check to start failing.
To simulate the app crasing, let's kill the underlying service so it stops responding. Execute via Eclipse Che **Terminal**:

`oc  rsh dc/inventory pkill java`

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

**17. Managing Application Configuration**

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

**18. Externalize Thorntail (Inventory) Configuration**

Thorntail supports multiple mechanisms for externalizing configurations such as environment variables, 
Maven properties, command-line arguments and more. The recommend approach for the long-term for externalizing 
configuration is however using a [YAML file](https://docs.thorntail.io/2.3.0.Final/#configuring-a-thorntail-application-using-yaml-files_thorntail) 
which you have already packaged within the Inventory Maven project.

The YAML file can be packaged within the application JAR file and be overladed 
[using command-line or system properties](https://docs.thorntail.io/2.3.0.Final/#setting-system-properties-using-the-command-line_thorntail) which you will do in this lab.

> Check out `inventory/src/main/resources/project-defaults.yml` which contains the default configuration.

Create a YAML file with the PostgreSQL database credentials. Note that you can give an arbitrary 
name to this configuration (e.g. `prod`) in order to tell Thorntail which one to use:

~~~shell
$ cat <<EOF > ./project-defaults.yml
swarm:
  datasources:
    data-sources:
      InventoryDS:
        driver-name: postgresql
        connection-url: jdbc:postgresql://inventory-postgresql:5432/inventory
        user-name: inventory
        password: mysecretpassword
EOF
~~~

> The hostname defined for the PostgreSQL connection-url corresponds to the PostgreSQL 
> service name published on OpenShift. This name will be resolved by the internal DNS server 
> exposed by OpenShift and accessible to containers running on OpenShift.

And then create a config map that you will use to overlay on the default `project-defaults.yml` which is 
packaged in the Inventory JAR archive:

~~~shell
$ oc create configmap inventory --from-file=src/main/resources/project-defaults.yml
~~~

> If you don't like bash commands, Go to the **CoolStore Inventory Microservice Project** 
> project in OpenShift Web Console and then on the left sidebar, **Resources >> Config Maps**. Click 
> on **Create Config Maps** button to create a config map with the following info:
> 
> * Name: `inventory`
> * Key: `project-defaults.yml`
> * Value: *copy-paste the content of the above project-defaults.yml excluding the first and last lines (the lines that contain EOF)*

Config maps hold key-value pairs and in the above command an `inventory` config map 
is created with `project-defaults.yml` as the key and the content of the `project-defaults.yml` as the 
value. Whenever a config map is injected into a container, it would appear as a file with the same 
name as the key, at specified path on the filesystem.

> You can see the content of the config map in the OpenShift Web Console or by 
> using `oc describe cm inventory` command.

Modify the Inventory deployment config so that it injects the YAML configuration you just created as 
a config map into the Inventory container:

~~~shell
$ oc volume dc/inventory --add --configmap-name=inventory --mount-path=/app/config
~~~

The above command mounts the content of the `inventory` config map as a file inside the Inventory container 
at `/app/config/project-defaults.yaml`

The last step is the [aforementioned system properties](https://docs.thorntail.io/2.3.0.Final/#setting-system-properties-using-the-command-line_thorntail) on the Inventory container to overlay the Thorntail configuration, using the `JAVA_ARGS` environment variable. 

> The Java runtime on OpenShift can be configured using 
> [a set of environment variables](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/reference#configuration_environment_variables) 
> to tune the JVM without the need to rebuild a new Java runtime container image every time a new option is needed.

~~~shell
$ oc set env dc/inventory JAVA_ARGS="-s /app/config/project-defaults.yml"
~~~


The Inventory pod gets restarted automatically due to the configuration changes. Wait till it's ready, 
and then verify that the config map is in fact injected into the container by running 
a shell command inside the Inventory container:

~~~shell
$ oc rsh dc/inventory cat /app/config/project-defaults.yml
~~~

Also verify that the PostgreSQL database is actually used by the Inventory service. Check the 
Inventory pod logs:

~~~shell
oc logs dc/inventory | grep hibernate.dialect

2019-05-16 04:44:51,993 INFO  [org.hibernate.dialect.Dialect] (ServerService Thread Pool -- 4) HHH000400: Using dialect: org.hibernate.dialect.PostgreSQL81Dialect
~~~

You can also connect to Inventory PostgreSQL database and check if the seed data is 
loaded into the database.

In OpenShift Web Console, navigate the left sidebar, **Applications >>Pods >>inventory-database-xxxxx**. 

![inventory-posgresql-terminal]({% image_path inventory-posgresql-terminal.png %})

Click on **Terminal** tab menu to run the following info:

~~~shell
$ psql -U inventory -c "select * from inventory"

 itemid |               link                | location | quantity
--------+-----------------------------------+----------+----------
 329299 | http://maps.google.com/?q=Raleigh | Raleigh  |      736
 329199 | http://maps.google.com/?q=Raleigh | Raleigh  |      512
 165613 | http://maps.google.com/?q=Raleigh | Raleigh  |      256
 165614 | http://maps.google.com/?q=Raleigh | Raleigh  |       54
 165954 | http://maps.google.com/?q=Raleigh | Raleigh  |       87
 444434 | http://maps.google.com/?q=Raleigh | Raleigh  |      443
 444435 | http://maps.google.com/?q=Raleigh | Raleigh  |      600
 444436 | http://maps.google.com/?q=Seoul   | Seoul    |      230
(8 rows)

$ exit
~~~

You have now created a config map that holds the configuration content for Inventory and can be updated 
at anytime for example when promoting the container image between environments without needing to 
modify the Inventory container image itself. 

#### Summary

You learned a bit more about what Thorntail is, and how it can be used to create
modern Java microservice-oriented applications.

You created a new Inventory microservice representing functionality previously implmented in the monolithic
CoolStore application. For now this new microservice is completely disconnected from our monolith and is
not very useful on its own. In future steps you will link this and other microservices into the monolith to
begin the process of [strangling the monolith](https://www.martinfowler.com/bliki/StranglerApplication.html).

Thorntail brings in a number of concepts and APIs from the Java EE community, so your existing
Java EE skills can be re-used to bring your applications into the modern world of containers,
microservices and cloud deployments.

Thorntail is one of many components of Red Hat OpenShift Application Runtimes. In the next lab
you'll use Spring Boot, another popular framework, to implement additional microservices. Let's go!
