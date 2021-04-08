# kcc-demo
Using the GCP Config Connector to provision GCP infra from k8s

## Requirements
`kubectl` access to k8s cluster (tested with `Docker Desktop for Mac`, but `kind`, `Minikube` should all be ok) 

Ability to create roles i.e. 
`kubectl auth can-i create roles` 
- answer should be 'yes'

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
Install the KCC operator:
```
kubectl apply -f operator-system/configconnector-operator.yaml
```

Check all is ok (make sure containers all started)

```
kubectl wait -n cnrm-system \
      --for=condition=Ready pod --all
```

Create namespace for resources managed by kcc (can be anything) e.g.

kubectl create ns kcc

Annotate to control resources at project level:

```
kubectl annotate namespace \
 kcc cnrm.cloud.google.com/project-id="$PROJECT_ID"
```