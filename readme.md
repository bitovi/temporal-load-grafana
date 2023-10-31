# Temporal Load Testing

Let's do some load testing on Temporal.
<!-- 
## Create Cluster

```shell
eksctl create cluster \
--name vantage-temporal \
--version 1.28 \
--region us-east-2 \
--nodegroup-name temporal-nodes \
--node-type t2.large \
--nodes 3
```

### Rightsizing nodes & pods

https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/using-eni.html#AvailableIpPerENI

- 2 system pods per node
- 1 ip reserved per node

pod capacity per node = `(nics * addrs) - 3`

### Install Volume Handler

EKS doesn't come with a default volume handler. We need to install one.

First, check if it's already installed:

```shell


```shell
eksctl utils associate-iam-oidc-provider --region=us-east-2 --cluster=vantage-temporal --approve

eksctl create addon --name aws-ebs-csi-driver \
  --cluster vantage-temporal \
  --region us-east-2 \
  --service-account-role-arn arn:aws:iam::111122223333:role/AmazonEKS_EBS_CSI_DriverRole \
  # --force
``` -->
<!-- 
## Install Temporal & Monitoring

Using: https://github.com/VantageDiscovery/helm-charts/tree/main/charts/temporalio

In the Temporal chart's directory:

1. Change the `web` service type to `LoadBalancer` in `values.yaml`. This lets us access the UI from the itnernet.

    ```yaml
    web:
    enabled: true
    ...
    service:
      # set type to NodePort if access to web needs access from outside the cluster
      # for more info see https://kubernetes.io/docs/concepts/services-networking/service/#publishing-services-service-types
      type: ClusterIP  <--- CHANGE THIS TO 'LoadBalancer'
      port: 8080
      annotations: {}
      ```

    > For external access, NodePorts require an Ingress Controller. We've used a   LoadBalancer here because they require no further setup and are managed by the cloud provider (AWS).

1. Update helm dependencies

    ```shell
    helm dependencies update
    ```

1. Create the `temporal` namespace (you can call it whatever you want)

    ```shell
    k create ns temporal
    ```

1. Install the chart

    This installs Temporal, CassandraDB, Prometheus, and Grafana.

    ```shell
    helm install temporaltest . --timeout 15m -n temporal
    ```

    > `temporaltest` is the name of the Helm release. You can call it whatever you want. -->

## Load Test Deployment

Using: <https://github.com/temporalio/benchmark-workers/pkgs/container/benchmark-workers>

_NOTE: this document uses the alias `k` for `kubectl` (and you should too!)_

### Create a dir for our files

  ```shell
  mkdir load && cd load
  ```

### Create the deployment manifest
  
Copy the contents of <https://github.com/temporalio/benchmark-workers/blob/main/deployment.yaml> into a file named `deployment.yaml`.

### Deploy the load test

  ```shell
  > k apply -f deployment.yaml 
  deployment.apps/benchmark-workers created
  deployment.apps/benchmark-soak-test created
  ```

### Confirm the activity in Temporal UI

As soon as you deploy the load test, you should see activity in the Temporal UI. Click the reload button in the top right to see the latest activity: ↻

The ID hashes should be changing every few seconds, and you should see new workflows being created.

### Stop the test

The quick-and-dirty way to stop the test is to just delete the loading deployment:

  ```shell
  k delete deployment -f deployment.yaml
  ```

Alternatively, you can scale down the deployment:

  ```shell
  k scale deployment benchmark-workers --replicas=0
  ```

You can of course use your k8s interface of choice to do this as well (k9s, openlens, etc)

## appendix

### deploy kube-prom

1. `git clone https://github.com/prometheus-operator/kube-prometheus.git && cd kube-prometheus`
1. Create the monitoring stack using the config in the manifests directory:

    ```shell
    # Create the namespace and CRDs, and then wait for them to be available before creating the remaining resources
    # Note that due to some CRD size we are using kubectl server-side apply feature which is generally available since kubernetes 1.22.
    # If you are using previous kubernetes versions this feature may not be available and you would need to use kubectl create instead.
    kubectl apply --server-side -f manifests/setup
    kubectl wait \
      --for condition=Established \
      --all CustomResourceDefinition \
      --namespace=monitoring
    kubectl apply -f manifests/
    ```

1. Set up LoadBalancer/PortForwarding:
  
      ```shell
      kubectl port-forward --namespace monitoring svc/prometheus-k8s 9090
      ```
  
    Or, you can set the service type to LoadBalancer to avoid having to set up PF every time.

1. Teardown:

    ```shell
    kubectl delete --ignore-not-found=true -f manifests/ -f manifests/setup
    ```

### Import the Temporal dashboards

In the Grafana UI, paste the content of `./server-general.json` into the Dashboard import window.

- pulled from <https://github.com/temporalio/dashboards/blob/master/server/server-general.json>

## Log In to grafana

user/pass should be admin/admin