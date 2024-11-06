# pv and pvc

## nfs storageclass

## local storageclass
```yaml
kind: StorageClass
apiVersion: storage.k8s.io/v1
metadata:
  name: local-storage
provisioner: kubernetes.io/no-provisioner
volumeBindingMode: WaitForFirstConsumer
```
### pv
```yaml

apiVersion: v1
kind: PersistentVolume
metadata:
  name: kafka-pv-0
spec:
  capacity:
    storage: 300Gi
  accessModes:
  - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: local-storage
  local:
    path: /data/kafka
  nodeAffinity:
    required:
      nodeSelectorTerms:
      - matchExpressions:
        - key: kubernetes.io/hostname
          operator: In
          values:
          -  uat-k8s-kafka-10-110-54-197
```