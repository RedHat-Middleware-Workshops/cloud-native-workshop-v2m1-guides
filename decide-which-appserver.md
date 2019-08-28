## Lab1 - Decide which Application Server to use in OpenShift

In this step we will analyze an monolith application built for use with Oracle® WebLogic Server (WLS). This application is a Java EE application
using a number of different technologies, including standard Java EE APIs as well as proprietary Weblogic APIs and best practices.

The Red Hat Application Migration Toolkit can be installed and used in a few different ways:

* `Web Console` - The web console for Red Hat Application Migration Toolkit is a web-based system that allows a team of users to assess and prioritize migration and modernization efforts for a large number of applications. It allows you to group applications into projects for analysis and provides numerous reports that highlight the results.
* `Command Line Interface` - The CLI is a command-line tool that allows users to assess and prioritize migration and modernization efforts for applications. It provides numerous reports that highlight the analysis results.
* `Eclipse Plugin` - The Eclipse plugin for Red Hat Application Migration Toolkit provides assistance directly in Eclipse and Red Hat JBoss Developer Studio for developers making changes for a migration or modernization effort. It analyzes your projects using RHAMT, marks migration issues in the source code, provides guidance to fix the issues, and offers automatic code replacement when possible.

For this lab, we will use the Web Console on top of OpenShift Container Platform.
Access the AMT Web Console [HERE]({{ AMT_CONSOLE_URL }}).

####1. Login the RHAMT web console in OpenShift cluster

---

![RHAMT Login]({% image_path rhamt_login.png %})

Login using your Openshift credentials. When you login for the first time, you should be asked to change the password in order to comply with RH SSO policy.

> `NOTE`: You can use the current password to input a new password.

![RHAMT Change Pwd]({% image_path rhamt_change_pwd.png %})

####2. Create a new project

---

![RHAMT Landing Page]({% image_path rhamt_landing_page.png %})

Input a name and description to create a project.

![RHAMT Create Project]({% image_path rhamt_create_project.png %})

  * Name: `userXX-eap-migration`
  * Description: `USERXX EAP MIGRATION PROJECT`

> `NOTE`: Add your username as prefix(i.e. user1-eap-migration) to distinguish each attendee's project.

####3. Add a monolith application to the project

---

Select `Server Path` to analyze a monolithic application:

 * Server Path: `/opt/apps`

![RHAMT Add App]({% image_path rhamt_add_monolith_app.png %})

####4.Select "Migration to JBoss EAP 7" in Transformation Path and clicks the "Save & Run" button

---

![RHAMT Add App]({% image_path rhamt_check_monolith_app.png %})

> `NOTE`: Check `com, weblogic` for Included packages.

####5. Go to the Active Analysis page and clicks on the latest when it’s completed

---

Click the # of Analysis(For example, #2 in the screenshot).

![RHAMT Complete]({% image_path rhamt_complete_analysis.png %})

####6. Review the report

---

You should see the landing page for the report:

![RHAMT Langing Page]({% image_path rhamt_result_landing_page.png %})

The main landing page of the report lists the applications that were processed. Each row contains a high-level overview of the story points, number of incidents, and technologies encountered in that application.

Click on the `monolith.war` link to access details for the project:

![RHAMT Project Overview]({% image_path rhamt_project_overview.png %})

####7. Understanding the report

---

The Dashboard gives an overview of the entire application migration effort. It summarizes:

* The incidents and story points by category
* The incidents and story points by level of effort of the suggested changes
* The incidents by package

> Story points are an abstract metric commonly used in Agile software development to estimate the relative level of effort needed to implement a feature or change.
Red Hat Application Migration Toolkit uses story points to express the level of effort needed to migrate particular application constructs, and the application as a whole.
The level of effort will vary greatly depending on the size and complexity of the application(s) to migrate.

There are several other sub-pages accessible by the menu near the top. Click on each one and observe the results for each of these pages:

* `All Applications` Provides a list of all applications scanned.
* `Dashboard` Provides an overview for a specific application.
* `Issues` Provides a concise summary of all issues that require attention.
* `Application Details` provides a detailed overview of all resources found within the application that may need attention during the migration.
* `Unparsable` shows all files that RHAMT could not parse in the expected format. For instance, a file with a .xml or .wsdl suffix is assumed to be an XML file. If the XML parser fails, the issue is reported here and also where the individual file is listed.
* `Dependencies` displays all Java-packaged dependencies found within the application.
* `Remote Services` Displays all remote services references that were found within the application.
* `EJBs` contains a list of EJBs found within the application.
* `JBPM` contains all of the JBPM-related resources that were discovered during analysis.
* `JPA` contains details on all JPA-related resources that were found in the application.
* `About` Describes the current version of RHAMT and provides helpful links for further assistance.

> Some of the above sections may not appear depending on what was detected in the project.

You also investigate the report to see if there are any complex migrations.

Now that you are comfortable that migration will only take approximately a week, which is about the same time it would take to migration from Oracle Weblogic to WebSphere Liberty Profile you feel comfortable to add `Low` in the row `Migration effort` in the `JBoss EAP` column. When you present your recommendation to the team and the architectural board your recommendation to migrate to JBoss EAP to deploy your application on OpenShift is approved.

For the first sprint you will focus on migrating the application to JBoss EAP, the DBA will migrate the database to a PostgresQL instance on OpenShift while the ops engineer setup a project for you in the OpenShift cluster.
