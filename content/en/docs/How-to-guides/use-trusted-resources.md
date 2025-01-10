<!--
---
title: "Use Trusted Resources with Tekton"
linkTitle: "Use Trusted Resources"
weight: 3
description: >
  How to sign and verify Tekton resources
---
-->

This guide shows you how to:

1. Sign Tekton Tasks and Pipelines with cosign and KMS.
1. Verify signed Tekton Tasks and Pipelines.

## Prerequisites

1. To follow this How-to you must have a Kubernetes cluster up and running and
   [kubectl][kubectl] properly configured to issue commands to your cluster.


1. Install the latest release of Tekton Pipelines:
   
   ```bash
   kubectl apply --filename \
   https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
   ```

   See the [Pipelines installation documentation][pipelines-inst] for other
   installation options.

1. Install the [Tekton CLI, `tkn`][tkn-inst], on your machine.

1. Install [cosign][cosign].

## Signing Tasks and Pipelines

For this guide your are going two use the code from the [Getting Started with
Pipelines][pipelines-quicktart] guide. Namely, these files are:

+ The `hello.yaml` Task.
+ The `goodbye.yaml` Task.
+ The `pipeline.yaml` Pipeline.

You can find these samples at the end of this document.

To sign Tekton resources you can use two different tools, Cosign or a Key
Management System (KMS):

{{% tabs %}}

{{% tab "Cosign" %}}

1. Generate a key pair to sign the artifact provenance:

   ```bash
   cosign generate-key-pair
   ```

   You are prompted to enter a password for the private key. Type a password
   and press *Enter*. Public and private keys, `cosign.pub` and `cosign.key`
   respectively, are created in the current directory.

1. Sing the tasks YAML files with the private key using the Tekton CLI. You
   can find sample sample code in the [Code Samples](#code-samples)
   section.

   ```bash
   tkn task sign hello.yaml -K="cosign.key" -f="signed-hello.yaml"
   tkn task sign goodbye.yaml -K="cosign.key" -f="signed-goodbye.yaml"
   ```

   The command outputs two signed Tasks `signed-hello.yaml` and
   `signed-goodbye.yaml`.

1. Push the signed Tasks to a Git repository. For other remote storage options
   see the [remote resolution documentation][remote-reso].

1. Modify `pipeline.yaml` to use the signed Tasks using remote resolution. For
   example, the following snippet points to the "hello" Task on GitHub.

   ```yaml
   tasks:
     - name: hello
       taskRef:
         resolver: git
         params:
           - name: url
             value: https://github.com/<username>/<repository>
           - name: revision
             value: main
           - name: pathInRepo
             value: signed-hello.yaml
   ```

   You can find the full `pipeline.yaml` in the [Code Samples](#code-samples)
   section.

1. Sign the `pipeline.yaml` file:
     
   ```bash
   tkn pipeline sign pipeline.yaml -K="cosign.key" \
    -f="signed-pipeline.yaml"
   ```

   The output is the `signed-pipeline-cosign.yaml`.

1. Push the signed Pipeline to a Git repository. This repository can be the same
   as the one where you pushed the Tasks.

[remote-reso]: /docs/pipelines/resolution/
{{% /tab %}}

{{% tab "Google Cloud" %}}

1. Create a keyring if you haven't already:

   ```bash
   gloud kms keyrings create <key_ring> --location <location>
   ```

   Replace the following:

   + `key_ring`: Name of the keyring.
   +  `location`: [Cloud KMS location][kms-location] of the keyring

1. Set up a KMS asymmetric signing key:

   ```bash
   gcloud kms keys create <key_name> \
       --keyring <key_ring> \
       --location <location> \
       --purpose "asymmetric-signing" \
       --default-algorithm "<algorithm>"
   ```

   Replace the following:

   +  `key_name`: Name of the key.
   +  `key_ring`: Name of the keyring where you are storing the key.
   +  `location`: Cloud KMS location of the ring.
   +  `algorithm`: [Asymmetric signing alorithm][signing-alg] to use for the
      key. The recommended algorithm is `ec-sign-p256-sha256`.

   For more information see the [Google Cloud KMS documentation][kms-docs].

1. Log in to your Google Cloud account:

   ```bash
   gcloud auth application-default login
   ```

1. Sing the Taks YAML files with your KMS private key:

   ```bash
   tkn task sign hello.yaml \
   -m="<key>" \
   -f="signed-hello.yaml"


   tkn task sign goodbye.yaml \
   -m="<key>" \
   -f="signed-goodbye.yaml"
   ```

   Replace `<key>` with the KMS key URL. For example: `gcpkms://projects/user-test/locations/us/keyRings/trusted-task-demo/cryptoKeys/trusted-task/cryptoKeyVersions/1`

1. Push the signed Tasks to a Git repository. For other remote storage options
   see the [remote resolution documentation][remote-reso].

1. Modify `pipeline.yaml` to use the signed Tasks using remote resolution. For
   example, the following snippet points to the "hello" Task on GitHub.

   ```yaml
   tasks:
     - name: hello
       taskRef:
         resolver: git
         params:
           - name: url
             value: https://github.com/<username>/<repository>
           - name: revision
             value: main
           - name: pathInRepo
             value: signed-hello.yaml
   ```

   You can find the full `pipeline.yaml` in the [Code Samples](#code-samples)
   section.

1. Sign the `pipeline.yaml` file:
     
   ```bash
   tkn task sign pipeline.yaml \
   -m="<key>" \
   -f="signed-pipeline.yaml"
   ```

   The output is the `signed-pipeline-cosign.yaml`.

1. Push the signed Pipeline to a Git repository. This repository can be the same
   as the one where you pushed the Tasks.

[remote-reso]: https://github.com/tektoncd/pipeline/blob/main/docs/resolution.md
[kms-docs]: https://cloud.google.com/kms/docs/create-key#create-asymmetric-sign
[kms-location]: https://cloud.google.com/kms/docs/locations
[signing-alg]: https://cloud.google.com/kms/docs/algorithms#asymmetric_signing_algorithms
{{% /tab %}}

{{% /tabs %}}

## Configure your cluster

1. To verify the signatures, enable policy verification on your cluster. Create
   a file called `configmap.yaml`:

   ```yaml
   apiVersion: v1
   kind: ConfigMap
   metadata:
     name: feature-flags
     namespace: tekton-pipelines
     labels:
       app.kubernetes.io/instance: default
       app.kubernetes.io/part-of: tekton-pipelines
   data:
     trusted-resources-verification-no-match-policy: "fail"   
   ```

   To customize how the feature behaves, use one of the following values for
   `trusted-resources-verification-no-match-policy`:

   + `ignore`: Use it if you want to verify signed resources and skip unsigned
     resources. If no matching policies are found, skip the verification and
     don't log. This won't fail the TaksRun or PipelineRun. 
   + `warn`: If no matching policies are found, skipe the verification and log
     a warning. Dont' fail the Taskrun or PipelineRun.
   + `fail`: Fail the TaskRun or PipelineRun if no matching policies are found.

1. Apply the configuration to your cluster:

   ```bash
   kubectl apply -f configmap.yaml
   ```


## Artifact Verification

### Create a verification policy

{{% tabs %}}

{{% tab "Cosign" %}}

1.  Create a file called `verification-policy.yaml` and add the following:

    ```yaml
    apiVersion: tekton.dev/v1alpha1
    kind: VerificationPolicy
    metadata:
      name: cosign-policy
      namespace: trusted-resources
    spec:
      resources:
        - pattern: "https://github.com/user/sample-tekton-task"
        - pattern: "https://github.com/user/sample-tekton-pipeline"
      authorities:
        - name: cosign
          key:
            secretRef:
              name: verification-secrets
              namespace: tekton-pipelines
      mode: enforce
    ```

1.  Apply the policy to the cluster:

    ```bash
    kubectl apply -f verification-policy.yaml
    ```

1. Create a secret with your key

{{% /tab %}}

{{% tab "Google Cloud" %}}

1. Create a file called `verification-policy.yaml` and add the following:

   ```yaml
   apiVersion: tekton.dev/v1alpha1
   kind: VerificationPolicy
   metadata:
     name: kms-policy
     namespace: trusted-resources
   spec:
     resources:
       - pattern: "https://github.com/user/sample-tekton-task"
       - pattern: "https://github.com/user/sample-tekton-pipeline"
     authorities:
       - name: kms
         key:
           kms:
   gcpkms://projects/user-test/locations/us/keyRings/trusted-task-demo/cryptoKeys/trusted-task/cryptoKeyVersions/1
     mode: enforce
   ```

1. Apply the policy to the cluster:

   ```bash
   kubectl apply -f verification-policy.yaml
   ```

{{% /tab %}}

{{% /tabs %}}

### Verify Tekton resources

Submit your PipelineRun:

```bash
kubectl apply -f pipelinerun.yaml
```

The verification was completed successfully:

```none
Status:
  Child References:
    API Version:         tekton.dev/v1beta1
    Kind:                TaskRun
    Name:                example-pipelinerun-example-task
    Pipeline Task Name:  example-task
  Completion Time:       2023-06-16T19:30:17Z
  Conditions:
    Last Transition Time:  2023-06-16T19:30:17Z
    Message:               Tasks Completed: 1 (Failed: 0, Cancelled 0), Skipped: 0
    Reason:                Succeeded
    Status:                True
    Type:                  Succeeded
    Last Transition Time:  2023-06-16T19:30:07Z
    Status:                True
    Type:                  TrustedResourcesVerified
```

The `Status: True` of the `Type: TrustedResourcesVerified` confirms that the
resources passed the verification.

If the verification fails, the output is similar to the following:

```none
Status:
  Completion Time:  2023-06-16T21:29:08Z
  Conditions:
    Last Transition Time:  2023-06-16T21:29:08Z
    Message:               PipelineRun trusted-resources/example-pipelinerun-kms referred resource example-pipeline failed signature verification: resource verification failed: resource example-pipeline in namespace trusted-resources fails verification
    Reason:                ResourceVerificationFailed
    Status:                False
    Type:                  Succeeded
    Last Transition Time:  2023-06-16T21:29:08Z
    Message:               PipelineRun trusted-resources/example-pipelinerun-kms referred resource example-pipeline failed signature verification: resource verification failed: resource example-pipeline in namespace trusted-resources fails verification
    Status:                Falsenone
Status:
  Completion Time:  2023-06-16T21:29:08Z
  Conditions:
    Last Transition Time:  2023-06-16T21:29:08Z
    Message:               PipelineRun trusted-resources/example-pipelinerun-kms
referred resource example-pipeline failed signature verification: resource
verification failed: resource example-pipeline in namespace trusted-resources
fails verification
    Reason:                ResourceVerificationFailed
    Status:                False
    Type:                  Succeeded
    Last Transition Time:  2023-06-16T21:29:08Z
    Message:               PipelineRun trusted-resources/example-pipelinerun-kms
referred resource example-pipeline failed signature verification: resource
verification failed: resource example-pipeline in namespace trusted-resources
fails verification
    Status:                False
    Type:                  TrustedResourcesVerified
    Type:                  TrustedResourcesVerified
```

## Code samples:

+   **Sample Task**

    ```yaml
    kind: Task
    apiVersion: tekton.dev/v1beta1
    metadata:
      name: example-task
      namespace: trusted-resources
    spec:
      steps:
        - image: ubuntu
          name: echo
          command:
            - echo
          args:
            - "Hello World"
    ```

+   **Sample Pipeline**

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: example-pipeline
      namespace: trusted-resources
    spec:
      tasks:
        - name: example-task
          taskRef:
            resolver: git
            params:
              - name: url
                value: https://github.com/user/sample-tekton-task
              - name: revision
                value: main
              - name: pathInRepo
                value: signed-task.yaml
    ```

+   **Sample PipelineRun for Cosign-signed resource**

    ```yaml
    apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      name: example-pipelinerun
      namespace: trusted-resources
    spec:
      pipelineRef:
        resolver: git
        params:
          - name: url
            value: https://github.com/Yongxuanzhang/sample-tekton-pipeline
          - name: revision
            value: main
          - name: pathInRepo
            value: signed-pipeline-cosign.yaml
    ```

+   **Sample PipelineRun for KMS-signed resource**

    ```yaml
    apiVersion: tekton.dev/v1
    kind: PipelineRun
    metadata:
      name: example-pipelinerun-kms
      namespace: trusted-resources
    spec:
      pipelineRef:
        resolver: git
        params:
          - name: url
            value: https://github.com/Yongxuanzhang/sample-tekton-pipeline
          - name: revision
            value: main
          - name: pathInRepo
            value: signed-pipeline-kms.yaml
    ```

+   **Failed verification**

    ```
    Name:              pipelinerun-cgx99
    Namespace:         default
    Service Account:   default
    Labels:
     tekton.dev/pipeline=pipelinerun-cgx99
    Annotations:
     tekton.dev/signature=MEYCIQCFm6tAXRVf4O28WlqBRkUjj0SsUP6Z2NRCnF3yey3SCQIhANlEcJsQavVfS9xpErGb8X/TDmpBKZHzy6MCYVFKcJW1
    
    Status
    
    STARTED         DURATION   STATUS
    2 minutes ago   0s         Failed(ResourceVerificationFailed)
    
    Message
    
    PipelineRun default/pipelinerun-cgx99 referred resource hello failed signature verification: resource verification failed: resource hello in namespace  fails verification
    
    Timeouts
     Pipeline:   1h0m0s
    
    Params
    
     NAME       VALUE
     username   Tekton
    
    ```
    

[pipelines-inst]: /docs/pipelines/install/
[tkn-inst]: /docs/cli/
[kubectl]: https://kubernetes.io/docs/tasks/tools/#kubectl
[cosign]: https://docs.sigstore.dev/cosign/installation/
[pipelines-quicktart]: /docs/getting-started/pipelines/

