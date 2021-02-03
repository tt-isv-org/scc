# SCC tutorial (draft)

## Concepts

Refer to the article and other links...

## Prerequisites

TBD: Some of these may need to be explained. Ideally very little...

* Access to an OpenShift cluster
* Admin permissions (more specifics TBD)
* Install the OpenShift CLI (oc)
* Open a terminal
* Create and/or switch to a project to work in

### Conventions

Most of this tutorial will be run from a terminal command-line -- assuming bash or zsh (testing with zsh on MacOS).

In some cases, we're using the `export` command to set environment variables that will be used in following examples. This allows you to choose a name, e.g. a project name, that is different than ours, set it once and use the environment variable in the examples to avoid copy/pasting the wrong name. Just set it and then make sure you are working in a shell where you set it.

Before running the commands later in this tutorial, set your **PROJECT**, login with your credentials, and switch to the project using a terminal shell.

```bash
export PROJECT=your-project-name
oc login <your-credentials>
oc project $PROJECT
```

We are using YAML templates to allow these environment variables to be set when you run our example commands to create resources. You will copy our YAML to a file and run the `oc create` command. By using templates, we can pass in parameters instead of requiring you to edit the YAML file (e.g. no need to edit a hard-coded project name).

## Step 1: Successfully deploy our demo app with no security context

### Creating the demo app with `oc new-app`

> NOTE: Add the *--as-test=true* option if you'd prefer skip the deployment of the app with an empty *securityContext*. That will build the image we need in the later examples where we use a security context.

Run the following commands to build an image from our git repo and deploy the demo app. The app is a simple Python app that we will use in this tutorial.

> TODO: Will move this from github.com/markstur to github.com/IBM

```bash
oc new-app python~https://github.com/markstur/scc-tutorial-assets --context-dir=examples/mod-wsgi-test-app 
```

This command results in:

1. Building and image from the GitHub source and creating an image stream in our OpenShift image registry.
2. Deploying a pod which runs the application in a container.
3. Creating a service named **scc-tutorial-assets**.

### Exposing a route

To create a route based on the service so that you can try the app, run:

```bash
oc expose service scc-tutorial-assets
```

Use the following command to get the HOST/PORT to access the application's web page:

```bash
oc get route/scc-tutorial-assets
```

### Using the application

This application will run successfully with an empty security context, but it will log errors whenn you hit the web page. Ideally the app would like to:

* Run as a specific user ID
* Run as a specific group ID
* Mount storage in directory ...
* Allow write a file in...
* Set a group ownership on files in ...

After you hit the web page URL, you can also see the logs with the following command:

```bash
oc logs svc/scc-tutorial-assets
```

### Deleting

TODO: Clean-up section here or at the end?

## Step 2: Try to deploy the app with a security context -- AND FAIL

Since the application is intended to run with a specific user ID and other restricted capabilities, we really should use a `securityContext` to request those capabilities.

This YAML will deploy the same application, but this time we are requesting some specific security constraint requirements including:

* Run as user 1234
* Create a file system group ID of 5678
* Add capabilities: CHOWN and SETGID
* Drop capabilities: SETUID, KILL, MKNOD

### Deploying the application with security context settings

1. Copy (or download) this scc-template.yaml file.

```yaml
kind: Template
apiVersion: v1
metadata:
  name: ${APP}-template
objects:
- kind: Deployment
  apiVersion: apps/v1
  metadata:
    name: ${APP}-deployment
  spec:
    selector:
      matchLabels:
        app: ${APP}
    template:
      metadata:
        labels:
          app: ${APP}
      spec:
        containers:
        - image: image-registry.openshift-image-registry.svc:5000/${PROJECT}/scc-tutorial-assets:latest
          name: ${APP}-container
          ports:
          - containerPort: 8080
            protocol: TCP
          resources: {}
          securityContext:
            capabilities:
              add:
              - CHOWN
              - SETGID
              drop:
              - SETUID
              - KILL
              - MKNOD
            privileged: false
            runAsUser: 1234
          volumeMounts:
          - mountPath: /var/lib/${APP}/data
            name: ${APP}-data
        securityContext:
          fsGroup: 5678
        volumes:
        - emptyDir: {}
          name: ${APP}-data
parameters:
  - name: APP
  - name: PROJECT
```

1. Process the template with APP and PROJECT parameters and create the deployment

    Set a new APP name for your app. Make sure your PROJECT is also set in your shell.

    ```bash
    export APP=<your-app-name>
    export PROJECT=<your-app-name>
    ```

    Process the template using APP and PROJECT parameters and create the deployment:

    ```bash
    oc process -f  scc-template.yaml -pAPP=$APP -pPROJECT=$PROJECT | oc create -f -
    ```

1. Check for SCC errors

    When a deployment fails due to SCC, you need to check the status of the replica set. Describe the deployment to check replica status:

    ```bash
    oc describe deployment/$APP-deployment
    ```

    The output (trimmed here) should show:
    Find it like this:

    ```bash
    $ oc describe deployment/$APP-deployment
    [ ... ]
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      Progressing      True    NewReplicaSetCreated
      Available        False   MinimumReplicasUnavailable
      ReplicaFailure   True    FailedCreate
    OldReplicaSets:    <none>
    NewReplicaSet:     scctest1-deployment-697fd459b4 (0/1 replicas created)
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  3m    deployment-controller  Scaled up replica set scctest1-deployment-697fd459b4 to 1
    ```

    To get a more specific reason for the replica set failure. Use `oc get events`:

    ```bash
    $ oc get events | grep scctest1-deployment-697fd459b4
    [ ... ]
    3m10s       Warning   FailedCreate        replicaset/scctest1-deployment-697fd459b4     Error creating: pods "scctest1-deployment-697fd459b4-" is forbidden: unable to validate against any security context constraint: [fsGroup: Invalid value: []int64{5678}: 5678 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1234: must be in the ranges: [1000640000, 1000649999] capabilities.add: Invalid value: "CHOWN": capability may not be added capabilities.add: Invalid value: "SETGID": capability may not be added]
    ```

    We omitted some of the details, but what we really wanted to show is that we got back to our security context errors.

    We did not fail to create the deployment, we failed when the replica set tried to create the pod. The replica set uses a service account and the default service account uses the restricted SCC.

    These errors are expected because the deployment asked for specific capabilities, but the default service account is restricted and cannot provide these capabilities. So, instead of deploying an application that will produce errors, we made it fail earlier with specific reasons.

    Don't delete this deployment. We'll fix this in the next section.

## Step 3: Create and assign an SCC

### Create our SCC with the capabilities we will need

1. Save this YAML to a file named scc-tutorial-scc.yaml.

    ```yaml
    kind: SecurityContextConstraints
    apiVersion: v1
    metadata:
      name: scc-tutorial-scc
    allowPrivilegedContainer: true
    runAsUser:
      type: RunAsAny
    seLinuxContext:
      type: RunAsAny
    fsGroup:
      type: RunAsAny
    supplementalGroups:
      type: RunAsAny
    users:
    - my-admin-user
    groups:
    - my-admin-group
    defaultAddCapabilities:
    - CHOWN
    - SETGID
    - SETUID
    requiredDropCapabilities:
    - MKNOD
    - SYS_CHROOT
    ```

1. Run the following `oc create` command to create the SecurityContextConstraint.

    Create the SecurityContextContraint

    ```bash
    oc create -f scc-tutorial-scc.yaml
    ```

1. Use our custom SCC with a service account

    **Instead of modifying the project's default service account**, we'll create a new one and add the SCC to it.  Notice the `-z` in the usage.

    > Usage: oc adm policy add-scc-to-user SCC (USER | -z SERVICEACCOUNT) [USER ...] [flags]

    Run the following commands to create the service account and add our existing SCC to it.

    ```bash
    oc create sa scc-tutorial-sa
    oc adm policy add-scc-to-user scc-tutorial-scc -z scc-tutorial-sa
    ```

    Patch the deployment to use our new custom service account with the following command:

    ```bash
    oc patch deployment/$APP-deployment --patch '{"spec":{"template":{"spec":{"serviceAccountName": "scc-tutorial-sa"}}}}'
    ```

    Now replication set should successfully spin up a pod running the application.

1. See the working application as proof
    Run the following commands to make your app available.

    ```bash
    oc expose deployment/$APP-deployment
    oc expose svc/$APP-deployment
    ```

    See the success message on the applications web page:

    > Hello World from mod_wsgi hosted WSGI application!


## What did we learn?

Bueller? Bueller?
