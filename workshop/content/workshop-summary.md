This is the end of the workshop.

If you need to delete the custom content you deployed from this workshop, run:

```execute
oc delete all,serviceaccount,rolebinding,configmap -l app=lab-sample-workshop
```

To delete the workshop itself, from where you originally deployed the workshop, run:

```copy
oc delete all,serviceaccount,rolebinding,configmap -l app=lab-workshop-content
```

This workshop content can be found at:

* https://github.com/openshift-homeroom/lab-workshop-content

The Git repositories which you can use as a starting point for your own workshops can be found at:

* https://github.com/openshift-homeroom/lab-markdown-sample
* https://github.com/openshift-homeroom/lab-asciidoc-sample

Workshop content is used to create a custom image deriving from the workshop dashboard image found at:

* https://github.com/openshift-labs/workshop-dashboard

The workshop can be deployed standalone by following instructions in that repository.

Alternatively, if you need to deploy a workshop for multiple users, you need to use the workshop spawner found at:

* https://github.com/openshift-labs/workshop-spawner
