---
title: "Google Cloud DNS"
linkTitle: "Google Cloud DNS"
weight: 30
type: "docs"
---

This guide explains how to set up an `Issuer`, or `ClusterIssuer`, to use Google
Cloud DNS to solve DNS01 ACME challenges. It's advised you read the [DNS01
Challenge Provider](../) page first for a more general understanding of
how cert-manager handles DNS01 challenges.

> Note: This guide assumes that your cluster is hosted on Google Cloud Platform
> (GCP) and that you already have a domain set up with CloudDNS.

## Set up a Service Account

cert-manager needs to be able to add records to CloudDNS in order to solve the
DNS01 challenge. To enable this, a GCP service account must be created with the
`dns.admin` role.

> Note: For this guide the `gcloud` command will be used to set up the service
> account. Ensure that `gcloud` is using the correct project and zone before
> entering the commands. These steps could also be completed using the Cloud
> Console.

```bash
$ PROJECT_ID=myproject-id
$ gcloud iam service-accounts create cert-manager --display-name "Cert Manager"
```

In the command above, replace `myproject-id` with the ID of your project.

```bash
$ gcloud projects add-iam-policy-binding $PROJECT_ID \
   --member serviceAccount:cert-manager@$PROJECT_ID.iam.gserviceaccount.com \
   --role roles/dns.admin
```

> **Note**: The use of the `dns.admin` role in this example role is for convenience.
> If you want to ensure cert-manager runs under a least privilege service account,
> you will need to create a custom role with the following permissions:
>
>  * `dns.resourceRecordSets.*`
>  * `dns.changes.*`
>  * `dns.managedZones.list`

## Use Workload Identity

[Workload Identity](cloud.google.com/kubernetes-engine/docs/how-to/workload-identity) is the recommended way to access Google Cloud services from applications running 
within GKE due to its improved security properties and manageability.

To access this service account, you will need to annotate the Kubernetes Service Account of cert-manager by running the following command, replacing instances of `$PROJECT_ID` with the ID of your project. :

```bash
$ helm upgrade --install \
  cert-manager jetstack/cert-manager \
  --namespace cert-manager \
  --version v0.16.1 \
  --set installCRDs=true --set serviceAccount.annotations."iam\.gke\.io/gcp-service-account"=cert-manager@$PROJECT_ID.iam.gserviceaccount.com
```

If you did not create the service account `cert-manager` before, you need to
create it first.

```bash
$ gcloud iam service-accounts create cert-manager
```

Replace instances of `$PROJECT_ID` with the ID of your project. This command 
assumes you installed cert-manager in the cert-manager namespace. If you didn't, then amend the namespace accordingly.

```bash
$ gcloud iam service-accounts add-iam-policy-binding \
--role roles/iam.workloadIdentityUser \
--member serviceAccount:$PROJECT_ID.svc.id.goog[cert-manager/cert-manager] cert-manager@$PROJECT_ID.iam.gserviceaccount.com \
--project $PROJECT_ID
$ 
```

### Create an Issuer That Uses CloudDNS

Next, create an `Issuer` (or `ClusterIssuer`) with a `cloudDNS` provider. An
example `Issuer` manifest can be seen below with annotations.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    ...
    solvers:
    - dns01:
        cloudDNS:
          # The ID of the GCP project
          project: $PROJECT_ID
```

For more information about `Issuers` and `ClusterIssuers`, see
[Configuration](../../../).

Once an `Issuer` (or `ClusterIssuer`) has been created successfully, a
`Certificate` can then be added to verify that everything works.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    # The issuer created previously
    name: example-issuer
  dnsNames:
  - example.com
  - www.example.com
```

For more details about `Certificates`, see [Usage](../../../../usage/).

## GKE Workload Identity

If you are deploying cert-manager into a [Google Container Engine (GKE)
cluster](https://cloud.google.com/kubernetes-engine/) with workload identity
enabled, you can leverage workload identity to avoid creating and managing
static service account credentials. The [workload identity
how-to](https://cloud.google.com/kubernetes-engine/docs/how-to/workload-identity)
provides more detail on how workload identity functions, but briefly workload
identity allows you to link a Google service accounts (GSA) to Kubernetes
service accounts (KSA). This GSA/KSA linking is two-way, i.e., you must
establish the link in GCP _and_ Kubernetes. Once configured, workload identity
allows Kubernetes pods running under a KSA to access the GCP APIs with the
permissions of the linked GSA. The workload identity how-to also provides
detailed instructions on how to enable workload identity in your GKE cluster.
The instructions in the following sections assume you are deploying cert-manager
to a GKE cluster with workload identity already enabled.

### Link KSA to GSA in GCP

The cert-manager component that needs to modify DNS records is the pod created
as part of the cert-manager deployment. The [standard methods for deploying
cert-manger to Kubernetes](../../../../installation/kubernetes/) create the
cert-manager deployment in the cert-manager namespace and its pod spec specifies
it runs under the cert-manager service account. To link the GSA you created
above to the cert-manager KSA in the cert-manager namespace in your GKE cluster,
run the following command.

```bash
$ gcloud iam service-accounts add-iam-policy-binding \
    --role roles/iam.workloadIdentityUser \
    --member "serviceAccount:$PROJECT_ID.svc.id.goog[cert-manager/cert-manager]" \
    dns01-solver@$PROJECT_ID.iam.gserviceaccount.com
```

If your cert-manager pods are running under a different service account, replace
`goog[cert-manager/cert-manager]` with `goog[NAMESPACE/SERVICE_ACCOUNT]`, where
`NAMESPACE` is the namespace of the service account and `SERVICE_ACCOUNT` is the
name of the service account.

### Link KSA to GSA in Kubernetes

After deploying cert-manager, add the proper workload identity annotation to the
cert-manager service account.

```bash
$ kubectl annotate serviceaccount --namespace=cert-manager cert-manager \
    "iam.gke.io/gcp-service-account=dns01-solver@$PROJECT_ID.iam.gserviceaccount.com"
```

Again, if your cert-manager pods are running under a different service account,
replace `--namespace=cert-manager cert-manager` with `--namespace=NAMESPACE
SERVICE_ACCOUNT`, where `NAMESPACE` is the namespace of the service account and
`SERVICE_ACCOUNT` is the name of the service account.

If you are deploying cert-manager using its helm chart, you can use the
`serviceAccount.annotations` configuration parameter to add the above workload
identity annotation to the cert-manager KSA.

### Create an Issuer That Uses CloudDNS

Next, create an `Issuer` (or `ClusterIssuer`) with a `clouddns` provider. An
example `Issuer` manifest can be seen below with annotations. Note that the
issuer does not include a `serviceAccountSecretRef` property. Excluding this
instructs cert-manager to use the default credentials provided by GKE workload
identity.

```yaml
apiVersion: cert-manager.io/v1
kind: Issuer
metadata:
  name: example-issuer
spec:
  acme:
    ...
    solvers:
    - dns01:
        cloudDNS:
          # The ID of the GCP project
          project: $PROJECT_ID
```

For more information about `Issuers` and `ClusterIssuers`, see
[Configuration](../../../).

Once an `Issuer` (or `ClusterIssuer`) has been created successfully, a
`Certificate` can then be added to verify that everything works.

```yaml
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: example-com
  namespace: default
spec:
  secretName: example-com-tls
  issuerRef:
    # The issuer created previously
    name: example-issuer
  dnsNames:
  - example.com
  - www.example.com
```

For more details about `Certificates`, see [Usage](../../../../usage/).
