# 安裝redis cluster on k8s
- 用專用節點
- 191,192,193 8G 
- 都先限制2G for one node
# redis on k8s
- 1 node: 1m1s
## yamls
- ns: cps-system
- 照順序新增
1. config-map => common.conf
2. headless-service
3. statefulset => 多帶--cluster-announce-ip
   - pvc (gluster)-> 16Gi
   - node selector & taint
4. service
## create cluster
```
redis-cli -a dsaq1234 --cluster create redis-cluster-0.redis-cluster.cps-system.svc.cluster.local:6379 redis-cluster-1.redis-cluster.cps-system.svc.cluster.local:6379 redis-cluster-2.redis-cluster.cps-system.svc.cluster.local:6379 redis-cluster-3.redis-cluster.cps-system.svc.cluster.local:6379 redis-cluster-4.redis-cluster.cps-system.svc.cluster.local:6379 redis-cluster-5.redis-cluster.cps-system.svc.cluster.local:6379 --cluster-replicas 1

```