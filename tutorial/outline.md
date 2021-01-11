# Outline for SCC tutorial (draft)

Start with a "hello world" program that needs access to a protected resource such as a file system.
1. Deploy the program as a container in a pod, run the program.
   - `kubectl|oc apply <Deployment yaml file>`
   - No error (surprisingly!). If so, probably because kubectl/oc is logged in as the cluster admin, and he can do anything.
1. As a [normal user with limited access](https://www.openshift.com/blog/managing-sccs-in-openshift): Deploy the program as a container in a pod, run the program.
   - `kubectl|oc apply <Deployment yaml file> --as=my-unprivileged-user`
   - Error: Program cannot access file system. Now we're getting somewhere; by default, storage can't be accessed.
1. Configure the Deployment's container to request SCC capabilities for file system access and deploy that.
   - The Deployment spec will look something like this [Pod spec](https://www.openshift.com/blog/introduction-to-security-contexts-and-sccs):
    ```yaml
    apiVersion: v1
    kind: Pod
    metadata:
        name: security-context-demo-4
    spec:
        containers:
            - name: sec-ctx-4
              image: gcr.io/google-samples/node-hello:1.0
              securityContext:
                capabilities:
                  add: ["NET_ADMIN", "SYS_TIME"]
   ```
   - `kubectl|oc apply <Deployment yaml file> --as=my-unprivileged-user`
   - Error: The pod isn't allowed to request that capability. The request failed. (Maybe followed by the access error in the program.)
1. Find a predefined SCC with file system access (or define a new SCC).
   - List the eight predefined SCCs: `oc get scc`
   - Find one of them that allows file system access.
   - (For a capability that's not in a predefined SCC, we'll need to define a custom SCC.)
   - The SCC we'll use is: `<my scc>`

Now we've got a pod that needs a capability and an SCC that grants the capability. The trick is to associate the SCC with the pod, so that when the pod requests certain capabilities, the SCC hopefully has them and the request will be granted.
1. Define an RBAC role and add the SCC to the role.
   - Something like [this example](https://www.openshift.com/blog/managing-sccs-in-openshift):
     ```yaml
     apiVersion: rbac.authorization.k8s.io/v1
     kind: Role
     metadata:
        namespace: default
        name: role-name
     rules:
     - apiGroups: 
       - security.openshift.io
       resourceNames:
       - <my scc>
       resources:
       - securitycontextconstraints
       verbs:
       - use
     ```
1. Bind the role to the user.
   - Define a RoleBinding for the role (`role-name`) and the user (`my-unprivileged-user`).
1. Now the program should be able to access the file system.
   - Restart the pod or retest the program.
   - The program accesses the file system successfully. It does not give a no-access error. The pod does not give an error that the capability it requested can't be granted.

Notice that we didn't deploy the pod again, we just tested it again. Before, it didn't work. But after we gave the user a role that had an appropriate SCC, suddenly the pod now has access and the program works.
