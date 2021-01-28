## Demonstrate a security context failure

1. `deployment.yaml` deploys a pod (Open Liberty) with an empty security context.

```yaml
    spec:
      containers:
        - name: liberty
          image: icr.io/ibm/liberty:kernel
          securityContext: {}
          ports:
            - containerPort: 80
```

Look at the YAML in the manifest file, notice how short the file is.

Deploy the manifest:

```shell
kubectl apply -n bwoolf -f example/deployment.yaml
deployment.apps/liberty-deployment created
```

2. Deploying the manifest causes the cluster to create a resource whose configuration matches that described in the manifest. In this case, it creates a Deployment with three Pods (three replicas of the same Pod spec).

The pods start no problem. Notice that the status of all three is `Running`.

```shell
kubectl get pods -n bwoolf                        
NAME                                 READY   STATUS    RESTARTS   AGE
liberty-deployment-b9b4448f5-7qz77   1/1     Running   0          31s
liberty-deployment-b9b4448f5-khvxh   1/1     Running   0          31s
liberty-deployment-b9b4448f5-m66lj   1/1     Running   0          31s
```

3. `deployment-sc.yaml` defines the same pod, but the security context specifies some capabilities.

```yaml
    spec:
      containers:
        - name: liberty
          image: icr.io/ibm/liberty:kernel
          securityContext:
            capabilities:
              add: [ "NET_ADMIN", "SYS_TIME" ]
          ports:
            - containerPort: 80
```

Deploy it:

```shell
kubectl apply -n bwoolf -f example/deployment-sc.yaml
deployment.apps/liberty-sc-deployment created
```

4. The pods fail to create or start. The list of pods only shows the three from `liberty-deployment`, but not the three from `liberty-sc-deployment`, so they weren't even created. The deployment's details show the error message.

```shell
kubectl get deployment liberty-sc-deployment -n bwoolf -o yaml
apiVersion: apps/v1
kind: Deployment
...
"name":"liberty-sc-deployment","namespace":"bwoolf"
...
  - lastTransitionTime: "2021-01-28T23:15:04Z"
    lastUpdateTime: "2021-01-28T23:15:04Z"
    message: 'pods "liberty-sc-deployment-845cb6d6d9-" is forbidden: unable to validate
      against any security context constraint: [capabilities.add: Invalid value: "NET_ADMIN":
      capability may not be added capabilities.add: Invalid value: "SYS_TIME": capability
      may not be added]'
    reason: FailedCreate
    status: "True"
    type: ReplicaFailure
...
```

You can also see this failure message in the OpenShift console at the bottom of the Deployments > Deployment Details > liberty-sc-deployment page.

## What have we learned?

So the error message is:

> pods "liberty-sc-deployment-845cb6d6d9-" is forbidden: unable to validate against any security context constraint: [capabilities.add: Invalid value: "NET_ADMIN": capability may not be added capabilities.add: Invalid value: "SYS_TIME": capability may not be added]

The cluster tried to validate the pod against the SCC but:
- the cluster was unable to add a NET_ADMIN capability to the pod
- the cluster was unable to add a SYS_TIME capability to the pod

I thought the cluster would be able to create the pods but not start them. But in this experiment, it couldn't even create them. I still think there's a way to tell the cluster to only create the pod and that would work until you tried to start it, but I/we will need to investiage further.

## Next

The next step in the experiment is to create an SCC with some capabilities and associate that with the pod (via a user; probably easier than a role, just to get something working). When the SCC allows a couple of capabilities and those are the ones the pod requests, the pods should start properly.
