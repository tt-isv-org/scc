# User privileges and security

This note is regarding how to secure the privileges that the cluster admin is granting in an SCC.

To recap, we are:

* As cluster admin:
  1. Granting privileges to an SCC
  2. Granting the SCC to a role
  3. Assigning the role to a service account

So...  How do we control who gets to use the service account?

## Namespace (or Project)

The first and most obvious authorization is to use namespaces. Limit the service accounts in a project to the minimum privileges for that project.

* Remove service accounts / roles from projects that don't need them
* Keep the service accounts / roles / privileges in a namespace to the minimum needed by that project/application
* Avoid cluster-wide...

## Create / update resources (i.e. be a "Deployer")

* An average user should not be allowed to create a deployment -- or even a pod.

* An average user should not be allowed to impersonate other users.

* When you add CREATE / DEPLOYMENT permissions in a namespace to a user, then you are giving up everything you've given to all the service accounts in that namespace.

* So, the SA privileges (granted via roles) are in fact limited to users that have permission to create (or update...) deployments, pods, or other controllers

TBD: Is there really, really, really no way to restrict which SA a deployer can specify finer than project-level?

## Conclusion

1. Only give a project the minimum necessary SCCs
1. Grant them to roles, if that adds clarity
1. Only assign SAs the minimum roles (SCCs)
1. Only grant resource permissions (such as create deployment) to users that should have all the SA privileges

## Link to the authority

Seem like this statement might end the discussion, but since there are sooooo many ways to do things (see TBD above)...  In the meantime, refer to this link (text copied below) and secure your projects / roles / SCCs/ user auth.

Link: https://kubernetes.io/docs/reference/access-authn-authz/authorization/

The below text is copied from the above link.

## Privilege escalation via pod creation

Users who have the ability to create pods in a namespace can potentially escalate their privileges within that namespace. They can create pods that access their privileges within that namespace. They can create pods that access secrets the user cannot themselves read, or that run under a service account with different/greater permissions.

> Caution: System administrators, use care when granting access to pod creation. A user granted permission to create pods (or controllers that create pods) in the namespace can: read all secrets in the namespace; read all config maps in the namespace; and impersonate any service account in the namespace and take any action the account could take. This applies regardless of authorization mode.
