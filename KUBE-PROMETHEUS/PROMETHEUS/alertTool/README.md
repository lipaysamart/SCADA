## 内容改动

```
# 启动配置里的注释很清晰，需要修改默认配置请看这里
$ vim alertTool/prometheusAlert-config.yaml
```

```
# 增加了 /app/db 目录的数据持久化。防止容器重启，模板全部消失。
$ vim alertTool/prometheusAlert-deploy.yaml
---
# 这里改成了 utc-8
        env:
        - name: TZ
          value: "America/Los_Angeles"
# svc 暴露为 NodePort
  type: NodePort
```
## 相关仓库
[![standard-readme compliant](https://img.shields.io/badge/readme%20style-standard-brightgreen.svg?style=flat-square)](https://github.com/RichardLitt/standard-readme)
* https://github.com/feiyu563/PrometheusAlert - PrometheusAlert全家桶