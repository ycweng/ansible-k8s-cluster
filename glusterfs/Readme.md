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

# k8s上部屬glusterfs + heketi
- 要大改 ref: https://ithelp.ithome.com.tw/articles/10246200
- 使用`kube-system`
```bash 
./gk-deploy -g -n kube-system --admin-key admin@cps2024 --user-key user@cps2024 topology.json 

# 反悔
 ./gk-deploy -g --abort -n kube-system --admin-key admin@cps2024 --user-key user@cps2024
```

- 完成的話
```

heketi is now running and accessible via http://10.244.1.6:8080 . To run
administrative commands you can install 'heketi-cli' and use it as follows:

  # heketi-cli -s http://10.244.1.6:8080 --user admin --secret '<ADMIN_KEY>' cluster list

You can find it at https://github.com/heketi/heketi/releases . Alternatively,
use it from within the heketi pod:

  # /usr/bin/kubectl -n kube-system exec -i heketi-5cbc98f9d9-rnpqr -- heketi-cli -s http://localhost:8080 --user admin --secret '<ADMIN_KEY>' cluster list

For dynamic provisioning, create a StorageClass similar to this:

---
apiVersion: storage.k8s.io/v1beta1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.244.1.6:8080"
  restuser: "user"
  restuserkey: "user@cps2024"


Deployment complete!
```
## 建立storageclass
```
---
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: glusterfs-storage
provisioner: kubernetes.io/glusterfs
parameters:
  resturl: "http://10.110.54.201:32444"
  restuser: "admin"
  restuserkey: "admin@cps2024"
  volumetype: "replicate:3" 
```
- pvc 範例
```
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: cps-system-common
  annotations:
    volume.beta.kubernetes.io/storage-class: "glusterfs-storage"   
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 50Gi  
```

# troubleshoot
- gluster 硬碟重置
```
umount /dev/vdb
mkfs.xfs -f /dev/vdb
pvcreate /dev/vdb


lsblk
dmsetup remove 
```

# 測
## gluster 
- 進入pod
```
[root@uat-k8s-worker-10-110-54-204 /]# gluster peer status
Number of Peers: 2

Hostname: uat-k8s-worker-10-110-54-203
Uuid: 595a062d-8971-4baa-8447-eec002426b99
State: Peer in Cluster (Connected)

Hostname: 10.110.54.205
Uuid: e39dc0c2-274d-47ad-bdbe-7925b7453a77
State: Peer in Cluster (Connected)
[root@uat-k8s-worker-10-110-54-204 /]# gluster pool list
UUID                                    Hostname                        State
595a062d-8971-4baa-8447-eec002426b99    uat-k8s-worker-10-110-54-203    Connected 
e39dc0c2-274d-47ad-bdbe-7925b7453a77    10.110.54.205                   Connected 
0c6d700d-5d36-4ae9-af5e-1c9a62530473    localhost                       Connected 

[root@uat-k8s-worker-10-110-54-204 /]# gluster volume info all
 
Volume Name: heketidbstorage
Type: Replicate
Volume ID: 2da69497-38bd-4a67-aec0-5cdcaa0fd027
Status: Started
Snapshot Count: 0
Number of Bricks: 1 x 3 = 3
Transport-type: tcp
Bricks:
Brick1: 10.110.54.203:/var/lib/heketi/mounts/vg_00ec668f56645aadef68c144578ff1d1/brick_852eae24695453170e7ce1ec76a5396b/brick
Brick2: 10.110.54.205:/var/lib/heketi/mounts/vg_05984ffcc4e1e643eeb7ed301bed4839/brick_3877d40f17636161b70a82e96b2f286c/brick
Brick3: 10.110.54.204:/var/lib/heketi/mounts/vg_982bc9008644b846fa0624d6ae99fc46/brick_5df286573c229eb28d0cd15f0c3a6853/brick
Options Reconfigured:
user.heketi.id: a5bb465afcdd232b02675cfe842e6d0e
user.heketi.dbstoragelevel: 1
performance.readdir-ahead: off
performance.io-cache: off
performance.read-ahead: off
performance.strict-o-direct: on
performance.quick-read: off
performance.open-behind: off
performance.write-behind: off
performance.stat-prefetch: off
transport.address-family: inet
storage.fips-mode-rchecksum: on
nfs.disable: on
performance.client-io-threads: off
```
## heketi
```
wget https://github.com/heketi/heketi/releases/download/v10.4.0/heketi-client-v10.4.0-release-10.linux.amd64.tar.gz
tar -zxvf heketi-client-v10.4.0-release-10.linux.amd64.tar.gz

curl http://10.244.1.6:8080/hello
Hello from Heketi

HEKETI_CLI_USER=admin HEKETI_CLI_KEY=admin@cps2024 bin/heketi-cli -s http://10.244.1.6:8080 cluster list
Clusters:
Id:56a435888caea8e14a93b984a2cf5604 [file][block] 

root@uat-k8s-master-10-110-54-201:~/heketi-client# HEKETI_CLI_USER=admin HEKETI_CLI_KEY=admin@cps2024 bin/heketi-cli -s http://10.244.1.6:8080 node list
Id:d697afbc28574e598a069758aea5ebe2     Cluster:56a435888caea8e14a93b984a2cf5604
Id:db46c9ee993a5abcf3e84883ef4a779c     Cluster:56a435888caea8e14a93b984a2cf5604
Id:e879e5742bb4e1e109fa426eb54a9248     Cluster:56a435888caea8e14a93b984a2cf5604
root@uat-k8s-master-10-110-54-201:~/heketi-client# HEKETI_CLI_USER=admin HEKETI_CLI_KEY=admin@cps2024 bin/heketi-cli -s http://10.244.1.6:8080 node info d697afbc28574e598a069758aea5ebe2
Node Id: d697afbc28574e598a069758aea5ebe2
State: online
Cluster Id: 56a435888caea8e14a93b984a2cf5604
Zone: 1
Management Hostname: uat-k8s-worker-10-110-54-204
Storage Hostname: 10.110.54.204
Devices:
Id:a0ec53cce37d9bf41b9de47aa5c64130   Name:/dev/vdb            State:online    Size (GiB):299     Used (GiB):10      Free (GiB):289     Bricks:9 
```

## gluster storageclass