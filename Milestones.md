# Milestones

1. [x] Get the application deployed by exactly following the tutorial
2. [x] Use Skaffold
3. [ ] Deploy the [WordCount Example](https://github.com/apache/flink-kubernetes-operator/tree/main/examples/flink-beam-example)
4. [ ] Use a Dockerfile to build the Flink application image

## Milestone 1: Quickstart

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

Even better, we can split all the deployments up into their own modules:

1. Flink Operator Installation
2. State Machine Example
3. Word Count Example

This way, we can deploy each example separately.

```bash
skaffold delete -p bootstrap
skaffold run -p bootstrap
skaffold dev --module state-machine
```

### Setting up Word Count

```bash
curl -o word-count/word-count-example.yaml https://raw.githubusercontent.com/apache/flink-kubernetes-operator/refs/heads/main/examples/flink-beam-example/beam-example.yaml
curl -o word-count/pom.xml https://raw.githubusercontent.com/apache/flink-kubernetes-operator/refs/heads/main/examples/flink-beam-example/pom.xml
```

```bash
cd word-count
eval $(minikube docker-env)
DOCKER_BUILDKIT=1 docker build . -t flink-beam-example:latest
kubectl apply -f word-count-example.yaml
```

#### The Correct Jar

Turns out we needed `local:///code/target/original-flink-wordcount-example-1.0-SNAPSHOT.jar` instead of `local:///code/target/flink-wordcount-example-1.0-SNAPSHOT.jar`!

The reason is because of what the JAR Entries contained (specifically looking for `WordCount.class`):

```bash
wcygan@foobar word-count % jar tf target/original-flink-wordcount-example-1.0-SNAPSHOT.jar
META-INF/
META-INF/MANIFEST.MF
org/
org/apache/
org/apache/flink/
org/apache/flink/util/
META-INF/maven/
META-INF/maven/org.apache.flink/
META-INF/maven/org.apache.flink/flink-wordcount-example/
org/apache/flink/WordCount$Tokenizer.class
org/apache/flink/util/WordCountData.class
org/apache/flink/WordCount.class
META-INF/maven/org.apache.flink/flink-wordcount-example/pom.xml
META-INF/maven/org.apache.flink/flink-wordcount-example/pom.properties
wcygan@foobar word-count % jar tf target/flink-wordcount-example-1.0-SNAPSHOT.jar
META-INF/MANIFEST.MF
org/
org/apache/
org/apache/flink/
org/apache/flink/connector/
org/apache/flink/connector/datagen/
org/apache/flink/connector/datagen/source/
org/apache/flink/connector/datagen/source/GeneratorFunction.class
org/apache/flink/connector/datagen/source/DataGeneratorSource.class
org/apache/flink/connector/datagen/source/GeneratingIteratorSourceReader.class
org/apache/flink/connector/datagen/source/GeneratorSourceReaderFactory.class
META-INF/
META-INF/LICENSE
META-INF/NOTICE
```

Doing this in the root directory instead of `word-count`

```bash
eval $(minikube docker-env)
DOCKER_BUILDKIT=1 docker build -f word-count/Dockerfile -t flink-beam-example:latest word-count
kubectl apply -f word-count/word-count-example.yaml
kubectl delete -f word-count/word-count-example.yaml 
```