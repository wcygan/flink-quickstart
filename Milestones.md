# Milestones

1. [x] Get the application deployed by exactly following the tutorial
2. [x] Use Skaffold
3. [ ] Deploy the [WordCount Example](https://github.com/apache/flink-kubernetes-operator/tree/main/examples/flink-beam-example)
4. [ ] Use a Dockerfile to build the Flink application image
5. [ ] Potentially update this to be a Beam application

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

Next, we need to translate this into [word-count/skaffold.yaml](/word-count/skaffold.yaml) so that it can continuously build the application.

### Current problem: latest flink is not being updated by Skaffold

Maybe we need to update the [FlinkDeployment](/word-count/word-count-example.yaml) to specify the latest image in a pod template?

Documentation: 

1. https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.10/docs/custom-resource/overview/#flinkdeployment
2. https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.10/docs/custom-resource/reference/
3. https://nightlies.apache.org/flink/flink-kubernetes-operator-docs-release-1.10/docs/custom-resource/pod-template/

Here are some possible examples from GitHub: https://github.com/search?q=%22kind%3A+FlinkDeployment%22+%22podTemplate%22&type=code

Essentially, the problem is that we're not actually using the latest tagged image. Here's a clear example:

```bash
docker images --digests
REPOSITORY                                      TAG                                                                DIGEST                                                                    IMAGE ID       CREATED          SIZE
flink-beam-example                              2024-11-10_13-21-33.505_CST                                        <none>                                                                    38a8b7c09949   2 minutes ago    777MB
flink-beam-example                              38a8b7c09949946f68048e5fa83149db8b55e04ddc74789d688f8a3ffac628a9   <none>                                                                    38a8b7c09949   2 minutes ago    777MB
flink-beam-example                              2024-11-10_12-50-41.404_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_12-52-20.764_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_13-05-08.474_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_13-07-42.204_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_13-09-25.673_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_13-16-35.923_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_13-19-13.911_CST                                        <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              91e81bc4d7cbbf5c14e80c830d2805d49b9a2c36f64e4b4d9a7507b936d81ca0   <none>                                                                    91e81bc4d7cb   33 minutes ago   777MB
flink-beam-example                              2024-11-10_12-42-54.759_CST                                        <none>                                                                    df1ba5db0a5b   54 minutes ago   777MB
flink-beam-example                              ce1a856-dirty                                                      <none>                                                                    df1ba5db0a5b   54 minutes ago   777MB
flink-beam-example                              df1ba5db0a5bc6b413013685340beeb6a5169e9e260c63884adf1d5f7e3db288   <none>                                                                    df1ba5db0a5b   54 minutes ago   777MB
flink-beam-example                              472922ba941f119978e9831c8c2eae8954fc9eae32ff3237c9cb0a10aca7df31   <none>                                                                    472922ba941f   58 minutes ago   777MB
<none>                                          <none>                                                             <none>                                                                    129d5ddf25e1   2 hours ago      777MB
flink-beam-example                              latest                                                             <none>                                                                    ae09d8c7fed1   2 hours ago      777MB
ghcr.io/apache/flink-kubernetes-operator        c703255                                                            sha256:e9c2ce635b89cfede3e6a20c002d39803b1cc754f39dc2c6e87f8d4e0cf59d22   a292a12359b8   3 weeks ago      412MB
```

Now let's see what image that the FlinkDeployment is actually using:

```bash
docker inspect $(kubectl get pods -l app=beam-example -o json | jq -r '.items[0].status.containerStatuses[] | select(.name=="flink-main-container") | .imageID' | sed 's|docker://||') | jq '.[] | {
  Id,
  RepoTags,
  Comment,
  Created,
  LastTagTime: .Metadata.LastTagTime,
  Architecture,
  Variant,
  Os
}'
```

```json
{
  "Id": "sha256:ae09d8c7fed164f8665f39e217ce4e0a6d5b6f6e8291666787f4b4e3795b0892",
  "RepoTags": [
    "flink-beam-example:latest"
  ],
  "Comment": "buildkit.dockerfile.v0",
  "Created": "2024-11-10T17:34:23.247433299Z",
  "LastTagTime": "2024-11-10T18:16:20.173335422Z",
  "Architecture": "arm64",
  "Variant": "v8",
  "Os": "linux"
}
```

You see it's using image ID `ae09d8c7fed1` instead of `38a8b7c09949`! That's not what we want.

So, somewhere in our Skaffold or Flink Manifests we have a problem.

### Temporary Solution

Maybe the solution you can use for now is simply to delete all the docker images for `flink-beam-example`, then run `skaffold dev`.

```bash
docker images | rg flink-beam-example | awk '{print $3}' | xargs docker rmi -f 
```

### Observation

After deleing the containers & then running `DOCKER_BUILDKIT=1 docker build -f word-count/Dockerfile -t flink-beam-example:latest word-count`, I was able to get the image built and running again.

However, just by doing `skaffold dev --module word-count` it didn't solve the problem!

So, this probably means there is a constraint when developing Flink jobs particularly using Skaffold.

You might want to open an issue on the Skaffold GitHub repo to figure out if there's a solution

Otherwise, just do this whenever you need to update the Flink code:

```bash
DOCKER_BUILDKIT=1 docker build -f word-count/Dockerfile -t flink-beam-example:latest word-count && skaffold dev --module word-count
```

Why doesn't skaffold automatically do the same thing for me? Maybe I need a combination of useCLI and useBuildkit

Alternatively, maybe we should just package it with Jib instead? Here are two examples:

1. https://github.com/wcygan/monorepo-jib-grpc
2. https://github.com/Pozo/continuous-java-kubernetes