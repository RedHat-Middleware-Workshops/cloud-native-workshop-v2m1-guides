Cloud-Native Workshop Module 1[![Build Status](https://travis-ci.org/openshift-labs/cloud-native-guides.svg?branch=ocp-3.11)](https://travis-ci.org/openshift-labs/cloud-native-guides)
===
This half day hands-on cloud-native workshops(`module 1`) provides developers and introduction to cloud-natives applications and gives them an experience of building cloud-native applications using Red Hat Application Rumties(i.e. Throntail, Spring Boot) with OpenShift and more.

Agenda
===
* Getting Started with Cloud-Native Apps
* Decide which Application Server to use in OpenShift
* Migrate your application to JBoss EAP
* Breaking the monolith apart - I
* Breaking the monolith apart - II

Lab Instructions on OpenShift

To deploy the workshop, it is recommended you first create a fresh project into which to deploy it. The same project will then be used by the workshop as it steps you through the different ways workshops can be deployed.

```
oc new-project labs
```

In the active project you want to use, run:

```
oc new-app https://raw.githubusercontent.com/openshift-labs/workshop-dashboard/2.14.4/templates/production.json \
  --param TERMINAL_IMAGE="quay.io/quaa/cloud-native-workshop-v2m1-guides:latest" \
  --param APPLICATION_NAME=cn-workshop-v2m1 \
  --param AUTH_USERNAME=workshop
```

Access to the workshop environment will be password protected and you will see a browser popup for entering user credentials. The username to enter is `workshop`. The password is displayed in the output from deploying the workshop environment, but can also be queried by running:

```
oc set env --list dc/cn-workshop-v2m1 | grep AUTH_PASSWORD
```

With the password in hand, the hostname for accessing the sample workshop environment in your browser can be found by running:

```
oc get route cn-workshop-v2m1
```

When you are finished you can delete the project you created, or if you used an existing project, run:

```
oc delete all,serviceaccount,rolebinding,configmap -l app=cn-workshop-v2m1
```

Note that this will not delete anything which may have been deployed when you went through the sample workshop. Ensure that you go right through the workshop and execute any steps described in it for deleting any deployments it had you make. Alternatively, if you deploy the workshop environment in a fresh project, delete the project.
