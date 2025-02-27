---
title: Datadog Admission Controller
kind: documentation
aliases:
- /agent/cluster_agent/admission_controller
further_reading:
- link: "/agent/cluster_agent/clusterchecks/"
  tag: "Documentation"
  text: "Running Cluster Checks with Autodiscovery"
- link: "/agent/cluster_agent/troubleshooting/"
  tag: "Documentation"
  text: "Troubleshooting the Datadog Cluster Agent"
---

## Overview
The Datadog admission controller is a component of the Datadog Cluster Agent. The main benefit of the admission controller is to simplify your application pod configuration. For that, it has two main functionalities:

- Inject environment variables (`DD_AGENT_HOST`, `DD_TRACE_AGENT_URL` and `DD_ENTITY_ID`) to configure DogStatsD and APM tracer libraries into the user's application containers.
- Inject Datadog standard tags (`env`, `service`, `version`) from application labels into the container environment variables.

Datadog's admission controller is `MutatingAdmissionWebhook` type. For more details on admission controllers, see the [Kubernetes guide][1].

## Requirements

- Datadog Cluster Agent v7.39+

## Configuration
{{< tabs >}}
{{% tab "Helm chart" %}}
Starting from Helm chart v2.35.0, Datadog Admission controller is activated by default. No extra configuration is needed to enable the admission controller.

To enable the admission controller for Helm chart v2.34.6 and earlier, set the parameter `clusterAgent.admissionController.enabled` to `true`:

{{< code-block lang="yaml" filename="values.yaml" disable_copy="true" >}}
[...]
 clusterAgent:
[...]
  ## @param admissionController - object - required
  ## Enable the admissionController to automatically inject APM and
  ## DogStatsD config and standard tags (env, service, version) into
  ## your pods
  #
  admissionController:
    enabled: true

    ## @param mutateUnlabelled - boolean - optional
    ## Enable injecting config without having the pod label:
    ## admission.datadoghq.com/enabled="true"
    #
    mutateUnlabelled: false
[...]
{{< /code-block >}}
{{% /tab %}}

{{% tab "Datadog Operator" %}}

To enable the admission controller for the Datadog operator, set the parameter `clusterAgent.config.admissionController.enabled` to `true` in the custom resource:

```yaml
[...]
 clusterAgent:
[...]
    config:
      admissionController:
        enabled: true
        mutateUnlabelled: false
[...]
```
{{% /tab %}}

{{% tab "Manual setup" %}}

To enable the admission controller without using Helm or the Datadog operator, add the following to your configuration:

First, download the [Cluster Agent RBAC permissions][1] manifest, and add the following under `rules`:

{{< code-block lang="yaml" filename="cluster-agent-rbac.yaml" disable_copy="true" >}}
- apiGroups:
  - admissionregistration.k8s.io
  resources:
  - mutatingwebhookconfigurations
  verbs: ["get", "list", "watch", "update", "create"]
- apiGroups: [""]
  resources: ["secrets"]
  verbs: ["get", "list", "watch", "update", "create"]
- apiGroups: ["batch"]
  resources: ["jobs", "cronjobs"]
  verbs: ["get"]
- apiGroups: ["apps"]
  resources: ["statefulsets", "replicasets", "deployments"]
  verbs: ["get"]
{{< /code-block >}}

Add the following to the bottom of `agent-services.yaml`:

{{< code-block lang="yaml" filename="agent-services.yaml" disable_copy="true" >}}

apiVersion: v1
kind: Service
metadata:
  name: datadog-cluster-agent-admission-controller
  labels:
    app: "datadog"
    app.kubernetes.io/name: "datadog"
spec:
  selector:
    app: datadog-cluster-agent
  ports:
  - port: 443
    targetPort: 8000

{{< /code-block >}}

Add environment variables to the Cluster Agent deployment which enable the Admission Controller:

{{< code-block lang="yaml" filename="cluster-agent-deployment.yaml" disable_copy="true" >}}
- name: DD_ADMISSION_CONTROLLER_ENABLED
  value: "true"
- name: DD_ADMISSION_CONTROLLER_SERVICE_NAME
  value: "datadog-cluster-agent-admission-controller"

# Uncomment this to configure APM tracers automatically (see below)
# - name: DD_ADMISSION_CONTROLLER_MUTATE_UNLABELLED
#   value: "true"
{{< /code-block >}}

Finally, run the following commands:

- `kubectl apply -f cluster-agent-rbac.yaml`
- `kubectl apply -f agent-services.yaml`
- `kubectl apply -f cluster-agent-deployment.yaml`

[1]: https://raw.githubusercontent.com/DataDog/datadog-agent/master/Dockerfiles/manifests/cluster-agent/cluster-agent-rbac.yaml
{{% /tab %}}
{{< /tabs >}}

### APM
You can configure the Cluster Agent (version 7.39 and higher) to inject APM tracing libraries automatically. Read [APM Tracing Setup with Admission Controller][2] for more information


### DogStatsD

To configure DogStatsD clients or other APM libraries that do not support library injection, inject the environment variables `DD_AGENT_HOST` and `DD_ENTITY_ID` by doing one of the following:
- Add the label `admission.datadoghq.com/enabled: "true"` to your pod.
- Configure the Cluster Agent admission controller by setting `mutateUnlabelled` (or `DD_ADMISSION_CONTROLLER_MUTATE_UNLABELLED`, depending on your configuration method) to `true`.


#### Order of priority
The Datadog admission controller does not inject the environment variables `DD_VERSION`, `DD_ENV`, and `DD_SERVICE` if they already exist.

When these environment variables are not set, the admission controller uses standard tags value in the following order (highest first):

- Labels on the pod
- Labels on the `ownerReference` (ReplicaSets, DaemonSets, Deployments...)

#### Configure APM and DogstatsD communication mode
Starting from Datadog Cluster Agent v1.20.0, the Datadog Admission Controller can be configured to inject different modes of communication between the application and Datadog agent.

This feature can be configured by setting `admission_controller.inject_config.mode` or by defining a pod-specific mode using the `admission.datadoghq.com/config.mode` pod label.

Possible options:
| Mode               | Description                                                                                                       |
|--------------------|-------------------------------------------------------------------------------------------------------------------|
| `hostip` (Default) | Inject the host IP in `DD_AGENT_HOST` environment variable                                                        |
| `service`          | Inject Datadog's local-service DNS name in `DD_AGENT_HOST` environment variable (available with Kubernetes v1.22+)|
| `socket`           | Inject Unix Domain Socket path in `DD_TRACE_AGENT_URL` environment variable and the volume definition to access the corresponding path |

**Note**: Pod-specific mode takes precedence over the global mode defined at the Admission controller level.

#### Notes

- The admission controller needs to be deployed and configured before the creation of new application pods. It cannot update pods that already exist.
- To disable the admission controller injection feature, use the Cluster Agent configuration: `DD_ADMISSION_CONTROLLER_INJECT_CONFIG_ENABLED=false`
- By using the Datadog admission controller, users can skip configuring the application pods using downward API ([step 2 in Kubernetes Trace Collection setup][3]).
- In a Google Kubernetes Engine (GKE) Private Cluster, you need to [add a Firewall Rule for the control plane][4]. The webhook handling incoming connections receives the request on port `443` and directs it to a service implemented on port `8000`. By default, in the Network for the cluster there should be a Firewall Rule named like `gke-<CLUSTER_NAME>-master`. The "Source filters" of the rule match the "Control plane address range" of the cluster. Edit this Firewall Rule to allow ingress to the TCP port `8000`.


## Further Reading

{{< partial name="whats-next/whats-next.html" >}}

[1]: https://kubernetes.io/blog/2019/03/21/a-guide-to-kubernetes-admission-controllers/
[2]: /tracing/trace_collection/admission_controller/
[3]: https://docs.datadoghq.com/agent/kubernetes/apm/?tab=helm#setup
[4]: https://cloud.google.com/kubernetes-engine/docs/how-to/private-clusters#add_firewall_rules
