# Exporting GCP resources and managing them via KCC

```
gsutil cp gs://cnrm/latest/cli.tar.gz .
tar zxf cli.tar.gz
```



```
mv darwin/amd64/config-connector /usr/local/bin

gcloud services enable cloudasset.googleapis.com

gcloud auth application-default login

gcloud pubsub topics create sample-topic

TOPIC_RESOURCE_ID=$(gcloud pubsub topics describe sample-topic --format "value(name)")
echo $TOPIC_RESOURCE_ID
projects/stuart-dev-example-01/topics/sample-topic
stuart@Stuarts-MacBook-Pro cli % TOPIC_RESOURCE_NAME="//pubsub.googleapis.com/${TOPIC_RESOURCE_ID}"
stuart@Stuarts-MacBook-Pro cli % echo $TOPIC_RESOURCE_NAME
//pubsub.googleapis.com/projects/stuart-dev-example-01/topics/sample-topic
stuart@Stuarts-MacBook-Pro cli % config-connector export ${TOPIC_RESOURCE_NAME}
```
Create a resource manually e.g.

```
gcloud 
```


WIP - seems there is a perms issue trying to run export, i.e.
```
stuart@Stuarts-MacBook-Pro cli % config-connector export ${TOPIC_RESOURCE_NAME} --verbose
error in 'config-connector' version '1.45.0': error getting unstructured: error getting iam policy for 'PubSubTopic' with name 'sample-topic': error fetching live state for resource: error reading underlying resource: summary: Error when reading or editing Resource "pubsub topic \"projects/stuart-dev-example-01/topics/sample-topic\"" with IAM Policy: Error retrieving IAM policy for pubsub topic "projects/stuart-dev-example-01/topics/sample-topic": googleapi: Error 403: User not authorized to perform this action., detail:
```

See image in dir - possibly related?


Same with bulk export:

```
config-connector bulk-export --project $PROJECT_ID
error in 'config-connector' version '1.45.0': error exporting asset inventory: error response from exportassets request: googleapi: Error 403: Request denied by Cloud IAM., forbidden
```

Also attempted by using appropriate `--oauth2-token` but no joy :-(
