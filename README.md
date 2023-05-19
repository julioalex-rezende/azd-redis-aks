# azd-redis-aks
Incorporating redis in aks to expose a possible issue/bug with azd cli deployment

When deploying Redis on an AKS cluster and redeploying services through azd cli on the same namespace, the verification step if the deployment was made correctly fails, causing the command to interrupt.

After investigation, it seems that azd expects the `json` output from the AKS services to have targetPorts assigned to integer values only, whereas through redis deployment, this value is a literal string called `redis` and the ports are sorted at pod level


Things to notice: 
- deployment of services works correctly, and fails only on validation step
- command fails during json parsing when verifying deployment (`kubectl -get svc -n <environment_name> -json`)
- this interrupts the command, thus following services are not deployed 

It's possible to check the json file with the following command:
```bash
kubectl get service --namespace <namespace> -o json
```
For all Redis related services (redis-headless, redis-master, redis-replicas) the value for ports.TargetPort is a literal string. That's the cause of conflict with `azd`
```json
"spec": {
    ...
    "ports": [
        {
            "name": "tcp-redis",
            "port": 6379,
            "protocol": "TCP",
            "targetPort": "redis"
        }
    ],
    ...
}
```

Looking at [`azd` source code](https://github.com/Azure/azure-dev/blob/473ce2409aaffaf1f6988c3324969bf2cdc962f2/cli/azd/pkg/tools/kubectl/models.go#L110), `targetPort` field is specified as integer:
```go
type Port struct {
	Port       int    `json:"port"`
	TargetPort int    `json:"targetPort"`
	Protocol   string `json:"protocol"`
}
```


# To Reproduce

```bash
# login to azure
az login

# initiate azd environment. This create a new *.azure* folder where the environment variables used by AZD are stored
azd init
    <select an empty template>
    <select an environment name>

# deploy services and resources. This will deploy resources defined in infra folder and service described in azure.yaml file
azd up
    <select subscription>
    <select location>

# source azd environment variable file and update credentials to cluster
source ./.azure/<environment name>/.env
az aks get-credentials --resource-group $AZURE_RESOURCE_GROUP_NAME --name $AZURE_AKS_CLUSTER_NAME --overwrite-existing


# Deploy Redis on cluster through helm chart
helm repo add bitnami https://charts.bitnami.com/bitnami
helm uninstall redis
helm install redis bitnami/redis --namespace $AZURE_ENV_NAME 

# Deploy services again (either through *azd up* or *azd deploy*)
azd deploy [--debug]
...
...
  (x) Failed: Deploying service public-api-service

ERROR: failed deploying service 'public-api-service': failing invoking action 'deploy', failed retrieving service endpoints, failed waiting for resource, failed waiting for resource, failed unmarshalling resources JSON, json: cannot unmarshal string into Go struct field Port.items.spec.ports.targetPort of type int
```