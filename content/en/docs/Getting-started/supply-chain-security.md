<!--
---
title: "Getting Started Supply Chain Security"
linkTitle: "Getting Started Supply Chain Security"
weight: 4
description: >
  Create and sign artifact provenance with Tekton Chains
---
-->

This guide shows you how to:

- Create a Pipeline to build and push a container image to a local registry.
- Record and sign provenance of the image.
- Read back the provenance information.
- Verify the signature.

## Prerequisites

1. [Install minikube][minikube]. Only complete the step 1, "Installation".
1. [Install kubectl][kubectl].
1. [Install tkn, the Tekton CLI][tkn].
1. [Install jq][jq].
1. [Install cosign][cosign].

## Start minikube with a local registry enabled

1. Delete any previous clusters:

   ```bash
   minikube delete
   ```

1. Start up minikube with insecure registry enabled:

   ```bash
   minikube start --insecure-registry "10.0.0.0/24"
   ```

   The process takes a few seconds, you see an output similar to the following,
   depending on the [minikube driver](https://minikube.sigs.k8s.io/docs/drivers/)
   that you are using:
   
   ```
   😄  minikube v1.29.0
   ✨  Automatically selected the docker driver. Other choices: none, ssh
   📌  Using Docker driver with root privileges
   👍  Starting control plane node minikube in cluster minikube
   🚜  Pulling base image ...
   🔥  Creating docker container (CPUs=2, Memory=7900MB) ...
   🐳  Preparing Kubernetes v1.26.1 on Docker 20.10.23 ...
       ▪ Generating certificates and keys ...
       ▪ Booting up control plane ...
       ▪ Configuring RBAC rules ...
   🔗  Configuring bridge CNI (Container Networking Interface) ...
       ▪ Using image gcr.io/k8s-minikube/storage-provisioner:v5
   🌟  Enabled addons: storage-provisioner, default-storageclass
   🔎  Verifying Kubernetes components...
   🏄  Done! kubectl is now configured to use "minikube" cluster and "default" namespace by default
   ```

1. Enable the local registry plugin:

   ```bash
   minikube addons enable registry 
   ```
   The output confirms that the registry plugin is enabled:

   ```
   💡  registry is an addon maintained by Google. For any concerns contact minikube on GitHub.
   You can view the list of minikube maintainers at: https://github.com/kubernetes/minikube/blob/master/OWNERS
       ▪ Using image docker.io/registry:2.8.1
       ▪ Using image gcr.io/google_containers/kube-registry-proxy:0.4
   🔎  Verifying registry addon...
   🌟  The 'registry' addon is enabled
   ```

Now you can push images to a registry within your minikube cluster.

## Install and configure the necessary Tekton components

1. Install Tekton Pipelines:

   ```bash
   kubectl apply --filename \
   https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

1. Monitor the installation:

   ```bash
   kubectl get po -n tekton-pipelines -w
   ```

   When both `tekton-pipelines-controller` and `tekton-pipelines-webhook` show `1/1`
   under the `READY` column, you are ready to continue. For example:

   ```
   NAME                                          READY   STATUS    RESTARTS   AGE
   tekton-pipelines-controller-9675574d7-sxtm4   1/1     Running   0          2m28s
   tekton-pipelines-webhook-58b5cbb7dd-s6lfs     1/1     Running   0          2m28s
   ```

   Hit *Ctrl + C* to stop monitoring.

1. Install Tekton Chains:

   ```bash
   kubectl apply --filename \
   https://storage.googleapis.com/tekton-releases/chains/latest/release.yaml
   ``` 

1. Monitor the installation

   ```bash
   kubectl get po -n tekton-chains -w
   ```

   When `tekton-chains-controller` shows `1/1` under the `READY` column, you
   are ready to continue. For example:

   ```
   NAME                                        READY   STATUS    RESTARTS   AGE
   tekton-chains-controller-57dcc994b9-vs2f2   1/1     Running   0          2m23s
   ```

   Hit *Ctrl + C* to stop monitoring.

1. Configure Tekton Chains to store the provenance metadata locally:

   ```bash
   kubectl patch configmap chains-config -n tekton-chains \
   -p='{"data":{"artifacts.oci.storage": "", "artifacts.taskrun.format":"in-toto", "artifacts.taskrun.storage": "tekton"}}'
   ```

   The output confirms that the configuration was updated successfully:

   ```
   configmap/chains-config patched
   ```

1. Generate a key pair to sign the artifact provenance:

   ```bash
   cosign generate-key-pair k8s://tekton-chains/signing-secrets
   ```

   You are prompted to enter a password for the private key. For this guide,
   leave the password empty and press *Enter* twice. A public key, `cosign.pub`,
   is created in your current directory.

## Build and push a container image

1. Create a file called `pipeline.yaml` and add the following:

   {{% readfile file="samples/build-push.yaml" code="true" lang="yaml" %}}

1. Get your cluster IPs:

   ```bash
   kubectl get service --namespace kube-system
   ```

   This shows the IPs of the services on your cluster:

   ```
   NAME       TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)                  AGE
   kube-dns   ClusterIP   10.96.0.10      <none>        53/UDP,53/TCP,9153/TCP   48m
   registry   ClusterIP   10.101.134.48   <none>        80/TCP,443/TCP           47m
   ```

   Save your registry IP, in this case `10.101.134.48`, for the next step.

1. Create a file called `pipelinerun.yaml` and add the following:

   {{% readfile file="samples/build-push-run.yaml" code="true" lang="yaml" %}}

   Replace `<registry-ip>` with the value from the previous step.

1. Apply the Pipeline to your cluster:

   ```bash
   kubectl apply -f pipeline.yaml
   ```

   You see the following output:

   ```
   pipeline.tekton.dev/build-push created
   task.tekton.dev/create-dockerfile created
   task.tekton.dev/kaniko created
   ```

1. Run the Pipeline:

   ```bash
   kubectl create -f pipelinerun.yaml
   ```

   A new PipelineRun with a unique name is created:

   ```
   pipelinerun.tekton.dev/build-push-run-q22b5 created 
   ```

1. Use the PipelineRun name, `build-push-run-q22b5` , to monitor the
   execution:

   ```bash
   tkn pr logs build-push-run-q22b5 -f
   ```

   The output shows the Pipeline completed successfully:

   ```
   [kaniko-build : build-and-push] INFO[0000] Retrieving image manifest alpine:3.16
   [kaniko-build : build-and-push] INFO[0000] Retrieving image alpine:3.16 from registry index.docker.io
   [kaniko-build : build-and-push] INFO[0000] Built cross stage deps: map[]
   [kaniko-build : build-and-push] INFO[0000] Retrieving image manifest alpine:3.16
   [kaniko-build : build-and-push] INFO[0000] Returning cached image manifest
   [kaniko-build : build-and-push] INFO[0000] Executing 0 build triggers
   [kaniko-build : build-and-push] INFO[0000] Unpacking rootfs as cmd RUN echo "hello world" > hello.log requires it.
   [kaniko-build : build-and-push] INFO[0000] RUN echo "hello world" > hello.log
   [kaniko-build : build-and-push] INFO[0000] Taking snapshot of full filesystem...
   [kaniko-build : build-and-push] INFO[0000] cmd: /bin/sh
   [kaniko-build : build-and-push] INFO[0000] args: [-c echo "hello world" > hello.log]
   [kaniko-build : build-and-push] INFO[0000] Running: [/bin/sh -c echo "hello world" > hello.log]
   [kaniko-build : build-and-push] INFO[0000] Taking snapshot of full filesystem...
   [kaniko-build : build-and-push] INFO[0000] Pushing image to 10.101.134.48/tekton-test
   [kaniko-build : build-and-push] INFO[0001] Pushed image to 1 destinations

   [kaniko-build : write-url] 10.101.134.48/tekton-test
   ```

## Retrieve and verify the artifact provenance

Tekton Chains silently monitored the execution of the PipelineRun. It recorded
and signed the provenance metadata, information about the container that the
PipelineRun built and pushed.

1. Get the PipelineRun UID:

   ```bash
   export PR_UID=$(tkn pr describe --last -o  jsonpath='{.metadata.uid}')
   ```

1. Fetch the metadata and store it in a JSON file:

   ```bash
   tkn pr describe --last \
   -o jsonpath="{.metadata.annotations.chains\.tekton\.dev/signature-pipelinerun-$PR_UID}" \
   | base64 -d > metadata.json
   ```

1. View the provenance:

   ```bash
   cat metadata.json | jq -r '.payload' | base64 -d | jq .
   ```

   The output contains a detailed description of the build:

   {{% readfile file="samples/provenance.json" code="true" lang="json" %}}

1. To verify that the metadata hasn't been tampered with, check the signature
   with `cosign`:

   ```bash
   cosign verify-blob-attestation --insecure-ignore-tlog \
   --key k8s://tekton-chains/signing-secrets --signature metadata.json \
   --type slsaprovenance --check-claims=false /dev/null
   ```

   The output confirms that the signature is valid:
  
   ```
   Verified OK
   ```

## Further reading  

- [Learn about Tekton Chains and Supply Chain Security][chains-overview].
- [Getting To SLSA Level 2 with Tekton and Tekton Chains blog post][blog-post].
- [Check more examples on the Tekton Chains repository][chains-repo].
    
[chains-overview]: /docs/concepts/supply-chain-security/
[chains-repo]: https://github.com/tektoncd/chains/tree/main/examples
[minikube]: https://minikube.sigs.k8s.io/docs/start/
[kubectl]: https://kubernetes.io/docs/tasks/tools/#kubectl
[tkn]: /docs/cli/
[jq]: https://stedolan.github.io/jq/download/
[cosign]: https://docs.sigstore.dev/cosign/system_config/installation/
[blog-post]: /blog/2023/04/19/getting-to-slsa-level-2-with-tekton-and-tekton-chains/

