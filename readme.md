# Temporal Load Testing

Let's do some load testing on Temporal. This will help us configure our cluster for efficiency, robustness, and cost control.

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

<img width="1516" alt="Screenshot 2023-10-31 at 11 13 57 PM" src="https://github.com/bitovi/temporal-load-grafana/assets/8335079/2996692e-13d8-4253-bbda-0022b5435b1c">


### Observe the load in Grafana

In the Grafana UI, select the `Temporal Server` dashboard that you imported above.

You should see the load increasing in the various graphs. The peaks are increasing load, and the valleys are decreasing/no load.
<br>
<img width="1614" alt="Screenshot 2023-10-31 at 10 39 50 PM" src="https://github.com/bitovi/temporal-load-grafana/assets/8335079/fe487111-a06e-45c5-a4bf-809934cf22f2">

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

# What to look for

This benchmark creates a stable load test. What are the key metrics to look for?

The most critical metric is `state_transitions_count_count`. This is the throughput of your Temporal system. As you increase and decrease the load, you'll see the `state_transitions_count_count` react.

Specifically, the metric is defined as `sum(rate(state_transition_count_count[1m]))`

> That is not a typo, the metric name is `...count_count`.

At a certain load, you will see the `state_transition_count_count` plateau, or even start to drop. This is a sign that you've found a bottleneck in the system. Where is it? You can look at the metrics suggested above. You can also inspect the resource utilization of your cluster. Are any worker pods at CPU or Memory limits? Is throttling happening on the node?

To scale the service you've identified as a potential bottleneck, scale the deployment:

```shell
k scale deployment <deployment_name> --replicas=<desired_replicas> [-n <namespace]
```

If you're not sure of the specific name of the deployment:

```shell
k get deployments [-n <namespace>]
```

As always, `kubectl` is just one way to manage your cluster. K9s and openlens are highly recommended.

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
