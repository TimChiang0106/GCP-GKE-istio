# GCP - GKE - istio - ASM - Internal Load Balancer


# Background

We want the asm of cluster is able to communicate each other for using istio-ingressgateway in peering network in GCP.

# Prerequisites
We recommend you to use [cloud shell](https://cloud.google.com/shell/docs/launching-cloud-shell) for next steps.

Get the ip of cloud shell instance
```bash
curl -s checkip.dyndns.org | sed -e 's/.*Current IP Address: //' -e 's/<.*$//'
```

Download asmcli
```bash
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.15 > asmcli
chmod +x asmcli
```
# 1. Create two cluster in two different project.
# 2. Peering two VPC which cluster in.

# Before you begin
Optional: Create the following environment variables for the project ID, cluster zone or region, cluster name, and context. 
```bash
export PROJECT_1=PROJECT_ID_1
export LOCATION_1=CLUSTER_LOCATION_1
export CLUSTER_1=CLUSTER_NAME_1
export CTX_1="gke_${PROJECT_1}_${LOCATION_1}_${CLUSTER_1}"

export PROJECT_2=PROJECT_ID_2
export LOCATION_2=CLUSTER_LOCATION_2
export CLUSTER_2=CLUSTER_NAME_2
export CTX_2="gke_${PROJECT_2}_${LOCATION_2}_${CLUSTER_2}"
```
Optional: Configure kubectl to point to the cluster
```bash
gcloud container clusters get-credentials CLUSTER_NAME \
     --zone CLUSTER_LOCATION \
     --project PROJECT_ID
```
# Deploy hello world and istio with Internal Load Balancer
Install istio in Cluster 1
```bash
istioctl install -y  -f istio-operator.yaml
```
Create an Istio Gateway:
```bash
kubectl apply  -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway # use Istio default gateway implementation
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```
Configure routes for traffic entering via the Gateway:
```bash
kubectl apply  -f  - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: VirtualService
metadata:
  name: httpbin
spec:
  hosts:
  - "httpbin.example.com"
  gateways:
  - httpbin-gateway
  http:
  - match:
    - uri:
        prefix: /hello
    route:
    - destination:
        port:
          number: 5000
        host: helloworld
EOF
```
```bash
kubectl label namespace NAMESPACE istio-injection=enabled
```
### Firewall setting with port 15000-15100 in Control plane address range
Create the HelloWorld service:
```bash
kubectl create \
    -f samples/helloworld/helloworld.yaml \
    -l service=helloworld 
```
Deploy HelloWorld v1
```bash
kubectl create  \
  -f samples/helloworld/helloworld.yaml \
  -l version=v1 
```
Deploy the Sleep service for curl in pod
```bash
 kubectl apply  -f samples/sleep/sleep.yaml
```
Verify the internal load balancing.
```bash
kubectl exec -c sleep "$(kubectl get pod  -l app=sleep -o jsonpath='{.items[0].metadata.name}')" -- /bin/sh -c 'for i in $(seq 1 1); do curl -s -I -HHost:httpbin.example.com "http://IP:PORT/hello"; done'
```
Analyze Istio configuration and print validation messages
```bash
istioctl analyze
```

## Install ASM
1. Enable APIs
```bash
  gcloud services enable mesh.googleapis.com \
      --project=FLEET_PROJECT_ID
```
2. Enable the Anthos Service Mesh fleet feature
```bash
gcloud container fleet mesh enable --project FLEET_PROJECT_ID 
```
3. Register a GKE cluster using Workload Identity to a fleet:
```bash
 # For getting gke uri
 gcloud container clusters list --uri --project PROJECT_ID
 
 gcloud container fleet memberships register NAME \
  --gke-uri=https://container.googleapis.com/v1/projects/PROJECT/zones/zones/clusters/clusters\
  --enable-workload-identity \
  --project PROJECT_ID
```
4. Verify your cluster is registered:
```bash
gcloud container fleet memberships list --project FLEET_PROJECT_ID
```
# Install with asmcli
```bash
./asmcli install \
  --project_id PROJECT_ID \
  --cluster_name CLUSTER_NAME \
  --cluster_location CLUSTER_LOCATION \
  --fleet_id FLEET_ID \
  --enable_all \
  --ca mesh_ca
  
```

# Testing

In pod
```bash
kubectl exec --context="${CTX_1}" -n default -c sleep \
    "$(kubectl get pod --context="${CTX_1}" -n default -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- /bin/sh -c 'for i in $(seq 1 1); do curl -v -s -I -HHost:httpbin.example.com "http://192.168.0.28:80/hello"; done'
```

In VM 
```bash
curl -s -I -HHost:httpbin.example.com "http://192.168.0.28:80/hello;"
```

# Error
```bash
Error:
Internal error occurred: failed calling webhook "rev.namespace.sidecar-injector.istio.io": failed to call webhook: Post "https://istiod-asm-1152-6.istio-system.svc:443/inject?timeout=10s": context deadline exceeded


Solution:
Firewall role on control plane address range	
15000-15100
```

```bash
Error:
pods "istio-ingressgateway-5b648874cc-" is forbidden: error looking up service account asm-1152-6/user-ingressgateway-service-account: serviceaccount "user-ingressgateway-service-account" not found




Solution:
kubectl create namespace GATEWAY_NAMESPACE
kubectl create serviceaccount istio-ingressgateway-service-account \
    --namespace istio-system
```
```bash
Error:
Istiod encountered an error: failed to wait for resource: resources not ready after 5m0s: timed out waiting for the condition
	 


Solution:
Does not have minimum availability, there is not sufficient RAM or CPU to do the install

```
```bash
Error: 
asmcli: Installing ASM control plane...
Failed 




Solution:
Check your VPC route. is there any 0.0.0.0/0 default internet?
```
```bash
Error: ImagePullBackOff and Error: ErrImagePull errors with GKE



Solution:
has private nodes (aka no Public IP's)
There is no Cloud NAT for the region of that cluster
You don't have Private Access enabled on the subnet/vpc

```

# References

## Istio
https://istio.io/latest/docs/setup/install/istioctl/

https://istio.io/latest/docs/tasks/traffic-management/ingress/ingress-control/

## GCP
https://cloud.google.com/service-mesh/docs/unified-install/install-anthos-service-mesh

https://cloud.google.com/service-mesh/docs/unified-install/gke-install-multi-cluster#verify_cross-cluster_load_balancing



