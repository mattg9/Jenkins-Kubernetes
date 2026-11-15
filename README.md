# Jenkins on Kubernetes (kind)

A local proof of concept for running a Jenkins controller in a [kind](https://kind.sigs.k8s.io/) cluster and using Kubernetes agents.

> **Scope:** This repository is for local learning and demonstration, not production. Jenkins is exposed on the local machine, the agent installs Python at startup as root, and Jenkins configuration and agent provisioning are manual. Do not run untrusted jobs with this setup.

## What is included

- `kind-config.yaml` creates a single-node cluster and maps host port `8080` to the Jenkins service.
- `namespace.yaml` creates the `jenkins` namespace.
- `service-account.yaml` gives the controller namespace-scoped permissions to manage agent pods.
- `volume.yaml` defines local persistent storage for Jenkins home.
- `jenkins-deployment.yaml` and `jenkins-service.yaml` deploy and expose the controller.
- `jenkins-agent.yaml` is a complete Kubernetes Pod manifest intended for the Kubernetes plugin's pod-template YAML field. It is not applied directly by the setup commands.

## Prerequisites

- Docker running locally
- `kind` and `kubectl` installed and available on `PATH`
- Enough local resources for a Kubernetes node and a Jenkins controller (the controller requests 500 MiB and 500 millicores, with limits of 2 GiB and 1 CPU)

## Start Jenkins

From the repository root:

```sh
kind create cluster --config kind-config.yaml
kubectl apply -f namespace.yaml
kubectl apply -f service-account.yaml
kubectl apply -f volume.yaml
kubectl apply -f jenkins-deployment.yaml
kubectl apply -f jenkins-service.yaml
kubectl rollout status deployment/jenkins -n jenkins --timeout=5m
kubectl get pods,services,pvc -n jenkins
```

Open [http://localhost:8080](http://localhost:8080). The first-run administrator password can be read with:

```sh
kubectl exec -n jenkins deployment/jenkins -- cat /var/jenkins_home/secrets/initialAdminPassword
```

Treat that output as a password: do not add it to source control, screenshots, or shared logs.

## Configure a Kubernetes agent

Install the **Kubernetes** plugin from **Manage Jenkins > Plugins**. Then go to **Manage Jenkins > Nodes and Clouds > Configure Clouds > Add a new cloud > Kubernetes**.

Use these minimum cloud settings for this repository:

| Cloud setting | Value |
| --- | --- |
| Name | `kind-jenkins` (any unique name is fine) |
| Kubernetes URL | Leave blank to use in-cluster configuration. If needed, use `https://kubernetes.default.svc.cluster.local`. |
| Kubernetes server certificate key | Leave blank; in-cluster configuration provides the cluster CA. |
| Credentials | Leave blank; Jenkins uses the controller pod's service account. |
| Namespace | `jenkins` |
| Jenkins URL | `http://jenkins-service.jenkins.svc.cluster.local:8080` |
| Jenkins tunnel | `jenkins-service.jenkins.svc.cluster.local:50000` |
| WebSocket | Leave unchecked; this setup uses the inbound-agent service port. |

Use **Test Connection** to check Jenkins can reach the Kubernetes API. The controller deployment must use the `jenkins-controller` service account from `service-account.yaml`; its Role grants pod-management permissions only in the `jenkins` namespace.

Under **Kubernetes Pod Templates**, add a template with:

- **Name:** `python-agent`
- **Namespace:** `jenkins`
- **Labels:** `python-agent`
- **YAML:** paste the contents of `jenkins-agent.yaml`

The manifest includes `apiVersion` and `kind`, so it can be used as a Pod definition in the plugin's YAML field. The plugin supplies `JENKINS_URL`, `JENKINS_SECRET`, and `JENKINS_AGENT_NAME` to the inbound-agent container. The work directory is set explicitly in the manifest. Do not put an actual agent secret in the repository.

To confirm on-demand provisioning, run a pipeline that requests the template's label:

```groovy
pipeline {
    agent { label 'python-agent' }
    stages {
        stage('Verify agent') {
            steps {
                sh 'python3 --version'
            }
        }
    }
}
```

Watch `kubectl get pods -n jenkins -w` during the build; the plugin should create an agent pod for the job and remove it when finished.

The current template runs as root to install Python with `apt-get` whenever an agent starts. This is slow and grants more privilege than a portfolio-ready setup should need. A follow-up should build and pin a dedicated agent image with Python preinstalled, then run it as a non-root user. Leave the pod template's service account at its default unless the build itself needs Kubernetes API access; do not give build agents the controller's service account.

## Validate and inspect

With a reachable Kubernetes context selected, run client-side dry-runs before applying the manifests:

```sh
kubectl apply --dry-run=client -f namespace.yaml
kubectl apply --dry-run=client -f service-account.yaml
kubectl apply --dry-run=client -f volume.yaml
kubectl apply --dry-run=client -f jenkins-deployment.yaml
kubectl apply --dry-run=client -f jenkins-service.yaml
```

Inspect controller startup and events:

```sh
kubectl logs -n jenkins deployment/jenkins
kubectl get events -n jenkins --sort-by=.metadata.creationTimestamp
```

The local PV is pinned to the kind node named `jenkins-cluster-control-plane` and uses `/mnt` inside that node. If the PVC remains pending or the pod cannot mount its volume, verify the cluster name/node and local path.

## Stop and clean up

This removes the kind cluster and its local Jenkins data:

```sh
kind delete cluster --name jenkins-cluster
```

## Security and known limitations

- The Kubernetes service account is limited to agent-pod operations in the `jenkins` namespace; do not broaden it without a demonstrated requirement.
- The Jenkins service is reachable on the local host at port `8080`; this configuration is not intended for public or production exposure.
- Image tags are currently floating (`lts` and `latest`), so rebuilds may change over time. Pin and test exact image versions before presenting repeatability claims.
- The pipeline above is a manual smoke-test example, not a checked-in `Jenkinsfile`; Jenkins configuration and agent provisioning are not automated, and there is no end-to-end test yet.
- Suggested next steps: commit a QA `Jenkinsfile` that runs tests and publishes JUnit results (including a demonstrable failure case); build a pinned, non-root Python agent image and pin the controller image; add Kubernetes manifest linting and an agent lifecycle test; then add an architecture diagram and redacted run evidence.
- Before publishing, scan the complete Git history for secrets. A clean working-tree scan does not prove that old commits contain no credentials; rotate any credential that was ever committed.
