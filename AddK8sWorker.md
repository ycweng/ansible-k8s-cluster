# Add K8s worker
- 紀錄增加3個worker並給taint label
## add worker
- install kubectl kubelet kubeadm by ansible
- print
```bash
master:~# kubeadm token create --print-join-command
kubeadm join 10.110.54.201:6443 --token 3c4vzg.vz1b6z27g1fzgdd9 --discovery-token-ca-cert-hash sha256:bdd1b22b3f1628849140d9ddef003495fde72fe32c0494a0def41c7cef02802d
 
```
## add taint
```
kubectl taint nodes uat-redis-54-191 server-type=redis:NoSchedule
# 移除
kubectl taint nodes uat-redis-54-191 server-type:NoSchedule-
```
- 用在服務上 `pod toleration`
```
tolerations:
- key: "server-type"
  operator: "Equal"
  value: "redis"
  effect: "NoSchedule"
```
