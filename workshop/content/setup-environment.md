Before starting the workshop, check that you are logged in correctly and will be able to deploy applications. To do this run:

```execute
oc auth can-i create pods
```

Did you type the command in yourself? If you did, click on the command instead and you will find that it is executed for you. You can click on any command which has the <span class="fas fa-play-circle"></span> icon shown to the right of it, and it will be copied to the interactive terminal and run. If you would rather make a copy of the command so you can paste it to another window, hold down the shift key when you click on the command.

If the output from the `oc auth can-i` command is `yes`, you are all good to go, and you can scroll to the bottom of the page and click on "Start Workshop".

If instead of the output `yes`, you see:

```
no - no RBAC policy matched
```

or any other errors, then the environment is not deployed correctly. Check the [GitHub repository](https://github.com/openshift-homeroom/lab-workshop-content) for this workshop for details of how to deploy it.
