# Azure External DNS Automation in Rancher

This document is designed to instruct the user on how to setup ExternalDNS in Kuberentes on an Azure cluster that was create in Kubernetes.

## Prerequisites 
Login to Azure AD, and if needed set your subscription

## A couple of notes before you get started

Be aware the external template file in this document is using a number of optional settings.  

It is also in debug level logging so you can troubleshoot any errors.


## Setup External DNS on Azure AKS or Azure IaaS
1. Create the Azure DNS Record

```sh
RESOURCE_GROUP=MC_rancher-group_c-6vkts_eastus
DNS_ZONE=vanbrackel.net
az network dns zone create -g $RESOURCE_GROUP -n $DNS_ZONE
```
2. Delegate DNS as needed for your registrar

3. Create a service principal to act on behalf of Kubernetes.
```sh
SUBSCRIPTION_ID="$(az account show | jq '.id')" && SUBSCRIPTION_ID=${SUBSCRIPTION_ID//\"}
TENANT_ID=$(az account show | jq '.tenantId') && TENANT_ID=${TENANT_ID//\"}
SCOPE=$(az group show --name $RESOURCE_GROUP | jq '.id') && SCOPE=${SCOPE//\"}
PRINCIPAL=$(az ad sp create-for-rbac --role="Contributor" --scopes=$SCOPE -n ExternalDnsServicePrincipal)
CLIENT_ID=$(echo $PRINCIPAL | jq '.appId') && CLIENT_ID=${CLIENT_ID//\"}
CLIENT_SECRET=$(echo $PRINCIPAL | jq '.password') && CLIENT_SECRET=${CLIENT_SECRET//\"}
```

4. Create your a cloud provider config.
```sh
echo "{ \"tenantId\": \"$TENANT_ID\", \"subscriptionId\": \"$SUBSCRIPTION_ID\", \"aadClientId\": \"$CLIENT_ID\", \"aadClientSecret\": \"$CLIENT_SECRET\", \"resourceGroup\": \"$RESOURCE_GROUP\"}" >> azure.json
```

5. Create a kubernetes secret with the cloud provider configuration
```sh
> kubectl create secret generic azure-config-file --from-file=azure.json
secret "azure-config-file" created
```

6. For Azure IaaS Backed Clusters provisioned by Rancher, remove the default ingress controller from your cluster.
```sh
> kubectl get ns
NAME            STATUS    AGE
cattle-system   Active    1d
default         Active    1d
ingress-nginx   Active    1d
kube-public     Active    1d
kube-system     Active    1d
> kubectl delete ns/ingress-nginx
namespace "ingress-nginx" deleted
```

**Note** if you provisioned your cluster using AKS in Rancher, there will be no ingress controller provided.

7. Install the nginx ingress controller and configure it for ExternalDNS
  - Create the ingress-nginx deployment and service
  ```sh
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/mandatory.yaml
  kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/master/deploy/provider/cloud-generic.yaml
 
  ``` 


8. Since RBAC is enabled by default on Rancher based Kubernetes clusters create a yaml file called externaldns.yaml from the script below, or use the externaldns-template.yaml file in this repository.

```yaml
apiVersion: v1
kind: ServiceAccount
metadata:
  name: external-dns
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRole
metadata:
  name: external-dns
rules:
- apiGroups: [""]
  resources: ["services"]
  verbs: ["get","watch","list"]
- apiGroups: [""]
  resources: ["pods"]
  verbs: ["get","watch","list"]
- apiGroups: ["extensions"] 
  resources: ["ingresses"] 
  verbs: ["get","watch","list"]
---
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: external-dns-viewer
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: external-dns
subjects:
- kind: ServiceAccount
  name: external-dns
  namespace: default
---
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: external-dns
spec:
  strategy:
    type: Recreate
  template:
    metadata:
      labels:
        app: external-dns
    spec:
      serviceAccountName: external-dns
      containers:
      - name: external-dns
        image: registry.opensource.zalan.do/teapot/external-dns:v0.5.2
        args:
        - --source=service
        - --source=ingress
        - --domain-filter=vanbrackel.net # (optional) limit to only vanbrackel.net domains; change to match the zone created above.
        - --provider=azure
        - --azure-resource-group=MC_rancher-group_c-6vkts_eastus # (optional) use the DNS zones from above
        volumeMounts:
        - name: azure-config-file
          mountPath: /etc/kubernetes
          readOnly: true
      volumes:
      - name: azure-config-file
        secret:
          secretName: azure-config-file
```

```sh
EXTERNAL_DNS=$(cat externaldns-template.yaml)
EXTERNAL_DNS=${EXTERNAL_DNS//DOMAIN/$DOMAIN} && echo "${EXTERNAL_DNS//RESOURCE_GROUP/$RESOURCE_GROUP}" >> externaldns.yaml
kubectl create -f externaldns.yaml
```

## Verification

1. Create an nginx service in an ingress in the same manner we deployed ExternalDNS

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx
spec:
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - image: nginx
        name: nginx
        ports:
        - containerPort: 80
---
apiVersion: v1
kind: Service
metadata:
  name: nginx-svc
spec:
  ports:
  - port: 80
    protocol: TCP
    targetPort: 80
  selector:
    app: nginx
  type: ClusterIP
  
---
apiVersion: extensions/v1beta1
kind: Ingress
metadata:
  name: nginx
  annotations:
    kubernetes.io/ingress.class: nginx
spec:
  rules:
  - host: server.vanbrackel.net
    http:
      paths:
      - backend:
          serviceName: nginx-svc
          servicePort: 80
        path: /
```

```sh
NGINX=$(cat nginx-ingress-test-template.yaml) && echo "${NGINX//DOMAIN/$DOMAIN}" >> nginx-ingress-test.yaml
```

2. Create the nginx-ingress controller
```sh
kubectl create -f nginx-ingress-test.yaml
```

3. Give it a few minutes

4. Check to see if any records were created.
```sh
[jason@vblinux ~ ]$ az network dns record-set a list --resource-group $RESOURCE_GROUP --zone-name $DNS_ZONE
[
  {
    "arecords": [
      {
        "ipv4Address": "13.68.138.206"
      }
    ],
    "etag": "0fb3eaf9-7bf2-48c4-b8f8-432e05dce94a",
    "fqdn": "server.vanbrackel.net.",
    "id": "/subscriptions/c7e23d24-5dcd-4c7c-ae84-22f6f814dc02/resourceGroups/mc_rancher-group_c-6vkts_eastus/providers/Microsoft.Network/dnszones/vanbrackel.net/A/server",
    "metadata": null,
    "name": "server",
    "resourceGroup": "mc_rancher-group_c-6vkts_eastus",
    "ttl": 300,
    "type": "Microsoft.Network/dnszones/A"
  }
]
```


5. Check the logs.
```sh
kubectl logs external-dns-655df89959-7ztm2 
time="2018-06-13T23:57:11Z" level=info msg="config: {Master: KubeConfig: Sources:[service ingress] Namespace: AnnotationFilter: FQDNTemplate: CombineFQDNAndAnnotation:false Compatibility: PublishInternal:false ConnectorSourceServer:localhost:8080 Provider:azure GoogleProject: DomainFilter:[vanbrackel.net] ZoneIDFilter:[] AWSZoneType: AWSAssumeRole: AzureConfigFile:/etc/kubernetes/azure.json AzureResourceGroup:MC_rancher-group_c-6vkts_eastus CloudflareProxied:false InfobloxGridHost: InfobloxWapiPort:443 InfobloxWapiUsername:admin InfobloxWapiPassword: InfobloxWapiVersion:2.3.1 InfobloxSSLVerify:true DynCustomerName: DynUsername: DynPassword: DynMinTTLSeconds:0 InMemoryZones:[] PDNSServer:http://localhost:8081 PDNSAPIKey: Policy:sync Registry:txt TXTOwnerID:default TXTPrefix: Interval:1m0s Once:false DryRun:false LogFormat:text MetricsAddress::7979 LogLevel:debug}"
time="2018-06-13T23:57:11Z" level=info msg="Connected to cluster at https://10.0.0.1:443"
...
time="2018-06-14T00:02:11Z" level=debug msg="Retrieving Azure DNS zones."
time="2018-06-14T00:02:12Z" level=debug msg="Found 1 Azure DNS zone(s)."
time="2018-06-14T00:02:12Z" level=debug msg="Retrieving Azure DNS records for zone 'vanbrackel.net'."
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service default/kubernetes"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service default/nginx-svc"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service ingress-nginx/default-http-backend"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service ingress-nginx/ingress-nginx"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/full-guppy-nginx-ingress-controller"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/full-guppy-nginx-ingress-default-backend"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/heapster"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/kube-dns"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/kubernetes-dashboard"
time="2018-06-14T00:02:12Z" level=debug msg="No endpoints could be generated from service kube-system/tiller-deploy"
time="2018-06-14T00:02:12Z" level=debug msg="Endpoints generated from ingress: default/nginx: [server.vanbrackel.net 0 IN A 13.68.138.206]"
time="2018-06-14T00:02:12Z" level=debug msg="Retrieving Azure DNS zones."
time="2018-06-14T00:02:12Z" level=debug msg="Found 1 Azure DNS zone(s)."
time="2018-06-14T00:02:12Z" level=info msg="Updating A record named 'server' to '13.68.138.206' for Azure DNS zone 'vanbrackel.net'."
time="2018-06-14T00:02:13Z" level=info msg="Updating TXT record named 'server' to '\"heritage=external-dns,external-dns/owner=default,external-dns/resource=ingress/default/nginx\"' for Azure DNS zone 'vanbrackel.net'."
time="2018-06-14T00:03:11Z" level=debug msg="Retrieving Azure DNS zones."
time="2018-06-14T00:03:12Z" level=debug msg="Found 1 Azure DNS zone(s)."
time="2018-06-14T00:03:12Z" level=debug msg="Retrieving Azure DNS records for zone 'vanbrackel.net'."
time="2018-06-14T00:03:12Z" level=debug msg="Found A record for 'server.vanbrackel.net' with target '13.68.138.206'."
time="2018-06-14T00:03:12Z" level=debug msg="Found TXT record for 'server.vanbrackel.net' with target '\"heritage=external-dns,external-dns/owner=default,external-dns/resource=ingress/default/nginx\"'."
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service default/kubernetes"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service default/nginx-svc"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service ingress-nginx/default-http-backend"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service ingress-nginx/ingress-nginx"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/full-guppy-nginx-ingress-controller"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/full-guppy-nginx-ingress-default-backend"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/heapster"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/kube-dns"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/kubernetes-dashboard"
time="2018-06-14T00:03:12Z" level=debug msg="No endpoints could be generated from service kube-system/tiller-deploy"
time="2018-06-14T00:03:12Z" level=debug msg="Endpoints generated from ingress: default/nginx: [server.vanbrackel.net 0 IN A 13.68.138.206]"
time="2018-06-14T00:03:12Z" level=debug msg="Retrieving Azure DNS zones."
time="2018-06-14T00:03:12Z" level=debug msg="Found 1 Azure DNS zone(s)."
```