---
author: "Luke Hinds"
date: 2021-09-09T22:19:13+01:00
linktitle: Tekton CD Chains
title: Signing tekton pipelines and tasks using chains
weight: 10
tags: [
    "security",
    "crypto",
]
categories: [
    "Development",
    "crypto",
]
---

Tekton Chains is a Kubernetes Custom Resource Definition (CRD) controller that
allows you to manage your supply chain security in Tekton.

Chains is a project I have followed since it began and I have always wanted more
time to contribute, but found I got swallowed up into the beast that is sigstore.

I wanted to change that and now am hoping to work on some sort of inbound
verification and help folks out more on the project (not that they need it,
they have been on a blaze with adding sigstore integration with rekor entries
as well as provenance work including an In-toto attestation formatter).

This post is really a note dump, which I figure will be useful to others.

We will set up a kind cluster, tektoncd and chains, generate some keys
sign a pipeline run. Last of all we will dump out what we need to make a verification.

# Grab some tools

- [kind](https://kind.sigs.k8s.io/) will be used to set up a quick and lightweight Kubernetes cluster. Kubernetes vresion >= 1.15 is required. This was tested with `v0.8.1`.
- [kubectl](https://kubernetes.io/docs/tasks/tools/install-kubectl/)
- Additional Tekton CI/CD requirements to install:
    - [ko](https://github.com/google/ko) version >= v0.1 is required to work with and deploy development versions of Tekton. Make sure it's available in your `PATH`. This was tested with `v0.5.1`.
    - [tkn](https://github.com/tektoncd/cli) is the Tekton CLI for interacting with Tekton. This was tested with `v0.9.0`.

## Set up an local OCI registry

Use the following script `kind-with-registry.sh`

Grab it [here if easier](https://gist.githubusercontent.com/lukehinds/7da31f34c1f05942936f2bdb9d71c167/raw/7b7aee3333a8c511fc3cbeb0cc1d9574600caf89/kind-with-registry.sh)


```bash
#!/bin/sh
set -o errexit

# desired cluster name; default is "kind"
KIND_CLUSTER_NAME="${KIND_CLUSTER_NAME:-kind}"

# create registry container unless it already exists
reg_name='kind-registry'
reg_port='5000'
running="$(docker inspect -f '{{.State.Running}}' "${reg_name}" 2>/dev/null || true)"
if [ "${running}" != 'true' ]; then
  docker run \
    -d --restart=always -p "${reg_port}:5000" --name "${reg_name}" \
    registry:2
fi

# create a cluster with the local registry enabled in containerd
cat <<EOF | kind create cluster --name "${KIND_CLUSTER_NAME}" --config=-
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
containerdConfigPatches:
- |-
  [plugins."io.containerd.grpc.v1.cri".registry.mirrors."localhost:${reg_port}"]
    endpoint = ["http://${reg_name}:${reg_port}"]
EOF

# connect the registry to the cluster network
docker network connect "kind" "${reg_name}"

# tell https://tilt.dev to use the registry
# https://docs.tilt.dev/choosing_clusters.html#discovering-the-registry
for node in $(kind get nodes); do
  kubectl annotate node "${node}" "kind.x-k8s.io/registry=localhost:${reg_port}"
done
```

Run your script

```bash
./kind-with-registry.sh`
```

Set up env

```bash                                                                                        
export PATH="${PATH}:${GOPATH}/bin"
export KO_DOCKER_REPO="localhost:5000/mypipeline"
```

Lets now deploy pipelines

```bash
git clone https://github.com/tektoncd/pipeline
```


From the `tektoncd/pipelines` repository

```bash
ko apply -f config/
```

Deploy chains

```bash
git clone https://github.com/tektoncd/chains
```


```bash
# from the tektoncd/chains repository
ko apply -f config/
```

We now need to create some ecdsa keys

```bash
openssl ecparam -genkey -name prime256v1 > ec_private.pem                                                                                                          ✔  655  09:35:04
openssl ec -in ec_private.pem -pubout -out ecpubkey.pem                                                                                                            ✔  656  09:35:28
openssl pkcs8 -topk8 -nocrypt -in ec_private.pem -out x509.pem
```

Set the key as a signing-secret

```bash
kubectl create secret generic signing-secrets -n tekton-chains --from-file=x509.pem
```

We now need a simple pipeline to run

```yaml
# cat hello-world-pipeline.yaml                                                                                                                                         
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: sample-pipeline
spec:
  tasks:
    - name: echo-something
      taskRef:
        name: echo-hello-world
---
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: echo-hello-world
spec:
  steps:
    - name: echo
      image: registry.access.redhat.com/ubi8-minimal
      command:
        - echo
      args:
        - "hello world"
---
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  name: sample-pipeline-run
spec:
  pipelineRef:
    name: sample-pipeline
```

Create the pipeline

```bash
kubectl create -f hello-world-pipeline.yaml
```

Grab the name

```bash
tkn pipelinerun list
NAME                  STARTED          DURATION    STATUS
sample-pipeline-run   59 seconds ago   7 seconds   Succeeded
```

and the taskrun (need this later for getting the signature and payload)

```bash
tkn taskrun list
NAME                                       STARTED          DURATION    STATUS
sample-pipeline-run-echo-something-bj8r7   11 seconds ago   7 seconds   Succeeded
```

Lets find the chains controller to monitor our pipelinerun

```bash
kubectl get pods --all-namespaces
NAMESPACE            NAME                                                 READY   STATUS      RESTARTS   AGE
default              sample-pipeline-run-echo-something-bj8r7-pod-swrws   0/1     Completed   0          4m19s
kube-system          coredns-74ff55c5b-8297p                              1/1     Running     1          2d22h
kube-system          coredns-74ff55c5b-hd9jn                              1/1     Running     1          2d22h
kube-system          etcd-kind-control-plane                              1/1     Running     1          2d22h
kube-system          kindnet-656fx                                        1/1     Running     7          2d22h
kube-system          kube-apiserver-kind-control-plane                    1/1     Running     4          2d22h
kube-system          kube-controller-manager-kind-control-plane           1/1     Running     6          2d22h
kube-system          kube-proxy-nl6j4                                     1/1     Running     1          2d22h
kube-system          kube-scheduler-kind-control-plane                    1/1     Running     6          2d22h
local-path-storage   local-path-provisioner-78776bfc44-5dj5z              1/1     Running     16         2d22h
tekton-chains        tekton-chains-controller-65bbc4d959-6qxx7            1/1     Running     0          18h
tekton-pipelines     tekton-pipelines-controller-6dd7c9bfcf-b8zbj         1/1     Running     1          2d21h
tekton-pipelines     tekton-pipelines-webhook-d9c9f4589-kvsgh             1/1     Running     1          2d21h
```

And lets checkout the logs

```bash
kubectl logs -f tekton-chains-controller-65bbc4d959-6qxx7 -n tekton-chains
```

And there we see the key found

```bash
{"level":"info","ts":"2021-05-21T09:42:11.158Z","logger":"watcher","caller":"x509/x509.go:55","msg":"Found x509 key..."}
```

Let's dig out are sig, payload:

```bash
kubectl get taskrun sample-pipeline-run-echo-something-bj8r7 -o=json | jq -r |grep "payload"| cut -d '"' -f4 |base64 -d > payload
```

And the same for the signature

```bash
kubectl get taskrun sample-pipeline-run-echo-something-bj8r7 -o=json | jq -r |grep "signature"| cut -d '"' -f4 |base64 -d > signature
```

And finally lets verify

```bash
openssl dgst -sha256 -verify ecpubkey.pem -signature signature payload
Verified OK
```

To delete and start again

```bash
kubectl delete -f hello-world-pipeline.yaml
```
