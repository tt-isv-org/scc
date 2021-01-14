# OpenShift Security Practices using SCC

OpenShift **Security Context Contraints (SCCs)** are used to control permissions for deployed pods.

By default, pods are restricted from accessing any protected resources on the cluster. For typical stateless workloads, this is not an issue. But what if your application requires access to storage, networking services, or user management? This is where SCCs come into play.

## What is an SCC?

Similar to the way that RBAC resources control user access, administrators can use SCCs to control permissions for pods.

Each Openshift cluster contains 8 default SCCs, each specifying a set of allowed capabilities:

* **restricted** -  denies access to all host features and requires pods to be run with a UID, and SELinux context that are allocated to the namespace.
* **anyuid** - same as restricted, but allows users to run with any UID and GID.
* **hostaccess** - allows access to all host namespaces but still requires pods to be run with a UID and SELinux context that are allocated to the namespace.
* **hostmount-anyuid** - provides all the features of the restricted SCC but allows host mounts and any UID by a pod.  This is primarily used by the persistent volume recycler.
* **hostnetwork** - allows using host networking and host ports but still requires pods to be run with a UID and SELinux context that are allocated to the namespace.
* **node-exporter** -  is only used for the Prometheus node exporter.
* **nonroot** - provides all features of the restricted SCC but allows users to run with any non-root UID.
* **privileged** - allows access to all privileged and host features and the ability to run as any user, any group, any fsGroup, and with any SELinux context.

SCCs can be assigned to specific RBAC roles created on the OpenShift platform. Users associated with those roles are then limited to the capabilites allowed by the SCC.

Here is an overview of how the process works:

![flow](images/flow.png)

1. Role is assigned an SCC
1. User is associated with a role
1. User creates a pod manifest that requests capabilities
1. The pod manifest is process by OpenShift
1. OpenShift compares the pod request against the SCC
1. If the request is allowed by the SCC, OpenShift deploys the pod

## Managing SCCs

Administrators can manage the SCCs on the OpenShift platform via the OepnShift CLI.

```bash
oc get scc
oc get scc <scc name> -o yaml
oc describe scc <scc name>
oc edit scc <scc name>
oc delete scc <scc name>
```

When determining which SCC to assign, it is important to remember that less is better. If your pod requires capability A, don't select an SCC that provides capalities A, B, and C.

If none of the default SCCs provide exactly what you are looking for, you can create a custom one. It requires you sumbit a YAML file, such as the following:

```yaml
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-new-scc
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
```

Submit the SCC definition file using the `oc create` command:

```bash
oc create -f my-scc.yaml
```

## Assign SCC to RBAC role

On OpenShift, an administrator can create a **Role** with a rule to define which SCCs will be available for all the users associated with that role.

Roles are defined by a YAML file. For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
...
  name: test-role
  namespace: my-namespace
...
rules:
...
- apiGroups:
  - security.openshift.io           # location of SCCs
  resourceNames:
  - my-new-scc                      # SCC name
  resources:
  - securitycontextconstraints      # indicates this is an SCC
  verbs:
  - use                             # "use" is only allowed action on an SCC
```

## Deploying a Pod

To deploy a pod, it must have a JSON or YAML file called a pod manifest.

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    name: web-app
spec:
  securityContext:             # Pod security
    runAsUser: 1000
    runAsGroup: 3000
    fsGroup: 2000
  containers:
  - name: web-app-container
    image: nginx
    securityContext:           # Container security
      capabilities:
        add: ["NET_ADMIN", "SYS_TIME"]
    resources:
      limits:
        memory: "128Mi"
        cpu: "500m"
    ports:
      - containerPort: 80
...
```

The `securityContext` object is used to request capabilities for both the pod, and for all containers within in the pod. To be accepted, the capabilites must match what is allowed by the associated SCC.

## SCC Admission Process

As shown in the workflow diagram above, when a user makes a pod request, OpenShift compares the capabilities requested by the pod against what the associated SCC allows.

When multiple SCCs are available to the user, OpenShift will prioritize them.

SCCs have a priority field that affects the ordering when a pod request is validated. A higher priority SCC is moved to the front of the set when sorting. When the complete set of available SCCs are determined they are ordered by:

* Highest priority first, nil is considered a 0 priority
* If priorities are equal, the SCCs will be sorted from most restrictive to least restrictive
* If both priorities and restrictions are equal the SCCs will be sorted by name

## availablecapabilities/constraints (SELinuxpolicies,AppArmorprofiles,  etc.)

