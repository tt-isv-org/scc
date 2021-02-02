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

## Creating and assigning a SecurityContextConstraint

Create our example SecurityContextContraint and assign it to a user.

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

1. Setup application image

Run the following commands to build an image from our git repo. The app is a simple Python app that we will use in this tutorial. We're using `--as-test=true` here to build the image. We'll deploy it in the next section.

```bash
APP=scc-demo-app
oc new-app python~https://github.com/markstur/python-test-app --context-dir=examples/mod-wsgi-test-app --name=app3 --as-test=true
```

## Try to deploy the app with a security context -- AND FAIL

We'll do the same test with a deployment config. The concept is the same, but the result looks a little different.

1. Save this YAML to a file named scc-redis-dc.yaml

    ```yaml
    kind: Deployment
    apiVersion: apps/v1
    metadata:
      name: demo1
      labels:
        app: demo1
    spec:
      replicas: 1
      selector:
        matchLabels:
          app: demo1
      template:
        metadata:
          labels:
            app: demo1
        spec:
          securityContext:
            fsGroup: 5678
          serviceAccountName: default
          volumes:
            - name: demo1-app-data
              emptyDir: {}
          containers:
            - name: demo1-container
              image: 'image-registry.openshift-image-registry.svc:5000/markstur-project2/app3:latest'
              ports:
                - containerPort: 8080
                  protocol: TCP
              volumeMounts:
                - name: demo1-app-data
                  mountPath: /var/lib/demo1/data
              securityContext:
                capabilities:
                  add:
                    - CHOWN
                    - SETGID
                  drop:
                    - SETUID
                    - KILL
                    - MKNOD
                runAsUser: 1234
                fsGroup:
                  type: MustRunAs
                  ranges:
                  - max: 6000
                    min: 5000
                privileged: false
    ```

    When a Deployment fails due to SCC, you need to check the status of the ReplicationSet.

    Find it like this:

    ```bash
    $ oc describe  deployment/demo1
    [ ... ]
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      Available        False   MinimumReplicasUnavailable
      **ReplicaFailure   True    FailedCreate**
      Progressing      False   ProgressDeadlineExceeded
    OldReplicaSets:    <none>
    NewReplicaSet:     demo1-686786dccc (0/1 replicas created)
    Events:
      Type    Reason             Age   From                   Message
      ----    ------             ----  ----                   -------
      Normal  ScalingReplicaSet  28m   deployment-controller  Scaled up replica set demo1-686786dccc to 1
    ```

    To see why the ReplicaSet failed to create, use your OpenShift web console to find the Replica Set and see the Events tab.  Or check all the events from the command line like this:

    ```bash
    $ oc get events
    [ ... ]
    7m36s       Warning   FailedCreate        replicaset/demo1-686786dccc    Error creating: pods "demo1-686786dccc-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1234: must be in the ranges: [1000640000, 1000649999] capabilities.add: Invalid value: "CHOWN": capability may not be added capabilities.add: Invalid value: "SETGID": capability may not be added]
    ```

    We omitted some of the details, but what we really wanted to show is that we got back to our security context errors.

    We did not fail to create the deployment, we failed when the replica set tried to create the pod. The replica set uses a service account and the default service account uses the restricted SCC.

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
    oc patch deployment/demo1 --patch '{"spec":{"template":{"spec":{"serviceAccountName": "scc-tutorial-sa"}}}}'
    ```

    Now replication set should successfully spin up a pod running the application.

    Run the following commands to make your app available.

    ```bash
    oc expose deployment/demo1
    oc expose svc/demo1
    ```
