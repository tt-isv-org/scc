# Red Hat OpenShift security practices using security context contraints - Part II

This article is Part II of the two-part series on Red Hat OpenShift Security Context Contraints (SCCs).

[Part I](https://github.ibm.com/TT-ISV-org/scc/blob/main/article/intro.md) provided background information on what SCCs are, and how they play a key role in providing security on an OpenShift cluster.

This article builds on those concepts and digs into the details on how to properly utilize SCCs. This includes:

* Details on how SCCs allow permissions
* What pre-defined SCCs are and when they should be used
* How to create custom SCCs
* Details on how deployment manifests request permissions
* How to connect deployment manifests with SCCs

## OpenShift projects

Any discussion of SCCs needs to be in the context of an OpenShift project namespace, which plays a major role in how SCCs are defined, managed, and utilized.

OpenShift projects are the mechanism to scope resources in a cluster and are used by administrators to limit and isolate resources. These resources include users, deployments, and SCCs.

## Types of permissions that can be requested

Up to this point we have used the generic term "permissions" to describe what pods can request (via the deployment manifest) and what SCCs will allow. In reality, "permissions" consist of 3 specific types - privileges, access control, and capabilities. Each with its own set of rules and syntax.

In the following sections, we show some of available fields and the values that can be used - from both the deployment manifest side (asking permission), and the SCC side (granting permission).

**IMPORTANT**: To be specific, the **security context** section of the deployment manifest is where permission requests are made.

### Privileges  

These settings allow general authority the pod will have when deployed.

In the SCC, you allow the privilege by setting the value to **true**. Here are some example privileges:

* **allowPrivilegedContainer** - specifies if a pod can run privileged containers.
* **allowPrivilegeEscalation** - specifies if a child process of a container can gain more privileges than its parent.

In the deployment manifest, you can request these privileges at the container level. For example:

* **containers.securityContext.privileged: true**

### Access control

Controls what specific user and group values a pod can run as.

In the SCC, The list of fields that can be set include:
  
* **runAsUser** - specifies the allowable range of user IDs used for running all the containers in the pod.
* **supplementalGroups** - specifies the allowable range of group IDs used for running all the containers in the pod.
* **fsGroup** -  specifies the allowable range of group IDs used for controlling pod storage volumes.
* **seLinuxContext** - specifies the allowable values used for setting the SELinux context, which includes SELinux user, role, type and level.

In the deployment manifest, these values can either be requested at the pod level and pertain to all containers running within the pod, or at the specific container level. The fields used are:

* **securityContext.runAsUser** - request to run under a specific user ID.
* **securityContext.runAsGroup** - request to run under a specific group ID.
* **securityContext.fsGroup** - request to run under a specific group ID for accessing storage volumes.
* **securityContext.seLinuxOptions** - request to run using a specific SELinux context set of labels.

**NOTE**: SELinux (Security-Enhanced Linux) is a Linux kernel module that provides additional access control security, and provides the following benefits:

* All processes and files are labeled. SELinux policy rules define how processes interact with files, as well as how processes interact with each other. Access is only allowed if an SELinux policy rule exists that specifically allows it.
* Fine-grained access control. Stepping beyond traditional UNIX permissions that are controlled at user discretion and based on Linux user and group IDs, SELinux access decisions are based on all available information, such as an SELinux user, role, type, and, optionally, a security level.
* SELinux policy is administratively-defined and enforced system-wide.
* Improved mitigation for privilege escalation attacks. Processes run in namespaces, and are therefore separated from each other. SELinux policy rules define how processes access files and other processes.

### Capabilities

Permission to perform specific actions, like access system time, or configure network settings. These capabilities are assigned to individual containers running within the pod, and take precedent over any pod settings.

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

Capabilities are contolled in the SCC using the following fields:

* **defaultAddCapabilities** - list of default capabilities automatically added to each container.
* **requiredDropCapabilities** - list of capabilities that will be forbidden to run on each container.
* **allowedCapabilities** - list of container capabilities that are allowed to be requested by the demployment manifest.

Capabilities are requested in the deployment manifest using:

* **containers.securityContext.capabilities.add**
* **containers.securityContext.capabilities.drop**

## Pre-defined SCCs

Each Openshift cluster contains 8 pre-defined SCCs, each specifying a set of allowed permissions.

|   |   |   |
| - | - | - |
| SCC | Description | Comments |
| **restricted** | Denies access to all host features and requires pods to be run with a user ID (UID), and SELinux context that are allocated to the OpenShift project. | This is the most secure SCC and is always used by default. Will work for most typical stateless workloads. |
| **anyuid** | Same as restricted, but allows users to run with any UID and group ID (GID). | This is very dangerous as it allows running as root user, both inside and outside the container. If used, SELinux controls can play an important role here adding a layer of protection. It's also a good idea to use **seccomp** to filter non desired system calls. |
| **hostmount-anyuid** | Provides all the features of the restricted SCC but allows host mounts and any UID by a pod.  This is primarily used by the "persistent volume recycler", a trusted workload that is an essential infrastructure piece to the cluster. | Same warnings as using **anyuid**, but goes further by allowing the mounting of host volumes. |
| **hostaccess** | Allows access to all host project namespaces but still requires pods to be run with a UID and SELinux context that are allocated to the project. | Access to all host namespaces is dangerous, even though it does restrict UID and SELinux. This should only be used for necessary trusted workloads. |
| **hostnetwork** | Allows using host networking and host ports but still requires pods to be run with a UID and SELinux context that are allocated to the project. | This allows the pod to "see and use" the host network stack directly. Requiring the pod run with a non-zero UID and pre-allocated SELinux context will add some security. |
| **node-exporter** | Only used for the Prometheus node exporter (Prometheus is a popular Kubenetes monitoring tool). | This SCC was designed specifically for Prometheus to retrieve metrics from the cluster. It allows access to the host network, host PIDS, and host volumes, but not host IPC. Also allows **anyuid**. This should **not** be used. |
| **nonroot** | Provides all features of the **restricted** SCC but allows users to run with any non-root UID. | Suitable for applications that need predictable non root UIDs, but can function with all the other limitations set by the **restricted** SCC. |
| **privileged** | Allows access to all privileged and host features, and the ability to run as any user, group, or fsGroup, and with any SELinux context. This is the most relaxed SCC policy. | This SCC allows a pod to control everything in the host and worker nodes, as well as other containers. Only trusted workloads should use this. There is a case to be made that this should never be used in production, as it allows the pod to completely control the host. |
|   |   |   |
<br>

>**IMPORTANT:** You should never modify or delete any of the pre-defined SCCs.

### Examining the restricted SCC

SCCs can be viewed using the following OpenShift CLI commands:

```bash
oc get scc <scc name> -o yaml
oc describe scc/<scc name>
```

Here is a snippet of what the **restricted** SCC YAML file looks like (note that comment lines have been added to the YAML for clarity):

```yaml
$ oc get scc restricted  -o yaml
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: restricted
...
# Privileges
allowPrivilegedContainer: false
allowHostNetwork: false
allowHostPID: false
...
# Capabilities
allowedCapabilities: null
defaultAddCapabilities: null
requiredDropCapabilities:
- KILL
- MKNOD
- SETUID
- SETGID
...
# Access Control
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

Note how locked down this SCC is - it doesn't allow any special privileges or capabilities. It does, however, allow any supplemental group ID to be specified. This is to provide access to shared resources at the group level.

Regarding the access control section, the following types can be specified:

* **MustRunAs** and **MustRunAsRange** enforces the range of ID values that can be requested by a container, and also assigns a default value if needed.
* **RunAsAny** indicates that no range checking is performed, thus allowing any ID to be requested. Note that this allows UID 0 (root) to be specified, which is significantly less secure than a non-root UID.
* **MustRunAsNonRoot** indicates that any non-root (UID 0) ID value can be requested. This is similar to **RunAsAny** but much more secure.

## Creating custom SCCs

When determining which SCC to assign, it is important to remember that less is better. If your pod requires permission A, don't select an SCC that provides permissions A, B, and C.

If none of the pre-defined SCCs provide exactly what you are looking for, you can create a custom one.

One way to create a custom SCC is by creating a YAML file, such as the following:

```yaml
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-custom-scc
# Privileges
allowPrivilegedContainer: false
# Access Control
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
# Capabilities
defaultAddCapabilities:
  - CHOWN
  - SYS_TIME
requiredDropCapabilities:
  - MKNOD
allowedCapabilites:
  - NET_ADMIN  
```

If we named this SCC definition file `my-custom-scc.yaml`, we would create the SCC with the following command:

```bash
oc create -f my-custom-scc.yaml
```

To view the created SCC, use the command:

```bash
oc get scc my-custom-scc -o yaml
```

You can also view the SCC with the `describe` command:

```bash
$ oc describe scc/my-custom-scc
Name:                           my-custom-scc
Priority:                       <none>
Access:
  Users:                        <none>
  Groups:                       <none>
Settings:
  Allow Privileged:             false
  Allow Privilege Escalation:   true
  Default Add Capabilities:     CHOWN,SYS_TIME
  Required Drop Capabilities:   MKNOD
  Allowed Capabilities:         <none>
  Allowed Seccomp Profiles:     <none>
  Allowed Volume Types:         awsElasticBlockStore,azureDisk,azureFile,cephFS,cinder,configMap,csi,downwardAPI,emptyDir,fc,flexVolume,flocker,gcePersistentDisk,gitRepo,glusterfs,iscsi,nfs,persistentVolumeClaim,photonPersistentDisk,portworxVolume,projected,quobyte,rbd,scaleIO,secret,storageOS,vsphere
  Allowed Flexvolumes:          <all>
  Allowed Unsafe Sysctls:       <none>
  Forbidden Sysctls:            <none>
  Allow Host Network:           false
  Allow Host Ports:             false
  Allow Host PID:               false
  Allow Host IPC:               false
  Read Only Root Filesystem:    false
  Run As User Strategy: MustRunAsRange
    UID:                        <none>
    UID Range Min:              1000
    UID Range Max:              2000
  SELinux Context Strategy: RunAsAny
    User:                       <none>
    Role:                       <none>
    Type:                       <none>
    Level:                      <none>
  FSGroup Strategy: MustRunAs
    Ranges:                     5000-6000
  Supplemental Groups Strategy: MustRunAs
    Ranges:                     5000-6000
```

A quick note on some of the permissions annotated in the SCC:

### SELinux

When using SELinuxContext in SCC's, note that:

1. Using **MustRunAs** requires **seLinuxContext.seLinuxOptions** to be set, either in the SCC or at the project level. These values are then validated against the **seLinuxOptions** requested in the deployment manifest.
1. Using **RunAsAny** means no default values are provided. Allows any **seLinuxOptions** to be specified in the deployment manifest.

### Seccomp

Seccomp is a Linux kernel security feature. When enabled this prevents a majority of system calls from being made by the container, eliminating most common vulnerabilities. Seccomp is maintained by a whitelist profile that can be added to for custom use and is unique to each base image profile.

Here is an example of a RedHat linux image [RedHat Linux capabilites and Seccomp](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_atomic_host/7/html/container_security_guide/linux_capabilities_and_seccomp).

## Making SCCs available

> **IMPORTANT** - This section includes the use of Role-based access control (RBAC) resources and assumes you have a fundamental knowledge about RBAC -- such as roles and bindings, and how administrators make them work together to manage user permissions and access. For reference, check out these pages in the Red Hat OpenShift documentation:
>* [Using RBAC to define and apply permissions](https://docs.openshift.com/container-platform/4.6/authentication/using-rbac.html)
>* [Understanding and creating service accounts](https://docs.openshift.com/container-platform/4.6/authentication/understanding-and-creating-service-accounts.html)

Once we have decided which SCC to use (pre-existing or custom), how do we make it available to deployments?

The key resource is the **service account**, which provides the link to an SCC.

Once that association exists, it is easy to point to that service account from our deployment.

### Service accounts

Service accounts are basically user accounts, but are more flexible in that you don't have to share a regular user's credentials.

Each service account's user name is derived from its project and name:

```bash
system:serviceaccount:<project>:<name>
```

Use the following command to create a new service account in our current project:

```bash
oc create sa my-custom-sa
```

### Assign SCCs to service accounts

One approach is to directly assign an SCC to a service account.

Here is the command to assign our custom SCC to our service account:

```bash
oc adm policy add-scc-to-user my-custom-scc -z my-custom-sa
```

**NOTE**: The preferred approach is to associate an SCC with a service account via an RBAC role: Create a role, associate the SCC with the role, and bind the service account to the role. See [Role-based access to security context constraints](https://docs.openshift.com/container-platform/4.7/authentication/managing-security-context-constraints.html#role-based-access-to-ssc_configuring-internal-oauth) for more details on how to accomplish this.

## Deployment manifests

The final piece in the puzzle is the deployment manifest. It needs to include the security context and specify the service account that's expected to grant the capabilities requested in the service context. As we've already shown, the service account links directly or indirectly to the SCC that grants capabilities.

A deployment manifest is used to create and build a deployment, which can be then used to deploy a pod.

Here is a snippet of what a deployment manifiest YAML file looks like:

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: my-test-app
spec:
  selector:
    matchLabels:
      app: scc-article-sc-sa
  template:
    metadata:
      labels:
        app: scc-article-sc-sa
    spec:
      serviceAccountName: my-custom-sa
      securityContext:
        fsGroup: 5555
      containers:
      - image: ubi8/ubi-minimal
        name: ubi-minimal
        command: ['sh', '-c', 'echo "Hello from user $(id -u)" && sleep infinity']
        securityContext:
          runAsUser: 1234
          runAsGroup: 5678
          capabilities:
            add: ["SYS_TIME"]
        volumeMounts:
        - mountPath: /var/opt/app/data
          name: data
      volumes:
      - emptyDir: {}
        name: data      
```

**NOTE**: This YAML file contains 2 **securityContext** sections. The first (and most outer-level) is applied to the pod, and the second is specifically for the container. While this YAML doesn't show it, there can be a **securityContext** section for each container in the pod.

On behalf of the running containers of the pod, the deployment manifest is responsible for requesting the permissions for the container applications to run.

The `serviceAccountName` object defines the service account to use during deployment. As shown earlier, it can be tied to an SCC either directly or via its associated role.

The `securityContext` object is used to request capabilities for both the pod and for all containers within in the pod. To be accepted, the capabilites must match what is allowed by the associated SCC.

The first set of `securityContext` values are associated with the pod. The entries relate to what ID values are to be assigned to all containers running in the pod. For example,

* **fsGroup: 5555** - requests that the owner for mounted volumes and files created in that volume will be set to GID 5555.

The second set of `securityContext` values are associated with specific containers running inside the pod. For example:

* **runAsUser: 1234** - requests that the container will run as user ID 1234.
* **runAsGroup: 5678** - request that the container will run as group ID 5678.
* **capabilities:add** - requests that the container be allowed to run `SYS_TIME`.

Assuming the manifest YAML is in a file named "my-test-app.yaml", use the following command to create the deployment:

```bash
oc create -f my-test-app.yaml
```

Once the workload is deployed, you can use the following commands to manage it:

```bash
oc describe deployment/my-test-app
oc get events | grep replicaset/my-test-app
oc delete deployment/my-test-app
```

## Putting it all together

Using the examples from above, let's walk through the OpenShift SCC admission process to determine if our pod gets deployed or not.

First, let's take a look at the OpenShift project, which plays a vital role in isolation and containing the resources used during deployment. It also provides default values and ranges when not specified in the SCC. If we display the YAML file, we see some important annotation field values:

```yaml
$ oc get project scc-test-project -o yaml 
...
apiVersion: project.openshift.io/v1
kind: Project
metadata:
  name: default
  annotations:
    openshift.io/sa.scc.mcs: s0:c26,c5 
    openshift.io/sa.scc.supplemental-groups: 1000000000/10000
    openshift.io/sa.scc.uid-range: 1000000000/10000
...
```

The annotations define default values for (in order):

* SELinux options (MCS stands for multi-category security)
* group IDs
* user IDs

These values are only used if the corresponding SCC access control setting is NOT RunAsAny, AND when not specified in the SCC or deployment manifest.

**NOTE**: The range 1000000000/10000 means values between 1000000000 to 1000009999.

### Deployment vs restricted SCC

Using our example deployment manifest, let's see how it will fare against the **restricted** SCC.

![pod-vs-restricted](images/pod-vs-restricted.png)

![pod-vs-restricted-text](images/pod-vs-restricted-text.png)

### Deployment vs custom SCC

Now let's try the same deployment against our **custom** SCC.

![pod-vs-custom](images/pod-vs-custom.png)

![pod-vs-custom-text](images/pod-vs-custom-text.png)

>**NOTE**: Note that the SCC items #1 and #3 are set to **mustRunAs**, which typically would indicate setting a value. But in this case a range is used.

## SCC admission process

The SCC admission process is used to by OpenShift to compare the permissions requested by the pod against what the associated SCC allows.

But what happens when multiple SCCs are available? This can occur, for example, when the **deployment manifest** links to a **service account** that is bound to multiple **RBAC roles**, each pointing to a different **SCC**.

In this case, OpenShift must prioritize the SCCs.

SCCs have a priority field that affects the ordering when a pod request is validated. A higher priority SCC is moved to the front of the set when sorting. 

Here we show the YAML for an SCC that sets its priority to 11:

```yaml
kind: SecurityContextConstraints
apiVersion: security.openshift.io/v1
metadata:
  name: scc-priority-sample
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
users:
- my-admin-user
groups:
- my-admin-group

# PRIORITY FIELD
priority: 11
```

When the complete set of available SCCs is determined, they are ordered by:

* Highest priority first - nil is considered a 0 priority (lowest).
* If priorities are equal, the SCCs will be sorted from most restrictive to least restrictive.
* If both priorities and restrictions are equal, the SCCs will be sorted by name.

It should also be noted that the admission process:

* Does not give extra weight to cluster-wide SCCs over project-based SCCs.
* Does not merge SCCs - one is accepted and the rest are ignored.

## Summary

Hopefully you now have a good understanding of what SCCs are, and how you can use them to enforce pod security on an OpenShift cluster.

Key concepts covered include:

* To enable accessing protected resources, a deployer writes a deployment manifest for the pod that specifies...
  * A security context (for the pod and/or for each container) requesting the access needed by the application. This includes privileges, access control, and capabilities.
  * And a service account that the deployer expects to be able to grant this access.
* For the request to be granted, the cluster-admin must associate the service account with a security context constraint (SCC) that grants this access.
* SCCs are used to restrict pod capabilities, and that they can be tailored to allow specific pod permissions.
* An SCC may be one of OpenShift's predefined SCCs or may be a custom SCC.
* The cluster-admin associates the service account with the SCC, either directly or typically via an RBAC role.
* If the SCC grants the access, the pod deploys successfully.
* Resources can be constrained and isolated using OpenShift project namespaces.

To get hands-on experience using SCCs, there is a tutorial that was created to accompany this article. It can be found at [https://github.ibm.com/TT-ISV-org/scc/blob/main/tutorial/index.md](https://github.ibm.com/TT-ISV-org/scc/blob/main/tutorial/index.md).
