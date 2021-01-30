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

Before running the commands later in this tutorial, set your PROJECT, login with your credentials, and switch to the project using a terminal shell.

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

1. Setup users

    To demonstrate Security Context Constraints (SCC) we will need users to show what can be done with and without the SCC.  Run the following commands to create two users.

    ```bash
    oc create user user-scc
    oc create user userzero
    ```

    Add our custom SCC to the user named "user-scc" (do not add it to userzero).

    ```bash
    oc adm policy add-scc-to-user scc-tutorial-scc user-scc
    ```

    We'll use a role to allow these users to create resources. Run the following commands to create a role and assign it to the users.

    ```bash
    export ROLE=scc-create-resources
    oc create role $ROLE --verb=create --resource imagestreams,buildconfigs,deploymentconfigs,deployments,services,pods
    oc adm policy add-role-to-user $ROLE user-scc --role-namespace=$PROJECT
    oc adm policy add-role-to-user $ROLE userzero --role-namespace=$PROJECT
    ```

## Test the security context constraint with a pod

1. Attempt to create a pod with security constraints -- and FAIL

    In the following YAML, we've added to the securityContext of the container spec. We are indicating that the container requires:

    * Must be run as user 1234
    * Will need a "FsGroup" 5678
    * Requires CHOWN and SETGID capabilities

    Save this YAML to a file named scc-redis-pod.yaml.

    ```yaml
    kind: Pod
    apiVersion: v1
    metadata:
      generateName: scc-redis-pod-
      managedFields:
    spec:
      serviceAccountName: default
      securityContext:
        fsGroup: 5678
      containers:
        - resources: {}
          name: scc-pod-container
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
          ports:
            - containerPort: 8080
              protocol: TCP
          imagePullPolicy: Always
          volumeMounts:
            - name: default-token-zh5m8
              readOnly: true
              mountPath: /var/run/secrets/kubernetes.io/serviceaccount
          terminationMessagePolicy: File
          image:  'image-registry.openshift-image-registry.svc:5000/openshift/redis:latest'
      volumes:
        - name: default-token-zh5m8
      dnsPolicy: ClusterFirst
    ```

    Try to create the pod **WITHOUT** using our custom SCC. Create the pod with `--as=userzero`.

    ```bash
    oc create -f scc-redis-pod.yaml --as=userzero
    ```

    This attempt should fail with an error like the following.

    ```bash
    $ oc create -f scc-redis-pod.yaml --as=userzero
    Error from server (Forbidden): error when creating "scc-redis-pod.yaml": pods "scc-redis-pod-" is forbidden: unable to validate against any security context constraint: [fsGroup: Invalid value: []int64{5678}: 5678 is not an allowed group spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1234: must be in the ranges: [1000650000, 1000659999] capabilities.add: Invalid value: "CHOWN": capability may not be added capabilities.add: Invalid value: "SETGID": capability may not be added]
    ```

    Notice that the error specifically calls out the fsGroup, the runAsUser and the capabilities that we added.

1. Attempt to create a pod with security constraints using our custom SCC -- and WIN

    Try to create the pod **WITH** our custom SCC. Create the pod with `--as=user-scc`.

    ```bash
    oc create -f scc-redis-pod.yaml --as=user-scc
    ```

    With the correct custom SCC assigned to the user, this should work. Check the pod status to see it running and get the generated pod name.

    ```bash
    oc get pods | grep scc-redis-pod
    ```

    Use the pod name to access the logs and see redis running.

    ```bash
    oc logs pod/<pod-name>
    ```

    You can start a shell session to see how things look inside the pod.

    ```bash
    % oc rsh pod/<pod-name>
    sh-4.2$ whoami
    1234
    sh-4.2$ id
    uid=1234(1234) gid=0(root) groups=0(root),5678
    sh-4.2$ cd
    sh-4.2$ touch testfile
    sh-4.2$ ls -l
    total 4
    drwxrwx---. 2 redis root 4096 Dec 25 04:34 data
    -rw-r--r--. 1 1234  root    0 Jan 29 06:32 testfile
    sh-4.2$ chown :5151 testfile
    chown: changing group of 'testfile': Operation not permitted
    sh-4.2$ chown :5678 testfile
    sh-4.2$ ls -l
    total 4
    drwxrwx---. 2 redis root 4096 Dec 25 04:34 data
    -rw-r--r--. 1 1234  5678    0 Jan 29 06:32 testfile
    sh-4.2$ exit
    ```

    Notice that the user ID is 1234 as requested. Also, the group 5678 was created and can be set in file ownership.

1. You can remove this pod with the `oc delete` command.

    ```bash
    oc delete pods/<pod-name>
    ```

## See how it works with a deployment config

We'll do the same test with a deployment config. The concept is the same, but the result looks a little different.

1. Save this YAML to a file named scc-redis-dc.yaml

    ```yaml
    kind: DeploymentConfig
    apiVersion: apps.openshift.io/v1
    metadata:
      name: scc-redis-dc
    spec:
      securityContext:
        fsGroup: 5678
      replicas: 1
      template:
        metadata:
          labels:
            name: scc-redis-dc
        spec:
          serviceAccountName: default
          volumes:
            - name: redis-data
              emptyDir: {}
          containers:
            - name: scc-dc-container
              image: >-
                image-registry.openshift-image-registry.svc:5000/openshift/redis
              ports:
                - containerPort: 6379
                  protocol: TCP
              volumeMounts:
                - name: redis-data
                  mountPath: /var/lib/redis/data
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
    ````

    Try to create the deployment config **WITHOUT** using our custom SCC. Create the dc with `--as=userzero`.

    ```bash
    oc create -f scc-redis-dc.yaml --as=userzero
    ```

    HEY!  It didn't fail.  What happened?

    When a DeploymentConfig fails due to SCC, you need to check the status of the ReplicationController.

    Find it like this:

    ```bash
    $ oc get rc
    NAME             DESIRED   CURRENT   READY     AGE
    scc-redis-dc-1   1         0         0         5m17s
    ```

    Get the details using `oc describe rc/<rc name>` like this:

    ```md
    $ oc describe rc/scc-redis-dc-1
    Name:         scc-redis-dc-1
    [... stuff deleted ...]
    Replicas:     0 current / 1 desired
    Pods Status:  0 Running / 0 Waiting / 0 Succeeded / 0 Failed
    [ ... ]
    Conditions:
      Type             Status  Reason
      ----             ------  ------
      ReplicaFailure   True    FailedCreate
    Events:
      Type     Reason        Age               From                    Message
      ----     ------        ----              ----                    -------
      Warning  FailedCreate  3m (x17 over 7m)  replication-controller  Error creating: pods "scc-redis-dc-1-" is forbidden: unable to validate against any security context constraint: [spec.containers[0].securityContext.securityContext.runAsUser: Invalid value: 1234: must be in the ranges: [1000650000, 1000659999] capabilities.add: Invalid value: "CHOWN": capability may not be added capabilities.add: Invalid value: "SETGID": capability may not be added]
    ````

    We omitted some of the details, but what we really wanted to show is that we got back to our security context errors.

    You might expect that the next step is to run using *--as=user-scc* like we did earlier. It turns out that it would not help. We did not fail to create the deployment config, we failed when the replication controller tried to create the pod (like we did earlier). The replication controller is not using your "--as" user. It uses a service account and the default service account uses the restricted SCC.

1. Use our custom SCC with a service account

    Instead of modifying the project's default service account, we'll create a new one and add the SCC to it.  Notice the `-z` in the usage.

    > Usage: oc adm policy add-scc-to-user SCC (USER | -z SERVICEACCOUNT) [USER ...] [flags]

    Run the following commands to create the service account and add our existing SCC to it.

    ```bash
    oc create sa scc-tutorial-sa
    oc adm policy add-scc-to-user scc-tutorial-scc -z scc-tutorial-sa
    ```

    Edit your **scc-redis-dc.yaml** and change the `ServiceAccountName`:

    * From: `ServiceAccountName: default`
    * To: `ServiceAccountName: scc-tutorial-sa`

    Delete the failed deployment config and try again.

    ```bash
    oc delete dc/scc-redis-dc
    oc create -f scc-redis-dc.yaml --as=userzero 
    ```

    This time the replication controller should successfully spin up a pod running redis.
