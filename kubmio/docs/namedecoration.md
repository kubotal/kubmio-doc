

# Name decoration

Before we describe each resource, we may say some words about the 'Name décoration' Kubmio feature.

As Kubmio resources (Buckets, AccessKey, Policies, ...) are namespaced resources, several one can be created with the
same name, in different namespace, resulting in clash on the MinIO server.

Besides this, a MinIO server can be accessed by several Kubernetes cluster, thus allowing another level of name conflict.

Requiring several independent teams to coordinate about object naming would be complex and error prone. 

To overcome this problem, Kubmio provide a mechanism of name decoration, allowing system administrators to define and
enforce naming rules aimed to guaranty uniqueness of MinIO resource name.

This decoration capability is built using the [Go Template](https://pkg.go.dev/text/template) with the the
[Sprig function template](http://masterminds.github.io/sprig/) library.

The template is defined by the `nameDecorator` tenant property. For example:

```
cat <<'EOF' | kubectl apply -f -
---
apiVersion: kubmio.com/v1alpha1
kind: Tenant
metadata:
  name: default
  namespace: kubmio
spec:
  endpoint: minio.myminioserver.myinfra.com
  credentials:
    secretName: minio-root
  ssl: false
  nameDecorator: "{{.Cluster}}-{{.Namespace}}-{{.Name}}"    
EOF
```

A Bucket of name `mybucket` created in the namespace `default` will result in a bucket created on the MinIO server
with the name `k8scluster1-default-mybucket`.

The MinIO resulting bucket name will be set in the `spec.minioName` property of the resource:

```
$ kubectl get buckets mybucket -o yaml
apiVersion: kubmio.com/v1alpha1
kind: Bucket
metadata:
  finalizers:
  - kubmio.com/finalizer
  name: mybucket
  namespace: default
  ... (Some metadata properties removed)
spec:
  minioName: k8scluster1-default-mybucket
  tenant:
    name: default
    namespace: kubmio
status:
  phase: READY

```

**For a tenant with the `nameDecorator` feature enabled, the `spec.minioName` of each resource is under control of Kubmio and must be left empty on object creation.**

The data model for Kubmio resource decoration templating host 6 variables:

- `Name`: The `name` property of the kubmio kubernetes resource.
- `Namespace`: The namespace hosting the kubmio kubernetes resource.
- `Cluster`: The cluster name, as defined by the configuration `clusterName` global property.
- `TenantName`: The tenant name.
- `TenantAlias`: The tenant alias
- `TenantNamespace`: The tenant namespace
