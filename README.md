The CCN Roadshow(Dev Track) Module 1 Guide 2019
===
This module provides developers and introduction to cloud-natives applications and gives them an experience of building cloud-native applications 
using Red Hat Application Rumties(i.e. Throntail, Spring Boot) with OpenShift and more.

Agenda
===
* Getting Started with Cloud-Native Apps
* Decide which Application Server to use in OpenShift
* Migrate your application to JBoss EAP
* Breaking the monolith apart - I
* Breaking the monolith apart - II

Lab Instructions on OpenShift
===

Note that if you have installed the lab infra via APB, the lab instructions are already deployed.

Here is an example Ansible playbook to deploy the lab instruction to your OpenShift cluster manually.

```
- name: Create Guides Module 1
  hosts: localhost
  tasks:
  - import_role:
      name: siamaksade.openshift_workshopper
    vars:
      project_name: "guide-m1"
      workshopper_name: "Cloud-Native Workshop V2 Module-1"
      project_suffix: "-XX"
      workshopper_content_url_prefix: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m1-guides/master
      workshopper_workshop_urls: https://raw.githubusercontent.com/RedHat-Middleware-Workshops/cloud-native-workshop-v2m1-guides/master/_cloud-native-workshop-module1.yml
      workshopper_env_vars:
        PROJECT_SUFFIX: "-XX"
        COOLSTORE_PROJECT: coolstore{{ project_suffix }}
        OPENSHIFT_CONSOLE_URL: "https://YOUR_OCP_MASTER_URL"
        ECLIPSE_CHE_URL: "http://che-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        GIT_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        NEXUS_URL: "http://nexus-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
        LABS_DOWNLOAD_URL: "http://gogs-labs-infra.YOUR_OCP_ROUTE_SUBFFIX"
         
      openshift_cli: "oc --server https://YOUR_OCP_MASTER_URL"
```
