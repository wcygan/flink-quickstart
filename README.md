# [Flink Quickstart](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/a)

Start Minikube:

```bash
minikube start --cpus=8 --memory=10240 --disk-size=50g
minikube status
```

## Install the Flink Operator

Install Cert Manager:

```bash
kubectl create -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```

Install Flink Operator:

```bash
helm repo add flink-operator-repo https://downloads.apache.org/flink/flink-kubernetes-operator-1.10.0/
helm install flink-kubernetes-operator flink-operator-repo/flink-kubernetes-operator
```

Note: we see that the `default` namespace is used by the flink operator:

```bash
 default       flink-kubernetes-operator-6b55b4664-zvwld  ●  2/2   Running         0 10.244.0.7    minikube  33m
```

```bash
kubectl get pods
helm list
```

## Submit a Flink Job

```bash
kubectl create -f https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.10/examples/basic.yaml
```

Check the job status:

```bash
kubectl get flinkdeployment
kubectl logs -f deploy/basic-example
```

## Undo it all

```bash
kubectl delete -f https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.10/examples/basic.yaml
helm uninstall flink-kubernetes-operator
kubectl delete -f https://github.com/jetstack/cert-manager/releases/download/v1.8.2/cert-manager.yaml
```
