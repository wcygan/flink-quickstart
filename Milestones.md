# Milestones

1. [x] Get the application deployed by exactly following the tutorial
2. [x] Use Skaffold
3. [ ] Deploy the [WordCount Example](https://github.com/apache/flink-kubernetes-operator/tree/main/examples/flink-beam-example)
4. [ ] Use a Dockerfile to build the Flink application image

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
curl -o flink/state-machine-example.yaml https://raw.githubusercontent.com/apache/flink-kubernetes-operator/release-1.10/examples/basic.yaml
```

Update `flink/skaffold.yaml`:

```yaml
apiVersion: skaffold/v4beta1
kind: Config
metadata:
  name: state-machine-example
manifests:
  rawYaml:
    - state-machine-example.yaml
deploy:
  kubectl: { }
```

Check the logs:

```bash
kubectl logs -f deploy/state-machine-example
```

## Milestone 3: Deploy the WordCount Example

First, we are going to split the deployments up so `Word Count` and `State Machine` can be deployed separately.

Here's what the new way of deploying the `State Machine` looks like:

```bash
skaffold dev --module state-machine-example
```

This will set us up to be able to deploy the `Word Count` example in the next step.
