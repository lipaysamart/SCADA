## 创建minio存储后端
___
```
$ kubectl create -f ci/minio.yaml  (需要修改文件内的 storageClass)

# 登录minio后台，创建buckets (username: minio, password: minio123)
- admin
- chunks
- ruler
```
## 创建redis做数据查询缓存
___
```
$ kubectl create -f ci/redis.yaml (默认注释了redis-export,如需要取消注释即可)
```
## loki-读写分离模式
___
```
$ kubectl create -f ci/loki-config.yaml (核心配置都在这个文件中)
$ vim loki-simple-scalable/values.yaml
---
# 使用先前创建的 Loki 配置文件. 来覆盖默认配置.(loki.existingSecretForConfig)
  existingSecretForConfig: loki-config
# 启用API管理
  adminApi:
    enabled: true
# 写入副本数为 1
write:
  replicas: 1
# 持久化数据
  persistence:
    size: 5Gi
    storageClass: nfs-csi
# 读取副本数为 1 
read:
  replicas: 1
# 启用 HPA 扩缩
  autoscaling:
    enabled: true
    # 最小副本数为 1
    minReplicas: 1
    # 最大副本数为 3
    maxReplicas: 3
    # 触发HPA规则 CPU 达到 80%
    targetCPUUtilizationPercentage: 60
    # 触发HPA规则 MEM 达到 80%
    targetMemoryUtilizationPercentage: 80
  resources: 
    requests:
      memory: 1Gi
      cpu: 500m
    limits:
      memory: 1Gi
      cpu: 500m
# 持久化数据
  persistence:
    size: 5Gi
    storageClass: nfs-csi
# 启用网关
gateway:
  enabled: true
# 指定副本为 1
  replicas: 3
# 启用 HPA 扩缩
  autoscaling:
    enabled: true
    # 最小副本数为 1
    minReplicas: 1
    # 最大副本数为 3
    maxReplicas: 3
    # 触发HPA规则 CPU 达到 80%
    targetCPUUtilizationPercentage: 60
    # 触发HPA规则 MEM 达到 80%
    targetMemoryUtilizationPercentage: 80
  resources: 
    requests:
      memory: 256Mi
      cpu: 128m
    limits:
      memory: 256Mi
      cpu: 128m
```