# kcc-demo
Using the GCP (Kubernetes) Config Connector (KCC) to provision GCP infra directly from k8s.

- [kcc-demo](#kcc-demo)
  - [Introduction](#introduction)
  - [Prerequisites](#prerequisites)
  - [Before you begin](#before-you-begin)
  - [Configure target GCP project](#configure-target-gcp-project)
  - [Configure cluster](#configure-cluster)
  - [Managing GCP resources](#managing-gcp-resources)
    - [Creating a managed resource](#creating-a-managed-resource)
  - [Clean up](#clean-up)
  - [Resources](#resources)

## Introduction
This repo show how to provision GCP infra from a k8s cluster outside of GCP (i.e. non-GKE). Note the approach below is purely for demoing from a local cluster - it should NOT be seen as a secure way of deploying infra in production envs; for that see the installation on GKE clusters using Workload Identity. It does however demonstrate the principle of using the k8s control plane and declarative methods to control the state of infrastructure. In particular, we can demonstrate how the principle of how [Kubernetes Controller loops](https://kubernetes.io/docs/concepts/architecture/controller/) can be used to maintain state of objects both inside and outside of k8s clusters. Combining with GitOps CD tools such as ArgoCD, we can show how we can control infrastructure state directly from Git using the KCC.

This demos controlling resources at the _project_ level - though this approach can be used to control resources at folder and Org levels.


## Prerequisites
`kubectl` access to k8s cluster (tested with `Docker Desktop for Mac`, but `kind`, `minikube`, `k3s` etc should all be ok).

NOTE: KCC can be pretty resource hungry - if demoing on local machine eg Docker For Desktop k8s on Mac - ensure you have enough CPU/RAM assigned (suggest min 4CPUs/8Gb). Otherwise you may be better using a GKE cluster specc'd appropriately

Ability to create roles i.e. 
`kubectl auth can-i create roles` - answer should be 'yes'

Access to an appropriate GCP project via `gcloud` with ability to create service accounts, enable APIs etc


## Before you begin

Set up env vars for following steps:
```
PROJECT_ID=stuart-dev-example-01
SERVICE_ACCOUNT_NAME=gcp-cc-sa
SECRET_NAME=gcp-cc-secret
```

## Configure target GCP project

Ensure access via `gcloud` and current context
`gcloud config list`

Enable resource manager API
`gcloud services enable cloudresourcemanager.googleapis.com`

Create a service account for creds used by KCC - note requires 'owner' or 'editor' role (editor appears fine for project level control)

```
gcloud iam service-accounts create $SERVICE_ACCOUNT_NAME

gcloud projects add-iam-policy-binding $PROJECT_ID \
    --member="serviceAccount:$SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com" \
    --role="roles/editor" 
```

This will be used to retrieve credentials to use within the k8s cluster

```
gcloud iam service-accounts keys create --iam-account \
    $SERVICE_ACCOUNT_NAME@$PROJECT_ID.iam.gserviceaccount.com key.json
```

## Configure cluster
See https://cloud.google.com/config-connector/docs/how-to/install-other-kubernetes - (note this is different for GKE clusters as these can use Workload Identity - see appropriate docs). The following 
works ok on a local Docker/k8s single node cluster

Pull the latest distribution of the GCP Config Connector: 

```
gsutil cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz

tar zxvf release-bundle.tar.gz
```

Create a namespace for the kcc - this _must_ be named `cnrm-system` - and add the secret you created from the above service account:
```
kubectl create namespace cnrm-system 

kubectl create secret generic $SECRET_NAME \
    --from-file key.json \
    --namespace cnrm-system
```

Delete the key(!)
```
rm key.json
```

Install the KCC operator:
```
kubectl apply -f operator-system/configconnector-operator.yaml
```

Configure the operator - amend the [](configconnector.yaml) file to include a reference to the secret you created in the `cnrm-system` ns, and apply as follows:
```
kubectl apply -f configconnector.yaml
```

Check all is ok (make sure containers all started)

```
kubectl wait -n cnrm-system \
      --for=condition=Ready pod --all
```
You should see
```
pod/cnrm-controller-manager-0 condition met
pod/cnrm-deletiondefender-0 condition met
pod/cnrm-resource-stats-recorder-848dbbf897-jhhmq condition met
pod/cnrm-webhook-manager-5ccc747594-78gbh condition met
pod/cnrm-webhook-manager-5ccc747594-zmbxl condition met
```

Create namespace for resources managed by kcc (can be anything) e.g.

```
kubectl create ns kcc
```

Annotate this ns to control resources at project level, eg:

```
kcc-demo % kubectl annotate namespace \
 kcc cnrm.cloud.google.com/project-id="YOUR_PROJECT_ID"
```

Should be good to go at this point. You can see the types of GCP resources you are able to create with KCC via 
```
 kubectl get crds | grep '.cnrm.cloud.google.com'
 ```
 or 
 ```
 kubectl get crds --selector cnrm.cloud.google.com/managed-by-kcc=true
 ```
or look at the resources documentation [here](https://cloud.google.com/config-connector/docs/reference/overview)


## Managing GCP resources

### Creating a managed resource
Given the correct project API permissions, you can now use 

Enable API [](samples/enable_pubsub.yaml)
```
kubectl apply -f samples/enable_pubsub.yaml -n kcc
```

Create the resource
```
kubectl apply -f samples/pubsub_topic.yaml -n kcc
```

Monitor it to see if it came up ok:
```

```


View what's happening with your resources

```
kubectl get events -n kcc
```

Get all managed resources:
```
kubectl get gcp -n kcc
```

To list resources you have created e.g. storage buckets:
```
kubectl get storagebuckets -n kcc

kubectl describe storagebucket stuart-dev-example-01-b1 -n kcc
```

KCC is eventually consistent - i.e. there may be some time between a mutating change being applied and being visible. 

The reconciliation loop of the KCC is around 10 mins (not currently changeable?) - so if you mod stuff directly in GCP, it will not resolve to the definition of state in your custer for a period of a few mins at least - you can always force the behaviour by re-performing the `kubectl apply` if you want to demonstrate config drift being automatically remediated by your cluster.

## Clean up

Delete all managed resources first 

(note if you want to keep the resources running for whichever reason you must first 'unmanage' them from KCC - unless you had tagged them as 'abandon')

To remove the KCC CRDs and operators etc:

```
kubectl delete ConfigConnector configconnector.core.cnrm.cloud.google.com \
    --wait=true

kubectl delete -f operator-system/configconnector-operator.yaml  --wait=true
```

## Resources

https://cloud.google.com/config-connector/docs/reference/overview