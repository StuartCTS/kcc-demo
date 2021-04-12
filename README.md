# kcc-demo
Using the GCP Config Connector (KCC) to provision GCP infra from k8s.

This repo show how to provision GCP infra from a k8s cluster outside of GCP (i.e. non-GKE). Note the approach below is purely for demoing from a local cluster - it should NOT be seen as a secure way of deploying infra; for that see the installation on GKE clusters using Workload Identity. It does however demonstrate the principle of using the k8s control plane and declarative methods to control the state of infrastructure.


## Requirements
`kubectl` access to k8s cluster (tested with `Docker Desktop for Mac`, but `kind`, `Minikube`, `k3s` etc should all be ok).

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

Enable API
`gcloud services enable cloudresourcemanager.googleapis.com`

Create a service account for creds used by CC - note requires 'owner' or 'editor' role

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

See https://cloud.google.com/config-connector/docs/how-to/install-other-kubernetes - (note this is different for GKE clusters as these can use Workload Identity - see appropriate docs)

Pull the latest distribution of the GCP Config Connector: 

```
gsutil cp gs://configconnector-operator/latest/release-bundle.tar.gz release-bundle.tar.gz

tar zxvf release-bundle.tar.gz
```

Create a namespace for the kcc - this _must_ be named `cnrm-system`:
```
kubectl create namespace cnrm-system 

kubectl create secret generic $SECRET_NAME \
    --from-file key.json \
    --namespace cnrm-system
```

Delete the key
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

Annotate to control resources at project level, eg:

```
kcc-demo % kubectl annotate namespace \
 kcc cnrm.cloud.google.com/project-id="YOUR_PROJECT_ID"
```

Should be good to go at this point. You can see the types of GCP resources you are able to create via 
```
 kubectl get crds | grep '.cnrm.cloud.google.com'
 ```
or look at the resources documentation [here](https://cloud.google.com/config-connector/docs/reference/overview)


## Manage GCP resources

Enable API [](samples/enable_pubsub.yaml)
```
kubectl apply -f samples/enable_pubsub.yaml -n kcc
```

Create the resource
```
kubectl apply -f samples/pubsub_topic.yaml -n kcc
```

View what's happening with your resources

```
kubectl get events -n kcc
```

Get all managed resources:
```
kubectl get gcp -n kcc
```

## Clean up

Delete all managed resources first 

(note if you want to keep the resources running for whichever reason you must first 'unmanage' them from KCC)


## Resources

https://cloud.google.com/config-connector/docs/reference/overview