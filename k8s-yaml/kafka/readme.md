# install kafka cluster
- 197-199 加入k8s
- ref 1: https://juejin.cn/post/7330515218605637667
- ref 2: https://juejin.cn/post/7349437605857411083?from=search-suggest

# helm 安裝
```
helm repo add bitnami https://charts.bitnami.com/bitnami
helm pull bitnami/kafka --version 30.1.8
tar -xf kafka-30.1.8.tgz

# 修改配置
vim kafka/values.yaml


```
# taint & node select
```
kubectl taint nodes uat-kafka-54-197 server-type=kafka:NoSchedule

spec:
  container:
  nodeSelector:
    server-type: kafka
  tolerations:
    - key: "server-type"
      operator: "Equal"
      value: "kafka"
      effect: "NoSchedule"

```
# random uuid
```
[root@cps_rd_10-110-54-75 ~]# docker run --rm -it confluentinc/cp-kafka:7.6.2 kafka-storage random-uuid
wwAV2o3xRqCbHsd66CwzRA
```