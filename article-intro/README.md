# Red Hat OpenShift security practices using security context contraints

To better understand how Security Context Contraints (**SCC**) are used to control access on a Red Hat OpenShift Container Platform, let's walk-through a common deployment scenario.

First, let's identify the personas that are typically involved in the process of developing an application, configuring a pod to contain the application, and then deploying that application on OpenShift.

* **Programmer** - responsible for developing the application or service that will run in the pod.
* **Deployer** - responsible for creating the deployment manifest that will define how the application will be run.
* **OpenShift User** - a user account granted deployment permissions by the OpenShift administrator.
* **OpenShift Service Account** - created to be associated with one or more SCCs.
* **OpenShift Administrator** - ensures the security of the OpenShift platform by granting access to protected resources only as needed.

Administrators use Role-Based Access Control (**RBAC**) resources to control user access. In our scenario, the administrator will create an **OpenShift User** account that has the permission to deploy pods. The **Deployer** can then use that account to create and deploy the pod on OpenShift. The **Deployer** performs this task by creating a deployment manifest, which links to the application and provides instructions on how the application should be deployed and ran.

If the application is a typical stateless workload (i.e. requiring no special permissions), the **Deployer** will generate a basic deployment manifest, and the pod will be deployed without any issues.

But what if the application requires access to protected resources such as local storage, networking services, or needs to run under specific user or group identities? In this case the **Deployer** will need to enhance the deployment manifest to request the additional privileges. And who gets to decide if the requests should be allowed or not? This is where SCCs and RBAC come into play.

The SCC specifies what privileges are allowed, what user and group IDs can be assigned, and what additional capabilities can be granted. SCCs are associated with users, groups, and/or service accounts - typically via RBAC roles. Therefore, when the **Deployer** attempts to deploy the pod, the SCC invoked is the one associated with the role assigned to the **OpenShift User** account.

>**NOTE**: Another way to associate a deployment manifest with an SCC is by specifying an **OpenShift Service Account** name in the manifest. This will be discussed later in the article.

If the SCC allows all of the requested capabilities made by the deployment manifest, the deployment is created and the pod/application is run.

>**NOTE**: As mentioned previously, there are multiple ways that a deployment manifest can request additional privileges, access, and capabilities. Throughout this article we will refer to all these request types collectively as simply "capabilities".

If the pod is denied the requested capabilities, the **OpenShift Administrator** will need to:

* Determine if the additional requests made in the manifest are in fact needed.
* Determine what SCC will allow the requested capabilities, or if none exist, create a new one.
* Assign the SCC to a role for the appropriate users, groups, and/or service accounts, or create such a role if needed.
* Assign the new role to the **OpenShift User** account.

Here is an overview of how user and service accounts, roles, and SCCs are involved in the deployment process:

![flow](images/flow.png)

1. The **Programmer** develops an application or service, and ...
1. Delivers the application to the **Deployer**, who creates a deployment manifest for the application.
1. The **OpenShift Administrator** creates roles which are assigned a security context constraint.
1. The **OpenShift Administrator** creates an **OpenShift User Account** and assigns it a role that provides deployment permissions. Also created is an **OpenShift Service Account** that is associated with one or more SCCs.
1. The **Deployer** logs into OpenShift using the new **OpenShift User Account**, and deploys the application using the deployment manifest. The manifest may contain a request for additional capabilities, and may reference which **OpenShift Service Account** to use when deploying the pod.
1. OpenShift processes the manifest and attempts to deploy the pod. The deployment process will compare the capabilities requested by the deployment manifest against the capabilities allowed by the associated SCCs.
1. If the associated SCC cannot provide all of the capabilities the deployment manifest requests, the deployment will fail.
1. Otherwise, the deployment will create the pod and run the application.

TODO: do we need more detail on the actual deployment? what actually fails - deployment, pod creation, pod starting...

Now that we have a high-level view of how deployment manifests work with SCCs on an OpenShift container platform, let's dig into the details.

## Capabilities requested vs. capabilities allowed

In the flow diagram above, you see that the pod will only be deployed if the requested capabilites defined in the deployment manifest match the allowed capabilites defined by the SCC.

Another way to envision this relationship is to think of the SCC as a lock protecting system resources, and the manifest being the key. The app only gets access to the resources if the key fits.

![capabilities](images/capabilities.png)

## Capability types

Up to this point we have used the generic term "capabilities" to describe all the permissions that are secured by SCCs. In reality, "capabilites" consist of 3 specific types - privileges, access, and capabilities. Each with its own set of rules and syntax.

### Privileges  

* allowPrivilegedContainer - can a pod run privileged containers
* allowHostDirVolumePlugin

### Access

Controls what specific user or group ID a pod may run as.

In the SCC, The list of fields that can be set include:
  
* **RunAsUser** - specifies the allowable range of user IDs used for running all the containers in the pod.
* **SupplementalGroups** - specifies the allowable range of group IDs used for rulling all the containers in the pod.
* **FSGroup** -  specifies the allowable range of group IDs used for controlling pod storage volumes.
* **SELinuxContext** - specifies the allowable values used for setting SELinux user, role, type and level.

In the deployment manifest, these values are set at the pod level and pertain to all containers running within the pod. The fields used are:

* **securityContext.runAsUser**
* **securityContext.runAsGroup**
* **securityContext.fsGroup**
* **securityContext.XXXXXX**

### Capabilities

Permission to perform specific actions, like access system time, or configure network settings. These capabilities are assigned to individual containers running within the pod. 

The list of actions includes:

* CHOWN - change file ownership and group ownership
* KILL - can send signal to process without having matching user ID
* NET_BROADCAST - can broadcast and listening to multicast
* NET_ADMIN - can configure interfaces, routing tables, multicast, admin of IP firewall, etc.
* SYS_CHROOT - can use chroot command
* SYS_ADMIN - can set domain and host names, run mount and unmount, lock/unlock shared memory, etc.
* SYS_TIME - can manipulate system clock
* MKNOD - provides privileged aspects of mknod()
* SETCAP - can set or remove capabilities on files 

A full list of values can be found [here](https://github.com/torvalds/linux/blob/master/include/uapi/linux/capability.h)

Capabilities are specified in the SCC using the following fields:

* **defaultAddCapabilities** - list of default capabilities added to each container.
* **requiredDropCapabilities** - list of capabilities that will be forbidden to run on each container.
* **allowedCapabilities** - list of container capabilities that are allowed to be requested by the demployment manifest.

Capabilities are requested in the deployment manifest using the **securityContext.capabilities.add** field.

## Pre-defined SCCs

Each Openshift cluster contains 8 pre-defined SCCs, each specifying a set of allowed capabilities:

* **restricted** -  denies access to all host features and requires pods to be run with a user ID (UID), and SELinux context that are allocated to the namespace.
* **anyuid** - same as restricted, but allows users to run with any UID and group ID (GID).
* **hostaccess** - allows access to all host namespaces but still requires pods to be run with a UID and SELinux context that are allocated to the namespace.
* **hostmount-anyuid** - provides all the features of the restricted SCC but allows host mounts and any UID by a pod.  This is primarily used by the persistent volume recycler.
* **hostnetwork** - allows using host networking and host ports but still requires pods to be run with a UID and SELinux context that are allocated to the namespace.
* **node-exporter** -  is only used for the Prometheus node exporter.
* **nonroot** - provides all features of the restricted SCC but allows users to run with any non-root UID.
* **privileged** - allows access to all privileged and host features and the ability to run as any user, any group, any fsGroup, and with any SELinux context.

If not specified, the **restricted** SCC will be used.

## Managing SCCs

Administrators can manage the SCCs on the OpenShift platform via the OpenShift CLI.

```bash
oc get scc
oc get scc <scc name> -o yaml
oc describe scc <scc name>
oc edit scc <scc name>
oc delete scc <scc name>
```

Here is a snippet of what the **restricted** SCC yaml looks like:

```yaml
oc get scc restricted  -o yaml
...
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: restricted
fsGroup:
  type: MustRunAs
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
allowedCapabilities: null
defaultAddCapabilities: null  
...
```

Here we show the fields used to define and limit the values specified by a deployment manifest - user and group IDs and SELinux options.

**MustRunAs** and **MustRunAsRange** enforces the range of ID values that can be requested by a container, and also assigns a default value if needed.

**RunAsAny** indicates that no range checking is performed and no default value is assigned, thus allowing any ID to be requested.

## Creating custom SCCs

When determining which SCC to assign, it is important to remember that less is better. If your pod requires capability A, don't select an SCC that provides capalities A, B, and C.

If none of the default SCCs provide exactly what you are looking for, you can create a custom one. It requires you submit a YAML file, such as the following:

```yaml
oc get scc my-custom-scc -o yaml
...
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-custom-scc
fsGroup:
  type: MustRunAs 
  ranges:
  - min: 2000
    max: 3000
runAsUser:
  type: MustRunAsRange 
  uidRangeMin: 1000
  uidRangeMax: 2000
seLinuxContext: 
  type: MustRunAs
  SELinuxOptions: 
    user: u0
    role: r0
    type: t0
    level: l0
supplementalGroups:
  type: MustRunAs 
  ranges:
  - min: 3000
    max: 4000
defaultAddCapabilities:
- CHOWN
- SYS_TIME
requiredDropCapabilities:
- MKNOD
allowedCapabilites:
- NET_ADMIN   
...
```

Submit the SCC definition file using the `oc create` command:

```bash
oc create -f my-custom-scc.yaml
```

## Ways to associate an SCC with a deployment manifest

### Assign SCCs to RBAC roles

So how is it determined which SCC is used when a deployment manifest is processed? One way is to create a service account and assign it to an RBAC role. The SCC can then be assigned to that role.

On OpenShift, an administrator can create a **Role** with a rule to define which SCCs will be available for all the users associated with that role.

Roles are defined by a YAML file. For example:

```yaml
apiVersion: rbac.authorization.k8s.io/v1
kind: Role
metadata:
...
  name: my-custom-role
  namespace: my-namespace
...
rules:
...
- apiGroups:
  - security.openshift.io                       # location of SCCs
  resourceNames:
  - my-custom-scc                               # SCC name
  resources:
  - securitycontextconstraints                  # indicates this is an SCC
  verbs:
  - use                                         # "use" is the only allowed action on an SCC
  
```

### Assign user or service accounts to an SCC

Here are the commands to create a service account, then assign it to our custom SCC:

```bash
oc create sa my-service-account
oc adm policy add-scc-to-user my-custom-scc -z my-service-account
```

This will update the SCC to include a list of assigned users:

```yaml
oc get scc my-custom-scc -o yaml
...
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-custom-scc
users:
- my-service-acount
...
```

## Deployment Manifests

The final piece in the SCC security puzzle is the deployment manifest, which is used to create and build a deployment, which can be then used to deploy a pod.

Here is a snippet of what a deployment manifiest yaml looks like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-test-deployment
  labels:
    app: my-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: my-app
  template:
    metadata:
      labels:
        app: my-app
    spec:
      serviceAccountName: my-custom-account
      securityContext:
        runAsUser: 1000
        runAsGroup: 3000
        fsGroup: 2000
      containers:
      - name: web-app-container
        image: nginx:1.14.2
        securityContext:
          capabilities:
            add: ["NET_ADMIN", "SYS_TIME"]
        ports:
        - containerPort: 80
...
```

The `securityContext` object is used to request capabilities for both the pod and for all containers within in the pod. To be accepted, the capabilites must match what is allowed by the associated SCC.

The first set of `securityContext` values are associated with the pod. The entries relate to what ID values are to be assigned to all containers running in the pod. For example,

* **runAsUser: 1000** - requests that all containers in the pod will run as user ID 1000.
* **runAsGroup: 3000** - request that all containers in the pod will run as group ID 3000.
* **fsGroup: 2000** - requests that the owner for mounted volumes and files created in that volume will be set to GID 2000.

The second set of `securityContext` values are associated with specific containers running inside the pod. For example:

* **capabilities - add** - requests that the `web-app-containter` container be allowed the `NET_ADMIN` and `SYS_TIME` capabilities .

## SCC Admission Process

As described earlier, OpenShift compares the capabilities requested by the pod against what the associated SCC allows.

But what happens when multiple SCCs are available? In this case, OpenShift will prioritize them.

SCCs have a priority field that affects the ordering when a pod request is validated. A higher priority SCC is moved to the front of the set when sorting. When the complete set of available SCCs are determined they are ordered by:

* Highest priority first, nil is considered a 0 priority
* If priorities are equal, the SCCs will be sorted from most restrictive to least restrictive
* If both priorities and restrictions are equal the SCCs will be sorted by name

TODO: show deployment yaml after deployment to see which SCC was used
TODO: add more detail. Are the SCCs merged? What if one SCC allows, but is lower priority of another SCC that fails?

## Putting it all together

Using the examples above, let's walk through the OpenShift SCC admission process to determine if our pod gets deployed or not.

First, we need to mention the OpenShift project yaml. The project plays a vital role as it provides default values when they are not specified in either the deployment manifest or SCC:

```yaml
oc get project default -o yaml 
...
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: default
  annotations:
    openshift.io/sa.scc.mcs: s0:c1,c0 
    openshift.io/sa.scc.supplemental-groups: 1000000000/10000
    openshift.io/sa.scc.uid-range: 1000000000/10000
...
```

Note that the following examples may refer back to this project yaml for default values.

### Deployment Vs. Restricted SCC

Let's check how our deployment manifest will fare against the **restricted** SCC.

![pod-vs-restricted](images/pod-vs-restricted.png)

![pod-vs-restricted-text](images/pod-vs-restricted-text.png)

### Deployment Vs. Custom SCC

Now let's try it against our **custom** SCC.

![pod-vs-custom](images/pod-vs-custom.png)

![pod-vs-custom-text](images/pod-vs-custom-text.png)

## Misc notes - TODO  *** **IGNORE FOLLOWING TEXT** ****


Cover ## availablecapabilities/constraints (SELinuxpolicies,AppArmorprofiles,  etc.)

When an application is deployed it will run as a user ID unique to the project it is running in. This overrides the user ID which the application image defines it wants to be run as. That the application is run as a different user ID can result in it failing to start.

The best solution is to build the application image so it can be run as an arbitrary user ID. This avoids the risks associated with having to run an application as the root user ID, or other fixed user ID which may be shared with applications in other projects.

If an image can't be modified, you can elect to override the default security configuration of OpenShift and have it run as the user the image specifies, but this can only be done by an administrator of the OpenShift cluster. This cannot be done by normal developers, nor a project administrator. This cannot be done on hosting services such as OpenShift Online.

The change required to override the default security configuration, is to grant rights to the **service account the application is run under**, to run images as a set user ID.

By default applications would run under the restricted SCC. The MustRunAsRange value for RUNASUSER is what indicates that the application needs to run within the user ID range associated with the project.

To allow an application to be run as any user ID, including the root user ID, you want to use the anyuid SCC

### pod manifest

runAsUser: 1000 - means all containers in the pod will run as user UID 1000
fsGroup: 2000 - means the owner for mounted volumes and files created in that volume will be GID 2000







### SCC yaml

Access:
  Users: <none>   // which users and service accounts the SCC is applied to
  Groups: system:authenticated   // which groups the SCC is applied to

runAsUser: RunAsAny  // the SCC can allow arbitrary IDs, and ID that falls into a range, or exact user ID specific to the request 

Add an SCC to a group:
```bash
$ oc adm policy add-scc-to-group <scc_name> <group_name>
```

Add an SCC to all service accounts in a namespace:
```bash
$ oc adm policy add-scc-to-group <scc_name> \
    system:serviceaccounts:<serviceaccount_namespace>
```

## Capabilities matrix

| SCC Capability | Description |
| - | - |
| allowedCapabilites | A list of capabilities that a pod can request. An empty list means that none of capabilities can be requested while the special symbol * allows any capabilities. |
| defaultAddCapabilites | A list of additional capabilities that are added to any pod. |
| fsGroup | Dictates the allowable values for the security context. |
| groups | The groups that can access this SCC. |
| requiredDropCapabilities | A list of capabilities that are be dropped from a pod. |
| runAsUser | Dictates the allowable values for the Security Context. |
| seLinuxContext | Dictates the allowable values for the Security Context. |
| supplementalGroups | Dictates the allowable supplemental groups for the Security Context. |
| users | The users who can access this SCC. |

## Test

Volume security - https://docs.openshift.com/enterprise/3.1/install_config/persistent_storage/pod_security_context.html

### Project yaml

```yaml
oc get project default -o yaml 
...
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: default
  annotations:
    openshift.io/sa.scc.mcs: s0:c1,c0 
    openshift.io/sa.scc.supplemental-groups: 1000000000/10000
    openshift.io/sa.scc.uid-range: 1000000000/10000
...
```

The annotations define default values for (in order) SELinux options, group IDs, and user IDs. These values are only used if the corresponding SCC strategy is NOT **RunAsAny**, AND when not specified in the SCC yaml or deployments yaml.

### Restricted SCC yaml

```yaml
oc get scc restricted  -o yaml
...
fsGroup:
  type: MustRunAs
runAsUser:
  type: MustRunAsRange
seLinuxContext:
  type: MustRunAs
supplementalGroups:
  type: RunAsAny
...
```

This SCC is the most restrictive of the pre-defined SCCs that are provided with OpenShift.

**fsGroup**: set to **MustRunAs**, which enforces group ID range checking and provides the container’s groups default. Since the range is omitted from the SCC, the default would be 1000000000 (derived from the project).

>**NOTE**: The other supported type, **RunAsAny**, does not perform range checking, thus allowing any group ID, and produces no default groups.

**runAsUser**: set to **MustRunAsRange**, which enforces user ID range checking and provides a UID default. Since the minimum and maximum range are omitted from the SCC, the default user ID would be 1000000000 (derived from the project).

>**NOTE**: **MustRunAsNonRoot** and **RunAsAny** are *the other supported types.

**seLinuxContext**: set to **MustRunAs**, which means the container is created with the SCC’s SELinux options, or the MCS default defined in the project.

>**NOTE**: A type of **RunAsAny** indicates that SELinux context is not required, and, if not defined in the pod, is not set in the container.

**supplementalGroup**: set to **RunAsAny**, which means no range checking is performed, thus allowing any group ID, and produces no default groups.

### Custom SCC yaml

```yaml
oc get scc my-custom-scc -o yaml
...
fsGroup:
  type: MustRunAs 
  ranges:
  - min: 5000
    max: 6000
runAsUser:
  type: MustRunAsRange 
  uidRangeMin: 65534
  uidRangeMax: 65634
seLinuxContext: 
  type: MustRunAs
  SELinuxOptions: 
    user: u0
    role: r0
    type: t0
    level: l0
supplementalGroups:
  type: MustRunAs 
  ranges:
  - min: 5000
    max: 6000
...
```

**fsGroup**: set to **MustRunAs**, which enforces group ID range checking and provides the container’s groups default. Based on this SCC definition, the default is 5000 (the minimum ID value). The other supported type, **RunAsAny**, does not perform range checking, thus allowing any group ID, and produces no default groups.

**runAsUser**: set to **MustRunAsRange**, which enforces user ID range checking and provides a UID default. Based on this SCC, the default UID is 65534 (the minimum value). The range of allowed IDs can be defined to include any user IDs required for the target storage.

**seLinuxContext**: set to **MustRunAs**, which means the container is created with the SCC’s SELinux options, or the MCS default defined in the project. In this case, we have defined the SELinux user name, role name, type, and levels.

### Pod yaml

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    name: web-app
spec:
  securityContext:
    runAsUser: 65534
    runAsGroup: 3000
    fsGroup: 5000
...
```

### Results

| Test | Restricted SCC | Custom SCC
| - | - | - |
| fsGroup | **FAIL** - 2000 does not fall within range specified by project | **PASS** - 2000 falls within range 5000-6000 allowed by SCC |
| runAsUser | **FAIL** - 65534 does not fall within range specified by project | **PASS** - 65534 falls within range 65534-65634 allowed by SCC|
| seLinuxContext | **PASS** - will assign value "s0:c1,c0"  | **PASS** - will assign value "u0:r0:t0:l0" |
| supplementalGroups | **PASS** | **PASS** |

