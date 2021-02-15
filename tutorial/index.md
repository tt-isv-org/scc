# SCC tutorial (draft)

In this tutorial, you will get hands-on experience with OpenShift security context constraints. You will:

1. Create the test application showing errors due to lack of privileges
    * Ensures you have a working environment
    * Creates the base image that will be used througout the tutorial
    * Provides a baseline to demonstrate the effect of security contexts and security context constraints
1. Attempt to redeploy the application with a security context
    * Demonstrates how an application deployment requesting special privileges in the security context is expected to fail when those privileges are restricted
1. Create and assign a security context constraint
    * Shows how to create an SCC granting special privileges
    * Shows how to add an SCC to a service account
1. Patch the deployment to use your new service account (and SCC!)
    * Proves that restricted capabilities can be added (and dropped)
    * Demonstrates how you can control your runtime user and groups
    * Shows how volume access uses predefined group IDs

> TODO: Probably a "flow diagram" type graphic to introduce this
## Concepts

Summarize and refer to the article and other links...

## Prerequisites

> TODO: Some of these may need to be explained. Ideally very little...

* Access to an OpenShift cluster
* Admin permissions (more specifics TBD)
* Install the OpenShift CLI (oc)
* Open a terminal
* Create and/or switch to a project to work in

### Conventions

Most of this tutorial will be run from a terminal command-line -- assuming bash or zsh (tested with zsh on MacOS).

We're using the `export` command to set environment variables that will be used in following examples. This allows you to choose names (i.e. a project name and application name) that are different than ours, set them in your environment, and use them in the rest of the commands to avoid copy/pasting the wrong names. Make sure you are always working in a shell where you set them.

### Start in a terminal

Before running the commands later in this tutorial, use your terminal shell to:

1. Login with your credentials
1. Set your **PROJECT** environment variable
1. Switch to the project

```bash
oc login <your-credentials>
export PROJECT=your-project-name
oc project $PROJECT
```

We are using YAML templates to allow these environment variables to be set when you run our example commands to create resources. You will copy our YAML to a file and run the `oc process` command. By using templates, we can pass in parameters instead of requiring you to edit the YAML file (e.g. no need to edit a hard-coded project name).

## Step 1. Create the test application showing errors due to lack of privileges

### Creating the demo app with `oc new-app`

Run the following commands to build an image from our git repo and deploy the demo app. The app is a simple Python app that we will use in this tutorial.

```bash
oc new-app python~https://github.com/markstur/scc-tutorial-assets --context-dir=examples/mod-wsgi-test-app 
```

> NOTE: Add the *--as-test=true* option if you'd prefer to skip the deployment of the app. That will build the image we need in the later examples where we use a security context and you can skip ahead without these baseline results.

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

The application is a Python WSGI Flask application. It uses a minimal amount of code to run some tests and report the results on its default web page. It also logs the results, so you can check the service logs or pod logs.

This application will run successfully with an empty security context, but it will log errors when you hit the web page. Ideally the application would like to:

* Run as a specific user ID
* Run as a specific group ID
* Mount a volume at path /var/opt/app-root/src/data
* Use a specific group ID to allow read/write on the volume
* Change the system clock
* Use setuid to run a program as another user
* Test and log the results of all of the above

After you hit the web page URL, you can also check the pod logs with the following command:

```bash
oc logs svc/scc-tutorial-assets
```

If you prefer to use the web console:

![new_app_topology.png](images/new_app_topology.png)

1. Use the pulldown on the left to select the `Developer` view.
1. Click on `Topology`.
1. Click on the application named `scc-tutorial-assets`. The right sidebar will slide out. It shows you how to `View logs` for the pods and provides a link for the service route.
1. Click on the route link.
    * TIP: There is an `Open URL` glyph attached to the application on the Topology canvas that you can also use to browse to the route.
1. Go back to the sidebar and click on `View logs` to see where an app would really show you warnings, errors, and info.

### What to expect

You should have successfully deployed an application from our source code, exposed a public service route, browsed to its web page, and found the pod logs.

By default, OpenShift runs this application using the **default** service account with the **restricted** SCC. If your cluster/project has different defaults, your results might be different. That's okay, we'll use a new service account and CSS later in the tutorial.

We did not add a volume or any special privileges. So expect to see red errors on the web page and the same errors in the log. In the following sections we'll see how SCCs work and fix those errors.

![new_app_page.png](images/new_app_page.png)

If you view the pod log, you'll see that the *ERROR* and *INFO* messages are written to the log every time you hit the web page.

![new_app_log.png](images/new_app_log.png)

## Step 1. Attempt to redeploy the application with a security context

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
              - mountPath: /var/opt/app-root/src/data
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
    oc describe deployment/$APP
    ```

    The output should show a ReplicaFailure:

    ```bash
    $ oc describe deployment
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
    $ oc get events | grep $APP
    51s         Warning   FailedCreate        replicaset/scctest-74f6895984                 Error creating: pods "scctest-74f6895984-" is forbidden: unable to validate against any security context constraint: [fsGroup: Invalid value: []int64{5678}: 5678 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1234: must be in the ranges: [1000650000, 1000659999] capabilities.add: Invalid value: "SETGID": capability may not be added capabilities.add: Invalid value: "SETUID": capability may not be added capabilities.add: Invalid value: "SYS_TIME": capability may not be added]
    [...]
    ```

    We omitted some of the details, but what we really wanted to show is that we got back to our security context errors.

    We did not fail to create the deployment, we failed when the replica set tried to create the pod. The replica set uses a service account and the default service account uses the restricted SCC.

    These errors are expected because the deployment asked for specific capabilities, but the default service account is restricted and cannot provide these capabilities. So, instead of deploying an application that will produce errors, we made it fail earlier with specific reasons.

    Don't delete this deployment. We'll fix this in the next section.

## Step 3. Create and assign a security context constraint

### Create our SCC with the capabilities we will need

1. Save this YAML to a file named scc-tutorial-scc.yaml.

    ```yaml
    kind: SecurityContextConstraints
    apiVersion: v1
    metadata:
      name: scc-tutorial-scc3
    allowPrivilegedContainer: false
    runAsUser:
      type: MustRunAsRange 
      uidRangeMin: 1000
      uidRangeMax: 2000
    seLinuxContext:
      type: RunAsAny
    fsGroup:
      type: MustRunAs 
      ranges:
      - min: 5000
        max: 6000
    supplementalGroups:
      type: MustRunAs 
      ranges:
      - min: 5000
        max: 6000
    defaultAddCapabilities:
    - SYS_TIME
    - SETGID
    - SETUID
    requiredDropCapabilities:
    - KILL
    - MKNOD
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

## Step 4: Patch the deployment to use your new SCC

1. Patch the deployment to use our new custom service account with the following command:

    ```bash
    oc patch deployment/$APP --patch '{"spec":{"template":{"spec":{"serviceAccountName": "scc-tutorial-sa"}}}}'
    ```

    Now replication set should successfully spin up a pod running the application.

1. See the working application as proof
    Run the following commands to make your app available.

    ```bash
    oc expose deployment/$APP
    oc expose svc/$APP
    ```

    See the success message on the applications web page:


## What did we learn?

Bueller? Bueller?

## Deleting resources

> TODO: help clean up