## Getting Started with Cloud-Native Apps

In this module you'll work with an existing Java EE application for a retail webshop and migrate it and modernize it.
The current version of the webshop is a Java EE application built for Oracle Weblogic Application Server and as part of a modernization strategy and to be able to automate releases and eventually apply advanced deployment scenarios (like Canary or Blue/Green deployments) there has been a decision to move this application to a container native environment from Red Hat called OpenShift.

During the first planning sprint you have investigated the possibility to deploy Oracle Weblogic to Red Hat OpenShift, but since Oracle Weblogic is limited in support and also not recommended to use in an orchestrated container native platform like OpenShift you have two options. Either you migrate to WebSphere Liberty Profile or you migrate to JBoss EAP. You have tested both and since both are supported on OpenShift you compare them side-by-side in this matrix.

| Charateristics                                                    | Oracle Weblogic   | JBoss EAP         |
|-------------------------------------------------------------------|-------------------|-------------------|
|Supported on OpenShift                                             | Limited           | Yes               |
|Supports Java EE 8 Web Profile                                     | Yes               | Yes               |
|Supports Java EE 8                                                 | Yes               | Yes               |
|Has an S2I build image                                             | No                | Yes               |
|Has a container image that is supported by the vendor              | No                | Yes               |
|Autoconfiguration of data sources and messaging                    | No                | Yes               |
|Autoconfiguration of SSO                                           | No                | Yes               |
|Capable of handling transaction recovery in an ephemeral container | Unknown           | Yes               |
|Adapted for clustering in Kubernetes (session failover, etc)       | No                | Yes               |
|Single vendor support                                              | Yes               | Yes               |
|Support and maintenance cost                                       | High              | Included          |
|Migration effort                                                   | High              | Low               |

After the investigation the team agrees that JBoss EAP seems to be the much better choice with better support for running your application in OpenShift, but the last question mark on “Migration cost” is worrying. You decide to contact Red Hat to try to find out what the migration cost might actually be. Red Hat uses a tool called Red Hat Application Migration Toolkit (RHAMT) which help customers analyze their applications and report on the estimated effort to migrate and also provides detailed instructions on how to actually migrate the code.

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
the [RHAMT documentation](https://access.redhat.com/documentation/en/red-hat-application-migration-toolkit){:target="_blank"} and contact your local Red Hat representative when embarking on a real world migration and modernization strategy.

##### More RHAMT Resources

* [Documentation](https://access.redhat.com/documentation/en/red-hat-application-migration-toolkit){:target="_blank"}
* [Developer Homepage](https://developers.redhat.com/products/rhamt/overview/){:target="_blank"}

#### What comes after migration?

---

You might wonder about developing **new** cloud-native apps once you complete your migration of existing apps via RHAMT.

No worries!! Red Hat Runtimes allows you to get started building cloud-native apps quickly on multiple rumtimes via the it's **Launcher**.
The Launcher enables developers to create and/or import modern applications, built and deployed on OpenShift Container Platform.

![launch-landing]({% image_path launch-landing.png %})

Suppored backend Runtimes are here:

* [.NET](https://docs.microsoft.com/en-us/dotnet/core/){:target="_blank"} - Create stand-alone, production-grade .NET Core Applications that you can "just run".
* [Node.Js](https://nodejs.org/en/){:target="_blank"} - A JavaScript runtime built on Chrome's V8 JavaScript engine, using an event-driven, non-blocking I/O model for lightweight efficiency.
* [Quarkus](https://quarkus.io/){:target="_blank"} - A Kubernetes Native Java stack tailored for GraalVM and OpenJDK HotSpot, crafted from the best of breed Java libraries and standards
* [Spring Boot](https://spring.io/projects/spring-boot){:target="_blank"} - Create stand-alone, production-grade Spring based Applications that you can "just run".
* [Thorntail](https://thorntail.io/){:target="_blank"} - An innovative approach to packaging and running Java EE applications, packaging them with just enough of the server runtime to "java -jar" your application.
* [Wildfly](https://wildfly.org/){:target="_blank"} - A classic approach to packaging and running Java EE applications.
* [Vert.x](https://projects.eclipse.org/projects/rt.vertx){:target="_blank"} - A tool-kit for building reactive applications on the JVM.

We will use several of these runtimes during the course of this workshop. To try out other runtimes later using Launcher when you have spare time, open the [Launcher](https://developers.redhat.com/launch/login){:target="_blank"} (you will first have to login or register an account for free with the Red Hat Developer Program).

