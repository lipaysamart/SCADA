## serviceMonitor 示例
___
```
apiVersion: monitoring.coreos.com/v1
# 使用的CRD资源
kind: ServiceMonitor
metadata:
    name: ingress-nginx-metrics
    namespace: monitoring
spec:
# 指定的入口
  endpoints:
# 多久抓取一次指标
  - interval: 15s
# service.ports.name 名称
    port: prometheus
# 指定 svc 所在命名空间
  namespaceSelector:
    matchNames:
    - ingress-nginx
# 匹配 svc 的标签
  selector:
    matchLabels:
      app.kubernetes.io/name: ingress-nginx
```