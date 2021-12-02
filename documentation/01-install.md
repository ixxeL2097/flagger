# 01 - Install Flagger

Official doc :
- https://docs.flagger.app/install/flagger-install-on-kubernetes
- https://github.com/fluxcd/flagger
## Kubernetes
### Helm

Add helm repo and fetch the chart:
```
helm repo add flagger https://flagger.app
helm fetch --untar flagger/flagger
```

Install flagger (istio):
```
helm upgrade -i flagger flagger/flagger --namespace=istio-system --set crd.create=false \
                                                                 --set meshProvider=istio \
                                                                 --set metricsServer=http://prom-kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090 \
                                                                 --set configTracking.enabled=true \
                                                                 --set podMonitor.enabled=true \
                                                                 --set podMonitor.additionalLabels.release=prom \
                                                                 --set podMonitor.namespace=istio-system \
                                                                 --set serviceAccount.create=true \
                                                                 --set serviceAccount.name=flagger
```

### ArgoCD

ArgoCD app definition:
```yaml
apiVersion: argoproj.io/v1alpha1
kind: Application
metadata:
  name: flagger
spec:
  destination:
    name: ''
    namespace: istio-system
    server: 'https://kubernetes.default.svc'
  source:
    path: ''
    repoURL: 'https://flagger.app'
    targetRevision: 1.16.0
    chart: flagger
    helm:
      valueFiles:
        - values.yaml
      parameters:
        - name: meshProvider
          value: istio
        - name: metricsServer
          value: >-
            http://prom-kube-prometheus-stack-prometheus.monitoring.svc.cluster.local:9090
        - name: podMonitor.enabled
          value: 'true'
        - name: serviceAccount.name
          value: flagger
      values: |-
        podMonitor:
          additionalLabels:
            release: prom
  project: default
  syncPolicy:
    automated:
      prune: true
      selfHeal: true
    syncOptions:
      - ApplyOutOfSyncOnly=true
```

### Check install

Check everything is ok executing command:
```bash
kubectl logs deploy/flagger | jq .msg
```

### Monitoring

It's recommended to install the Grafana dashboard ID #15187 :
- https://grafana.com/grafana/dashboards/15158

:warning: WARNING !! :warning:

This dashboard fail to display some metrics. You need to create a new variable called `flagger_ns` in Grafana dashboard and then use the query `label_values(flagger_info, namespace)` to fetch the actual flagger namespace.
Then replace the variable `$namespace` by `$flagger_ns` in the promql query from panel 
- Primary : Flagger weighting
- Flagger canary : last result
- Canary: flagger weighting

Modified dashboard is available [here](../resources/dashboards/flagger.json)