# k8s-fluent-bit-stackdriver

This repository is for explaining how to configure a GKE Kubernetes cluster to collect logs and other data using Fluent Bit instead of Fluentd.

## Why Fluent Bit vs Fluentd

Fluentd is a log collector, processor, and aggregator. The Fluentd process can be quite resource hungry and end up consuming a non-trivial share of CPU and RAM available in the cluster. In GKE, aggregation by Fluentd is not necessary as the logs will be forwarded and then aggregated by Stackdriver by default, therefore the aggregation feature is not necessary.
Fluent Bit is based on the design and architecture of Fluentd, but restricts itself to log collecting, processing, and forwarding. It is more performant and less resource intensive than Fluentd.
See [this page](https://docs.fluentbit.io/manual/about/fluentd_and_fluentbit) for a side-by-side comparison of both projects.

## Creating a cluster

See [this page](https://cloud.google.com/kubernetes-engine/docs/how-to/creating-a-container-cluster) for instructions on how to create a cluster via `gcloud`. The only difference is that the `--no-enable-cloud-logging` flag must be used when creating the cluster. This will disable the default logging setup. This setting can be updated on a running cluster, including from the UI.

## Service accounts

Both Kubernetes and GCP IAM have a concept of service accounts, care will be taken to differentiate between them.

To forward logs to Stackdriver, Fluent Bit will need a GCP service account with the right permissions, specifically: `roles/logging.logWriter`, `roles/monitoring.viewer`, `roles/monitoring.metricWriter`.
You can create the service account and download its key manually, alternatively a script is provided that will do that, as well as create a k8s secret in the specified namespace from the key, and then delete the key. To do so, run the following command:

```bash
NO_SERVICE_KEY=1 NAMESPACE=logging ./gcp-create-service-account "fluent-bit-logging" roles/logging.logWriter roles/monitoring.viewer roles/monitoring.metricWriter
```

We will be installing everything in a namespace called `logging`. The `gcp-create-service-account` script will create the namespace if it does not exist already.

## Deploying the Kubernetes resources

First, we need a k8s service account for Fluent Bit. This will provide an identity to the pods when they interact with the k8s API server.

```bash
kubectl -n logging create -f service-account.yaml
```

Next, we need to specify a role with allowed permissions:

```bash
kubectl -n logging create -f role.yaml
```

We now need a `ClusterRoleBinding` to associate the role to the service account:

```bash
kubectl -n logging create -f role-binding.yaml
```

We now deploy the configuration for Fluent Bit itself. This is contained in a `ConfigMap`:

```bash
kubectl -n logging create -f configmap.yaml
```

Finally, we deploy the Fluent Bit container itself. We need a pod to run on each node in our cluster, for this we require a `DaemonSet`:

```bash
kubectl -n logging create -f gke.yaml
```

The way the service account credentials are passed to Fluent Bit might seem confusing. Let us go over how this is done in `gke.yaml`:

1. First we specify a volume based on the secret we created above
2. Next, it is mounted by the container at the following path: `/var/secrets/google`
3. Kubernetes will automatically populate the path with files based on the keys present in the secret
4. We specify an environment variable `GOOGLE_SERVICE_CREDENTIALS` with the path to the service account file that Fluent Bit expects for its Stackdriver plugin.
