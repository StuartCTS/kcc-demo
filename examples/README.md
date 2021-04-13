This just contains some random samples for provisioning infra via KCC.

After you have set up kcc and an appropriately configured namespace, you should be able to create resources via `kubectl apply -f <resource_file.yaml> -n <kcc_namespace>`

Note the redis example will require the 'default' network in the host project

()[enable_pubsub.yaml] is an example of enabling a required API via KCC