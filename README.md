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
    - [Updating a resource](#updating-a-resource)
    - [Dependent Resources](#dependent-resources)
    - [Reverse engineering existing resources into YAML](#reverse-engineering-existing-resources-into-yaml)
    - [Deleting a resource](#deleting-a-resource)
  - [General stuff for KCC](#general-stuff-for-kcc)
  - [Troubleshooting](#troubleshooting)
  - [GitOps managed Infra](#gitops-managed-infra)
    - [Using ArgoCD](#using-argocd)
    - [Using Google ConfigSync](#using-google-configsync)
  - [Clean up](#clean-up)
  - [Future ideas](#future-ideas)
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
Given the correct project API permissions, you can now use `kubectl` operations to create, manage, modify and delete GCP resources. Firstly, make sure the correct APIs are enabled for the resources you are considering creating - whether via the console or, if you wish, via KCC, e.g. for pubsub

Enable API [](samples/enable_pubsub.yaml)
```
kubectl apply -f samples/enable_pubsub.yaml -n kcc
```

(To view APIs you enabled using KCC, use ```kubectl get service.serviceusage -n kcc ```)

Check the API has been enabled: ```kubectl describe service.serviceusage pubsub.googleapis.com -n kcc```

Then create the resource:
```
kubectl apply -f examples/pubsub_topic.yaml -n kcc
```

See all objects of that type in the namespace
```
kubectl get pubsubtopic -n kcc
NAME        AGE   READY   STATUS     STATUS AGE
kcc-topic   64s   True    UpToDate   62s
```

Examine it to see if it came up ok:
```
kubectl describe pubsubtopic kcc-topic -n kcc
```

You should now see your desired resource in the console also

### Updating a resource
You can demonstrate how the k8s reconciliation loop of the ConfigConnector is used to resolve the declared state in the cluster with the actual resource by modifying the yaml used to recreate - e.g. add a label to the existing pubsub topic:
```
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  labels:
    env: test
    director: kubrick
  name: kcc-topic
```
Re-apply the yaml - do a `describe` to see that the object in the cluster has been updated with the  new label. You should see the updated label next to the topic (you may need to refresh the console).

Now _in the console_ - delete the label you just added to the topic. Over time, the ConfigConnector controller will check the status of the resource it is managing, and will attempt to resolve any discrepancies it discovers - in this case, the removed label. (Unfortunately, the default polling period is 10mins, so you may have to wait a while before this gets resolved - but eventually the label will re-appear so that the resource matches the definition in the cluster -i.e. it is being _managed_ by KCC.

Note: not all properties can be altered in this fashion, depending on whether they are deemed immutable, or indeed whether operations are simply not allowed e.g. you can _increase_ the storage for a PostgreSQL instead, but not reduce it (you need to delete/recreate).

### Dependent Resources  
Resources can be declared with references to other resources - see the [](gcp-gitops/deps) folder for an example of a pubsub subscription referencing an existing topic

### Reverse engineering existing resources into YAML
In principle, it should be possible to export existing GCP resources into yaml, to ten subsequently impor and manage under KCC. However this was not seen to work due ot what appears to be an IAM related issue and Cloud ASset manager - see [notes](export-WIP/export-import%20WIP.md) to reproduce. 

This could potentially be very powerful for reverse engineering manually created infra e.g for PoCs or demos and then allowing for it to be spun up automatically from KCC - however without testing this remains an unknown in terms of limitations.

### Deleting a resource
Just delete the resource like any k8s resource, i.e.
```
kubectl delete pubsubtopic kcc-topic -n kcc
```
*This will result in the deletion of the resource itself on GCP*

If you don't want the deletion of the manged resource object to result in the deletion of the _actual_ resource, you need to _unmanage_ the resource from KCC - you do this by adding the following to your resource yam definition:

```cnrm.cloud.google.com/deletion-policy: abandon```

i.e.
```
apiVersion: pubsub.cnrm.cloud.google.com/v1beta1
kind: PubSubTopic
metadata:
  annotations:
    cnrm.cloud.google.com/deletion-policy: abandon
  labels:
...
```

## General stuff for KCC
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


## Troubleshooting
As mentioned, KCC can be very resource hungry - sometimes its quicker just to drop and restart the cluster if having memory or timeout issues.

For general stuff - https://cloud.google.com/config-connector/docs/troubleshooting

## GitOps managed Infra  
### Using ArgoCD
If you have ArgoCD setup, you can easily use this to manage resources in kcc - the [ArgoCD quickstart]()https://argo-cd.readthedocs.io/en/stable/getting_started/ docs should be sufficient for your cluster (this has been tested with ArgoCD running on the same cluster as KCC, but no reason why it couldn't be hosted elsewhere). For demo purposes, using argo's reconciliation loop also gets round the issue of waiting for KCC to resolve (the default argo reconciliation is 3mins). 

Firstly - make sure you have forked this repo (if not done already) to somewhere where you have commit access, as you will be wanting to commit changes and see them reflected.

For convenience a set of commands needed for a local cluster are included [here](examples/argo-test/argo_install.txt) - create a simple argo application with defaults pointing to the [argo-test](examples/argo-test/) folder to setup a deployment of the nginx [yaml](examples/argo-test/nginx_deployment.yaml). Turn on auto-sync, and test that changing say, the number of replicas in th deployment is reflected on your cluster.

If that all works ok, you should be good to create an application that points to the [](gcp-gitops/buckets) folder - the target namespace needs to be the one you annotated for usage by KCC as above. You should see argo deploy the KCC definitions as above and see the infra being managed - similarly, pushing changes to the git repo will be reflected the next time argo syncs the changes from the repo to you cluster. 

### Using Google ConfigSync
As a GCP alternative, [ConfigSync](https://cloud.google.com/kubernetes-engine/docs/add-on/config-sync) could be installed to manage the repo syncing - however its primary use case is for synchronizing configurations and policies. You need to define a separate repo, but you can use it to manage to manage GCP infra in the same fashion.

ConfigController and ConfigSync are combined in this way in [Anthos Config Management](https://cloud.google.com/anthos-config-management/docs/overview) along with BinAuthz, PolicyController etc.


## Clean up

Delete all managed resources first 

(note if you want to keep the resources running for whichever reason you must first 'unmanage' them from KCC - unless you had tagged them as 'abandon')

To remove the KCC CRDs and operators etc:

```
kubectl delete ConfigConnector configconnector.core.cnrm.cloud.google.com \
    --wait=true

kubectl delete -f operator-system/configconnector-operator.yaml  --wait=true
```

## Future ideas
- Build this demo on GKE using WorkloadIdentity and KCC AddOn
- Use ConfigSync in place of ArgoCD
- Do this multi-cluster across GCP and Azure/AWS using _either_ 
  - Anthos managed clusters and ACM  (see https://seroter.com/2021/01/12/how-gitops-and-the-krm-make-multi-cloud-less-scary/)
  - GKE + AKS/AWS clusters, ArgoCD and appropriate AWS/AKS service operators
- Get assistance from Platform Eng to attempt to resolve permissions issue with reverse engineering assets
  

## Resources
https://cloud.google.com/config-connector/docs/reference/overview