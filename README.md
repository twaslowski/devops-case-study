# devops-case-study

This README outlines some brief details about the DevOps case study, including local run instructions
and a brief overview of the solution.

## Architectural Decisions & Trade-Offs

### The Prometheus Setup

Prometheus is set up via the `kube-prometheus-stack` Helm chart, which provides a comprehensive monitoring solution for
Kubernetes clusters. This makes it extremely easy to get started, as this chart includes not only Prometheus but also
Alertmanager, Grafana, and various exporters for Kubernetes components. The trade-off is that it introduces more
complexity: It heavily relies on the Prometheus Operator, which trades annotation-based scrape target autodiscovery
for a more declarative approach using custom resources. This is more flexible and powerful, but also introduces
complexity and maintenance effort.

There are enough benefits from using it though that I deemed it worth the trade-off:

- Easy integration of `kube-state-metrics`, which includes FluxCD monitoring and makes it easy to monitor other CRDs.
  This might turn out to be beneficial if we were to create backups declaratively with e.g. `k8up` to monitor the status
  of backups.
- Easy integration of Alertmanager and Grafana.

### Grafana

Access Grafana via port forwarding. You have to retrieve the admin password first, which is stored in a Kubernetes
secret.

```bash
kubectl get secret kube-prometheus-stack-grafana -n monitoring -o jsonpath='{.data.admin-password}' | base64 --decode
kubectl port-forward -n monitoring svc/kube-prometheus-stack-grafana 3000:80
```

There is one dashboard per component. There are other ways to structure dashboards: For instance, by application layer.
It could be feasible to have a dashboard for everything HTTP-related, another one for inference etc. However, I believe
this requires more in-depth understanding of the system and its components. Creating one dashboard for the ml-api
and one for the backend-api is much more straightforward.

### Improvements

- Grafana is currently being deployed via the Prometheus Operator on `monitoring/prometheus`.
  This creates a cross-dependency between the Prometheus Operator and Grafana, which is not ideal.
  In a future refactor, I would recommend extracting Grafana into a separate stack.

- No authentication is performed between Prometheus, Grafana and Loki, which is a security risk.
  In a production environment, I would recommend implementing authentication and authorization mechanisms to secure
  access
  to these monitoring tools.

- Depending on who uses these tools, I would consider running a central Grafana instance that can
  select different customer's Prometheus instances as data sources.