## Lab4 - Breaking the monolith apart - II

In the previous lab, you learned how to take an existing monolithic app and refactor a single _inventory_ service using
Quarkus. The previous lab resulted in you creating an inventory service, but so far we haven't started
_strangling_ the monolith. That is because the inventory service is never called directly by the UI. It's a backend service
that is only used only by other backend services. In this lab, you will create the catalog service and the catalog
service will call the inventory service. When you are ready, you will change the route to tie the UI calls to new service.

To implement this, we are going to use the Spring Framework. The reason for using Spring for this service is to introduce you
to Spring Development, and how [Red Hat Runtimes](https://www.redhat.com/en/products/runtimes){:target="_blank"} helps to
make Spring development on Kubernetes easy. In real life, the reason for choosing Spring vs. others mostly depends on
personal preferences, like existing knowledge, etc. At the core Spring and Java EE are very similar.

The goal is to produce something like:

![Greeting]({% image_path catalog-goal.png %}){:width="700px"}

#### What is Spring Framework?

---

Spring is one of the most popular Java Frameworks and offers an alternative to the Java EE programming model. Spring
is also very popular for building applications based on microservices architectures. Spring Boot is a popular tool in
the Spring ecosystem that helps with organizing and using 3rd-party libraries together with Spring and also provides a
mechanism for boot strapping embeddable runtimes, like Apache Tomcat. Bootable applications (sometimes also called _fat jars_)
fits the container model very well since in a container platform like OpenShift responsibilities like starting, stopping and
monitoring applications are then handled by the container platform instead of an Application Server.

#### Aggregate microservices calls

---

Another thing you will learn in this lab is one of the techniques to aggregate services using service-to-service calls.
Other possible solutions would be to use a microservices gateway or combine services using client-side logic.

####1. Setup a Catalog proejct

---

Run the following commands to set up your environment for this lab and start in the right directory:

In the project explorer, expand the _catalog_ project.

![catalog-setup]({% image_path catalog-project.png %}){:width="500px"}

####2. Examine the Maven project structure

---

The sample project shows the components of a basic Spring Boot project laid out in different
subdirectories according to Maven best practices.

> Click on the catalog folder in the project explorer and navigate to see its folders and files.

As you can see, there are some files that we have prepared for you in the project. Under _src/main/resources/static/index.html_
we have for example prepared a simple html-based UI file for you. This matches very well what you would get if you generated an empty project from the
[Spring Initializr](https://start.spring.io){:target="_blank"} web page.

One this that differs slightly is the `pom.xml`. Please open the and examine it a bit closer (but do not change anything
at this time)

As you review the content, you will notice that there are a lot of _TODO_ comments. **Do not remove them!** These comments are used as a marker and without them, you will not be able to finish this lab.

Notice that we are not using the default BOM (Bill of material) that Spring Boot projects typically use. Instead, we are using a BOM provided by Red Hat as part of the [Snowdrop](http://snowdrop.me/){:target="_blank"} project.

~~~xml
<dependencyManagement>
<dependencies>
  <dependency>
    <groupId>me.snowdrop</groupId>
    <artifactId>spring-boot-bom</artifactId>
    <version>${spring-boot.bom.version}</version>
    <type>pom</type>
    <scope>import</scope>
  </dependency>
</dependencies>
</dependencyManagement>
~~~

We use this bill of material to make sure that we are using the version of for example Apache Tomcat that Red Hat supports.

####3. Adding web (Apache Tomcat) to the application

---

Since our applications (like most) will be a web application, we need to use a servlet container like Apache Tomcat or
Undertow. Since Red Hat offers support for Apache Tomcat (e.g., security patches, bug fixes, etc.), we will use it.

> NOTE: Undertow is another an open source project that is maintained by Red Hat and therefore Red Hat plans to
add support for Undertow shortly.

To add Apache Tomcat to our project all we have to do is to add the following lines in _pom.xml_. Open the file to automatically add these lines at the `<!-- TODO: Add web (tomcat) dependency here -->` marker:

~~~xml
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
~~~

We will also make use of Java Persistance API (JPA) so we need to add the following to _pom.xml_ at the `<!-- TODO: Add data jpa dependency here -->` marker:

~~~xml
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
~~~

We will go ahead and add a bunch of other dependencies while we have the pom.xml open. These will be explained later. Add these at the
`<!-- TODO: Add actuator, feign and hystrix dependency here -->` marker:

~~~xml
        <dependency>
          <groupId>org.springframework.boot</groupId>
          <artifactId>spring-boot-starter-actuator</artifactId>
        </dependency>

        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-feign</artifactId>
          <version>1.4.7.RELEASE</version>
        </dependency>

        <dependency>
          <groupId>org.springframework.cloud</groupId>
          <artifactId>spring-cloud-starter-hystrix</artifactId>
          <version>1.4.7.RELEASE</version>
        </dependency>
~~~

Use the command palette and select 'Build' to build and package the app using Maven to make sure the changed code still compiles:

![catalog_build]({% image_path catalog-build.png %})

If it builds successfully (you will see **BUILD SUCCESS**), you have now successfully executed the first step in this lab.

Now you've seen how to get started with Spring Boot development on Red Hat Runtimes.

In next step of this lab, we will add the logic to be able to read a data from the database.

####4. Create Domain Objects

---

Before we create the database repository class to access the data it's good practice to create test cases for the different methods that we will use.

Right-click on the `src/test/java/com.redhat.coolstore.service` package, and select _New > Java Class_. Type `ProductRepositoryTest` into the dialog box and press **OK** which will create an empty class file.

![che]({% image_path che-right-click.png %}){:width="700px"}

Replace the content of this new file with the below code:

~~~java
package com.redhat.coolstore.service;

import java.util.List;
import java.util.stream.Collectors;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.junit4.SpringRunner;

import static org.assertj.core.api.Assertions.assertThat;

import com.redhat.coolstore.model.Product;


@RunWith(SpringRunner.class)
@SpringBootTest()
public class ProductRepositoryTest {

    //TODO: Insert Catalog Component here

    //TODO: Insert test_readOne here

    //TODO: Insert test_readAll here

}
~~~

Next, inject a handle to the future repository class which will provide access to the underlying data repository. It is
injected with Spring's **@Autowired** annotation which locates, instantiates, and injects runtime instances of classes automatically,
and manages their lifecycle (much like Java EE and it's CDI feature). Add these at the `<!-- TODO: Insert Catalog Component here -->` marker:

~~~java
    @Autowired
    ProductRepository repository;
~~~

The _ProductRepository_ should provide a method called _findById(String id)_ that returns a product and collect that from the database. We test this by querying for a product with id "444434" which should have name **Pebble Smart Watch**. The pre-loaded data comes from the _src/main/resources/schema.sql_ file.

Add these at the `<!-- TODO: Insert test_readOne here -->` marker:

~~~java
    @Test
    public void test_readOne() {
        Product product = repository.findById("444434");
        assertThat(product).isNotNull();
        assertThat(product.getName()).as("Verify product name").isEqualTo("Pebble Smart Watch");
        assertThat(product.getQuantity()).as("Quantity should be ZERO").isEqualTo(0);
    }
~~~

The _ProductRepository_ should also provide a methods called _readAll()_ that returns a list of all products in the catalog. We test this by making sure that the list contains a "Red Fedora", "Forge Laptop Sticker" and "Oculus Rift".
Again, add these at the `<!-- TODO: Insert test_readAll here -->` marker:

~~~java
    @Test
    public void test_readAll() {
        List<Product> productList = repository.readAll();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> names = productList.stream().map(Product::getName).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift");
    }
~~~

####5. Implement the database repository

---

We are now ready to implement the database repository.

Use the same procedure to create a new Java Class in the `src/main/java/com/redhat/coolstore/service` package called `ProductRepository`. Replace the empty class with the following code:

~~~java
package com.redhat.coolstore.service;

import java.util.List;

import com.redhat.coolstore.model.Product;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.jdbc.core.JdbcTemplate;
import org.springframework.jdbc.core.RowMapper;
import org.springframework.stereotype.Repository;

@Repository
public class ProductRepository {

    //TODO: Autowire the jdbcTemplate here

    //TODO: Add row mapper here

    //TODO: Create a method for returning all products

    //TODO: Create a method for returning one product

}
~~~

> NOTE: This class is annotated with `@Repository`. This is a feature of Spring that makes it possible to avoid a lot of boiler plate code and only write the implementation details for this data repository. It also makes it very easy to switch to another data storage, like a NoSQL database.

Spring Data provides a convenient way for us to access data without having to write a lot of boiler plate code. One way to do that is to use a _JdbcTemplate_. First we need to autowire that as a member to _ProductRepository_. Add these at the `<!-- TODO: Autowire the jdbcTemplate here -->` marker:

~~~java
    @Autowired
    private JdbcTemplate jdbcTemplate;
~~~

The _JdbcTemplate_ require that we provide a _RowMapper_ so that it can map between rows in the query to Java Objects. We are going to define the _RowMapper_ like this.
Add these at the `<!-- TODO: Add row mapper here -->` marker:

~~~java
    private RowMapper<Product> rowMapper = (rs, rowNum) -> new Product(
            rs.getString("itemId"),
            rs.getString("name"),
            rs.getString("description"),
            rs.getDouble("price"));
~~~

Now we are ready to create the methods that are used in the test. Let's start with the `readAll()`. It should return a `List<Product>` and then we can write the query as `SELECT * FROM catalog` and use the rowMapper to map that into `Product` objects.
Add these at the `<!-- TODO: Create a method for returning all products -->` marker:

~~~java
    public List<Product> readAll() {
        return jdbcTemplate.query("SELECT * FROM catalog", rowMapper);
    }
~~~

The _ProductRepositoryTest_ also used another method called _findById(String id)_ that should return a Product. The implementation of that method using the _JdbcTemplate_ and _RowMapper_ looks like this. Add these at the `<!-- TODO: Create a method for returning one product -->` marker:

~~~java
    public Product findById(String id) {
        return jdbcTemplate.queryForObject("SELECT * FROM catalog WHERE itemId = '" + id + "'", rowMapper);
    }
~~~

The _ProductRepository_ should now have all the components, but we still need to tell spring how to connect to the database. For local development we will use the H2 in-memory database. When deploying this to OpenShift we are instead going to use the PostgreSQL database, which matches what we are using in production.

The Spring Framework has a lot of sane defaults that can always seem magical sometimes, but basically all we have to do to setup the database driver is to provide some configuration values. Open _src/main/resources/application-default.properties_ and add the following properties where the comment says `#TODO: Add database properties`.

~~~java
spring.datasource.url=jdbc:h2:mem:catalog;DB_CLOSE_ON_EXIT=FALSE
spring.datasource.username=sa
spring.datasource.password=sa
spring.datasource.driver-class-name=org.h2.Driver
~~~

The Spring Data framework will automatically see if there is a `schema.sql` in the class path and run that when initializing.

Now we are ready to run the test to verify that everything works. Right-click on the `src/test/java/com/redhat/coolstore/service` package and select _Run Test > Run JUnit Test_.

![catalog-test-run]({% image_path catalog-test-run.png %}){:width="600px"}

The test should be successful and you should see green color _test_realAll_, _test_realOne_ in Default Suite window.

![catalog-test-success]({% image_path catalog-test-success.png %}){:width="600px"}

You have now successfully executed the second step in this lab.

Now you've seen how to use Spring Data to collect data from the database and how to use a local H2 database for development and testing.

In next step of this lab, we will add the logic to expose the database content from REST endpoints using JSON format.

####6. Create Catalog Service

---

Now you are going to create a service class. Later on the service class will be the one that controls the interaction with the inventory service, but for now it's basically just a wrapper of the repository class.

Again, create a new class `CatalogService` in the `src/main/java/com/redhat/coolstore/service` package.

Replace the empty class with this code:

~~~java
package com.redhat.coolstore.service;

import java.util.List;
import java.util.stream.Collectors;
import java.util.stream.IntStream;

//import com.redhat.coolstore.client.InventoryClient;
import com.redhat.coolstore.model.Product;

import org.json.JSONArray;
import org.json.JSONObject;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Service;

@Service
public class CatalogService {

    @Autowired
    private ProductRepository repository;

    //TODO: Autowire Inventory Client

    public Product read(String id) {
        Product product = repository.findById(id);
        //TODO: Update the quantity for the product by calling the Inventory service
        return product;
    }

    public List<Product> readAll() {
        List<Product> productList = repository.readAll();
        //TODO: Update the quantity for the products by calling the Inventory service
        return productList;
    }

}
~~~

As you can see there is a number of `TODO` in the code, and later we will use these placeholders to add logic for calling the Inventory Client to get the quantity.

Now we are ready to create the endpoints that will expose REST service. Let's again first start by creating a test case for our endpoint. We need two endpoints, one that exposes for GET calls to `/services/products` that will return all product in the catalog as JSON array, and the second one exposes GET calls to `/services/produc/{prodId}` which will return a single Product as a JSON Object. Let's again start by creating a test case.

Create the test case by creating a new class file called `CatalogEndpointTest` in the `src/test/java/com/redhat/coolstore/service` package.

Add the following code to the test case and make sure to *review* it without any codes change so that you understand how it works.

~~~java
package com.redhat.coolstore.service;

import com.redhat.coolstore.model.Inventory;
import com.redhat.coolstore.model.Product;
import io.specto.hoverfly.junit.rule.HoverflyRule;
import org.junit.ClassRule;
import org.junit.Test;
import org.junit.runner.RunWith;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.boot.test.web.client.TestRestTemplate;
import org.springframework.core.ParameterizedTypeReference;
import org.springframework.http.HttpMethod;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.test.context.junit4.SpringRunner;

import java.util.List;
import java.util.concurrent.TimeUnit;
import java.util.stream.Collectors;

import static io.specto.hoverfly.junit.dsl.HttpBodyConverter.json;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.success;
import static io.specto.hoverfly.junit.dsl.ResponseCreators.serverError;
import static io.specto.hoverfly.junit.dsl.matchers.HoverflyMatchers.startsWith;
import static org.assertj.core.api.Assertions.assertThat;
import static io.specto.hoverfly.junit.core.SimulationSource.dsl;
import static io.specto.hoverfly.junit.dsl.HoverflyDsl.service;

@RunWith(SpringRunner.class)
@SpringBootTest(webEnvironment = SpringBootTest.WebEnvironment.RANDOM_PORT)
public class CatalogEndpointTest {

    @Autowired
    private TestRestTemplate restTemplate;

    //TODO: Add ClassRule for HoverFly Inventory simulation

    @Test
    public void test_retriving_one_proudct() {
        ResponseEntity<Product> response
                = restTemplate.getForEntity("/services/product/329199", Product.class);
        assertThat(response.getStatusCode()).isEqualTo(HttpStatus.OK);
        assertThat(response.getBody())
                .returns("329199",Product::getItemId)
                .returns("Forge Laptop Sticker",Product::getName)
    //TODO: Add check for Quantity
                .returns(8.50,Product::getPrice);
    }


    @Test
    public void check_that_endpoint_returns_a_correct_list() {

        ResponseEntity<List<Product>> rateResponse =
                restTemplate.exchange("/services/products",
                        HttpMethod.GET, null, new ParameterizedTypeReference<List<Product>>() {
                        });

        List<Product> productList = rateResponse.getBody();
        assertThat(productList).isNotNull();
        assertThat(productList).isNotEmpty();
        List<String> names = productList.stream().map(Product::getName).collect(Collectors.toList());
        assertThat(names).contains("Red Fedora","Forge Laptop Sticker","Oculus Rift");

        Product fedora = productList.stream().filter( p -> p.getItemId().equals("329299")).findAny().get();
        assertThat(fedora)
                .returns("329299",Product::getItemId)
                .returns("Red Fedora", Product::getName)
    //TODO: Add check for Quantity
                .returns(34.99,Product::getPrice);
    }

}
~~~

Now we are ready to implement the _CatalogEndpoint_.

Start by creating a new class called `CatalogEndpoint` in the `src/main/java/com/redhat/coolstore/service` package.

Replace the contents with this code:

~~~java
package com.redhat.coolstore.service;

import java.util.List;

import com.redhat.coolstore.model.Product;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.stereotype.Controller;
import org.springframework.web.bind.annotation.*;

@Controller
@RequestMapping("/services")
public class CatalogEndpoint {

    @Autowired
    private CatalogService catalogService;

    @ResponseBody
    @GetMapping("/products")
    public ResponseEntity<List<Product>> readAll() {
        return new ResponseEntity<List<Product>>(catalogService.readAll(),HttpStatus.OK);
    }

    @ResponseBody
    @GetMapping("/product/{id}")
    public ResponseEntity<Product> read(@PathVariable("id") String id) {
        return new ResponseEntity<Product>(catalogService.read(id),HttpStatus.OK);
    }

}
~~~

The Spring MVC Framework default uses Jackson to serialize or map Java objects to JSON and vice versa. Because Jackson extends upon JAX-B and does can automatically parse simple Java structures and parse them into JSON and vice verse and since our `Product.java` is very simple and only contains basic attributes we do not need to tell Jackson how to parse between Product and JSON.

Now you can run the _CatalogEndpointTest_ and verify that it works via **Run Junit Test**. Right-click on the `CatalogEndpointTest` and select _Run Test > Run JUnit Test_.

![catalog-endpoint-test-run]({% image_path catalog-endpoint-test-run.png %}){:width="700px"}

The test should be successful and you should see green color _test_retriving_one_proudct_, _check_that_endpoint_returns_a_correct_list<> in Default Suite window.

![catalog-endpoint-test-success]({% image_path catalog-endpoint-test-success.png %})

You can also run the following command via `CodeReady Workspaces Terminal` to verify the test cases.

`cd /projects/cloud-native-workshop-v2m1-labs/catalog/`

`mvn verify -Dtest=CatalogEndpointTest`

Since we now have endpoints that returns the catalog we can also start the service and load the default page again, which should now return the products.

Start the application via CodeReady Workspaces **RUN** Menu:

![catalog-spring-run]({% image_path catalog-spring-run.png %})

Wait for the application to start. Then we can verify the endpoint by running the following command in Eclipse Terminal:

`curl http://localhost:8081/services/products | jq`

You should get a full JSON array consisting of all the products:

~~~json
  {
    "itemId": "329299",
    "name": "Red Fedora",
    "desc": "Official Red Hat Fedora",
    "price": 34.99,
    "quantity": 0
  },
  { ... }
~~~

You have now successfully executed the third step in this lab.

Now you've seen how to create REST application in Spring MVC and create a simple application that returns product.

In the next step, we will also call another service to enrich the endpoint response with inventory status.

> NOTE: Make sure to stop the service by closing `run spring-boot` tab window in CodeReady Workspace.

####7. Get inventory data

---

So far our application has been kind of straight forward, but our monolith code for the catalog is also returning the inventory status. In the monolith since both the inventory data and catalog data are in the same database we used a `OneToOne` mapping in JPA like this:

~~~java
@OneToOne(cascade = CascadeType.ALL,fetch=FetchType.EAGER)
@PrimaryKeyJoinColumn
private InventoryEntity inventory;
~~~

When redesigning our application to Microservices using domain driven design we have identified that Inventory and Product Catalog are two separate domains. However our current UI expects to retrieve data from both the Catalog Service and Inventory service in a singe request.

####Service interaction

Our problem is that the user interface requires data from two services when calling the REST service on `/services/products`. There are multiple ways to solve this like:

**I. Client Side integration** - We could extend our UI to first call `/services/products` and then for each product item call `/services/inventory/{prodId}` to get the inventory status and then combine the result in the web browser. This would be the least intrusive method, but it also means that if we have 100 of products the client will make 101 request to the server. If we have a slow internet connection this may cause issues.

**II. Microservices Gateway** - Creating a gateway in-front of the `Catalog Service` that first calls the Catalog Service and then based on the response calls the inventory is another option. This way we can avoid lots of calls from the client to the server. [Apache Camel](http://camel.apache.org){:target="_blank"} provides nice capabilities to do this and if you are interested to learn more about this, please checkout the Coolstore Microservices example: [Here](http://github.com/jbossdemocentral/coolstore-microservice){:target="_blank"}

**III. Service-to-Service** - Depending on use-case and preferences another solution would be to do service-to-service calls instead. In our case means that the Catalog Service would call the Inventory service using REST to retrieve the inventory status and include that in the response.

There are no right or wrong answers here, but since this is a workshop on application modernization using Red Hat Runtimes we will not choose option I or II here. Instead we are going to use option III and extend our Catalog to call the Inventory service.

####8. Extending the test

---

In the [Test-Driven Development](https://en.wikipedia.org/wiki/Test-driven_development){:target="_blank"} style, let's first extend our test to test the Inventory functionality (which doesn't exist).

Open _src/test/java/com/redhat/coolstore/service/CatalogEndpointTest.java_ again.

Now at the markers **//TODO: Add check for Quantity** add the following line:

`                .returns(9999,Product::getQuantity)`

And add it to the second test as well at the remaining _//TODO: Add check for Quantity_ marker:

`                .returns(9999,Product::getQuantity)`

Now you can run the _CatalogEndpointTest_ and verify that it **fails** via _Run Junit Test_:

![catalog-endpoint-test-run]({% image_path catalog-endpoint-test-run.png %}){:width="700px"}

The test _should fail_ and you should see red color **test_retriving_one_proudct**, **check_that_endpoint_returns_a_correct_list** in Default Suite window.

![catalog-endpoint-test-failure]({% image_path catalog-endpoint-test-failure.png %})

The test fails because we are trying to call the Inventory service which is not runninmg.

We will soon implement the code to call the inventory service, but first
we need a away to test this service without having to rely on the inventory services to be up and running. For that we are going to use an API Simulator
called [HoverFly](http://hoverfly.io){:target="_blank"} and particular it's capability to simulate remote APIs. HoverFly is very convenient to use with Unit test and all we have to do is
to add a **ClassRule** that will simulate all calls to inventory. Open the file to insert the
code at the `//TODO: Add ClassRule for HoverFly Inventory simulation` marker in _CatalogEndpointTest_ class:

~~~java
    @ClassRule
    public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
            service("inventory:8080")
    //                    .andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")
                    .get(startsWith("/services/inventory"))
    //                    .willReturn(serverError())
                    .willReturn(success("[{\"itemId\":\"329199\",\"quantity\":9999}]", "application/json"))

    ));
~~~

This _ClassRule_ means that if our tests are trying to call our inventory url, HoverFly will intercept this and respond with our hard coded response instead.

We will soon use the `// commented-out` lines, so keep them in there!

####9. Implementing the Inventory Client

 ---

Since we now have a nice way to test our service-to-service interaction we can now create the client that calls the Inventory. Netflix has provided some nice extensions to the Spring Framework that are mostly captured in the Spring Cloud project, however Spring Cloud is mainly focused on Pivotal Cloud Foundry and because of that Red Hat and others have contributed Spring Cloud Kubernetes to the Spring Cloud project, which enables the same functionallity for Kubernetes based platforms like OpenShift.

The inventory client will use a Netflix project called _Feign_, which provides a nice way to avoid having to write boilerplate code. Feign also integrate with Hystrix which gives us capability to Circute Break calls that doesn't work. We will discuss this more later, but let's start with the implementation of the Inventory Client. Using Feign all we have todo is to create a interface that details which parameters and return type we expect, annotate it with    `@RequestMapping` and provide some details and then annotate the interface with `@Feign` and provide it with a name.

Create the `InventoryClient` class in the `src/main/java/com/redhat/coolstore/client/` package in the project explorer.

Add the following code to the file:

~~~java
package com.redhat.coolstore.client;

import feign.hystrix.FallbackFactory;
import org.springframework.http.MediaType;
import org.springframework.stereotype.Component;
import org.springframework.cloud.netflix.feign.FeignClient;
import org.springframework.web.bind.annotation.PathVariable;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RequestMethod;

@FeignClient(name="inventory")
public interface InventoryClient {

    @RequestMapping(method = RequestMethod.GET, value = "/services/inventory/{itemId}", consumes = {MediaType.APPLICATION_JSON_VALUE})
    String getInventoryStatus(@PathVariable("itemId") String itemId);

    //TODO: Add Fallback factory here

}
~~~

There is one more thing that we need to do which is to tell Feign where the inventory service is running. Before that notice that we are setting the `@FeignClient(name="inventory")`.

Open the `src/main/resources/application-default.properties file.

Add these properties to it at the `#TODO: Configure netflix libraries` marker:

~~~java
inventory.ribbon.listOfServers=inventory:8080
feign.hystrix.enabled=true
~~~

By setting _inventory.ribbon.listOfServers_ we are hard coding the actual URL of the service to **inventory:8080**. If we had multiple servers we could also add those using a comma. However using Kubernetes there is no need to have multiple endpoints listed here since Kubernetes has a concept of _Services_ that will internally route between multiple instances of the same service. Later on we will update this value to reflect our URL when deploying to OpenShift.


Now that we have a client we can make use of it in our _CatalogService_.

Open _src/main/java/com/redhat/coolstore/service/CatalogService.java_

And autowire (e.g. inject) the client into it by inserting this at the `//TODO: Autowire Inventory Client` marker:

~~~java
    @Autowired
    InventoryClient inventoryClient;
~~~

Next, update the _read(String id)_ method at the comment `//TODO: Update the quantity for the product by calling the Inventory service` add the following:

~~~java
        JSONArray jsonArray = new JSONArray(inventoryClient.getInventoryStatus(product.getItemId()));
        List<String> quantity = IntStream.range(0, jsonArray.length())
            .mapToObj(index -> ((JSONObject)jsonArray.get(index))
            .optString("quantity")).collect(Collectors.toList());
        product.setQuantity(Integer.parseInt(quantity.get(0)));
~~~

Also, don't forget to add the import statement by un-commenting the import statement **//import com.redhat.coolstore.client.InventoryClient** near the top

~~~java
import com.redhat.coolstore.client.InventoryClient;
~~~

Also in the _readAll()_ method replace the comment `//TODO: Update the quantity for the products by calling the Inventory service` with the following:

~~~java
        productList.forEach(p -> {
            JSONArray jsonArray = new JSONArray(inventoryClient.getInventoryStatus(p.getItemId()));
            List<String> quantity = IntStream.range(0, jsonArray.length())
                .mapToObj(index -> ((JSONObject)jsonArray.get(index))
                .optString("quantity")).collect(Collectors.toList());
            p.setQuantity(Integer.parseInt(quantity.get(0)));
        });
~~~

> NOTE: Class `JSONArray` is an ordered sequence of values. Its external text form is a string wrapped in square brackets with commas separating the values. The internal form is an object having get and opt methods for accessing the values by index, and element methods for adding or replacing values.

Now you can run the _CatalogEndpointTest_ and verify that it works via **Run Junit Test**:

![catalog-endpoint-test-run]({% image_path catalog-endpoint-test-run.png %}){:width="700px"}

The test should be successful and you should see green color **test_retriving_one_proudct**, **check_that_endpoint_returns_a_correct_list** in Default Suite window.

![catalog-endpoint-test-success]({% image_path catalog-endpoint-test-success.png %})

So even if we don't have any inventory service running we can still run our test. However to actually run the service using `mvn spring-boot:run` we need to have an inventory service or the calls to `/services/products/` will fail. We will fix this in the next step.

#####Congratulations!
You now have the framework for retrieving products from the product catalog and enriching the data with inventory data from
an external service. But what if that external inventory service does not respond? That's the topic for the next step.


####10. Create a fallback for inventory

---

In the previous step we added a client to call the Inventory service. Services calling services is a common practice in Microservices Architecture, but as we add more and more services the likelihood of a problem increases dramatically. Even if each service has 99.9% update, if we have 100 of services our estimated up time will only be ~90%. We therefor need to plan for failures to happen and our application logic has to consider that dependent services are not responding.

In the previous step we used the Feign client from the Netflix cloud native libraries to avoid having to write
boilerplate code for doing a REST call. However Feign also have another good property which is that we easily create
fallback logic. In this case we will use static inner class since we want the logic for the fallback to be part of the
Client and not in a separate class.

Open: _src/main/java/com/redhat/coolstore/client/InventoryClient.java_

And paste this into it at the `//TODO: Add Fallback factory here` marker:

~~~java
    //TODO: Add Callback Factory Component
    @Component
    static class InventoryClientFallbackFactory implements FallbackFactory<InventoryClient> {
        @Override
        public InventoryClient create(Throwable cause) {
            return new InventoryClient() {
                @Override
                public String getInventoryStatus(@PathVariable("itemId") String itemId) {
                    return "[{'quantity':-1}]";
                }
            };
        }
    }
~~~

After creating the fallback factory all we have todo is to tell Feign to use that fallback in case of an issue, by adding the fallbackFactory property to the `@FeignClient` annotation. and replace the existing `@FeignClient(name="inventory")` line with this line:

~~~java
@FeignClient(name="inventory",fallbackFactory = InventoryClient.InventoryClientFallbackFactory.class)
~~~

####11. Test the Fallback

---

Now let's see if we can test the fallback. Optimally we should create a different test that fails the request and then verify the fallback value, however because we are limited in time we are just going to change our test so that it returns a server error and then verify that the test fails.

Open _src/test/java/com/redhat/coolstore/service/CatalogEndpointTest.java_ and change the following lines:

~~~
@ClassRule
public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
        service("inventory:8080")
//                    .andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")
                .get(startsWith("/services/inventory"))
//                    .willReturn(serverError())
                .willReturn(success("[{\"itemId\":\"329199\",\"quantity\":9999}]", "application/json"))

));
~~~

TO

~~~
@ClassRule
public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
        service("inventory:8080")
//                    .andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")
                .get(startsWith("/services/inventory"))
                .willReturn(serverError())
//                    .willReturn(success("[{\"itemId\":\"329199\",\"quantity\":9999}]", "application/json"))

));
~~~

Notice that the Hoverfly Rule will now return `serverError` for all requests to inventory.

Now you can run the _CatalogEndpointTest_ and verify that it **fails** via **Run Junit Test**:

![catalog-endpoint-test-run]({% image_path catalog-endpoint-test-run.png %}){:width="700px"}

The test _should fail_ and you _should see red color `test_retriving_one_proudct_, _check_that_endpoint_returns_a_correct_list` in Default Suite window.

![catalog-endpoint-test-failure]({% image_path catalog-endpoint-test-failure.png %})

So since even if our inventory service fails we are still returning inventory quantity -1. The test fails because we are expecting the quantity to be 9999.

Change back the class rule by re-commenting out the _.willReturn(serverError())_ line so that we don't fail the tests like this:

~~~
@ClassRule
public static HoverflyRule hoverflyRule = HoverflyRule.inSimulationMode(dsl(
        service("inventory:8080")
//                    .andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")
                .get(startsWith("/services/inventory"))
//                    .willReturn(serverError())
                .willReturn(success("[{\"itemId\":\"329199\",\"quantity\":9999}]", "application/json"))

));
~~~

Make sure the test works again by re-running the `CatalogEndpointTest` JUnit Test.

####12. Slow running services

---

Having fallbacks is good but that also requires that we can correctly detect when a dependent services isn't responding correctly. Besides from not responding a service can also respond slowly causing our services to also respond slow. This can lead to cascading issues that is hard to debug and pinpoint issues with. We should therefore also have sane defaults for our services. You can add defaults by adding it to the configuration.

Open _src/main/resources/application-default.properties_

And add this line to it at the **#TODO: Set timeout to for inventory to 500ms** marker:

~~~java
hystrix.command.inventory.execution.isolation.thread.timeoutInMilliseconds=500
~~~

Open _src/test/java/com/redhat/coolstore/service/CatalogEndpointTest.java_ and un-comment the **.andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")**

Now you can run the _CatalogEndpointTest_ and verify that it **fails** via **Run Junit Test**:

![catalog-endpoint-test-run]({% image_path catalog-endpoint-test-run.png %}){:width="700px"}

The test _should fail_ and you should see red color `test_retriving_one_proudct`, `check_that_endpoint_returns_a_correct_list` in Default Suite window.

![catalog-endpoint-test-failure]({% image_path catalog-endpoint-test-failure.png %})

This shows that the timeout works nicely. However, since we want our test to be successful **you should now comment out `.andDelay(2500, TimeUnit.MILLISECONDS).forMethod("GET")`** again and then verify that the test works by re-running the JUnit test.

####Congratulations!
You have now successfully executed the fourth step in this lab.
In this step you've learned how to add Fallback logic to your class and how to add timeout to service calls.
In the next step we now test our service locally before we deploy it to OpenShift.

####13. Test Locally

---

As you have seen in previous steps, using the Spring Boot maven plugin (predefined in _pom.xml_), you can conveniently run the application locally and test the endpoint.

Start the application via CodeReady Workspaces **RUN** Menu:

![catalog-spring-run]({% image_path catalog-spring-run.png %})

Wait for the application to start. Then we can verify the endpoint by running the following command in Eclipse Terminal:

`curl http://localhost:8081/services/product/329299 ; echo`

You would see a JSON response like this:

~~~json
{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":-1}%
~~~

> NOTE: Since we do not have an inventory service running locally the value for the quantity is -1, which matches the fallback value that we have configured.

The REST API returned a JSON object representing the inventory count for this product. Well done!

> NOTE: Make sure to stop the service by closing run spring-boot tab window in CodeReady Workspace.

You have now successfully created your the Catalog service using Spring Boot and implemented basic REST
API on top of the product catalog database. You have also learned how to deal with service failures.

In next step of this lab we will deploy our application to OpenShift Container Platform and then start
adding additional features to take care of various aspects of cloud native microservice development.

####14. Create the OpenShift project

---

We have already deployed our coolstore monolith and inventory to OpenShift. In this step we will deploy our new Catalog microservice for our CoolStore application,
so let's create a separate project to house it and keep it separate from our monolith and our other microservices.

Click on the name of the **userXX-catalog** project:

![create_new]({% image_path create_new_catalog.png %})

This will take you to the project overview. There's nothing there yet, but that's about to change.

Next, we'll deploy your new microservice to OpenShift.

####15. Deploy to OpenShift

---

Now that you've logged into OpenShift, let's deploy our new catalog microservice:

Our production catalog microservice will use an external database (PostgreSQL) to house inventory data.
First, deploy a new instance of PostgreSQL by executing via CodeReady Workspaces Terminal:

`oc project userXX-catalog`

~~~shell
oc new-app -e POSTGRESQL_USER=catalog \
             -e POSTGRESQL_PASSWORD=mysecretpassword \
             -e POSTGRESQL_DATABASE=catalog \
             openshift/postgresql:10 \
             --name=catalog-database
~~~

This will deploy the database to our new project.

![catalog_posgresql]({% image_path catalog_posgresql.png %})

You can also check if the deployment is complete via CodeReady Workspaces Terminal:

`oc rollout status -w dc/catalog-database`

####16. Update configuration

---

Open the file _src/main/resources/application-default.properties_ in CodeReady Workspace.

Comment the local variables and add a remote variables. You can replace the whole contents with the following variables to the file:

 * You have to replace **userXX** with your username in `inventory.ribbon.listOfServers=inventory-quarkus.userXX-inventory.svc.cluster.local:8080`.

~~~java
# Tomcat port - To avoid port conflict we set this to 8081 in the local environment
#server.port=8081

#TODO: Add database properties
#spring.datasource.url=jdbc:h2:mem:catalog;DB_CLOSE_ON_EXIT=FALSE
#spring.datasource.username=sa
#spring.datasource.password=sa
#spring.datasource.driver-class-name=org.h2.Driver

#TODO: Configure netflix libraries
#inventory.ribbon.listOfServers=inventory:8080
#feign.hystrix.enabled=true

#TODO: Set timeout to for inventory to 500ms
hystrix.command.inventory.execution.isolation.thread.timeoutInMilliseconds=500

server.port=8080
spring.datasource.url=jdbc:postgresql://catalog-database:5432/catalog
spring.datasource.username=catalog
spring.datasource.password=mysecretpassword
spring.datasource.driver-class-name=org.postgresql.Driver

inventory.ribbon.listOfServers=inventory-quarkus.userXX-inventory.svc.cluster.local:8080
~~~

![catalog_posgresql]({% image_path catalog_changed_properties.png %})

####17. Build and Deploy

---

Build and deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m1-labs/catalog/`

`mvn clean package spring-boot:repackage -DskipTests`

The build and deploy may take a minute or two. Wait for it to complete. You should see a **BUILD SUCCESS** at the
end of the build output.

Then deploy the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces Terminal:

`oc new-build registry.access.redhat.com/redhat-openjdk-18/openjdk18-openshift:1.5 --binary --name=catalog-springboot -l app=catalog-springboot`

This build uses the new [Red Hat OpenJDK Container Image](https://access.redhat.com/documentation/en-us/red_hat_jboss_middleware_for_openshift/3/html/red_hat_java_s2i_for_openshift/index){:target="_blank"}, providing foundational software needed to run Java applications, while staying at a reasonable size.

And then start and watch the build, which will take about a minute to complete:

`oc start-build catalog-springboot --from-file=target/catalog-1.0.0-SNAPSHOT.jar --follow`

Once the build is done, we'll deploy it as an OpenShift application:

`oc new-app catalog-springboot`

and expose your service to the world:

`oc expose service catalog-springboot`

Finally, make sure it's actually done rolling out:

`oc rollout status -w dc/catalog-springboot`

Wait for that command to report replication controller "catalog-springboot-1" successfully rolled out before continuing.

> NOTE: Even if the rollout command reports success the application may not be ready yet and the reason for that is that we currently don't have any liveness check configured, but we will add that in the next steps.

And now we can access using curl once again to find a certain inventory:

`export URL="http://$(oc get route | grep catalog | awk '{print $2}')"`

`curl $URL/services/product/329299 ; echo`

The expected result data is here:

`{"itemId":"329299","name":"Red Fedora","desc":"Official Red Hat Fedora","price":34.99,"quantity":736}`

> **NOTE** if you do not get the expected output, make sure you replaced `userXX` in the `application-default.properties` file! If you forgot to do this, go back and make the change and re-build using the previous `mvn` command and re-deploy to OpenShift with the previous `oc start-build` command.

So now **Catalog** service is deployed to OpenShift. You can also see it in the Project Status in the OpenShift Console
with running in 1 pod, along with the Postgres database pod.

####18. Access the application running on OpenShift

---

This sample project includes a simple UI that allows you to access the Inventory API. This is the same
UI that you previously accessed outside of OpenShift which shows the CoolStore inventory. Click on the
route URL at **Networking > Routes** in [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"} to access the sample UI.

> You can also access the application through the link on Resources tab in the Project Status page.

![catalog-route-link]({% image_path catalog-route-link.png %})

The UI will refresh the catalog table every 2 seconds, as before.

![catalog-sample-ui]({% image_path catalog-sample-ui.png %})

> NOTE: Since we previously have a inventory service running you should now see the actual quantity value and not the fallback value of -1.

`Congratulations!` You have deployed the Catalog service as a microservice which in turn calls into the Inventory service to retrieve inventory data.

####19. Strangling the monolith

---

So far we haven't started [strangling the monolith](https://www.martinfowler.com/bliki/StranglerApplication.html){:target="_blank"}. To do this we are going to make use of routing capabilities in OpenShift. Each external request coming into OpenShift (unless using ingress, which we are not) will pass through a route. In our monolith the web page uses client side REST calls to load different parts of pages.

For the home page the product list is loaded via a REST call to *http://<monolith-hostname>/services/products*. At the moment calls to that URL will still hit product catalog in the monolith. Now we will route these calls to our newly created catalog services instead and end up with something like:

![Greeting]({% image_path catalog-goal.png %}){:width="700px"}

Flow the steps below to create **Cross-origin resource sharing (CORS)** based route. CORS is a mechanism that allows restricted resources on a web page to be requested from another domain outside the domain from which the first resource was served.

Create **CORSProvider** class in _src/main/java/com/redhat/coolstore_ of **inventory** project to allow restricted resources on a _catalog_ service. Copy the following all codes in the class:

~~~java
package com.redhat.coolstore;

import org.jboss.resteasy.plugins.interceptors.CorsFilter;

import javax.ws.rs.core.Feature;
import javax.ws.rs.core.FeatureContext;
import javax.ws.rs.ext.Provider;

@Provider
public class CORSProvider implements Feature {
    @Override
    public boolean configure(FeatureContext context) {
        CorsFilter filter = new CorsFilter();
        filter.getAllowedOrigins().add("*");
        filter.setAllowedMethods("GET, POST, DELETE, OPTIONS, HEAD");
        filter.setAllowedHeaders("accept, content-type, origin");
        context.register(filter);
        return true;
    }
}
~~~

Repackage the **inventory** application via clicking on **Package for OpenShift** in Commands Palette:

![codeready-workspace-maven]({% image_path quarkus-dev-run-packageforOcp.png %})

Restart and watch the build, which will take about a minute to complete. Replace your username with **userXX**:

`cd /projects/cloud-native-workshop-v2m1-labs/inventory/`

`oc start-build inventory-quarkus --from-file target/*-runner.jar --follow -n userXX-inventory`

Once the build is done, the inventory pod will be deployed automatically via DeploymentConfig Trigger in OpenShift.

Open **CatalogEndpoint** class in _src/main/java/com/redhat/coolstore/service_ of **catalog** project to allow restricted resources on a _product_ page of the monolith application. Add *@CrossOrigin* annotation on _CatalogEndpoint_ class:

~~~java
@Controller
@CrossOrigin
@RequestMapping("/services")
~~~

Repackage the project using the following command, which will use the maven plugin to deploy via CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m1-labs/catalog/`

`mvn clean package spring-boot:repackage -DskipTests`

The build and deploy may take a minute or two. Wait for it to complete. You should see a **BUILD SUCCESS** at the
end of the build output.

Restart and watch the build, which will take about a minute to complete. Replace your username with **userXX**:

`oc start-build catalog-springboot --from-file=target/catalog-1.0.0-SNAPSHOT.jar --follow -n userXX-catalog`

Once the build is done, the catalog pod will be deployed automatically via DeploymentConfig Trigger in OpenShift.

Let's update the catalog endpoint in monolith application. Copy the route URL of catalog service using following **oc** command in CodeReady Workspaces Terminal. Replace your username with **userXX**:

`echo "http://$(oc get route -n userXX-catalog | grep catalog | awk '{print $2}')"`

In the **monolith** project, open `catalog.js` in `src/main/webapp/app/services` and add a line as shown in the image to define the value of `baseUrl`.

`baseUrl="YOUR_CATALOG_ROUTE_URL/services/products";`

> Replace `YOUR_CATALOG_ROUTE_URL` with the URL emitted from the previous `echo` command

![strangler]({% image_path catalog_js_strangler.png %})

Rebuild the project in CodeReady Workspaces Terminal:

`cd /projects/cloud-native-workshop-v2m1-labs/monolith/`

`mvn clean package -Popenshift`

Wait for the build to finish and the `BUILD SUCCESS` message!

Restart and watch the build, which will take about a minute to complete. Replace your username with **userXX**:

`oc start-build coolstore --from-file=deployments/ROOT.war --follow -n userXX-coolstore-dev`

Once the build is done, the coolstore pod will be deployed automatically via DeploymentConfig Trigger in OpenShift. Ensure it's rolled out:

`oc rollout status -w dc/coolstore -n userXX-coolstore-dev` (replace `userXX` with your username)

####20. Test the UI

---

Open the monolith UI at by selecting the `userXX-coolstore-dev` project in the [OpenShift web console]({{ CONSOLE_URL}}){:target="_blank"}, navigate to _Networking > Routes_ and click on the link to the monolith UI.

Observe that the new catalog is being used along with the monolith:

![Greeting]({% image_path coolstore-web.png %})

The screen will look the same, but notice that the earlier product *Atari 2600 Joystick* is now gone,
as it has been removed in our new catalog microservice.

> Note: If the web page is still same then you should clean cookies and caches in your web browser.

#####Congratulations!
You have now successfully begun to _strangle_ the monolith. Part of the monolith's functionality (Inventory and Catalog) are
now implemented as microservices.

##### Summary

---

In this lab you learned a bit more about what Spring Boot and how it can be used together with OpenShift and OpenShift
Kubernetes.

You created a new product catalog microservice representing functionality previously implemented in the monolithic
CoolStore application. This new service also communicates with the inventory service to retrieve the inventory status
for each product.
