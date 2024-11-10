# Milestones

1. [x] Get the application deployed by exactly following the tutorial
2. [ ] Use Skaffold
3. [ ] Use a Dockerfile to build the Flink application image

## Milestone 2: Use Skaffold

### Part A: Operator Installation via Skaffold

```bash
skaffold run -p bootstrap
```

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
skaffold delete -p bootstrap
```

### Part B: Job Submission via Skaffold

Next, we want to be able to submit the job via Skaffold using `skaffold dev`.

```bash
curl -o flink/basic.yaml https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.10/examples/basic.yaml
```

Update `flink/skaffold.yaml`:

```yaml
apiVersion: skaffold/v4beta1
kind: Config
metadata:
  name: flink-config
manifests:
  rawYaml:
    - basic.yaml
deploy:
  kubectl: { }
```