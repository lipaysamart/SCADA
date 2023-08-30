## 内容改动

```yaml
# 添加一个额外挂载
---
extraVolumes:
  - name: syslog
    hostPath:
      path: /var/log/messages
# Extra volume mounts together. Corresponds to `extraVolumes`.
extraVolumeMounts: 
  - name: syslog
    mountPath: /var/log/messages
    readOnly: true
# 开启监控
serviceMonitor:
  # -- If enabled, ServiceMonitor resources for Prometheus Operator are created
  enabled: true
# 启用sidecar
sidecar:
  configReloader:
    enabled: true
```