# Kubernetes / MinIO impedance mismatch.

Each Kubmio resources has two instantiations:

- Its k8s representation, as k8s object
- Its Minio instantiation.

These two implementation does not follow the same rules. This usual behavior for Kubernetes is to relax (do not implements)
all referential constraints between objects. For example, one can successfully define a RoleBinding linking an un-existing
role to an un-existing serviceAccount. May be the role and the serviceAccount will be defined later.

We need to preserve this behavior, as it is at the hearth of Kubernetes. (For example, all deployments tool, such as helm,
are unable to managed dependencies between resources).

For the `PolicyBinding => Policy` and `PolicyBinding => subject`, there is a problem, as theses relationships can't be
established on the Minio server if one or both side does not exists.

The same problem apply for the `GroupBinding = Group` and `GroupBinding => User` relationship

So, to be compliant with the k8s behavior, a `PolicyBinding` or `GroupBinding` k8s resource will be successfully created in all cases, and if one
of its reference is missing, it will be set in an UNBOUND state. No action on the Minio server will be performed in such case.

Then, the PolicyBinding or GroupBinding controllers will watch for User, Group and Policy creation (or deletion) to adjust its status accordingly.

For example, if you crate a PolicyBinding between two un-existing resources, you will have the following resulting status:

```
status:
  isGroup: false
  phase: UNBOUND
  policyMinioName: ""
  subjectMinioNameOrDn: ""
```

And, if you create User2, the status will be:

```
status:
  isGroup: false
  phase: UNBOUND
  policyMinioName: ""
  subjectMinioNameOrDn: user2
```

And when you create Policy2, the status will be:

```
status:
  isGroup: false
  phase: READY
  policyMinioName: policy2
  subjectMinioNameOrDn: user2
```

And the attachment will be effective on the MinIO server.

