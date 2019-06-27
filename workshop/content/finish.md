---
Sort: 3
Title: Workshop Summary
PrevPage: exercises/14-deploy-workshops
ExitSign: Finish Workshop
---

This is the end of the workshop. On this page it is a good idea to provide a summary of the workshop and any links to resources relevant to the workshop. This provides to anyone doing the workshop material they can research later to learn more.

The link for this sample workshop content can be found at:

* https://github.com/openshift-labs/lab-workshop-content

The workshop content is used to create a custom image deriving from the workshop dashboard image found at:

* https://github.com/openshift-labs/workshop-dashboard

The workshop can be deployed standalone by following instructions in that repository.

Alternatively, if you need to deploy a workshop for multiple users, you need to use the workshop spawner found at:

* https://github.com/openshift-labs/workshop-spawner

For further information on using `workshop-spawner`, see the workshops:

* https://github.com/openshift-labs/lab-learning-portal

Now that you have gone through this workshop, you can download it as a starting point for your own workshop. Delete the pages under the `workshop/content/exercises` and `workshop/slides` directory in your copy, add your own pages, then setup meta data for page titles and navigation in pages to define the navigation path for users of your workshop.

When editing your workshop content on your local computer, you can use editors such as Atom and VSCode. Because images are placed in the same directory as the markdown files, preview modes of these editors will work, including embedded images, and you can see what your content should look like without needing to first deploy it.

If you need to delete the custom content you deployed from this workshop, run:

```execute
oc delete all,serviceaccount,rolebinding,configmap -l app=lab-sample-workshop
```

To delete the workshop itself, from where you originally deployed the workshop, run:

```copy
oc delete all,serviceaccount,rolebinding,configmap -l app=lab-workshop-content
```
