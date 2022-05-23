# NSM + Istio example for a single kind cluster

## Setup Cluster

### KIND
Setup

```bash
go install sigs.k8s.io/kind@v0.13.0 

kind create cluster --config kind-cluster-config.yaml --name demo-cluster
```

## SPIRE

Install spire
```bash
kubectl apply -k https://github.com/networkservicemesh/deployments-k8s/examples/spire?ref=v1.3.1
```

## NSM

```bash
kubectl create ns nsm-system
kubectl apply -k ./nsm/single-cluster
```

## Istio

Install Istio for second cluster:
```bash
curl -sL https://istio.io/downloadIstioctl | sh -
export PATH=$PATH:$HOME/.istioctl/bin
istioctl install --set profile=minimal -y
istioctl proxy-status
```

## Run experiment

```bash
kubectl label namespace default istio-injection=enabled
```

Install networkservice:
```bash
kubectl apply -f networkservice.yaml
```

Start `auto-scale` networkservicemesh endpoint:
```bash
kubectl apply -k nse-auto-scale 
```

Start `workload2` endpoint:
```bash
cat > workload2-deploy.yaml <<EOF
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload2
  labels:
    app: workload2
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workload2
  template:
    metadata:
      labels:
        app: workload2
    spec:
      containers:
      - name: hello-world
        image: strm/helloworld-http
        imagePullPolicy: IfNotPresent
        stdin: true
        tty: true
EOF

kubectl apply -f workload2-deploy.yaml

cat > workload2-svc.yaml <<EOF
apiVersion: v1
kind: Service
metadata:
  name: workload2
spec:
  ports:
  - name: http
    port: 80
    targetPort: 80
  selector:
    app: workload2
  type: ClusterIP
EOF

kubectl apply -f workload2-svc.yaml

```

### Create NSC on cluster1

```bash
NAMESPACE2=($(kubectl create -f https://raw.githubusercontent.com/networkservicemesh/deployments-k8s/7d08174431e049a31fdaf574e03f12ea965c4f5b/examples/interdomain/usecases/namespace.yaml)[0])
NAMESPACE2=${NAMESPACE2:10}

```

```bash
cat > workload1-deployment.yaml <<EOF
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: workload1
spec:
  replicas: 1
  selector:
    matchLabels:
      app: workload1
  template:
    metadata:
      labels:
        app: workload1
      annotations:
        networkservicemesh.io: "kernel://autoscale-istio-proxy/nsm-1?app=workload1"
    spec:
      containers:
        - name: hello-world
          image: strm/helloworld-http
          imagePullPolicy: IfNotPresent
          stdin: true
          tty: true
EOF

kubectl apply -n ${NAMESPACE2} -f workload1-deployment.yaml 

kubectl wait --for=condition=ready --timeout=5m pod -l app=workload1 -n ${NAMESPACE2}
 
NSC=$(kubectl --context="${CTX_CLUSTER1}" get pods -l app=workload1 -n ${NAMESPACE2} --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
echo $NSC

```

Verify connectivity:
```bash
kubectl exec -n ${NAMESPACE2} deploy/workload1 -c cmd-nsc -- apk add curl
kubectl exec -n ${NAMESPACE2} deploy/workload1 -c cmd-nsc -- curl -s workload1.default
```

## Cleanup

```bash
WH=$(kubectl get pods -l app=admission-webhook-k8s -n nsm-system --template '{{range .items}}{{.metadata.name}}{{"\n"}}{{end}}')
kubectl delete mutatingwebhookconfiguration ${WH}

kubectl delete -k ./nsm/single-cluster

kubectl delete crd spiffeids.spiffeid.spiffe.io
kubectl delete ns spire
```