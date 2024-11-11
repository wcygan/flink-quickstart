# [Flink Quickstart](https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-main/docs/try-flink-kubernetes-operator/quick-start/a)

Start Minikube:

```bash
minikube start --cpus=8 --memory=10240 --disk-size=50g
minikube status
skaffold config set --global local-cluster true
eval $(minikube -p custom docker-env)
skaffold run -p bootstrap
```
## Run State Machine

```bash
skaffold dev --module state-machine
```

## Run Word Count

```bash
DOCKER_BUILDKIT=1 docker build -f word-count/Dockerfile -t flink-beam-example:latest word-count && skaffold dev --module word-count
```

It's a bit annoying that you have to build the docker image like this, but I haven't found a way around it yet...