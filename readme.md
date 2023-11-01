# Temporal Load Testing

Let's do some load testing on Temporal. This will help us configure our cluster for efficiency, robustness, and cost control.
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
    # helm install temporaltest . --timeout 15m -n temporal
    
    helm install \
    --set server.replicaCount=1 \
    --set cassandra.config.cluster_size=1 \
    --set prometheus.enabled=true \
    --set grafana.enabled=true \
    --set elasticsearch.enabled=false \
    temporaltest . --timeout 15m
    ```

    > `temporaltest` is the name of the Helm release. You can call it whatever you want. -->

## Load Test Deployment

Using: <https://github.com/temporalio/benchmark-workers/pkgs/container/benchmark-workers>

### Prerequisites

- A running Temporal cluster
- `kubectl` configured to access the cluster
- Access to the Grafana and Temporal UIs

_NOTE: this document uses the alias `k` for `kubectl` (and you should too!)_

### Clone this repo

  ```shell
  cd /path/to/your/projects
  git clone https://github.com/bitovi/temporal-load-grafana.git
  ```

### Import the Temporal dashboards

In the Grafana UI, paste the content of `./server-general.json` into the Dashboard import window.

- pulled from <https://github.com/temporalio/dashboards/blob/master/server/server-general.json>


### Deploy the load test harness

  ```shell
  k apply -f deployment.yaml [-n <namespace>]

  # you should see:
  # deployment.apps/benchmark-workers created
  # deployment.apps/benchmark-soak-test created
  ```

### Confirm the activity in Temporal UI

The provided deployment file is configured to start the load test immediately. You should see activity in the Temporal UI within a few seconds.

Click the refresh button to see the latest activity: ↻ <br>
You should see new workflows being created with each refresh.

### Observe the load in Grafana

In the Grafana UI, select the `Temporal Server` dashboard that you imported above.

You should see the load increasing in the various graphs.

### Scale up

You can increase the load by scaling up the deployment:

  ```shell
  k scale deployment benchmark-soak-test --replicas=10 [-n <namespace>]
  ```

### Scale down

You can decrease the load by scaling down the deployment:

  ```shell
  k scale deployment benchmark-soak-test --replicas=5 [-n <namespace>]
  ```

### Stop the test

The quick-and-dirty way to stop the test is to just delete the loading deployment:

  ```shell
  k delete -f deployment.yaml
  ```

Alternatively, you can scale the runner deployment to 0:

  ```shell
  k scale deployment benchmark-soak-test --replicas=0
  ```

This keeps the deployment alive for easy up-scaling when ready.

> Of course, you can use your k8s interface of choice instead of `kctl` to do these operations: [k9s](https://k9scli.io/), [openlens](https://github.com/MuhammedKalkan/OpenLens), etc.

## Viewing the metrics

In Grafana, you will see the load taking its toll on the Temporal cluster.

The peaks are increasing load, and the valleys are decreasing/no load.
<br>
<img width="1614" alt="Screenshot 2023-10-31 at 10 39 50 PM" src="https://github.com/bitovi/temporal-load-grafana/assets/8335079/fe487111-a06e-45c5-a4bf-809934cf22f2">


## Appendix

### Run with `tctl`

As an alternative to the above options, you can run benchmark tests directly with `tctl`.

Anywhere you have `tctl` available:

1. Run `export TEMPORAL_CLI_ADDRESS=<temporal-frontend-service-address:port>`
1. Execute:

    ```shell
    tctl workflow start --taskqueue benchmark \
    --workflow_type ExecuteActivity \
    --execution_timeout 60 \
    -i '{"Count":1,"Activity":"Sleep","Input":{"SleepTimeInSeconds":3}}'
    ```

This will start a workflow that executes a three-second `Sleep` activity once.<br>
To execute the activity multiple times, change the `Count` value.<br>
To change the sleep time, change the `SleepTimeInSeconds` value.

### Change the load test

In most cases, the default benchmark test provided by the `sleep` command above is adequate for load testing your Temporal cluster.

If necessary, the `soak-test` runner configuration can be adjusted either with Environment Variables or with command line flags. There aren't too many options, but see the official documentation for details: <https://github.com/temporalio/benchmark-workers/pkgs/container/benchmark-workers#runner>

### Log In to grafana

In some installs of Grafana, the default username is `admin` and the default password is `admin`.

In others you might have to shell into the Grafana pod to get the password:

```bash
kubectl get pods -n temporal
kubectl exec -it <grafana-pod-name> -n temporal -- /bin/bash
```

> You can get the pod name with `kubectl get pods -n temporal`
>
> K8s tools like k9s and openlens provide one-click access to shell into pods.

Then run:

```bash
env | grep -i password | sed s/[^=]*=//
```
