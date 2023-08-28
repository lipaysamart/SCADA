## Prerequisites
___
#### 安装环境
* Kube-Prometheus / main 分支
* Kubernetes / 1.26.7
## Getting Started
* 安装所需依赖资源，(不建议修改里面文件)
```
$ kubectl create -f manifests/setup
```
* 修改需要魔法拉取的镜像仓库
```
$ vim manifests/prometheusAdapter-deployment.yaml 
registry.k8s.io/prometheus-adapter/prometheus-adapter:v0.11.0 -> registry.cn-guangzhou.aliyuncs.com
$ vim manifests/kubeStateMetrics-deployment.yaml 
registry.k8s.io/kube-state-metrics/kube-state-metrics:v2.9.2 -> registry.cn-guangzhou.aliyuncs.com
```
* 修改Prometheus默认配置
```
$ vim manifests/prometheus-prometheus.yaml
---
# 添加一个额外标签, 方便多集群监控时当作唯一标识
  externalLabels: 
    cluster: local
# 去掉默认的 prometheus_replica 标签
  replicaExternalLabelName: ""
# 保存时间为 6h
  retention: 6h
# 启用 AdminAPI
  enableAdminAPI: true
# 持久化存储。(由于只是 6h的数据，我们不需要分配太多空间)
  storage:
    volumeClaimTemplates:
    - metadata:
        name: prom
      spec:
        accessModes: 
          - ReadWriteOnce
        storageClassName: openebs-hostpath
        resources:
          requests: 
            storage: 2Gi
# 远程写入 vmsingle
  remoteWrite:
    - url: http://vmsingle-victoria-metrics-single-server.monitoring.svc.cluster.local:8428/api/v1/write
# 默认是quay.io的镜像，有时会拉取不下来。
  image: registry.cn-guangzhou.aliyuncs.com/kubenetes-defualt/prometheus:v2.46.0
# 添加额外抓取配置
  additionalScrapeConfigs:
    name: additional-configs
    key: prometheus-additional.yaml
```
* 修改Grafana默认配置
```
$ vim manifests/grafana-deployment.yaml
---
# 自动挂载Token改为true
      automountServiceAccountToken: true
# 镜像改到 9.5.5 版本
        image: registry.cn-guangzhou.aliyuncs.com/kubernetes-default/grafana:9.5.5
# 删除 - emptyDir: {}
      volumes:
        name: grafana-storage
        persistentVolumeClaim:
          claimName: grafana-pvc
$ vim manifests/grafana-dashboardDatasources.yaml
---
# editable改为true可以有编辑权, url改写为vmsingle的dns地址
  datasources.yaml: |-
    {
        "apiVersion": 1,
        "datasources": [
            {
                "access": "proxy",
                "editable": true,
                "name": "prometheus",
                "orgId": 1,
                "type": "prometheus",
                "url": "http://vmsingle-victoria-metrics-single-server.monitoring.svc.cluster.local:8428",
                "version": 1
            }
        ]
    }
```
* 修改alert manager默认配置
```
$ vim manifests/alertmanager-alertmanager.yaml
---
# 副本数改为 1
  replicas: 1
# 数据保留时间
  retention: 9999h
# 指定secretName
  configSecret: alertmanager-main
# 持久化数据, 防止容器重启导致silence规则丢失
  storage:
    volumeClaimTemplate:
      spec:
        storageClassName: openebs-hostpath  # 填写你的 SC
        accessModes:
          - ReadWriteOnce
        resources:
          requests:
            storage: 1Gi

$ vim manifests/alertmanager-secret.yaml
---
# 一个简单的路由配置List:
apiVersion: v1
kind: Secret
metadata:
  labels:
    app.kubernetes.io/component: alert-router
    app.kubernetes.io/instance: main
    app.kubernetes.io/name: alertmanager
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.25.0
  name: alertmanager-main
  namespace: monitoring
stringData:
  alertmanager.yaml: |-
    global:
      resolve_timeout: 10m
    receivers:
      - name: default
        webhook_configs:
          - url: http://prometheus-alert-center.monitoring.svc:8080/prometheusalert?type=dd&tpl=default&ddurl=https://oapi.dingtalk.com/robot/send?access_token="修改你自己的token"
            send_resolved: true
      - name: workers
        webhook_configs:
          - url: http://prometheus-alert-center.monitoring.svc:8080/prometheusalert?type=dd&tpl=workers&ddurl=https://oapi.dingtalk.com/robot/send?access_token="修改你自己的token"
            send_resolved: true
    route:
      group_wait: 30s
      group_interval: 5m
      repeat_interval: 4h
      receiver: default
      routes:
        - receiver: default
        # 正则匹配告警名称
          matchers:
          - alertname =~ "KubeA(.+)|KubeStateMetrics(.+)|KubeCPU(.+)|KubeMemory(.+)|CPUThrottling"
        - receiver: workers
          matchers:
          - alertname =~ "KubeNode(.+)|Kubelet(.+)"
type: Opaque
```
* 修改blackbox默认配置
```
$ vim manifests/blackboxExporter-configuration.yaml
---
# 去除默认 irc、ssh 和 pop3 的检测模块, 新增 dns 模块。
apiVersion: v1
data:
  config.yml: |-
    "modules":
      "http_2xx":
        "http":
          "preferred_ip_protocol": "ip4"
          "valid_http_versions": ["HTTP/1.1", "HTTP/2"]
          "method": "GET"
        "prober": "http"
        "timeout": "5s"
      "http_post_2xx": # POST 请求
        "http":
          "method": "POST"
          "preferred_ip_protocol": "ip4"
        "prober": "http"
      "tcp_connect": # tcp 连接
        "prober": "tcp"
        "timeout": "10s"
        "tcp":
          "preferred_ip_protocol": "ip4"
      "dns":  # DNS 检测模块
        "prober": "dns"
        "dns":
          "transport_protocol": "udp"  # 默认是 udp, tcp
          "preferred_ip_protocol": "ip4"  # 默认是 ip6
          query_name: "kubernetes.default.svc.cluster.local" # 利用这个域名来检查dns服务器
      icmp:  # ping 检测服务器的存活
        prober: icmp
kind: ConfigMap
metadata:
  labels:
    app.kubernetes.io/component: exporter
    app.kubernetes.io/name: blackbox-exporter
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 0.24.0
  name: blackbox-exporter-configuration
  namespace: monitoring
```
* 添加自动发现配置
```
$ vim custom/additional-configs.yaml
---
- job_name: "endpoints"
  kubernetes_sd_configs:
    - role: endpoints
  relabel_configs: # 指标采集之前或采集过程中去重新配置
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scrape]
      action: keep # 保留具有 prometheus.io/scrape=true 这个注解的Service
      regex: true
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_path]
      action: replace
      target_label: __metrics_path__
      regex: (.+)
    - source_labels:
        [__address__, __meta_kubernetes_service_annotation_prometheus_io_port]
      action: replace
      target_label: __address__
      regex: ([^:]+)(?::\d+)?;(\d+) # RE2 正则规则，+是一次多多次，?是0次或1次，其中?:表示非匹配组(意思就是不获取匹配结果)
      replacement: $1:$2
    - source_labels: [__meta_kubernetes_service_annotation_prometheus_io_scheme]
      action: replace
      target_label: __scheme__
      regex: (https?)
    - action: labelmap
      regex: __meta_kubernetes_service_label_(.+)
      replacement: $1
    - source_labels: [__meta_kubernetes_namespace]
      action: replace
      target_label: kubernetes_namespace
    - source_labels: [__meta_kubernetes_service_name]
      action: replace
      target_label: kubernetes_service
    - source_labels: [__meta_kubernetes_pod_name]
      action: replace
      target_label: kubernetes_pod
    - source_labels: [__meta_kubernetes_node_name]
      action: replace
      target_label: kubernetes_node
```
* 修改RBAC配置
```
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  labels:
    app.kubernetes.io/component: prometheus
    app.kubernetes.io/instance: k8s
    app.kubernetes.io/name: prometheus
    app.kubernetes.io/part-of: kube-prometheus
    app.kubernetes.io/version: 2.46.0
  name: prometheus-k8s
rules:
  - apiGroups:
    # 为""则代表使用当前资源下的API
      - ""
    resources:
      - nodes
      - services
      - endpoints
      - pods
      - nodes/proxy
    verbs:
      - get
      - list
      - watch
  - apiGroups:
      - ""
    resources:
      - configmaps
      - nodes/metrics
    verbs:
      - get
  - nonResourceURLs:
      - /metrics
    verbs:
      - get
```
## Debug
___
本地调试可以创建这个路径下的文件`Prometheus/local`，来将svc都使用`NodePort`类型暴露
## TIPS
___
#### 配置多个AlertmanagerConfig
如果你有多个告警接收人和路由配置时，不想挤压在一个文件内，可以使用这个Kind资源`AlertmanagerConfig`,帮助我们创建管理告警配置。
#### 配置自定义监控
>在`Pormetheus-Operator`中给我们提供了`ServiceMonitor`对象,我们可以通过`ServiceMonitor`来关联`Metrics`数据接口的`Service`对象。

常见的监控对象在`PROMETHEUS/scrape`
#### 配置自动发现
通过添加注解`prometheus.io/scrape=true`来进行`service/pod`发现并进行自动监控,此前我们已经在Prometheus中进行配置`Prometheus.spec.additionalScrapeConfigs`。
#### 配置告警规则
>想要自定义一个报警规则，只需要创建一个能够被 `prometheus` 对象匹配的 `PrometheusRule` 对象即可。

常见的告警规则在`PROMETHEUS/rules`。
___
#### 监控kube_proxy组件
```
# kube-proxy默认监听的地址是127.0.0.1:10249，如果你是1.26的版本则默认不开启。
# 修改监听的端口，按如下方法:
$ k edit cm kube-proxy -nkube-system
# 将metricsBindAddress这段修改成metricsBindAddress: 0.0.0.0:10249
# 重启kube-proxy:
$ k get pods -nkube-system | grep kube-proxy | awk '{print $1}' | xargs kubectl delete pods -n kube-system
```
#### 监听etcd组件
```
# 在 master上修改 /etc/kubernetes/manifests/etcd.yaml 文件
# 将 command 中修改 --listen-metrics-urls=http://127.0.0.1:2381
--listen-metrics-urls=http://0.0.0.0:2381
```
#### 监听kube_scheduler组件&kube_controller_manager组件
```
# 在 master 路径下 /etc/kubernetes/manifests 找到这两个文件
# 将 command 中修改 --bind-address=127.0.0.1
--bind-address=0.0.0.0
```
#### 清理 Prometheus-Operator
```
$ k delete -f manifests/
$ k delete -f manifests/setup
```
## TroubleShooting
___

#### 多副本采集数据不一致
多副本的情况下正常来说请求是会去轮询访问后端的两个 `Prometheus` 实例, 这样可能就会导致每次请求到的`Prometheus`数据都不一样，解决这个问题的方法则可以在创建 `Service` 的时候添加 `sessionAffinity: ClientIP` 这样的属性，然后会根据 `ClientIP` 来做 `session` 亲和性。所以不用担心请求会到不同的副本上去