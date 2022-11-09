# GCP - GKE - istio - ASM


# Background

As you can see the below, we want the asm of cluster is able to communicate each other for using istio-ingressgateway in peering network in GCP.
![image1](https://storage.googleapis.com/github-image-bucket/istio-1.jpeg)

# Prerequisites
We recommend you to use [cloud shell](https://cloud.google.com/shell/docs/launching-cloud-shell) for next steps.

Download asmcli
```bash
curl https://storage.googleapis.com/csm-artifacts/asm/asmcli_1.15 > asmcli
chmod +x asmcli
```
### 1. Create two cluster in two different project.
### 2. Peering two VPC which cluster in.
# 3. Setting project and cluster variables
Create the following environment variables for the project ID, cluster zone or region, cluster name, and context. 
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
# Before you begin

## Install in Cluster 1
Configure kubectl to point to the cluster
```bash
gcloud container clusters get-credentials CLUSTER_NAME \
     --zone CLUSTER_LOCATION \
     --project PROJECT_ID
```
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
 gcloud container clusters list --uri
 
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


# Deploy hello world 
Create the HelloWorld service in both clusters:
```bash
kubectl create --context=${CTX_1} \
    -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
```
```bash
kubectl create --context=${CTX_2} \
    -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
    -l service=helloworld -n sample
```

Deploy HelloWorld v1 and v2 to each cluster
```bash
kubectl create --context=${CTX_1} \
  -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
  -l version=v1 -n sample
```
```bash
kubectl create --context=${CTX_2} \
  -f ${SAMPLES_DIR}/samples/helloworld/helloworld.yaml \
  -l version=v2 -n sample
```
# Deploy the Sleep service for curl in pod
```bash
for CTX in ${CTX_1} ${CTX_2}
do
    kubectl apply --context=${CTX} \
        -f ${SAMPLES_DIR}/samples/sleep/sleep.yaml -n sample
done
```

# Deploy Internal Load Balancer
```bash
kubectl apply -f - <<EOF
apiVersion: networking.istio.io/v1alpha3
kind: Gateway
metadata:
  name: httpbin-gateway
spec:
  selector:
    istio: ingressgateway
  servers:
  - port:
      number: 80
      name: http
      protocol: HTTP
    hosts:
    - "httpbin.example.com"
EOF
```
```bash
kubectl apply -f - <<EOF
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
        prefix: /status
    - uri:
        prefix: /delay
    route:
    - destination:
        port:
          number: 8000
        host: httpbin
	
EOF
```
```bash
kubectl apply -f istio-ilb-dev.yaml
kubectl apply -f istio-ilb-svc.yaml
```

# Testing

In pod
```bash
kubectl exec --context="${CTX_1}" -n sample -c sleep \
    "$(kubectl get pod --context="${CTX_1}" -n sample -l \
    app=sleep -o jsonpath='{.items[0].metadata.name}')" \
    -- /bin/sh -c 'for i in $(seq 1 1); do curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"; done'
```
In VM 
```bash
curl -s -I -HHost:httpbin.example.com "http://$INGRESS_HOST:$INGRESS_PORT/status/200"
```