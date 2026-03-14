# devops-case-study

This README outlines some brief details about the DevOps case study, including local run instructions
and a brief overview of the solution.

## Overview

### Solution Scope

I have created a monitoring stack that deploys Prometheus, Alertmanager and Grafana using the `kube-prometheus-stack`
Helm chart. Additionally, fluent-bit and Loki are deployed for log aggregation and visualization in Grafana.
All resources are deployed to the `monitoring` namespace and managed in the `./monitoring` directory.

The Prometheus, Loki and fluent-bit setup resides in `./monitoring/controllers`, whereas `./monitoring/config` contains
dashboards and alerting rules.

> [!IMPORTANT]  
> Loki and Prometheus in particular eat up a lot of memory. Your Cluster will have to be configured to have
> at least 4GB of memory available.

### Architectural Decisions & Trade-Offs

Prometheus is set up via the `kube-prometheus-stack` Helm chart, which provides a comprehensive monitoring solution for
Kubernetes clusters. This makes it extremely easy to get started, as this chart includes not only Prometheus but also
Alertmanager, Grafana, and various exporters for Kubernetes components. The trade-off is that it introduces more
complexity: It heavily relies on the Prometheus Operator, which trades annotation-based scrape target autodiscovery
for a more declarative approach using custom resources. This is more flexible and powerful, but also introduces
complexity and maintenance effort.

However, I ultimately decided to go with this solution because of the following benefits:

- Easy integration of `kube-state-metrics`, which includes FluxCD monitoring and makes it easy to monitor other CRDs.
  This might turn out to be beneficial if we were to create backups declaratively with e.g. `k8up` to monitor the status
  of backups.
- Easy integration of Alertmanager and Grafana.

## Accessing Grafana

You can access Grafana via port forwarding. You have to retrieve the admin password first, which is stored in a
Kubernetes secret.

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 --decode
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

## Dashboards and Alerts

### Dashboards

I have created two custom dashboards, as well as a few freebies. There are two functioning FluxCD dashboards to
visualize the health of FluxCD components and the status of Git repositories. There is an additional Kubernetes
namespace-level dashboard which can be helpful in gaining a holistic overview over the cluster.

There is one _Backend API_ and one _ML API_ dashboard. They both visualize key HTTP metrics, such as statuses and
latency. In both cases, I've made it a point to exclude `/health` and `/ready` endpoints, as they tend to skew the
metrics and are not relevant for monitoring the overall health of the APIs. p90 and p99 are included.

For the backend API, there are additional panels to visualize the number of active database connections as well as
the return statuses of database queries.

For the ML API, visualizations exist around the number of predictions made total – broken down by Pod – as well as
memory usage, which currently does not work. I have therefore opted to additionally visualize memory usage with the
`container_memory_usage_bytes` metric.

### Alerts

I have added alerts for a few key metrics that I think are important to monitor. For the backend API, the following
alerts are set up:

- Abnormally high amount of 500s (more than 5% of total requests in the last 5 minutes) using
  `backend_api_requests_total`.
- Abnormally high latency (p90 latency above 1s for more than 5 minutes) using
  `backend_api_request_duration_seconds_bucket`
- Sharp drop in connections, which could indicate DNS- or networking-related issues (more than 50% drop in inbound
  requests compared to the previous 5 minutes) using `backend_api_requests_total`
- Increased number of database errors (> 10% of total queries in the last 5 minutes) using
  `backend_api_db_queries_total{status="error"}`

For the ML API, the following alerts are set up:

- Abnormally high amount of 500s (more than 5% of total requests in the last 5 minutes) using `ml_api_requests_total`.
- Abnormally high latency (p90 latency above 1s for more than 5 minutes) using `ml_api_request_duration_seconds_bucket`
- Sharp drop in connections, which could indicate DNS- or networking-related issues (more than 50% drop in inbound
  requests compared to the previous 5 minutes) using `ml_api_requests_total`
- Increased memory usage (more than 80% of memory limit for more than 5 minutes) using `ml_api_memory_bytes`.

### Logs

Grafana _should_ be autodiscovering the Loki datasource. If this is not the case for you, you can manually
add it via the Grafana UI. The Loki datasource should be configured to point to `http://loki:3100`.

You can then query logs in the explorer. Here is an example query to get you started:

```logql
{app="ml-api"} | json | line_format "{{.message}}"
```

### Reasoning

I believe monitoring HTTP metrics is a great way to get insights into the overall health of the system.
It is how consumers interact with the system, and therefore a good indicator of whether the system is functioning as
expected. A spike in 500s and a spike in latency will indicate abnormal behavior of the system; a drop in connections
means that the system cannot be interacted with. These cases require immediate attention. I believe alerts
should be reserved for scenarios that require immediate attention, which is why I defined the alerts the way I did.

Of course, this means that less critical scenarios may not become exactly obvious. For instance, database errors
that are more subtle and do not cause 500s would currently not be caught. Similarly, there are *some*  cases of 503s
being returned when the cluster was just starting. There are no alerts for these cases, and they would have to be
discovered via the dashboards. However, I believe that this is an acceptable trade-off; I would align with developers
and operators to define the alerts and dashboards in a way that they complement each other.

Dashboards can be used to gain insights into the system and identify potential issues before they become critical.
Therefore, some more information is displayed there that may not be critical.

Of course, neither the alerts nor the dashboards are comprehensive. Alerts can and should be modified based
on domain knowledge and real incidents, just like the dashboards. I believe this is a strong starting point.

## Future Improvements

- Grafana dashboards are currently managed via the `grafana-dashboards` Configmap. This works well for a prototype,
  but does not allow for advanced capabilities, such as structuring dashboards in folders. If the Grafana setup grows,
  I would consider managing it via Terraform.

- No authentication is performed between Prometheus, Grafana and Loki, which is a security risk.
  In a production environment, I would recommend implementing authentication and authorization mechanisms to secure
  access to these monitoring tools.

- Depending on who uses these tools, I would consider running a central Grafana instance that can
  select different customer's Prometheus instances as data sources.
