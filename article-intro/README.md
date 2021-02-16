# Red Hat OpenShift security practices using security context contraints

To better understand how Security Context Contraints (**SCC**) are used to control access on a Red Hat OpenShift Container Platform, let's walk-through a common deployment scenario.

First, let's identify the personas that are typically involved in the process of developing an application, configuring a pod to contain the application, and then deploying that application on OpenShift.

* **Programmer** - responsible for developing the application or service that will run in the pod.
* **Deployer** - responsible for creating the deployment manifest that will define how the application will be run.
* **OpenShift User** - a user account granted deployment permissions by the OpenShift administrator.
* **OpenShift Service Account** - a special type of user account that can be created to be associated with one or more SCCs.
* **OpenShift Administrator** - ensures the security of the OpenShift platform by granting access to protected resources only as needed.

Administrators use Role-Based Access Control (**RBAC**) resources to control user access. In our scenario, the administrator will create an **OpenShift User** account that has the permission to deploy pods. The **Deployer** can then use that account to create and deploy the pod on OpenShift. The **Deployer** performs this task by creating a **deployment manifest**, which links to the application and provides instructions on how the application should be deployed and ran.

If the application is a typical stateless workload (i.e. requiring no special permissions), the **Deployer** will generate a basic deployment manifest, and the pod will be deployed without any issues.

But what if the application requires access to protected resources such as local storage, networking services, or needs to run under specific user or group identities? In this case the **Deployer** will need to enhance the deployment manifest to request the additional permissions. And who gets to decide if the requests should be allowed or not? This is where SCCs and RBAC come into play.

SCCs are the tool provided by OpenShift to control what permissions can be granted to a pod - specifically what privileges are allowed, what user and group IDs can be assigned, and what additional capabilities can be granted. SCCs are associated with users, groups, and/or service accounts - typically via RBAC roles. Therefore, when the  **Deployer** attempts to deploy the pod, the deployment manifest will specify what **OpenShift Service Accont** to use, which in turn is associated with a role and an SCC.

If the SCC allows all of the requested permissions made by the deployment manifest, the deployment is created and the pod/application is run.

If the pod is denied the requested permissions, the **OpenShift Administrator** will need to:

* Determine if the additional requests made in the manifest are in fact needed.
* Determine what SCC will allow the requested permissions, or if none exist, create a new one.
* Assign the SCC to a role for the appropriate users, groups, and/or service accounts, or create such a role if needed.
* Assign the new role to the **OpenShift Servvice Account**.

Here is an overview of how user and service accounts, roles, deployment manifiests, and SCCs are involved in the deployment process:

![flow](images/flow.png)

1. The **Programmer** develops an application or service, and ...
1. Delivers the application to the **Deployer**, who creates a deployment manifest for the application.
1. The **OpenShift Administrator** creates roles which are assigned a security context constraint.
1. The **OpenShift Administrator** creates an **OpenShift User Account** and assigns it a role that provides deployment permissions. Also created is an **OpenShift Service Account** that is associated with one or more SCCs.
1. The **Deployer** logs into OpenShift using the new **OpenShift User Account**, and deploys the application using the deployment manifest. The manifest may contain a request for additional permissions, and may reference which **OpenShift Service Account** to use when deploying the pod.
1. OpenShift processes the manifest and attempts to deploy the pod. The deployment process will compare the permissions requested by the deployment manifest against the permissions allowed by the associated SCCs.
1. If the associated SCC cannot provide all of the permissions the deployment manifest requests, the deployment will fail.
1. Otherwise, the deployment will create the pod and run the application.

TODO: do we need more detail on the actual deployment? what actually fails - deployment, pod creation, pod starting...

Now that we have a high-level view of how deployment manifests work with SCCs on OpenShift, let's dig into the details.

## Permissions requested vs. permissions allowed

In the flow diagram above, you see that the pod will only be deployed if the requested permissions defined in the deployment manifest match the allowed permissions defined by the SCC.

Another way to envision this relationship is to think of the SCC as a lock protecting system resources, and the manifest being the key. The app only gets access to the resources if the key fits.

![capabilities](images/capabilities.png)

## Types of permissions that can be requested

Up to this point we have used the generic term "permissions" to describe what pods can request and what SCCs can allow. In reality, "permissions" consist of 3 specific types - privileges, access, and capabilities. Each with its own set of rules and syntax.

### Privileges  

* **allowPrivilegedContainer** - can a pod run privileged containers
* **allowHostNetwork**
* **aloowHostPorts**

### Access

Controls what specific user or group ID a pod may run as.

In the SCC, The list of fields that can be set include:
  
* **RunAsUser** - specifies the allowable range of user IDs used for running all the containers in the pod.
* **SupplementalGroups** - specifies the allowable range of group IDs used for rulling all the containers in the pod.
* **FSGroup** -  specifies the allowable range of group IDs used for controlling pod storage volumes.
* **SELinuxContext** - specifies the allowable values used for setting the SELinux context, suas SELinux user, role, type and level.

In the deployment manifest, these values are set at the pod level and pertain to all containers running within the pod. The fields used are:

* **securityContext.runAsUser** - request to run under a specific user ID.
* **securityContext.runAsGroup** - request to run under a specific group ID.
* **securityContext.fsGroup** - request to run under a specific group ID for accessing storage volumes.
* **securityContext.XXXXXX** - request to run using a specific SELinux context.

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
oc get scc my-custom-scc -o yaml
```

## How to associate an SCC with a deployment manifest

Now that we have our custom SCC created, how do we get our deployment manifest to use it?

There a variety of ways to do this, but before we get into the details, let's make sure we know what objects are involved, and how to create them.

* Project
* RBAC Role
* Service Account
* SCC
* Deployment Manifest

### Project namespace

Using the OpenShift admin console, create a project that you can use to experiment with. Then using the CLI, set the default project to your new project name:

```bash
oc project rhagarty-project-1
```

We see the details of the project using the `describe` command:

```bash
oc describe project rhagarty-project-1
Name:			rhagarty-project-1
Labels:			<none>
Annotations:	openshift.io/description=
		    	openshift.io/display-name=
	    		openshift.io/sa.scc.mcs=s0:c25,c15
	    		openshift.io/sa.scc.supplemental-groups=1000630000/10000
	    		openshift.io/sa.scc.uid-range=1000630000/10000
...
```

Take note of the `openshift.io/sa.scc` values. These will be used as default values when processing SCCs later in this article.

The annotations define default values for (in order) SELinux options, group IDs, and user IDs. These values are only used if the corresponding SCC strategy is NOT **RunAsAny**, AND when not specified in the SCC yaml or deployment yaml.

### Roles

On OpenShift, an administrator can create a **Role** with a rule to define which SCCs will be available for all the users associated with that role.

There are 2 types of RBAC roles, **local** roles are limited to specific projects, while **cluster** roles apply to the entire OpenShift platform and all projects. For our scenario, we will just focus on **local** roles.

One way to create a role is to use a yaml file. For example:

```yaml
apiVersion: v1
kind: Role
metadata:
  name: my-custom-role
rules:
- apiGroups:
  -  security.openshift.io
  resourceNames:
  - my-custom-scc
  resources:
  - securitycontextconstraints
  verbs:
  - use
```

Note that the role is associated with the custom SCC we created in the last section.

Create the role using the command:

```bash
oc create -f my-custom-role.yaml -n rhagarty-project-1
```

### Service accounts

Each service account's user name is derived from its project and name:

```bash
system:serviceaccount:<project>:<name>
```

```bash
oc create sa my-custom-service-account
```

Use the following command to get the service account details:

```yaml
$ oc get sa my-custom-service-account -o yaml
apiVersion: v1
imagePullSecrets:
- name: my-custom-service-account-dockercfg-zgzx9
kind: ServiceAccount
metadata:
  creationTimestamp: "2021-02-16T19:46:00Z"
  name: my-custom-service-account
  namespace: rhagarty-project-1
  resourceVersion: "8920053"
  selfLink: /api/v1/namespaces/rhagarty-project-1/serviceaccounts/my-custom-service-account
  uid: 8c915529-897b-4eaa-860b-5b7ec5ae4583
secrets:
- name: my-custom-service-account-token-b2p5k
- name: my-custom-service-account-dockercfg-zgzx9
```

### Tie them all together

So how is it determined which SCC is used when a deployment manifest is processed? One way is to create a service account and assign it to an RBAC role. The SCC can then be assigned to that role - which we completed in the previous step.

#### Assign SCCs to RBAC roles

Once the role is created and associated with an SCC, we use the following commands to assign the role to a user account:

```bash
oc adm policy add-role-to-user my-custom-role -z my-custom-service-account -n rhagarty-poject-1
oc describe rolebinding.rbac -n rhagarty-project-1
```

#### Assign service accounts to an SCC

Another approach is to directly assign a service account to an SCC.

Here are the commands to create a service account, then assign it to our custom SCC:

```bash
oc adm policy add-scc-to-user my-custom-scc -z my-service-account
```

This will update the SCC to include a list of assigned users:

```yaml
$ oc get scc my-custom-scc -o yaml
...
kind: SecurityContextConstraints
apiVersion: v1
metadata:
  name: my-custom-scc
users:
- my-custom-service-acount
...
```

## Deployment Manifest details

A deployment manifest is used to create and build a deployment, which can be then used to deploy a pod.

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

Using the examples from above, let's walk through the OpenShift SCC admission process to determine if our pod gets deployed or not.

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

oc auth can-i get pods --subresource=log
oc policy add-rule-to-user edit user1

As an example, let's say that we have an application that needs special networking configurations. Let's say that we need to configure one interface, open a port on the system's firewall, create a NAT rule for that and punt a new custom route on the system's routing table. But you don't need to make arbitrary changes to any file in the system. We can set CAP_NET_ADMIN instead of running the process as a privileged one.

Beyond privileges and capabilities we have SELinux and AppArmor that are both kernel security modules that can be added on top of capabilities to get even more fine grained security rules by using access control security policies or program profiles. In addition, we have Seccomp which is a secure computing mode kernel facility that reduces the available system calls to the kernel for a given process.

When first requesting a pod to the API server, the credential to authorize the pod will be the user account requesting it. After that, the pod itself will be running under its service account. If we don't specify a service account for our pod it is automatically assigned the default service account available on the namespace it’s running. But every pod will run under a service account. So based on the user, service account and/or groups that the service account belongs to, the admission process responsible for checking the requested privileges will find the set of SCCs available and verify if there is a match between the requested resource security context and the constraints. If there is a match the pod is accepted otherwise it's rejected.


Cover ## availablecapabilities/constraints (SELinuxpolicies,AppArmorprofiles,  etc.)

When an application is deployed it will run as a user ID unique to the project it is running in. This overrides the user ID which the application image defines it wants to be run as. That the application is run as a different user ID can result in it failing to start.

The best solution is to build the application image so it can be run as an arbitrary user ID. This avoids the risks associated with having to run an application as the root user ID, or other fixed user ID which may be shared with applications in other projects.

If an image can't be modified, you can elect to override the default security configuration of OpenShift and have it run as the user the image specifies, but this can only be done by an administrator of the OpenShift cluster. This cannot be done by normal developers, nor a project administrator. This cannot be done on hosting services such as OpenShift Online.

The change required to override the default security configuration, is to grant rights to the **service account the application is run under**, to run images as a set user ID.

By default applications would run under the restricted SCC. The MustRunAsRange value for RUNASUSER is what indicates that the application needs to run within the user ID range associated with the project.

To allow an application to be run as any user ID, including the root user ID, you want to use the anyuid SCC

## Test

Volume security - https://docs.openshift.com/enterprise/3.1/install_config/persistent_storage/pod_security_context.html

```bash
# set current project
oc project rhagarty-project-1
# create a simple role that can view pods
oc create role scc-rhagarty-role-1 --verb=get --resource=pod -n rhagarty-project-1
# make sure it exists by viewing all roles in the current project
oc describe rolebinding.rbac
# get more details on just our new role
oc describe roles/scc-rhagarty-role-1
# add this new role to our user (-z indicates user is a service account)
oc adm policy add-role-to-user scc-rhagarty-role-1 -z rich-scc-test-1


oc create role scc-rhagarty-role-1 --verb=use --resource=securitycontextcontraints -n rhagarty-project-1

oc adm policy who-can use securitycontextcontraints
```

```
oc policy add-role-to-user view system:serviceaccont:top
```
