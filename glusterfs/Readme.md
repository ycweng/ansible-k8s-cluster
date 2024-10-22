# 安裝glusterfs +heketi, 
- 參考https://www.infvie.com/ops-notes/kubernetes-glusterfs-heketi.html
- 需有獨立硬碟 
```bash=
df -h

/dev/vdb             300G  2.2G  298G   1% /data
```
# k8s調整
- 安裝3個replica 203,204,205
## labeled node
```bash
kubectl label node uat-k8s-worker-10-110-54-203 storagenode=glusterfs
```
# glusterfs config
- ref: https://github.com/gluster/gluster-kubernetes.git
- topology.json
- manage: 節點名稱
- storage: ip
- device: 硬碟名稱 `/dev/sdb`

# k8s上部屬glusterfs
- 使用`kube-system`
```bash 
./gk-deploy -g -n kube-system
# 反悔
 ./gk-deploy -g --abort -n kube-system
```