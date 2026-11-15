# Jenkins on Kubernetes (kind)

A local proof of concept for running a Jenkins controller in a [kind](https://kind.sigs.k8s.io/) cluster and using Kubernetes agents.

> **Scope:** This repository is for local learning and demonstration, not production. Jenkins is exposed on the local machine, the agent installs Python at startup as root, and Jenkins configuration and agent provisioning are manual. Do not run untrusted jobs with this setup.

## What is included

- `kind-config.yaml` creates a single-node cluster and maps host port `8080` to the Jenkins service.
- `namespace.yaml` creates the `jenkins` namespace.
- `service-account.yaml` gives the controller namespace-scoped permissions to manage agent pods.
- `volume.yaml` defines local persistent storage for Jenkins home.
- `jenkins-deployment.yaml` and `jenkins-service.yaml` deploy and expose the controller.
- `jenkins-agent.yaml` is a pod-template fragment for an inbound agent; it is not a standalone Kubernetes resource.

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

The controller and agent template are not fully connected automatically. Complete the Jenkins setup, install and configure the Kubernetes plugin, and configure its cloud to use this cluster before creating a job that requests an agent. Use `jenkins-agent.yaml` as the pod template YAML.

The template's command expects the Jenkins URL, agent name, agent secret, and work directory to be supplied at runtime through `JENKINS_URL`, `JENKINS_AGENT_NAME`, `JENKINS_SECRET`, and `JENKINS_AGENT_WORKDIR`; `REMOTING_OPTS` is optional. Supply the actual agent secret through Jenkins/plugin-managed configuration—never commit it in a manifest or job definition.

The current template runs as root to install Python with `apt-get` whenever an agent starts. This is slow and grants more privilege than a portfolio-ready setup should need. A follow-up should build and pin a dedicated agent image with Python preinstalled, then run it as a non-root user.

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
- There is no automated Jenkins configuration, sample QA pipeline, or end-to-end test yet. A strong next portfolio step is a small pipeline that runs a test, publishes its results, and demonstrates a failing case as well as a passing one.
- Before publishing, scan the complete Git history for secrets. A clean working-tree scan does not prove that old commits contain no credentials; rotate any credential that was ever committed.
