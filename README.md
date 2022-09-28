# cadvisor-k8s-fix
When using Rancher monitoring with Kubernetes 1.24 cAdvisor doesn't work properly due to the dockershim removal

Deploy using Kustomize or CD tool

## Extra instructions for prometheus (Rancher-monitoring-stack version 100.1.3+up19.0.3)

* Click edit as yaml
* Change defaultRules.rules.k8s to false
* Add the following to `additionalPrometheusRulesMap`

```yaml
additionalPrometheusRulesMap:
  k8s-custom-rules:
    groups:
      - name: k8s.rules
        rules:
          - expr: >-
              sum by (cluster, namespace, pod, container) (
                irate(container_cpu_usage_seconds_total{job="cadvisor", metrics_path="/metrics", image!=""}[5m])
              ) * on (cluster, namespace, pod) group_left(node) topk by
              (cluster, namespace, pod) (
                1, max by(cluster, namespace, pod, node) (kube_pod_info{node!=""})
              )
            record: >-
              node_namespace_pod_container:container_cpu_usage_seconds_total:sum_irate
          - expr: >-
              container_memory_working_set_bytes{job="cadvisor",
              metrics_path="/metrics", image!=""}

              * on (namespace, pod) group_left(node) topk by(namespace, pod) (1,
                max by(namespace, pod, node) (kube_pod_info{node!=""})
              )
            record: node_namespace_pod_container:container_memory_working_set_bytes
          - expr: >-
              container_memory_rss{job="cadvisor", metrics_path="/metrics",
              image!=""}

              * on (namespace, pod) group_left(node) topk by(namespace, pod) (1,
                max by(namespace, pod, node) (kube_pod_info{node!=""})
              )
            record: node_namespace_pod_container:container_memory_rss
          - expr: >-
              container_memory_cache{job="cadvisor", metrics_path="/metrics",
              image!=""}

              * on (namespace, pod) group_left(node) topk by(namespace, pod) (1,
                max by(namespace, pod, node) (kube_pod_info{node!=""})
              )
            record: node_namespace_pod_container:container_memory_cache
          - expr: >-
              container_memory_swap{job="cadvisor", metrics_path="/metrics",
              image!=""}

              * on (namespace, pod) group_left(node) topk by(namespace, pod) (1,
                max by(namespace, pod, node) (kube_pod_info{node!=""})
              )
            record: node_namespace_pod_container:container_memory_swap
          - expr: >-
              kube_pod_container_resource_requests{resource="memory",job="kube-state-metrics"} 
              * on (namespace, pod, cluster)

              group_left() max by (namespace, pod) (
                (kube_pod_status_phase{phase=~"Pending|Running"} == 1)
              )
            record: >-
              cluster:namespace:pod_memory:active:kube_pod_container_resource_requests
          - expr: |-
              sum by (namespace, cluster) (
                  sum by (namespace, pod, cluster) (
                      max by (namespace, pod, container, cluster) (
                        kube_pod_container_resource_requests{resource="memory",job="kube-state-metrics"}
                      ) * on(namespace, pod, cluster) group_left() max by (namespace, pod) (
                        kube_pod_status_phase{phase=~"Pending|Running"} == 1
                      )
                  )
              )
            record: namespace_memory:kube_pod_container_resource_requests:sum
          - expr: >-
              kube_pod_container_resource_requests{resource="cpu",job="kube-state-metrics"} 
              * on (namespace, pod, cluster)

              group_left() max by (namespace, pod) (
                (kube_pod_status_phase{phase=~"Pending|Running"} == 1)
              )
            record: >-
              cluster:namespace:pod_cpu:active:kube_pod_container_resource_requests
          - expr: |-
              sum by (namespace, cluster) (
                  sum by (namespace, pod, cluster) (
                      max by (namespace, pod, container, cluster) (
                        kube_pod_container_resource_requests{resource="cpu",job="kube-state-metrics"}
                      ) * on(namespace, pod, cluster) group_left() max by (namespace, pod) (
                        kube_pod_status_phase{phase=~"Pending|Running"} == 1
                      )
                  )
              )
            record: namespace_cpu:kube_pod_container_resource_requests:sum
          - expr: >-
              kube_pod_container_resource_limits{resource="memory",job="kube-state-metrics"} 
              * on (namespace, pod, cluster)

              group_left() max by (namespace, pod) (
                (kube_pod_status_phase{phase=~"Pending|Running"} == 1)
              )
            record: >-
              cluster:namespace:pod_memory:active:kube_pod_container_resource_limits
          - expr: |-
              sum by (namespace, cluster) (
                  sum by (namespace, pod, cluster) (
                      max by (namespace, pod, container, cluster) (
                        kube_pod_container_resource_limits{resource="memory",job="kube-state-metrics"}
                      ) * on(namespace, pod, cluster) group_left() max by (namespace, pod) (
                        kube_pod_status_phase{phase=~"Pending|Running"} == 1
                      )
                  )
              )
            record: namespace_memory:kube_pod_container_resource_limits:sum
          - expr: >-
              kube_pod_container_resource_limits{resource="cpu",job="kube-state-metrics"} 
              * on (namespace, pod, cluster)

              group_left() max by (namespace, pod) (
               (kube_pod_status_phase{phase=~"Pending|Running"} == 1)
               )
            record: >-
              cluster:namespace:pod_cpu:active:kube_pod_container_resource_limits
          - expr: |-
              sum by (namespace, cluster) (
                  sum by (namespace, pod, cluster) (
                      max by (namespace, pod, container, cluster) (
                        kube_pod_container_resource_limits{resource="cpu",job="kube-state-metrics"}
                      ) * on(namespace, pod, cluster) group_left() max by (namespace, pod) (
                        kube_pod_status_phase{phase=~"Pending|Running"} == 1
                      )
                  )
              )
            record: namespace_cpu:kube_pod_container_resource_limits:sum
          - expr: |-
              max by (cluster, namespace, workload, pod) (
                label_replace(
                  label_replace(
                    kube_pod_owner{job="kube-state-metrics", owner_kind="ReplicaSet"},
                    "replicaset", "$1", "owner_name", "(.*)"
                  ) * on(replicaset, namespace) group_left(owner_name) topk by(replicaset, namespace) (
                    1, max by (replicaset, namespace, owner_name) (
                      kube_replicaset_owner{job="kube-state-metrics"}
                    )
                  ),
                  "workload", "$1", "owner_name", "(.*)"
                )
              )
            labels:
              workload_type: deployment
            record: namespace_workload_pod:kube_pod_owner:relabel
          - expr: |-
              max by (cluster, namespace, workload, pod) (
                label_replace(
                  kube_pod_owner{job="kube-state-metrics", owner_kind="DaemonSet"},
                  "workload", "$1", "owner_name", "(.*)"
                )
              )
            labels:
              workload_type: daemonset
            record: namespace_workload_pod:kube_pod_owner:relabel
          - expr: |-
              max by (cluster, namespace, workload, pod) (
                label_replace(
                  kube_pod_owner{job="kube-state-metrics", owner_kind="StatefulSet"},
                  "workload", "$1", "owner_name", "(.*)"
                )
              )
            labels:
              workload_type: statefulset
            record: namespace_workload_pod:kube_pod_owner:relabel
```