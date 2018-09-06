# streamsets-microservices-istio

Demo of StreamSets Data Collector Microservices Pipelines running with Istio

## Prerequisites

* Kubernetes Cluster and Cluster Admin privileges

## Setup

### Install the Kubernetes CLI

1. See the official [installation docs](https://kubernetes.io/docs/tasks/tools/install-kubectl/) for details on getting `kubectl`
    1. macOS: `brew install kubernetes-cli`

### Install Helm and Tiller

1. See the official [installation docs](https://docs.helm.sh/using_helm/#installing-helm) for details on getting the `helm` CLI and tiller installed into your cluster.
    1. macOS: `brew install helm` to install the CLI
    1. `helm init` to install the tiller pod into your cluster

### Install Istio

1. Download an [Istio release](https://github.com/istio/istio/releases/tag/1.0.1) for your platform
1. Extract the release to a directory (referred to as `$ISTIO_DIST` in subsequent steps) and either copy the istioctl binary to a directory on your path such as `/usr/local/bin` or add the `$ISTIO_DIST/bin` to your path
1. Run `istioctl` to verify that it is available on the path
1. Ensure you're in the `$ISTIO_DIST` directory.
1. To install Istio into the cluster, run `helm install install/kubernetes/helm/istio --name istio --namespace istio-system`
1. If installation completed successfully, we can also enable a few of the Istio add-on components. In `$ISTIO_DIST` create a file called `istio-values.yaml`. Add the following to the file:

    ```yaml
    ---
    grafana:
      enabled: true
    tracing:
      enabled: true
    kiali:
      enabled: true
    ```

1. Run `helm upgrade istio install/kubernetes/helm/istio --namespace istio-system --values istio-values.yaml`
1. Create a namespace with [Istio sidecar injection enabled](https://istio.io/docs/setup/kubernetes/sidecar-injection/#automatic-sidecar-injection). This means that all pods created in the namespace will automatically have an Istio sidecar added to them. *Note: This requires the mutating webhook admission controller, which is not currently supported on AWS EKS*
    1. `kubectl create namespace datacollector`
    1. `kubectl label namespace datacollector istio-injection=enabled`

### Install StreamSets Control Agent

This demo currently requires use of a Golang port of control agent in order to work properly with Istio.

1. Edit `streamsets/control-agent.yaml` and modify the lines with `<add unique identifier>` to some unique name that doesn't conflict with other agents in your Control Hub organization.
1. Edit `streamsets/control-agent.yaml` with values for the placeholders `<fill me in>` that include your **orgId** and **token** which is a provisioning agent token generated in Control Hub under *Administration > Provisioning Agents*.

### Define Deployments in Control Hub

1. In the Control Hub UI under *Execute > Deployments* create a deployment for an authoring data collector using the contents of the `streamsets/authoring-deployment.yaml` file.
1. In the Control Hub UI under *Execute > Deployments* create a deployment for a microservices execution data collector using the contents of the `streamsets/microservice-deployment.yaml` file.
1. Start both deployments simultaneously. *Note: There is currently a known issue where waiting to start them one at a time may result in one of them becoming unregistered*
1. In the *Execute > Data Collectors* screen find your authoring data collector and label it with a unique label. Remember this label. For example `authoring-istio`.
1. In the *Execute > Data Collectors* screen find your microservice data collector and label it with a unique label. Remember this label. For example `microservice-istio`.

### Import and Deploy the demo pipelines

1. Import all of the pipelines in the `pipelines/` subdirectory into your Control Hub pipeline repository.
2. Create Jobs for each pipeline, with microservices running on Data Collectors labeled with microservice label you defined previously, for example `microservice-istio`. Do the same for your authoring Data Collector when deploying the **Microservices Istio Demo** pipeline.

### Istio Monitoring

Assuming your jobs have been deployed and are running correctly, you can monitor them with the Istio addons we installed earlier such as Grafana and Kiali. In order to connect to them follow these next steps.

1. Create a port forward to Grafana with: `kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=grafana -o jsonpath='{.items[0].metadata.name}') 3000:3000 &`
1. Create a port forward to Kiali with: `kubectl -n istio-system port-forward $(kubectl -n istio-system get pod -l app=kiali -o jsonpath='{.items[0].metadata.name}') 20001:20001 &`
1. Browse to [http://localhost:3000](http://localhost:3000) to access Grafana
1. Browse to [http://localhost:20001](http://localhost:20001) to access Kiali. The default login is `admin`/`admin`.

To remove the port forward, run `killall kubectl`.

## Troubleshooting
