
# Quick start

## Deployment

To install Kubmio, just use the corresponding Helm chart:

```
helm -n kubmio upgrade -i --create-namespace kubmio oci://quay.io/kubotal/charts/kubmio --version 0.4.0-snapshot
```

Helm chart default values should be fine for a standard installation.

Three pods should now be running on the `kubmio` namespace:

```
$ kubectl -n kubmio get pods
NAME                                 READY   STATUS    RESTARTS      AGE
kubmio-supervisor-66ff799f45-r8mh8   1/1     Running   9             10h
kubmio-sweeper-5d7f7b798b-xsmjq      1/1     Running   0             10h
kubmio-webhooks-68c4f4cb68-5g6nx     1/1     Running   0             10h
```

## Tenant creation

Once kubmio is deployed, you must configure a connection to your MinIO server.
Such connection, called `tenant` is defined as a kubernetes resources.

But prior to this, you must create a secret hosting the MinIO access credentials. For example:

```
kubectl -n kubmio create secret generic minio-root --from-literal=rootUser=minio --from-literal=rootPassword=xxxxxxxxx
```

And then the tenant can be created:

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
EOF
```

Of course, you must adjust the `spec.endoint` value. And, if you MinIO access is encrypted with SSL, see below for its configuration

> Deploying this first tenant in the `kubmio` namespace, and naming it `default` is meaningful. See below

After few time, your tenant should reach the READY state:

```
$ kubectl -n kubmio get tenants
NAME      ALIAS     DESCRIPTION   ENDPOINT                       SSL    STATUS   DISAB.   DISCOM.   AGE
default   default                 minio1-ext.ingress.kubo2.mbp   true   READY    false    false     3m50s
```

A specific tenant operator deployment has been setup, with two replicas:

```
$ kubectl -n kubmio get pods
NAME                                              READY   STATUS    RESTARTS   AGE
kubmio-operator-kubmio-default-66bfdbdcc7-5hjf8   1/1     Running   0          2m42s
kubmio-operator-kubmio-default-66bfdbdcc7-9qqhz   1/1     Running   0          2m42s
kubmio-supervisor-5589f55c8-nfl8w                 1/1     Running   0          4h21m
kubmio-sweeper-5d7f7b798b-s9vq7                   1/1     Running   0          5h25m
kubmio-webhooks-677864df6d-w9wdh                  1/1     Running   0          4h21m
```

Several `tenants` can be defined in the same Kubernetes cluster. This will allow to address uses case such as:

- Several MinIO servers
- Access to the same server, but with different configuration.

Each other Kubmio resource may define a specific tenant. If they don't, they will use the one named `default` in the `kubmio` namespace.

### SSL configuration

If your MinIO server is secured with SSL (It should be), then you must enable its support and provide the corresponding
Certificate Authority:

```
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
  ssl: true
  certificateAuthority:
    value: "LS0tLS1CRUdJTiBDRVJUSUZJQ................................................LS0tRU5EIENFUlRJRklDQVRFLS0tLS0K"
```

The `ca` property is the base64 encoded certificate authority (CA). The CA is generally provided to you as a file with a content which look like:

```
-----BEGIN CERTIFICATE-----
MIIGSzCCBDOgAwIBAgIJAN3rPrHNIFfAMA0GCSqGSIb3DQEBCwUAMHUxCzAJBgNV
BAYTAkZSMQ4wDAYDVQQIDAVQYXJpczEOMAwGA1UEBwwFUGFyaXMxGTAXBgNVBAoM
EE9wZW5EYXRhUGxhdGZvcm0xFjAUBgNVBAsMDUlUIERlcGFydG1lbnQxEzARBgNV
....
J2D93BTA8z5cto4I5oCtfQ2GjlkfEJG863gcIT/3ieu3AI/+LATFO7+TYVqYY8SI
wDQVxs1wOpHZOEekfO4fKW12BQ+f+K9m+j0ISFzUCA==
-----END CERTIFICATE-----
```

On a Linux or Mac OS system, if the file is named `ca.crt`, the value to paste in the `ca:` property of the above configuration is the result of:

```
cat ca.crt | base64 
```

For testing purpose, you may skip this certificate validation:

```
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
  ssl: true
  noCheckCertificate: true
```


## Bucket creation

Once a tenant is setup successfully, you may create your first 'kubmio managed' MinIO bucket:

```
cat <<'EOF' | kubectl apply -f -
---
apiVersion: kubmio.com/v1alpha1
kind: Bucket
metadata:
    name: bucket1
spec:
EOF
```

After few time, your bucket should reach the READY state:

```
$ kubectl get --all-namespaces buckets
NAMESPACE   NAME      TENANT NS   TENANT    MINIO NAME   STATUS   DESCRIPTION   AGE
default     bucket1   kubmio      default   bucket1      READY                  29s
```

> May be the word `bucket` is also used by another application. For such case, `kubmio` provide an alias: `mbucket` or
`mbuckets`. Of course, you can also use the fully qualified name: <br>`kubectl get buckets.kubmio.com`

You can check this with the MinIO console, or the `mc` MinIO CLI (here `minio` is the alias of your server).

```
$ mc ls minio
[2024-07-11 17:37:40 CEST]     0B bucket1/

$ mc stat minio/bucket1
Name      : bucket1
Date      : 2024-07-11 18:21:51 CEST
Size      : N/A
Type      : folder

Properties:
  Versioning: Un-versioned
  Location: us-east-1
  Anonymous: Disabled
  ILM: Disabled
```

You can also check the resulting K8s bucket manifest:

```
$ kubectl get bucket bucket1 -o yaml
apiVersion: kubmio.com/v1alpha1
kind: Bucket
metadata:
  finalizers:
  - kubmio.com/finalizer
  name: bucket1
  namespace: default
  ... (Some metadata properties removed)
spec:
  minioName: bucket1
  tenant:
    name: default
    namespace: kubmio
status:
  phase: READY

```

Some points to mention:

- There was no namespace provided in the bucket creation. So the bucket K8s object lands on the `default` namespace.
  By default, a bucket can be created in any namespace. This can be restricted in the `tenant` definition.
- The `spec.minioName` property is set to the resource name by default. It is the name of the bucket on the MinIO server.
- The `spec.tenant` property is set to the default tenant, a tenant named `default` in the `kubmio` namespace.
  Default tenant name and namespace can be configured during initial Helm deployment.
- The `status.phase` is set to READY, meaning the bucket creation on the MinIO server is successful.
- There is a `metadata.finalizers`. This will be used during the Bucket deletion processing.
- The `status.phase` provide information about the current object state. `READY` means the MinIO object counterpart has
  been successfully created.


## Follow up

You can now deploy others Kubmio/Minio resources such as Users, Groups, AccessKey, Policy and PolicyBinding. You will find
indication and sample on each resource kind definition below in this document.

