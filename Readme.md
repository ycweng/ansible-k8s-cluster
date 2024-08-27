# create k8s cluster
- creating 1.23.2 with `mirrors.aliyun.com`:
  - kubelet-1.23.2
  - kubeadm-1.23.2
  - kubectl-1.23.2
- run:
```=
# rename hostsname
ansible-playbook -i inventory/server sethostname.yml
ansible-playbook -i inventory/server install-k8s.yml
# set /etc/hosts for kubelet
ansible-playbook -i inventory/server set-etc-hosts.yml

# then run commands below in master
kubeadm init \
--apiserver-advertise-address=10.110.54.80 \
--apiserver-bind-port=6443 \
--service-cidr=10.1.0.0/16 \
--pod-network-cidr=10.244.0.0/16 \
--kubernetes-version=1.23.2

export KUBECONFIG=/etc/kubernetes/admin.conf

kubectl apply -f https://raw.githubusercontent.com/coreos/flannel/master/Documentation/kube-flannel.yml

cat /etc/cni/net.d/10-flannel.conflist

## run this on each worker
kubeadm join 127.0.0.80:6443 --token rwpx3l.c7iorc94r148sohn --discovery-token-ca-cert-hash sha256:ca4293a94694d2d44b9f9a4314c2498ce34feda0a408279f3e05c68a7275f44c

```

# install dashboard
```=
kubectl apply -f https://raw.githubusercontent.com/kubernetes/dashboard/v2.7.0/aio/deploy/recommended.yaml
# forward 30000:443
kubectl edit svc kubernetes-dashboard -n kubernetes-dashboard

# init dashboard permmision
kubectl get -n kubernetes-dashboard ServiceAccount
kubectl apply -f dashboard-admin.yml
kubectl apply -f dashboard-admin-secret.yml
# get token
kubectl get secret admin-user -n kubernetes-dashboard -o jsonpath={".data.token"} | base64 -d
```

# install glusterfs and mnt to k8s
- go to any 3 server 
```=
mkdir /etc/glusterfs
mkdir /var/lib/glusterd
mkdir /var/log/glusterfs
mkdir -p /gluster/data

# on node 1
docker run --name gluster1 -v /etc/glusterfs:/etc/glusterfs:z -v /var/lib/glusterd:/var/lib/glusterd:z -v /var/log/glusterfs:/var/log/glusterfs:z -v /sys/fs/cgroup:/sys/fs/cgroup:rw -d --privileged=true --net=host -v /gluster/data:/data --cgroupns=host ghcr.io/gluster/gluster-containers:centos
# on node 2
docker run --name gluster2 -v /etc/glusterfs:/etc/glusterfs:z -v /var/lib/glusterd:/var/lib/glusterd:z -v /var/log/glusterfs:/var/log/glusterfs:z -v /sys/fs/cgroup:/sys/fs/cgroup:rw -d --privileged=true --net=host -v /gluster/data:/data --cgroupns=host ghcr.io/gluster/gluster-containers:centos
# on node 3
docker run --name gluster3 -v /etc/glusterfs:/etc/glusterfs:z -v /var/lib/glusterd:/var/lib/glusterd:z -v /var/log/glusterfs:/var/log/glusterfs:z -v /sys/fs/cgroup:/sys/fs/cgroup:rw -d --privileged=true --net=host -v /gluster/data:/data --cgroupns=host ghcr.io/gluster/gluster-containers:centos

# back to node 1, run in container
docker exec -it gluster1 /bin/bash
## check network
gluster peer probe 127.0.0.node1-3
## create gluster volume for example is replica 3
gluster volume create cps-jfrog replica 3 127.0.0.82:/data/cps-jfrog 127.0.0.83:/data/cps-jfrog 127.0.0.84:/data/cps-jfrog force
gluster volume start cps-jfrog

## excute if need to check on file system
mount -t glusterfs 10.110.54.82:/cps-jfrog /mnt/glusterfs/cps-jfrog/

# make k8s availabe to use per namespace
kubectl apply -f gluster-endpoint-svc.yml

```

# install traefik
```bash=
helm repo add traefik https://traefik.github.io/charts

helm repo update

helm install -n kube-system traefik traefik/traefik

helm uninstall traefik traefik/traefik

```
## traefik dashboard
```bash=
# deployment加上
--api.insecure=true
# kube apply web-ui svc && ingress

```
https://godleon.github.io/blog/Kubernetes/k8s-Install-traefik-as-Ingress-Controller/
## traefik dashboard
```bash
kubectl create namespace monitor
```

---
# add master
```bash
[root@cps-k8s-master-10-110-54-80 ~]# kubeadm init phase upload-certs --upload-certs
I0822 17:10:48.389485  255325 version.go:255] remote version is much newer: v1.31.0; falling back to: stable-1.23
[upload-certs] Storing the certificates in Secret "kubeadm-certs" in the "kube-system" Namespace
[upload-certs] Using certificate key:
0c22f7afc7c6a5615bcbe2a9ff117eb36fc16448183bbded24eff56b7f039473


[root@cps-k8s-master-10-110-54-80 ~]# kubeadm token create --print-join-command                                                                                                              
kubeadm join 10.110.54.80:6443 --token qn84nk.9awexr86o5gwmguh --discovery-token-ca-cert-hash sha256:ca4293a94694d2d44b9f9a4314c2498ce34feda0a408279f3e05c68a7275f44c

### 兩段拼接 在新的節點輸入

kubeadm join 10.110.54.80:6443 --token 258rcg.jyx2efen3g6tirkn --discovery-token-ca-cert-hash sha256:ca4293a94694d2d44b9f9a4314c2498ce34feda0a408279f3e05c68a7275f44c --control-plane --certificate-key 0c22f7afc7c6a5615bcbe2a9ff117eb36fc16448183bbded24eff56b7f039473 --cri-socket /run/containerd/containerd.sock
```
- edit
```bash
kubectl -n kube-system edit cm kubeadm-config

---
apiVersion: v1
data:
  ClusterConfiguration: |
    apiServer:
      extraArgs:
        authorization-mode: Node,RBAC
      timeoutForControlPlane: 4m0s
    apiVersion: kubeadm.k8s.io/v1beta3
    certificatesDir: /etc/kubernetes/pki
    clusterName: kubernetes
    controllerManager: {}
    dns: {}
    etcd:
      local:
        dataDir: /var/lib/etcd
    imageRepository: k8s.gcr.io
    kind: ClusterConfiguration
    kubernetesVersion: v1.23.2
    controlPlaneEndpoint: "10.110.54.80:6443"  ## <-這
    networking:
      dnsDomain: cluster.local
      podSubnet: 10.244.0.0/16
      serviceSubnet: 10.1.0.0/16
    scheduler: {}
```