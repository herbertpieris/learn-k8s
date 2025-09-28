## Prepare to update system and install some packages

```bash
sudo apt-get update
sudo apt-get -y install wget curl vim openssl git
```

## Configure /etc/hosts for each hosts

```bash
sudo nano /etc/hosts
```

Add (append) lines below to the content of `/etc/hosts`

```bash
10.102.12.112 master-01.kubernetes.local master-01
10.102.12.111 node-01.kubernetes.local node-01
```

Try to ping the host by its hostname

```bash
ping master-01
ping node-01
ping master-01.kubernetes.local
ping node-01.kubernetes.local
```

## Download all required binary

Create a new directory named k8s/1.32.3

```bash
mkdir -p k8s/1.32.3
cd k8s/1.32.3
```

```bash
## run this on both master and node
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubectl -o kubectl

## Server components, run this on master only
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-apiserver -o kube-apiserver
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-controller-manager -o kube-controller-manager
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-scheduler -o kube-scheduler

## Node components, run this on node only
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kube-proxy -o kube-proxy
curl -L https://dl.k8s.io/v1.32.3/bin/linux/amd64/kubelet -o kubelet

## Container components, run this on node only
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.32.0/crictl-v1.32.0-linux-amd64.tar.gz -o crictl-v1.32.0-linux-amd64.tar.gz
curl -L https://github.com/opencontainers/runc/releases/download/v1.3.0-rc.1/runc.amd64 -o runc
curl -L https://github.com/containernetworking/plugins/releases/download/v1.6.2/cni-plugins-linux-amd64-v1.6.2.tgz -o cni-plugins-linux-amd64-v1.6.2.tgz
curl -L https://github.com/containerd/containerd/releases/download/v2.1.0-beta.0/containerd-2.1.0-beta.0-linux-amd64.tar.gz -o containerd-2.1.0-beta.0-linux-amd64.tar.gz

## Etcd components, run this on master only
curl -L https://github.com/etcd-io/etcd/releases/download/v3.6.0-rc.3/etcd-v3.6.0-rc.3-linux-amd64.tar.gz -o etcd-v3.6.0-rc.3-linux-amd64.tar.gz
```

Make all binary executeable and move to `/usr/local/bin`

```bash

## run this on master
chmod +x kubectl kube-apiserver kube-controller-manager kube-scheduler
sudo mv kubectl kube-apiserver kube-controller-manager kube-scheduler /usr/local/bin

## run this on node
chmod +x kubectl kubelet kube-proxy runc
sudo mv kubectl kubelet kube-proxy runc /usr/local/bin
```

Try checking kubectl version

```bash
kubectl version --client
```

## Generate Public Key Infrastructure

Generate certificate in your local host but make sure `openssl` and `kubectl` is available in your local machine.
Make adjustment to ca.conf file and create dedicated folder to hold the certificates

```bash
mkdir -p certs
```

### Certificate Authority

```bash
openssl genrsa -out ./certs/ca.key 4096
openssl req -x509 -new -sha512 -noenc \
    -key ./certs/ca.key -days 3653 \
    -config ./ca.conf \
    -out ./certs/ca.crt

### results
certs/ca.crt
certs/ca.key
```

### Generate required server certificates

```bash
## kube-apiserver
openssl genrsa -out ./certs/kube-apiserver.key 4096
openssl req -new -key ./certs/kube-apiserver.key -sha256 \
    -config ./ca.conf -section kube-apiserver \
    -out ./certs/kube-apiserver.csr
openssl x509 -req -days 3653 -in ./certs/kube-apiserver.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-apiserver.crt

## kube-controller-manager
openssl genrsa -out ./certs/kube-controller-manager.key 4096
openssl req -new -key ./certs/kube-controller-manager.key -sha256 \
    -config ./ca.conf -section kube-controller-manager \
    -out ./certs/kube-controller-manager.csr
openssl x509 -req -days 3653 -in ./certs/kube-controller-manager.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-controller-manager.crt

## kube-scheduler
openssl genrsa -out ./certs/kube-scheduler.key 4096
openssl req -new -key ./certs/kube-scheduler.key -sha256 \
    -config ./ca.conf -section kube-scheduler \
    -out ./certs/kube-scheduler.csr
openssl x509 -req -days 3653 -in ./certs/kube-scheduler.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-scheduler.crt

## service account
openssl genrsa -out ./certs/service-accounts.key 4096
openssl req -new -key ./certs/service-accounts.key -sha256 \
    -config ./ca.conf -section service-accounts \
    -out ./certs/service-accounts.csr
openssl x509 -req -days 3653 -in ./certs/service-accounts.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/service-accounts.crt
```

### Generate required client certificates

```bash
## kubelet
openssl genrsa -out ./certs/node-01.key 4096
openssl req -new -key ./certs/node-01.key -sha256 \
    -config ./ca.conf -section node-01 \
    -out ./certs/node-01.csr
openssl x509 -req -days 3653 -in ./certs/node-01.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/node-01.crt

## kube proxy
openssl genrsa -out ./certs/kube-proxy.key 4096
openssl req -new -key ./certs/kube-proxy.key -sha256 \
    -config ./ca.conf -section kube-proxy \
    -out ./certs/kube-proxy.csr
openssl x509 -req -days 3653 -in ./certs/kube-proxy.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/kube-proxy.crt
```

### Generate required admin access certificates

```bash
openssl genrsa -out ./certs/admin.key 4096
openssl req -new -key ./certs/admin.key -sha256 \
    -config ./ca.conf -section admin \
    -out ./certs/admin.csr
openssl x509 -req -days 3653 -in ./certs/admin.csr \
    -copy_extensions copyall \
    -sha256 -CA ./certs/ca.crt \
    -CAkey ./certs/ca.key \
    -CAcreateserial \
    -out ./certs/admin.crt
```

### Distribute the server certificates

```bash
# replace 'debian' with your ssh username
ssh debian@10.102.12.112 mkdir -p /home/debian/k8s/1.32.3/certs
scp ./certs/ca.key ./certs/ca.crt ./certs/kube-apiserver.key ./certs/kube-apiserver.crt ./certs/kube-controller-manager.crt ./certs/kube-controller-manager.key ./certs/kube-scheduler.crt ./certs/kube-scheduler.key ./certs/service-accounts.key ./certs/service-accounts.crt ./certs/admin.crt ./certs/admin.key debian@10.102.12.112:/home/debian/k8s/1.32.3/certs/
```

### Distribute the client certificates

```bash
# replace 'debian' with your ssh username
ssh debian@10.102.12.111 mkdir -p /home/debian/k8s/1.32.3/certs
scp ./certs/ca.crt debian@10.102.12.111:/home/debian/k8s/1.32.3/certs/ca.crt
scp ./certs/node-01.crt debian@10.102.12.111:/home/debian/k8s/1.32.3/certs/kubelet.crt
scp ./certs/node-01.key debian@10.102.12.111:/home/debian/k8s/1.32.3/certs/kubelet.key
scp ./certs/kube-proxy.crt debian@10.102.12.111:/home/debian/k8s/1.32.3/certs/kube-proxy.crt
scp ./certs/kube-proxy.key debian@10.102.12.111:/home/debian/k8s/1.32.3/certs/kube-proxy.key
```

## Generate Kubernetes Configuration Files

### Run on Master

Generate kubeconfig directory

```bash
mkdir -p kubeconfigs
```

#### kube-controller-manager

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=./certs/kube-controller-manager.crt \
    --client-key=./certs/kube-controller-manager.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-controller-manager \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-controller-manager.kubeconfig
```

#### kube-scheduler

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
    --client-certificate=./certs/kube-scheduler.crt \
    --client-key=./certs/kube-scheduler.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-scheduler \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-scheduler.kubeconfig
```

#### Admin user

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

kubectl config set-credentials admin \
    --client-certificate=./certs/admin.crt \
    --client-key=./certs/admin.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=admin \
    --kubeconfig=./kubeconfigs/admin.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/admin.kubeconfig
```

### Run on Nodes

Generate kubeconfig directory

```bash
mkdir -p kubeconfigs
```

### kubelet

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config set-credentials system:node:node-01 \
    --client-certificate=./certs/kubelet.crt \
    --client-key=./certs/kubelet.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:node:node-01 \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kubelet.kubeconfig
```

### kube-proxy

```bash
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://master-01.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
    --client-certificate=./certs/kube-proxy.crt \
    --client-key=./certs/kube-proxy.key \
    --embed-certs=true \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

kubectl config set-context default \
    --cluster=learn-k8s \
    --user=system:kube-proxy \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig

kubectl config use-context default \
    --kubeconfig=./kubeconfigs/kube-proxy.kubeconfig
```

## Generate Data Encryption Config

Generate an encryption key at master

```bash
head -c 32 /dev/urandom | base64
```

Create a file named 'encryption-config.yaml', and put generated key into file

```bash
mkdir -p configs
touch configs/encryption-config.yaml

### encryption-config.yaml content
---
kind: EncryptionConfiguration
apiVersion: apiserver.config.k8s.io/v1
resources:
  - resources:
      - secrets
    providers:
      - aescbc:
          keys:
            - name: key1
              secret: NsW8ANa6mXssM1WtCfIZrhTocMflSJrjfLHq8Rf+jyg= ## replace this with generated key
      - identity: {}
```

## Etcd setup on master

Extract downloaded etcd

```bash
tar -xvf etcd-v3.6.0-rc.3-linux-amd64.tar.gz

## create required directory for etcd
sudo mkdir -p /etc/etcd /var/lib/etcd
sudo chmod 700 /var/lib/etcd

sudo cp ./etcd-v3.6.0-rc.3-linux-amd64/etcd /usr/local/bin
sudo cp ./etcd-v3.6.0-rc.3-linux-amd64/etcdctl /usr/local/bin
sudo cp ./etcd-v3.6.0-rc.3-linux-amd64/etcdutl /usr/local/bin

sudo chmod +x /usr/local/bin/etcd /usr/local/bin/etcdctl /usr/local/bin/etcdutl

## create etcd systemd files
sudo nano /etc/systemd/system/etcd.service

## put this into the file content
[Unit]
Description=etcd
Documentation=https://github.com/etcd-io/etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \
  --name controller \
  --initial-advertise-peer-urls http://127.0.0.1:2380 \
  --listen-peer-urls http://127.0.0.1:2380 \
  --listen-client-urls http://127.0.0.1:2379 \
  --advertise-client-urls http://127.0.0.1:2379 \
  --initial-cluster-token etcd-cluster-0 \
  --initial-cluster controller=http://127.0.0.1:2380 \
  --initial-cluster-state new \
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start etcd service
sudo systemctl daemon-reload
sudo systemctl enable etcd
sudo systemctl restart etcd

## check if etcd is running
etcdctl member list
```

## Kubernetes Control Plane

Run these command in the master

```bash
## generate required directory
sudo mkdir -p /etc/kubernetes/config
sudo mkdir -p /var/lib/kubernetes
sudo mkdir -p /var/log/kubernetes

## copy required certificate to kubernetes data directory
sudo cp \
    ./certs/ca.crt ./certs/ca.key \
    ./certs/kube-apiserver.crt ./certs/kube-apiserver.key \
    ./certs/service-accounts.crt ./certs/service-accounts.key \
    ./configs/encryption-config.yaml \
    /var/lib/kubernetes
```

### Kube Api Server

```bash
## create kube-apiserver systemd files
sudo nano /etc/systemd/system/kube-apiserver.service

## put this into the file content
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \
 --allow-privileged=true \
 --audit-log-maxage=30 \
 --audit-log-maxbackup=3 \
 --audit-log-maxsize=100 \
 --audit-log-path=/var/log/kubernetes/audit.log \
 --authorization-mode=Node,RBAC \
 --bind-address=0.0.0.0 \
 --client-ca-file=/var/lib/kubernetes/ca.crt \
 --enable-admission-plugins=NamespaceLifecycle,NodeRestriction,LimitRanger,ServiceAccount,DefaultStorageClass,ResourceQuota \
 --etcd-servers=http://127.0.0.1:2379 \
 --event-ttl=1h \
 --encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \
 --kubelet-certificate-authority=/var/lib/kubernetes/ca.crt \
 --kubelet-client-certificate=/var/lib/kubernetes/kube-apiserver.crt \
 --kubelet-client-key=/var/lib/kubernetes/kube-apiserver.key \
 --runtime-config='api/all=true' \
 --service-account-key-file=/var/lib/kubernetes/service-accounts.crt \
 --service-account-signing-key-file=/var/lib/kubernetes/service-accounts.key \
 --service-account-issuer=https://master-01.kubernetes.local:6443 \
 --service-node-port-range=30000-32767 \
 --tls-cert-file=/var/lib/kubernetes/kube-apiserver.crt \
 --tls-private-key-file=/var/lib/kubernetes/kube-apiserver.key \
 --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start kube-apiserver service
sudo systemctl daemon-reload
sudo systemctl enable kube-apiserver
sudo systemctl restart kube-apiserver

## check kube-apiserver status
sudo systemctl status kube-apiserver
sudo journalctl -xe -u kube-apiserver
```

### Kube Controller Manager

```bash
sudo cp ./kubeconfigs/kube-controller-manager.kubeconfig /var/lib/kubernetes

## create kube-controller-manager systemd files
sudo nano /etc/systemd/system/kube-controller-manager.service

## put this into the file content
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \
  --bind-address=0.0.0.0 \
  --cluster-cidr=10.201.0.0/16 \
  --cluster-name=kubernetes \
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.crt \
  --cluster-signing-key-file=/var/lib/kubernetes/ca.key \
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \
  --root-ca-file=/var/lib/kubernetes/ca.crt \
  --service-account-private-key-file=/var/lib/kubernetes/service-accounts.key \
  --service-cluster-ip-range=10.231.0.0/24 \
  --use-service-account-credentials=true \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start kube-controller-manager service
sudo systemctl daemon-reload
sudo systemctl enable kube-controller-manager
sudo systemctl restart kube-controller-manager

## check kube-controller-manager status
sudo systemctl status kube-controller-manager
sudo journalctl -xe -u kube-controller-manager
```

### Kube Scheduler

```bash
sudo cp ./kubeconfigs/kube-scheduler.kubeconfig /var/lib/kubernetes

### create kube-scheduler configuration
sudo nano /etc/kubernetes/config/kube-scheduler.yaml

### put this in the file content
---
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true

## create kube-scheduler systemd files
sudo nano /etc/systemd/system/kube-scheduler.service

## put this into the file content
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \
  --config=/etc/kubernetes/config/kube-scheduler.yaml \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start kube-scheduler service
sudo systemctl daemon-reload
sudo systemctl enable kube-scheduler
sudo systemctl restart kube-scheduler

## check kube-scheduler status
sudo systemctl status kube-scheduler
sudo journalctl -xe -u kube-scheduler
```

Give about 1 minute for kubernetes control plane finished bootstraping.

Make sure kubernetes control plane is running and healthy

```bash
kubectl cluster-info --kubeconfig ./kubeconfigs/admin.kubeconfig

### output
Kubernetes control plane is running at https://master-01.kubernetes.local:6443
```

### RBAC for Kubelet Authorization

```bash
## create kube-apiserver-to-kubelet file
nano configs/kube-apiserver-to-kubelet.yaml

## put this into file content
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRole
metadata:
  annotations:
    rbac.authorization.kubernetes.io/autoupdate: "true"
  labels:
    kubernetes.io/bootstrapping: rbac-defaults
  name: system:kube-apiserver-to-kubelet
rules:
  - apiGroups:
      - ""
    resources:
      - nodes/proxy
      - nodes/stats
      - nodes/log
      - nodes/spec
      - nodes/metrics
    verbs:
      - "*"
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kube-apiserver
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
  - apiGroup: rbac.authorization.k8s.io
    kind: User
    name: kubernetes

## apply the rbac
kubectl apply -f configs/kube-apiserver-to-kubelet.yaml --kubeconfig ./kubeconfigs/admin.kubeconfig
```

### Verification

Verify if the cluster is running

```bash
kubectl get componentstatuses --kubeconfig ./kubeconfigs/admin.kubeconfig

### expected output
Warning: v1 ComponentStatus is deprecated in v1.19+
NAME                 STATUS    MESSAGE   ERROR
etcd-0               Healthy   ok
scheduler            Healthy   ok
controller-manager   Healthy   ok

curl --cacert ./certs/ca.crt https://master-01.kubernetes.local:6443/version

### expected output
{
  "major": "1",
  "minor": "32",
  "gitVersion": "v1.32.3",
  "gitCommit": "32cc146f75aad04beaaa245a7157eb35063a9f99",
  "gitTreeState": "clean",
  "buildDate": "2025-03-11T19:52:21Z",
  "goVersion": "go1.23.6",
  "compiler": "gc",
  "platform": "linux/amd64"
}
```

## Kubernetes Worker Nodes

Run these command in the each nodes

```bash
## install some required packages
sudo apt-get -y install socat conntrack ipset kmod

## disable swap
sudo swapon --show
sudo swapoff -a

## create required directory
sudo mkdir -p \
    /etc/cni/net.d \
    /opt/cni/bin \
    /var/lib/kubelet \
    /var/lib/kube-proxy \
    /var/lib/kubernetes \
    /var/run/kubernetes

## copy required certificates
sudo cp certs/ca.crt certs/kubelet.crt certs/kubelet.key /var/lib/kubelet

## copy required kubeconfig
sudo cp kubeconfigs/kubelet.kubeconfig /var/lib/kubelet/kubeconfig
sudo cp kubeconfigs/kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig

## extract and copy crictl binary
tar -xvf crictl-v1.32.0-linux-amd64.tar.gz
chmod +x crictl
sudo mv crictl /usr/local/bin

## extract and copy containerd binary
tar -xvf containerd-2.1.0-beta.0-linux-amd64.tar.gz
chmod +x bin/containerd-shim-runc-v2 bin/containerd bin/containerd-stress bin/ctr
sudo mv bin/containerd-shim-runc-v2 bin/containerd bin/containerd-stress bin/ctr /usr/local/bin

## extract and copy cni plugins binary
tar -xvf cni-plugins-linux-amd64-v1.6.2.tgz
chmod +x ipvlan tap loopback host-device portmap \
    ptp vlan bridge firewall macvlan dummy bandwidth \
    vrf tuning static dhcp host-local sbr
sudo mv ipvlan tap loopback host-device portmap \
    ptp vlan bridge firewall macvlan dummy bandwidth \
    vrf tuning static dhcp host-local sbr \
    /opt/cni/bin

## load and configure some kernel module
sudo modprobe br-netfilter
echo "br-netfilter" | sudo tee -a /etc/modules-load.d/modules.conf
echo "net.bridge.bridge-nf-call-iptables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
echo "net.bridge.bridge-nf-call-ip6tables = 1" | sudo tee -a /etc/sysctl.d/kubernetes.conf
```

### Configure CNI

```bash
## create bridge configuration
sudo nano /etc/cni/net.d/10-bridge.conf

## put this into file content
{
  "cniVersion": "1.0.0",
  "name": "bridge",
  "type": "bridge",
  "bridge": "cni0",
  "isGateway": true,
  "ipMasq": true,
  "ipam": {
    "type": "host-local",
    "ranges": [
      [{"subnet": "SUBNET"}]
    ],
    "routes": [{"dst": "10.91.12.0/22"}] # make sure this replaced with your node subnet
  }
}

## create loopback configuration
sudo nano /etc/cni/net.d/99-loopback.conf

## put this into file content
{
  "cniVersion": "1.1.0",
  "name": "lo",
  "type": "loopback"
}
```

### Configure Containerd

```bash
sudo mkdir -p /etc/containerd

## create containerd config
sudo nano /etc/containerd/config.toml

## put this into file content
version = 2

[plugins."io.containerd.grpc.v1.cri"]
  [plugins."io.containerd.grpc.v1.cri".containerd]
    snapshotter = "overlayfs"
    default_runtime_name = "runc"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
    runtime_type = "io.containerd.runc.v2"
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
[plugins."io.containerd.grpc.v1.cri".cni]
  bin_dir = "/opt/cni/bin"
  conf_dir = "/etc/cni/net.d"

## create containerd systemd files
sudo nano /etc/systemd/system/containerd.service

## put this into file content
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/usr/local/bin/containerd
Restart=always
RestartSec=5
Delegate=yes
KillMode=process
OOMScoreAdjust=-999
LimitNOFILE=1048576
LimitNPROC=infinity
LimitCORE=infinity

[Install]
WantedBy=multi-user.target

## enable and start containerd service
sudo systemctl daemon-reload
sudo systemctl enable containerd
sudo systemctl restart containerd

## check containerd status
sudo systemctl status containerd
sudo journalctl -xe -u containerd
```

### Kubelet

```bash
## create kubelet config file
sudo nano /var/lib/kubelet/config.yaml

## put this into file content
---
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
address: "0.0.0.0"
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubelet/ca.crt"
authorization:
  mode: Webhook
cgroupDriver: systemd
containerRuntimeEndpoint: "unix:///var/run/containerd/containerd.sock"
enableServer: true
failSwapOn: false
maxPods: 16
memorySwap:
  swapBehavior: NoSwap
port: 10250
resolvConf: "/etc/resolv.conf"
registerNode: true
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/kubelet.crt"
tlsPrivateKeyFile: "/var/lib/kubelet/kubelet.key"

## create kubelet systemd file
sudo nano /etc/systemd/system/kubelet.service

## put this into file content
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \
  --config=/var/lib/kubelet/config.yaml \
  --kubeconfig=/var/lib/kubelet/kubeconfig \
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start kubelet service
sudo systemctl daemon-reload
sudo systemctl enable kubelet
sudo systemctl restart kubelet

## check kubelet status
sudo systemctl status kubelet
sudo journalctl -xe -u kubelet
```

### Kubeproxy

```bash
## create kube-proxy configuration file
sudo nano /var/lib/kube-proxy/config.yaml

## put this into file content
---
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.201.0.0/16" # Cluster CIDR must be matched with ClusterCIDR on kube-controller-manager

## create kube-proxy systemd file
sudo nano /etc/systemd/system/kube-proxy.service

## put this into file content
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \
  --config=/var/lib/kube-proxy/config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target

## enable and start kube-proxy service
sudo systemctl daemon-reload
sudo systemctl enable kube-proxy
sudo systemctl restart kube-proxy

## check kube-proxy status
sudo systemctl status kube-proxy
sudo journalctl -xe -u kube-proxy
```

### Verification

At this point, the node should be registered to api server, however the node status still `NotReady` which is normal because we still need to install CNI to this cluster.

```bash
kubectl get nodes --kubeconfig kubeconfigs/admin.kubeconfig

## expected output
NAME      STATUS     ROLES    AGE     VERSION
node-01   NotReady   <none>   9m16s   v1.32.3
```

## Configure Kubectl For Remote Access

Run these commands in your local environment, to setup remote access to our cluster.

```bash
## add master ip address to /etc/hosts
## append this in your /etc/hosts file
10.91.12.53     server.kubernetes.local

## setup kubeconfig
kubectl config set-cluster learn-k8s \
    --certificate-authority=./certs/ca.crt \
    --embed-certs=true \
    --server=https://server.kubernetes.local:6443 \
    --kubeconfig=./kubeconfigs/kubeconfig

kubectl config set-credentials admin \
    --client-certificate=./certs/admin.crt \
    --client-key=./certs/admin.key \
    --kubeconfig=./kubeconfigs/kubeconfig

kubectl config set-context learn-k8s \
    --cluster=learn-k8s \
    --user=admin \
    --kubeconfig=./kubeconfigs/kubeconfig

kubectl config use-context learn-k8s \
    --kubeconfig=./kubeconfigs/kubeconfig

export KUBECONFIG=./kubeconfigs/kubeconfig

## now test accessing the cluster from your local environment
kubectl get nodes
### expected output
NAME      STATUS     ROLES    AGE   VERSION
node-01   NotReady   <none>   14m   v1.32.3
```

## Installing CNI ( Cilium )

First, make sure helm is available at your system and is running helm v3.

```bash
helm version

## add cilium helm repository
helm repo add cilium https://helm.cilium.io/

## install cilium
helm install cilium cilium/cilium --version 1.18.1 \
  --namespace kube-system \
  --set envoy.enabled=false \
  --set k8sServiceHost=master-01.kubernetes.local \
  --set k8sServicePort=6443

## take a moment to wait until cilium component is ready.
## since we only have a single node, there will be a pending pod for cilium-operator
## this is normal because by default, cilium-operator replica is set to 2
kubectl -n kube-system get pods | grep cilium

## expected output something like this
cilium-62ndx                       1/1     Running   0          2m9s
cilium-operator-56cb5f76bb-b9w2v   1/1     Running   0          2m9s
cilium-operator-56cb5f76bb-nn86q   0/1     Pending   0          2m9s

## once the cilium pods is ready, the node now become `Ready`
kubectl get nodes

NAME      STATUS   ROLES    AGE   VERSION
node-01   Ready    <none>   30m   v1.32.3
```

## Testing

Let's test the basic functionality of our cluster.

### Create a deployment

```bash
kubectl create deployment nginx --image=nginx --replicas=2

## wait until pod is ready
kubectl get pods

## expected some outputs
NAME                     READY   STATUS    RESTARTS   AGE
nginx-5869d7778c-kmcbh   1/1     Running   0          29s
nginx-5869d7778c-msq29   1/1     Running   0          29s

## let's try to port-forward from one pod
kubectl port-forward nginx-5869d7778c-kmcbh 8080:80

## expected some outputs
Forwarding from 127.0.0.1:8080 -> 80
Forwarding from [::1]:8080 -> 80
Handling connection for 8080
Handling connection for 8080
```

Open your browser, and access http://localhost:8080, you should be able to see the nginx welcome page.
