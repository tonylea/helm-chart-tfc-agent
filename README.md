# HCP Terraform Agent Helm Chart

![Downloads](https://img.shields.io/github/downloads/tonylea/helm-chart-tfc-agent/total)

Deploys the [HCP Terraform agent](https://developer.hashicorp.com/terraform/cloud-docs/agents) from its official image, [`hashicorp/tfc-agent`](https://hub.docker.com/r/hashicorp/tfc-agent), published by HashiCorp.
The agent connects out to HCP Terraform and runs plans and applies from inside your cluster, so HCP Terraform needs no inbound access to your network.

The chart is opinionated: it fixes the image, the user and most of the settings, and leaves values only for what changes with where it's installed.
The agent runs as one replica, replaced with `Recreate` rather than rolled, because HCP Terraform's free tier allows one agent.
A rolling update's second pod could not register while the first was still running.
Runs are throwaway, so the agent's working directory, `/home/tfc-agent/.tfc-agent`, is an `emptyDir` and nothing persists.

The agent serves no metrics endpoint, so the chart has no Service, ServiceMonitor or dashboard.
It pushes its traces and metrics over OTLP instead, when `otlpAddress` is set.

## Values

| Value            | Default                                                                                                                                                      |
| ---------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| `existingSecret` | none; required. See [Requirements](#requirements).                                                                                                           |
| `otlpAddress`    | none; the agent sends no telemetry. The `host:port` of an OTLP gRPC receiver, such as `otel-collector.monitoring.svc:4317`, set as `TFC_AGENT_OTLP_ADDRESS`. |
| `resources`      | none. Requests of 2m CPU and 29Mi memory are a good starting point.                                                                                          |

## Fixed settings

- **`TFC_AGENT_NAME` is the pod's name.**
  Each agent then shows in HCP Terraform under the name of its pod.
- **`TFC_AGENT_AUTO_UPDATE` is `disabled`.**
  The chart pins the agent's version, and an agent updating itself would drift from it.
- **The pod runs as non-root `999:999`**, the image's own `tfc-agent` user and group, with all capabilities dropped, privilege escalation off and the `RuntimeDefault` seccomp profile.

## Requirements

`existingSecret` names a Secret in the release's namespace with this key:

| Key     | What it holds                                                                                                               |
| ------- | --------------------------------------------------------------------------------------------------------------------------- |
| `token` | An [agent token](https://developer.hashicorp.com/terraform/cloud-docs/agents/agent-pools) from an HCP Terraform agent pool. |

The agent exits if the token is missing, revoked or invalid, so the pod restarts until the token is valid.

## Usage

```sh
helm repo add tfc-agent https://tonylea.github.io/helm-chart-tfc-agent
helm install tfc-agent tfc-agent/tfc-agent --values my-values.yaml
```

```yaml
existingSecret: tfc-agent-token
otlpAddress: otel-collector.monitoring.svc:4317
```

## Development

Pull requests run:

- `helm lint --strict`, and the rendered chart checked by kubeconform against Kubernetes 1.37.
- helm-unittest's tests in `tests/`.

There is no install test in a kind cluster.
The agent exits without a valid HCP Terraform token, and CI never holds one, so the pod could not stay up.
The unit tests check the security context and the working-directory mount instead.

Commits follow Conventional Commits.
Releases are versioned and tagged automatically from `main`, and published to the Helm repository on this repo's GitHub Pages.
