<!--
---
title: "Clone a git repository with Tekton"
linkTitle: "Clone a git repository with Tekton"
weight: 1
description: >
  How to clone source code from git with Tekton Pipelines.
---
-->

This guide shows you how to:

1.  Create a Task to clone source code from a git repository.
1.  Create a second Task to read the source code from a shared Workspace.

If you are already familiar with Tekton and just want to see the example, you
can [skip to the full code samples](#full-code-samples).

## Prerequisites

1.  To follow this How-to you must have a Kubernetes cluster up and running and
    [kubectl][kubectl] properly configured to issue commands to your cluster.

1.  Install Tekton Pipelines:
    
    ```bash
    kubectl apply --filename \
    https://storage.googleapis.com/tekton-releases/pipeline/latest/release.yaml
    ```

    See the [Pipelines installation documentation][pipelines-inst] for other
    installation options.

1.  Install the [Tekton CLI, `tkn`][tkn-inst], on your machine.

[pipelines-inst]: /docs/pipelines/install/
[tkn-inst]: /docs/cli/
[kubectl]: https://kubernetes.io/docs/tasks/tools/#kubectl

If this is your first time using Tekton Pipelines, we recommend that you
complete the [Getting Started tutorials][getting-started] before proceeding with
this guide.

## Pull source code from git

In this section you are going to create a Pipeline containing a Task to pull
code from a git repository.

### Create the Pipeline

One practical aspect of Tekton Tasks and Pipelines is that they are reusable.
There's a [community hub][tekton-hub] with a curated list of Pipelines and Tasks
that you can include in your own CI/CD workflow. You are going to reuse one of
those Tasks in this guide.

1.  Create a new Pipeline, `pipeline.yaml`:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: Pipeline
    metadata:
      name: clone-read
    ```

    The name `clone-read` is going to be used by the PipelineRun to refer to
    this Pipeline. This name can also be used with the CLI, to check logs,
    delete the pipeline, etc.

1.  Add the repository URL to the list of Pipeline [parameters][params]:

    ```yaml
    spec:
      description: | 
        This pipeline clones a git repo, then echoes the README file to the stdout.
      params:
      - name: repo-url
        type: string
        description: The git repo URL to clone from.
    ```

    The `params` section contains a list of parameters to be used by the Tasks
    in this Pipeline. For now there is only one, `repo-url`.

1.  Add a [Workspace][workspace], a shared volume to store the code this task is
    going to download: 

    ```yaml
    workspaces:
    - name: shared-data
      description: | 
        This workspace contains the cloned repo files, so they can be read by the
        next task.
    ```

1.  Create the [Task][tasks] that is going to use the parameter and the
    Workplace that you just defined:

    ```yaml
    tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
      - name: output
        workspace: shared-data
      params:
      - name: url
        value: $(params.repo-url)
    ```

    This Task, `fetch-source`, refers to another Task, `git-clone`; to be
    installed from the community hub. A Task has its own `params` and
    `workspaces` passed down from the ones defined at Pipeline level. In this
    case, the names for the  parameter `url` and the Workspace `output` are the
    the ones expected by the *git clone* Task spec. See the [git
    clone][git-clone] documentation for more parameters and options.

Check the [full code samples](#full-code-samples) to see how all the pieces fit
together in the file.

[params]: /docs/pipelines/pipelines/#specifying-parameters
[workspace]: /docs/pipelines/pipelines/#specifying-workspaces
[tasks]: /docs/pipelines/tasks/#configuring-a-task

### Create the PipelineRun

1.  Now that you have a Pipeline, to instantiate it and set the actual values,
    create a PipelineRun, `pipelinerun.yaml`:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: PipelineRun
    metadata:
      generateName: clone-read-run-
    spec:
      pipelineRef:
        name: clone-read
      podTemplate:
        securityContext:
          fsGroup: 65532
    ```

    This PipelineRun instantiates `clone-read`, as specified by [the
    `pipelineRef` target][pipelineref] in the `spec` section.

1.  Instantiate the Workspace:
  
    ```yaml
    workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    ```

    This creates a [Persistent Volume Claim][volume-claim] to store the cloned
    files. The name `shared-data` matches the name used in the Pipeline.

1.  Set the value of the repository URL parameter:    

    ```yaml
    params:
    - name: repo-url
      value: https://github.com/tektoncd/website
    ```
    
    For this example you are going to clone the Tekton documentation website
    source code.

Check the [full code samples](#full-code-samples) to see full PipelineRun code.

[pipelineref]: /docs/pipelines/pipelineruns/#specifying-the-target-pipeline
[volume-claim]: /docs/pipelines/workspaces/#using-persistentvolumeclaims-as-volumesource

### Git authentication

For the sake of this example, we are using a public repository, which requires
no authentication. If you want to clone a private repository, you must create a
[Kubernetes Secret][secrets] with your credentials, then pass that secret to
your Task as a Workspace.

Before you proceed, you have to set up SSH authentication with your git
provider. The process may be slightly different in each case:

-   [GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
-   [GitLab](https://docs.gitlab.com/ee/user/ssh.html)
-   [Bitbucket](https://support.atlassian.com/bitbucket-cloud/docs/set-up-an-ssh-key/)

The following steps show you how to authenticate using an SSH key. For other
forms of authentication check the [git-clone Task documentation][git-clone] and
the [Git authentication section][pipelines-git] in the Pipelines documentation.

1.  Create a Kubernetes Secret with your credentials, for example:
     
    ```yaml
    apiVersion: v1
    kind: Secret
    metadata:
      name: git-credentials
    data:
      id_rsa: AS0tLS...
      known_hosts: AG033S...
      config: GS0FFL...
    ``` 

    The values for the fields under `data` are the corresponding base64-encoded
    files in the `.ssh` directory. For example, for `id_rsa` copy-paste the
    output of:

    ```bash
    cat ~/.ssh/id_rsa | base64 -w0
    ```

1.  Update `pipeline.yaml`, add a new Workspace to both the Pipeline and the
    Task:

    ```yaml
    workspaces:
    - name: shared-data
      description: | 
        This workspace contains the cloned repo files, so they can be read by the
        next task.
    - name: git-credentials
      description: My ssh credentials
    tasks:
    - name: fetch-source
      taskRef:
        name: git-clone
      workspaces:
      - name: output
        workspace: shared-data
      - name: ssh-directory
        workspace: git-credentials
    ```

    The Workspace `git-credentials` is defined at Pipeline level and then passed
    down to the Task as `ssh-directory`, which is the name the Task expects.

1.  Update `pipelinerun.yaml` to use the Secret as a Workspace and change the
    git URL from https to SSH:

    ```yaml
    workspaces:
    - name: shared-data
      volumeClaimTemplate:
        spec:
          accessModes:
          - ReadWriteOnce
          resources:
            requests:
              storage: 1Gi
    - name: git-credentials
      secret:
        secretName: git-credentials
    params:
    - name: repo-url
      value: git@github.com:tektoncd/website.git
    ```

    The new Workspace name, `git-credentials`, matches the Workspace added to
    the Pipeline.

[pipelines-git]: /docs/pipelines/auth/#configuring-authentication-for-git

## Use the source code in a second task

To learn how to share data between Tasks using a Workspace, you are going to
create a second Task that displays  the `README` file from the cloned git
repository. You can find more useful examples in the [How-to section][how-tos]
and the [examples folder][repo-examples] in the Pipelines git repository.

1.  Add a new entry to the `tasks` section of `pipeline.yaml`:

    ```yaml
    - name: show-readme
      runAfter: ["fetch-source"]
      taskRef:
        name: show-readme
      workspaces:
      - name: source
        workspace: shared-data
    ```

    This is referencing the Task `show-readme`. Unlike *git clone*, this Task is
    not from the Tekton community hub, you have to create it yourself. 

1.  Create a file called `show-readme.yaml` and add the following:

    ```yaml
    apiVersion: tekton.dev/v1beta1
    kind: Task
    metadata:
      name: show-readme
    spec:
      description: Read and display README file.
      workspaces:
      - name: source
      steps:
      - name: read
        image: alpine:latest
        script: | 
          #!/usr/bin/env sh
          cat $(workspaces.source.path)/README.md
    ```

    This Task uses a Workspace that was already set. You don't have to update
    `pipelinerun.yaml`.

[how-tos]: /docs/how-to-guides/
[repo-examples]: https://github.com/tektoncd/pipeline/tree/main/examples

## Run the Pipeline

### Install a Task from Tekton Hub

To use the *git clone* Task in your pipeline, you have to install it on your
cluster first. You can do this with the `tkn` command:

```bash
tkn hub install task git-clone
```

Or using `kubectl`:

```bash
kubectl apply -f \
https://raw.githubusercontent.com/tektoncd/catalog/main/task/git-clone/0.6/git-clone.yaml
```
> [!IMPORTANT]
> When using **Git-Clone** you should make sure that your private key is in the correct format. Libcrypto requires you to have a new line at the end of your private_key (PEM) format. Checkout: https://github.com/tektoncd/catalog/issues/1220 for more information.

Alternatively, you can [bundle a Task or a Pipeline][bundles] and let your
Pipeline fetch it directly from a registry.

### Apply the files

Now you are ready to test the code.

1.  Apply the `show-readme` Task:

    ```bash
    kubectl apply -f show-readme.yaml
    ```

1.  Apply the Pipeline:

    ```bash
    kubectl apply -f pipeline.yaml
    ```

1.  Create the PipelineRun:

    ```bash
    kubectl create -f pipelinerun.yaml
    ```

    This creates a PipelineRun with a unique name each time:

    ```
    pipelinerun.tekton.dev/clone-read-run-4kgjr created
    ```


1.  Use the PipelineRun name from the output of the previous step to monitor the
    Pipeline execution:

    ```bash
    tkn pipelinerun logs  clone-read-run-4kgjr -f
    ```

    You may have to wait a few seconds. The output confirms that the respository
    was cloned succesfully and displays the README file at the end:

    ```
    [fetch-source : clone] + '[' false '=' true ]
    [fetch-source : clone] + '[' false '=' true ]
    [fetch-source : clone] + '[' false '=' true ]
    [fetch-source : clone] + CHECKOUT_DIR=/workspace/output/
    [fetch-source : clone] + '[' true '=' true ]
    [fetch-source : clone] + cleandir
    [fetch-source : clone] + '[' -d /workspace/output/ ]
    [fetch-source : clone] + rm -rf '/workspace/output//*'
    [fetch-source : clone] + rm -rf '/workspace/output//.[!.]*'
    [fetch-source : clone] + rm -rf '/workspace/output//..?*'
    [fetch-source : clone] + test -z 
    [fetch-source : clone] + test -z 
    [fetch-source : clone] + test -z 
    [fetch-source : clone] + /ko-app/git-init '-url=https://github.com/tektoncd/website' '-revision=' '-refspec=' '-path=/workspace/output/' '-sslVerify=true' '-submodules=true' '-depth=1' '-sparseCheckoutDirectories='
    [fetch-source : clone] {"level":"info","ts":1652300245.5099113,"caller":"git/git.go:170","msg":"Successfully cloned https://github.com/tektoncd/website @ 4930334b17edeaa737e2e6d0c7f7139b0afb1896 (grafted, HEAD) in path /workspace/output/"}
    [fetch-source : clone] {"level":"info","ts":1652300245.5349698,"caller":"git/git.go:208","msg":"Successfully initialized and updated submodules in path /workspace/output/"}
    [fetch-source : clone] + cd /workspace/output/
    [fetch-source : clone] + git rev-parse HEAD
    [fetch-source : clone] + RESULT_SHA=4930334b17edeaa737e2e6d0c7f7139b0afb1896
    [fetch-source : clone] + EXIT_CODE=0
    [fetch-source : clone] + '[' 0 '!=' 0 ]
    [fetch-source : clone] + printf '%s' 4930334b17edeaa737e2e6d0c7f7139b0afb1896
    [fetch-source : clone] + printf '%s' https://github.com/tektoncd/website

    [show-readme : read] # TektonCD Website
    [show-readme : read] 
    [show-readme : read] This repo contains the code behind [the Tekton org's](https://github.com/tektoncd)
    [show-readme : read] website at [tekton.dev](https://tekton.dev).
    [show-readme : read] 
    [show-readme : read] For more information on the Tekton Project, see
    [show-readme : read] [the community repo](https://github.com/tektoncd/community).
    [show-readme : read] 
    [show-readme : read] For more information on contributing to the website see:
    [show-readme : read] 
    [show-readme : read] * [CONTRIBUTING.md](CONTRIBUTING.md)
    [show-readme : read] * [DEVELOPMENT.md](DEVELOPMENT.md)
    ```

## Full code samples

The Pipeline:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Pipeline
metadata:
  name: clone-read
spec:
  description: | 
    This pipeline clones a git repo, then echoes the README file to the stout.
  params:
  - name: repo-url
    type: string
    description: The git repo URL to clone from.
  workspaces:
  - name: shared-data
    description: | 
      This workspace contains the cloned repo files, so they can be read by the
      next task.
  - name: git-credentials
    description: My ssh credentials
  tasks:
  - name: fetch-source
    taskRef:
      name: git-clone
    workspaces:
    - name: output
      workspace: shared-data
    - name: ssh-directory
      workspace: git-credentials
    params:
    - name: url
      value: $(params.repo-url)
  - name: show-readme
    runAfter: ["fetch-source"]
    taskRef:
      name: show-readme
    workspaces:
    - name: source
      workspace: shared-data
```

The PipelineRun:

```yaml
apiVersion: tekton.dev/v1beta1
kind: PipelineRun
metadata:
  generateName: clone-read-run-
spec:
  pipelineRef:
    name: clone-read
  podTemplate:
    securityContext:
      fsGroup: 65532
  workspaces:
  - name: shared-data
    volumeClaimTemplate:
      spec:
        accessModes:
        - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi
  - name: git-credentials
    secret:
      secretName: git-credentials
  params:
  - name: repo-url
    value: git@github.com:tektoncd/website.git
```

The `show-readme` Task:

```yaml
apiVersion: tekton.dev/v1beta1
kind: Task
metadata:
  name: show-readme
spec:
  description: Read and display README file.
  workspaces:
  - name: source
  steps:
  - name: read
    image: alpine:latest
    script: | 
      #!/usr/bin/env sh
      cat $(workspaces.source.path)/README.md
```

The Kubernetes Secret. These values are not real, check the section about [git
authentication](#git-authentication) to figure out how to encode your
credentials.

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: git-credentials
data:
  id_rsa: AS0tLS...
  known_hosts: AG033S...
```

## Further reading

- [Build and push and image with Kaniko and Tekton][kaniko-build].
- [Full Tasks and Pipelines documentation](/docs/pipelines/).

[tekton-hub]: https://hub.tekton.dev/
[git-clone]: https://hub.tekton.dev/tekton/task/git-clone
[workspaces]: /docs/pipelines/workspaces/
[bundles]: /docs/pipelines/pipelines/#tekton-bundles
[getting-started]: /docs/getting-started/
[secrets]: https://kubernetes.io/docs/concepts/configuration/secret/
[kaniko-build]: /docs/how-to-guides/kaniko-build-push/
