## Getting Started with Cloud-Native Apps

The current version of the webshop is a Java EE application build for Oracle Weblogic Applications Server and as part of Coderlands modernization strategy and to be able to automate releases and eventually apply advanced deployment scenarios (like Canary releases) there has been a decision to move this application to a container native environment from Red Hat called OpenShift.

During the first planing sprint you have investigated the possibility to deploy Oracle Weblogic to Red Hat OpenShift, but since Oracle Weblogic is not supported and also not recommended to use in an orchestrated container native platform like OpenShift you have two options. Either you migrate to WebSphere Liberty Profile or you migrate to JBoss EAP. You have tested both and since both are supported on OpenShift you compare them side-by-side in this matrix.

| Charateristics                                                    | Oracle Weblogic   | JBoss EAP         |
|-------------------------------------------------------------------|-------------------|-------------------|
|Supported on OpenShift                                             | No                | Yes               |
|Supports Java EE 8 Web Profile                                     | Yes               | Yes               |
|Supports Java EE 8                                                 | Yes               | Yes               |
|Has an S2I build image                                             | No                | Yes               |
|Has a container image that is supported by the vendor              | No                | Yes               |
|Autoconfiguration of data sources and messaging                    | No                | Yes               |
|Autoconfiguration of SSO                                           | No                | Yes               |
|Capable of handling transaction recovery in an ephemeral container | Unknown           | Yes               |
|Adapted for clustering in Kubernetes (session failover, etc)       | No                | Yes               |
|Single vendor support                                              | No                | Yes               |
|Support and maintenance cost                                       | High              | Included          |
|Migration effort                                                   | High              | Low               |

After the investigation the team agrees that JBoss EAP seems to be the much better choice with better support for running your application in OpenShift, but the last question mark on “Migration cost” is worrying. You decide to contact Red Hat to try to find out what the migration cost might actually be. Red Hat recommends that you use a tool called Red Hat Application Migration Toolkit (RHAMT) which will help you analyze your application give you a report on the estimated effort to migrate and also gives you instructions on how to actually migrate the report.

#### What is Red Hat Application Migration Toolkit?

---

![RHAMT Logo]({% image_path rhamt_logo.png %})

Red Hat Application Migration Toolkit (RHAMT) is an extensible and customizable rule-based tool that helps simplify migration of Java applications.

It is used by organizations for:

* Planning and work estimation
* Identifying migration issues and providing solutions
* Detailed reporting
* Using built-in rules and migration paths
* Rule extension and customizability
* Ability to analyze source code or application archives

RHAMT examines application artifacts, including project source directories and application archives, then produces an HTML report that highlights areas needing changes. RHAMT can be used to migrate Java applications from previous versions of Red Hat JBoss Enterprise Application Platform or from other containers, such as Oracle® WebLogic Server or IBM® WebSphere® Application Server.

#### How Does Red Hat Application Migration Toolkit Simplify Migration?

---

Red Hat Application Migration Toolkit looks for common resources and highlights technologies and known trouble spots when migrating applications. The goal is to provide a high-level view into the technologies used by the application and provide a detailed report organizations can use to estimate, document, and migrate enterprise applications to Java EE and Red Hat JBoss Enterprise Application Platform.

> RHAMT is usually part of a much larger application migration and modernization program that involves well defined and repeatable phases over weeks or months and involves many people from a given business. Do not be fooled into thinking that every single
migration is a simple affair and takes an hour or less! To learn more about Red Hat's philosophy and proven methodology, check out
the [RHAMT documentation](https://access.redhat.com/documentation/en/red-hat-application-migration-toolkit) and contact your local Red Hat representative when embarking on a real world migration and modernization strategy.

##### More RHAMT Resources

* [Documentation](https://access.redhat.com/documentation/en/red-hat-application-migration-toolkit)
* [Developer Homepage](https://developers.redhat.com/products/rhamt/overview/)

You might have a question in terms of how I get started to develop a new apps in the `cloud-native way` once I complete to migreate existing apps via RHAMT.

No worries!! `Red Hat Runtimes` allows you to generate the cloud-native apps quickly on multiple rumtimes via the tool called `Launcher`.

#### What is Red Hat Runtimes Launcher?

---

LAUNCHER enables `developers` to create and/or import `modern applications`, built and deployed on OpenShift Container Platform.

![launch-landing]({% image_path launch-landing.png %})

Suppored backend Runtimes are here:

* [.NET](https://docs.microsoft.com/en-us/dotnet/core/) - Create stand-alone, production-grade .NET Core Applications that you can "just run".
* [Node.Js](https://nodejs.org/en/) - A JavaScript runtime built on Chrome's V8 JavaScript engine, using an event-driven, non-blocking I/O model for lightweight efficiency.
* [Quarkus](https://quarkus.io/) - A Kubernetes Native Java stack tailored for GraalVM and OpenJDK HotSpot, crafted from the best of breed Java libraries and standards
* [Spring Boot](https://spring.io/projects/spring-boot) - Create stand-alone, production-grade Spring based Applications that you can "just run".
* [Thorntail](https://thorntail.io/) - An innovative approach to packaging and running Java EE applications, packaging them with just enough of the server runtime to "java -jar" your application.
* [Wildfly](https://wildfly.org/) - A classic approach to packaging and running Java EE applications.
* [Vert.x](https://projects.eclipse.org/projects/rt.vertx) - A tool-kit for building reactive applications on the JVM.

Open the [Launcher](https://developers.redhat.com/launch/login). When you click on start, you will first have to login or register an account for free with the Red Hat Developer Program.

![launch-login]({% image_path launch-login.png %})

You can log in with your existing OpenShift Online, Red Hat Developer Program, or Red Hat Customer Portal account, 
or register for a Red Hat account if you don't have one already.


#### How to Create a New Application?

---

You start your own new application by picking the capabilities you want (Http Api, Persistence, ...). We take care of setting everything's up to get you started.

![launch-create-app]({% image_path launch-create-app.png %})

![launch-create-app-detail]({% image_path launch-create-app-detail.png %})

You should configure a `Frontend` and a `Backend` for your application.

When you configure a Frontend for your application then you will be able to bootstrap the frontend application in a few seconds.

![launch-create-app-frontend]({% image_path launch-create-app-frontend.png %})

> `NOTE`: Choose `None` as your Frontend Application, which will skip the selection.

Next, you select one of runtimes for your `Backend` appliction among Red Hat supported runtimes. For example, we will walk you through `Quarkus` because we will develop and deploy Quarkus application for the next labs.

![launch-create-app-backend-choose]({% image_path launch-create-app-backend-choose.png %})

Runtimes power the server-side processing of your application, and we can get you set up in one of several languages and frameworks. 
If you're looking to expose an HTTP API or interact with services like a database, choosing one here will hook that together for you.

![launch-create-app-backend-detail]({% image_path launch-create-app-backend-detail.png %})

Complete the configuration of the Backend application's capabilities(Select `Relational Persistence` and `PostgreSQL` when asked, `Health Checks`, and `HTTP API`) then click on `Save` button.

Now your application is ready to download:

![launch-create-app-download]({% image_path launch-create-app-download.png %}){:width="600"}

Extract the downloaded ZIP file then get started to developing your cloud-native applications.

Your new application contains a tool to help you deploy your new application on OpenShift. You can find instructions in the README.md.

![launch-download-app]({% image_path launch-download-app.png %}){:width="800"}

As soon as deployment is done, go checkout your new application capabilities.

We prepared a set of examples to let you directly start playing with your new application.

Those examples are there to get you started, soon it will be time for you to remove them and start developing your awesome application.

More importantly, `Fabric8 Launcher Operator` helps enabling the Launcher on an Openshift cluster for enterprise developers who
want to build the cloud-native application development tool on local machine or on-premise infrastructure. You can deploy the launcher in 5 mins
along with [Install and Configure the Fabric8 Launcher Tool](https://access.redhat.com/documentation/en-us/red_hat_openshift_application_runtimes/1/html-single/install_and_configure_the_fabric8_launcher_tool/index). 
You can also go through the [Operator Git Repository](https://github.com/fabric8-launcher/launcher-operator) to see how it works.

When you deploy the launcher by `Operator`, you will see a similar overview in OpenShift dashboard:

![launch-operator]({% image_path launch-operator.png %})

##### Note! Your Connection is not secure?

When you access OpenShift web console or the other route URL via `HTTPS` protocol, you will see `Your Connection is not secure` warning message.
Because, OpenShift uses self-certification to create TLS termication route as default. For example, if you're using `Firefox`, you will see the following screen.

Click on `Advanced > Add Exception...` then, you can access the `HTTPS` page when you click on `Confirm Security Exception`!!!

![warning]({% image_path browser_warning.png %})