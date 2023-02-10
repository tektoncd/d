To install Tekton Pipelines on OpenShift, you must first apply the `anyuid`
security context constraint to the `tekton-pipelines-controller` service
account. This is required to run the webhook Pod.  See [Security Context
Constraints][security-con] for more information.

1. Log on as a user with `cluster-admin` privileges. The following example
   uses the default `system:admin` user:

   ```bash
   oc login -u system:admin
   ```

1. Set up the namespace (project) and configure the service account:

   ```bash
   oc new-project tekton-pipelines
   oc adm policy add-scc-to-user anyuid -z tekton-pipelines-controller
   oc adm policy add-scc-to-user anyuid -z tekton-pipelines-webhook
   ```
1. Install Tekton Pipelines:

    Because OpenShift uses random user id (and user id range per namespace) for pods, we need to remove the `securityContext.runAsUser` and `securityContext.runAsGroup` from any container from the release.yaml.
    You will need to have [`yq`](https://mikefarah.gitbook.io/yq/) installed for this to work. Another way would be to download the yaml, search and replace (here replace with nothing) in your favourite editor.

   ```bash
   curl https://storage.googleapis.com/tekton-releases/pipeline/latest/release.notags.yaml | yq 'del(.spec.template.spec.containers[].securityContext.runAsUser, .spec.template.spec.containers[].securityContext.runAsGroup)' | oc apply -f -
   ```


   See the [OpenShift CLI documentation][openshift-cli] for more information on
   the `oc` command.

1. Monitor the installation using the following command until all components
   show a `Running` status:

   ```bash
   oc get pods --namespace tekton-pipelines --watch
   ```

   **Note:** Hit CTRL + C to stop monitoring.

Congratulations! You have successfully installed Tekton Pipelines on your
OpenShift environment.

To run OpenShift 4.x on your laptop (or desktop), take a look at [Red Hat
CodeReady Containers](https://github.com/code-ready/crc).

[openshift-cli]: https://docs.openshift.com/container-platform/4.3/cli_reference/openshift_cli/getting-started-cli.html
[security-con]: https://docs.openshift.com/container-platform/4.3/authentication/managing-security-context-constraints.html
