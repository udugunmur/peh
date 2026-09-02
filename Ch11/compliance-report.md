# Informe de Cumplimiento de la Plataforma

**Generado:** 2026-02-28 16:42:44 UTC
**Infracciones Totales:** 102

## Resumen por Severidad

| Severidad | Cantidad |
|----------|-------|
| CRITICAL | 0 |
| HIGH | 40 |
| MEDIUM | 62 |
| LOW | 0 |

## Infracciones por Política

| Política | Cantidad |
|--------|-------|
| K8sRequiredResources/require-resources | 20 |
| K8sRequireLabels/require-compliance-labels | 20 |
| K8sRequireLabels/require-labels | 20 |
| K8sRestrictLa imagenRegistries/restrict-image-registries | 20 |
| K8sRequireResourceLimits/require-resource-limits | 18 |
| K8sDenyPrivilegedEl contenedors/deny-privileged-containers | 4 |

## Infracciones por Espacio de Nombres (Namespace)

| Espacio de Nombres | Cantidad |
|-----------|-------|
| crossplane-system | 40 |
| monitoring | 31 |
| backstage | 10 |
| istio-system | 5 |
| production | 4 |
| local-path-storage | 4 |
| databases | 3 |
| observability | 3 |
| default | 2 |

## Infracciones Detalladas

| Severidad | Política | Espacio de Nombres | Recurso | Mensaje |
|----------|--------|-----------|----------|---------|
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-kube-prometheus-operator-77b866bdbb-2gxsn | El contenedor kube-prometheus-stack carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana-sc-datasources carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana-sc-datasources carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana-sc-dashboard carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana-sc-dashboard carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor grafana carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor config-reloader carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor config-reloader carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor alertmanager carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | local-path-storage | Pod/local-path-provisioner-67b8995b4b-t7sj2 | El contenedor local-path-provisioner carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | local-path-storage | Pod/local-path-provisioner-67b8995b4b-t7sj2 | El contenedor local-path-provisioner carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | istio-system | Pod/istiod-7c4fbc86db-6jpps | El contenedor discovery carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/provider-kubernetes-a3cbbe355fa7-5746645dcc-zqsxj | El contenedor package-runtime carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/provider-kubernetes-a3cbbe355fa7-5746645dcc-zqsxj | El contenedor package-runtime carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/provider-helm-4d90a08b9ede-7bdc97cd54-qvftm | El contenedor package-runtime carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/provider-helm-4d90a08b9ede-7bdc97cd54-qvftm | El contenedor package-runtime carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/function-patch-and-transform-6a1ab24d2512-75d54b5957-ctdxj | El contenedor package-runtime carece de solicitudes de recursos |
| HIGH | K8sRequiredResources/require-resources | crossplane-system | Pod/function-patch-and-transform-6a1ab24d2512-75d54b5957-ctdxj | El contenedor package-runtime carece de límites de recursos |
| HIGH | K8sRequiredResources/require-resources | backstage | Pod/backstage-postgresql-0 | El contenedor postgresql carece de límites de recursos |
| HIGH | K8sRequireLabels/require-compliance-labels | databases | Deployment/postgresql | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | databases | Deployment/postgresql | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-kubernetes-a3cbbe355fa7 | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-kubernetes-a3cbbe355fa7 | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-kubernetes-a3cbbe355fa7 | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-helm-4d90a08b9ede | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-helm-4d90a08b9ede | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/provider-helm-4d90a08b9ede | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| HIGH | K8sRequireLabels/require-compliance-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "cost-center",... |
| HIGH | K8sRequireLabels/require-compliance-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "cost-c... |
| HIGH | K8sRequireLabels/require-compliance-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'compliance-level'. Etiquetas requeridas: ["team", "c... |
| MEDIUM | K8sDenyPrivilegedEl contenedors/deny-privileged-containers | production | Pod/app-stable-75f6849556-wh4bf | El contenedor 'istio-init' no debe ejecutarse como root (UID 0) |
| MEDIUM | K8sDenyPrivilegedEl contenedors/deny-privileged-containers | production | Pod/app-stable-75f6849556-m99dd | El contenedor 'istio-init' no debe ejecutarse como root (UID 0) |
| MEDIUM | K8sDenyPrivilegedEl contenedors/deny-privileged-containers | production | Pod/app-stable-75f6849556-48fjn | El contenedor 'istio-init' no debe ejecutarse como root (UID 0) |
| MEDIUM | K8sDenyPrivilegedEl contenedors/deny-privileged-containers | production | Pod/app-canary-6cc6b969cf-zvlzb | El contenedor 'istio-init' no debe ejecutarse como root (UID 0) |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/provider-helm-4d90a08b9ede | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/provider-helm-4d90a08b9ede | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/function-patch-and-transform-6a1ab24d2512 | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane-rbac-manager | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | crossplane-system | Deployment/crossplane | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | backstage | Deployment/backstage | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | observability | DaemonSet/otel-collector | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | observability | DaemonSet/otel-collector | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | observability | DaemonSet/otel-collector | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireLabels/require-labels | monitoring | DaemonSet/monitoring-prometheus-node-exporter | Al recurso le falta la etiqueta requerida 'team'. Etiquetas requeridas: ["team", "owner", "cost... |
| MEDIUM | K8sRequireLabels/require-labels | monitoring | DaemonSet/monitoring-prometheus-node-exporter | Al recurso le falta la etiqueta requerida 'owner'. Etiquetas requeridas: ["team", "owner", "cos... |
| MEDIUM | K8sRequireLabels/require-labels | monitoring | DaemonSet/monitoring-prometheus-node-exporter | Al recurso le falta la etiqueta requerida 'cost-center'. Etiquetas requeridas: ["team", "owner"... |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/prometheus-monitoring-kube-prometheus-prometheus-0 | El contenedor 'prometheus' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/prometheus-monitoring-kube-prometheus-prometheus-0 | El contenedor 'init-config-reloader' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/prometheus-monitoring-kube-prometheus-prometheus-0 | El contenedor 'config-reloader' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-prometheus-node-exporter-6hb28 | El contenedor 'node-exporter' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-kube-state-metrics-7458844c7c-xdcw6 | El contenedor 'kube-state-metrics' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-kube-prometheus-operator-77b866bdbb-2gxsn | El contenedor 'kube-prometheus-stack' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor 'grafana-sc-datasources' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor 'grafana-sc-dashboard' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | El contenedor 'grafana' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor 'init-config-reloader' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor 'config-reloader' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | El contenedor 'alertmanager' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | local-path-storage | Pod/local-path-provisioner-67b8995b4b-t7sj2 | El contenedor 'local-path-provisioner' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | istio-system | Pod/istiod-7c4fbc86db-6jpps | El contenedor 'discovery' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | crossplane-system | Pod/provider-kubernetes-a3cbbe355fa7-5746645dcc-zqsxj | El contenedor 'package-runtime' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | crossplane-system | Pod/provider-helm-4d90a08b9ede-7bdc97cd54-qvftm | El contenedor 'package-runtime' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | crossplane-system | Pod/function-patch-and-transform-6a1ab24d2512-75d54b5957-ctdxj | El contenedor 'package-runtime' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRequireResourceLimits/require-resource-limits | backstage | Pod/backstage-postgresql-0 | El contenedor 'postgresql' debe definir solicitudes/límites de CPU y memoria |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/monitoring-kube-state-metrics-7458844c7c-xdcw6 | La imagen 'registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.18.0' is from un... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/monitoring-kube-prometheus-operator-77b866bdbb-2gxsn | La imagen 'quay.io/prometheus-operator/prometheus-operator:v0.89.0' is from unauthor... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | La imagen 'quay.io/kiwigrid/k8s-sidecar:2.5.0' is from unauthorized registry. Allowe... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/monitoring-grafana-58f5bc6ff-vglf7 | La imagen 'docker.io/grafana/grafana:12.4.0' proviene de un registro no autorizado. Permitidos:... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | La imagen 'quay.io/prometheus/alertmanager:v0.31.1' is from unauthorized registry. A... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | monitoring | Pod/alertmanager-monitoring-kube-prometheus-alertmanager-0 | La imagen 'quay.io/prometheus-operator/prometheus-config-reloader:v0.89.0' is from u... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | local-path-storage | Pod/local-path-provisioner-67b8995b4b-t7sj2 | La imagen 'docker.io/kindest/local-path-provisioner:v20251212-v0.29.0-alpha-105-g20c... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | istio-system | Pod/istiod-7c4fbc86db-6jpps | La imagen 'docker.io/istio/pilot:1.29.0' proviene de un registro no autorizado. Permitidos: ["g... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | istio-system | Pod/istio-ingressgateway-8d7447659-dnv9k | La imagen 'docker.io/istio/proxyv2:1.29.0' proviene de un registro no autorizado. Permitidos: [... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | istio-system | Pod/istio-egressgateway-f7fc5b56c-6sg6q | La imagen 'docker.io/istio/proxyv2:1.29.0' proviene de un registro no autorizado. Permitidos: [... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | default | Pod/platform-demo-app-5748c74749-m52kx | La imagen 'platform-demo-app:latest' proviene de un registro no autorizado. Permitidos: ["gcr.i... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | default | Pod/platform-demo-app-5748c74749-7s6gw | La imagen 'platform-demo-app:latest' proviene de un registro no autorizado. Permitidos: ["gcr.i... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | databases | Pod/postgresql-59b69f6b66-tc8jg | La imagen 'postgres:15-alpine' proviene de un registro no autorizado. Permitidos: ["gcr.io/", "... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | crossplane-system | Pod/provider-kubernetes-a3cbbe355fa7-5746645dcc-zqsxj | La imagen 'xpkg.upbound.io/crossplane-contrib/provider-kubernetes:v0.13.0' is from u... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | crossplane-system | Pod/provider-helm-4d90a08b9ede-7bdc97cd54-qvftm | La imagen 'xpkg.upbound.io/crossplane-contrib/provider-helm:v0.18.1' is from unautho... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | crossplane-system | Pod/function-patch-and-transform-6a1ab24d2512-75d54b5957-ctdxj | La imagen 'xpkg.upbound.io/crossplane-contrib/function-patch-and-transform:v0.7.0' i... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | crossplane-system | Pod/crossplane-rbac-manager-74494cb9bf-xqz79 | La imagen 'xpkg.crossplane.io/crossplane/crossplane:v2.2.0' is from unauthorized reg... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | crossplane-system | Pod/crossplane-5cb76b766d-8q5zs | La imagen 'xpkg.crossplane.io/crossplane/crossplane:v2.2.0' is from unauthorized reg... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | backstage | Pod/backstage-postgresql-0 | La imagen 'docker.io/bitnamilegacy/postgresql:15.4.0-debian-11-r10' is from unauthor... |
| MEDIUM | K8sRestrictLa imagenRegistries/restrict-image-registries | backstage | Pod/backstage-7cc8476cdc-dfhkw | La imagen 'ghcr.io/backstage/backstage:latest' is from unauthorized registry. Allowe... |