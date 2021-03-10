
# RBAC with SCC is sketchy

When trying create a role to use an SCC as we describe(d) in the article and seen in blogs and even official documentation, it does not work for me.  I get an error like this issue:

* error: can not perform 'use' on 'securitycontextconstraints' in group...

* link: https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner/issues/60

* Note: Currently looking like I'll basically use the workaround noted by the submitter (nobody has responded to the issue yet).

In addition, I've tried using add-role-to-user and when using local roles/rolebindings, I get a warning.  When using cluster roles/clusterrolebindings, it just doesn't work (no SCC to use). This feels like a namespace issue, but it really should be super simple to follow the command usage and it just doesn't work.

* Docs and lies: https://docs.okd.io/latest/authentication/managing-security-context-constraints.html#role-based-access-to-ssc_configuring-internal-oauth

So...

Stay tuned to the tutorial and article to see what works, but as I said. At this time, I have Role and RoleBinding YAML as described in the issue that I'll use as a workaround for those fine commands that let me down.

Alternatively, add-scc-to-user is the super simple working way of doing things w/o mucking it up -- but we'll make RBAC work anyway.

TODO: I guess I better update my `oc` CLI.  Could be that is the problem.


