# HA Kubernetes cluster with containerd 

### Prerequisites:
- 2 Ubuntu 24.04 LoadBalancer node's
- 3 Ubuntu 24.04 Kubernetes master node's
- 3 Ubuntu 24.04 Kubernetes worker node's

### Generate TLS certificates:

#### Prepare CFSSL on a Single Master Node (Certificate Authority Host):
*SSH into one of the master-node and run:*

```sh
{
  mkdir ~/certs && cd ~/certs
  wget https://pkg.cfssl.org/R1.2/cfssl_linux-amd64
  wget https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64
  chmod +x cfssl_linux-amd64 cfssljson_linux-amd64
  mv cfssl_linux-amd64 /usr/local/bin/cfssl
  mv cfssljson_linux-amd64 /usr/local/bin/cfssljson
}
```

#### Create CA Config and CSR for Root Certificate Authority:

```sh
{
cat > ca-config.json <<EOF
{
    "signing": {
        "default": {
            "expiry": "8766h"
        },
        "profiles": {
            "kubernetes": {
                "expiry": "8766h",
                "usages": ["signing","key encipherment","server auth","client auth"]
            }
        }
    }
}
EOF

cat > ca-csr.json <<EOF
{
  "CN": "Kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BLR",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "KA"
    }
  ]
}
EOF
}
```

#### Generate CA cert and key:

```bash
cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```
#### Create Server Certificate Signing Request (CSR) Including Full SANs:

```bash
cat > kubernetes-csr.json <<EOF
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster.local",
    "192.168.122.60",     # Load balancer VIP
    "192.168.122.101",    # master01 IP
    "192.168.122.102",    # master02 IP
    "192.168.122.103",    # master03 IP
    "192.168.122.51",     # load01 IP
    "192.168.122.52"      # load02 IP
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BLR",
      "O": "Kubernetes",
      "OU": "Cluster",
      "ST": "KA"
    }
  ]
}
EOF
```
**Note: Remove comments while creating the csr.**

#### Generate the Server Certificate Pair:

```sh
{
  cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes kubernetes-csr.json | \
   cfssljson -bare kubernetes
}
```

#### Generate Client/Admin Certificate for kubeadm and kubectl:
```sh
cat > admin-csr.json <<EOF
{
  "CN": "admin",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "IN",
      "L": "BLR",
      "O": "system:masters",
      "OU": "Cluster",
      "ST": "KA"
    }
  ]
}
EOF
```

#### Generate client cert:
```sh
{
  cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
}
```
`Copy admin.pem and admin-key.pem to your workstation/user where kubectl runs.`

#### Distribute Certificates Securely:
```sh
{
  mkdir -p /etc/kubernetes/pki
}
```

```sh
{
declare -a NODES=(192.168.122.101 192.168.122.102 192.168.122.103)

for node in ${NODES[@]}; do
  scp ca.pem kubernetes.pem kubernetes-key.pem root@$node:/etc/kubernetes/pki/
done
}
```

### Setup kubernetes master & worker nodes:

**Run on all master & worker nodes:**

#### Load Kerenel Modules:
```sh
{
cat << EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter
}
```
#### Add Kernel Settings:

```sh
{
cat <<EOF | sudo tee /etc/sysctl.d/99-kubernetes-cri.conf
net.bridge.bridge-nf-call-iptables = 1
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-ip6tables = 1
EOF

sysctl --system
}
```

#### Install container runtime: **containerd**
 
```sh
{
wget https://github.com/containerd/containerd/releases/download/v2.1.4/containerd-2.1.4-linux-amd64.tar.gz
tar Cxzvf /usr/local containerd-2.1.4-linux-amd64.tar.gz
rm containerd-2.1.4-linux-amd64.tar.gz
mkdir -p /etc/containerd
containerd config default | sudo tee /etc/containerd/config.toml
}
```
```sh
sed -i 's/SystemdCgroup \= false/SystemdCgroup \= true/g' /etc/containerd/config.toml
```

```sh
{
mkdir -pv /usr/local/lib/systemd/system/
wget https://raw.githubusercontent.com/containerd/containerd/main/containerd.service -O /usr/local/lib/systemd/system/containerd.service
}
```
```sh
{
systemctl daemon-reload
systemctl enable --now containerd
systemctl start containerd
systemctl status containerd
}
```

#### Disable swap permanently:

```sh
{
  swapoff -a
  sed -i '/ swap / s/^/#/' /etc/fstab
}
```
#### Install runc:
```sh
{
wget https://github.com/opencontainers/runc/releases/download/v1.3.0/runc.amd64
install -m 755 runc.amd64 /usr/local/sbin/runc
}
```

#### Installing CNI plugins:

```sh
{
wget https://github.com/containernetworking/plugins/releases/download/v1.7.1/cni-plugins-linux-amd64-v1.7.1.tgz
mkdir -p /opt/cni/bin
tar Cxzvf /opt/cni/bin cni-plugins-linux-amd64-v1.7.1.tgz
}
```
#### Install cricctl:
```sh
{
VERSION="v1.34.0"
curl -L https://github.com/kubernetes-sigs/cri-tools/releases/download/$VERSION/crictl-${VERSION}-linux-amd64.tar.gz --output crictl-${VERSION}-linux-amd64.tar.gz
sudo tar zxvf crictl-$VERSION-linux-amd64.tar.gz -C /usr/bin
rm -f crictl-$VERSION-linux-amd64.tar.gz
}
```

### Set containerd as default runtime for crictl:
```sh
{
cat << EOF | sudo tee /etc/crictl.yaml
runtime-endpoint: unix:///run/containerd/containerd.sock
image-endpoint: unix:///run/containerd/containerd.sock
timeout: 2
debug: false
pull-image-on-create: false
EOF
}
```

#### Add Apt repository & Install Kubernetes components:
```sh
{
apt-get install -y apt-transport-https ca-certificates curl gpg
mkdir /etc/apt/keyrings
curl -fsSL https://pkgs.k8s.io/core:/stable:/v1.34/deb/Release.key | sudo gpg --dearmor -o /etc/apt/keyrings/kubernetes-apt-keyring.gpg
cat << EOF | sudo tee /etc/apt/sources.list.d/kubernetes.list
deb [signed-by=/etc/apt/keyrings/kubernetes-apt-keyring.gpg] https://pkgs.k8s.io/core:/stable:/v1.34/deb/ /
EOF
}

```

```sh
{
apt-get update
apt-get install -y kubelet=1.34.0-1.1 kubeadm=1.34.0-1.1 kubectl=1.34.0-1.1
apt-mark hold kubelet kubeadm kubectl
}

```

### ETCD cluster creation`[On all etcd nodes]`:

*Note: Copy `ca.pem, kubernetes.pem, kubernetes-key.pem` to other master nodes.*

```sh
{
declare -a NODES=(192.168.122.101 192.168.122.102 192.168.122.103)

for node in ${NODES[@]}; do
  scp ca.pem kubernetes.pem kubernetes-key.pem root@$node: 
done
}
```

```sh
{
  mkdir -pv /etc/etcd /data/etcd
  mv ca.pem kubernetes.pem kubernetes-key.pem /etc/etcd/
  ll /etc/etcd/
}
```

#### Download etcd & etcdctl binaries from Github:

```sh
{
  ETCD_VER=v3.6.4
  wget -q --show-progress "https://github.com/etcd-io/etcd/releases/download/${ETCD_VER}/etcd-${ETCD_VER}-linux-amd64.tar.gz"
  tar zxf etcd-${ETCD_VER}-linux-amd64.tar.gz
  mv etcd-${ETCD_VER}-linux-amd64/etcd* /usr/local/bin/
  rm -rf etcd*
}
```
#### Create systemd unit file for etcd service:

*Set NODE_IP to the correct IP of the machine where you are running this*
```sh
{

NODE_IP="k8s-master-1"

ETCD1_IP="k8s-master-1"
ETCD2_IP="k8s-master-2"
ETCD3_IP="k8s-master-3"


cat <<EOF >/etc/systemd/system/etcd.service
[Unit]
Description=etcd

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${NODE_IP} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${NODE_IP}:2380 \\
  --listen-peer-urls https://${NODE_IP}:2380 \\
  --advertise-client-urls https://${NODE_IP}:2379 \\
  --listen-client-urls https://${NODE_IP}:2379,https://127.0.0.1:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster ${ETCD1_IP}=https://${ETCD1_IP}:2380,${ETCD2_IP}=https://${ETCD2_IP}:2380,${ETCD3_IP}=https://${ETCD3_IP}:2380 \\
  --initial-cluster-state new \\
  --data-dir=/data/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

}
```

### Enable and Start etcd service:
```sh
{
  systemctl daemon-reload
  systemctl enable --now etcd
  systemctl start etcd
  systemctl status etcd
}
```
### Verify Etcd cluster status:
```sh
{
ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://127.0.0.1:2379 \
  --cacert=/etc/etcd/kubernetes.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
}
```
`Example output:`
```sh
25ef0cb30a2929f1, started, k8s-master-1, https://k8s-master-1:2380, https://k8s-master-1:2379, false
5818a496a39840ca, started, k8s-master-2, https://k8s-master-2:2380, https://k8s-master-2:2379, false
c669cf505c64b0e8, started, k8s-master-3, https://k8s-master-3:2380, https://k8s-master-3:2379, false
```
*Repeat the above spets on other master nodes by replacing the ip address with respect to node ip address.*

### Initializing Master Node 1:

Create configuration file : 
```sh
{
HAPROXY_IP="haproxy-vip"

MASTER01_IP="k8s-master-1"
MASTER02_IP="k8s-master-2"
MASTER03_IP="k8s-master-3"

cat <<EOF > config.yaml
apiVersion: kubeadm.k8s.io/v1beta3
kind: ClusterConfiguration
kubernetesVersion: stable
apiServer:
  certSANs:
  - ${HAPROXY_IP}
  - ${MASTER01_IP}
  - ${MASTER02_IP}
  - ${MASTER03_IP}
  - "127.0.0.1"
  - "kubernetes"
  - "kubernetes.default"
  - "kubernetes.default.svc"
  - "kubernetes.default.svc.cluster.local"
  ExtraArgs:
    apiserver-count: "3"
controlPlaneEndpoint: "${HAPROXY_IP}:6443"
etcd:
  external:
    endpoints:
    - https://${MASTER01_IP}:2379
    - https://${MASTER02_IP}:2379
    - https://${MASTER03_IP}:2379
    caFile: /etc/etcd/ca.pem
    certFile: /etc/etcd/kubernetes.pem
    keyFile: /etc/etcd/kubernetes-key.pem
networking:
  podSubnet: 10.244.0.0/16
apiServerExtraArgs:
  apiserver-count: "3"
EOF
}
```
#### Initializing the master node:

```sh
kubeadm init --config=config.yaml --upload-certs
```

```sh
{    
  mkdir -p $HOME/.kube
  sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
  sudo chown $(id -u):$(id -g) $HOME/.kube/config
}
```

### Initializing Master Node 2 & 3 :

Use the kubeadm join command printed after master01 init, with the --control-plane flag and the certificate key uploaded:

```sh
sudo kubeadm join 192.168.122.60:6443 --token <token> \
  --discovery-token-ca-cert-hash sha256:<hash> \
  --control-plane --certificate-key <certificate-key>
```

### Adding Worker nodes: 

Get the join command from master node and run it on worker nodes to join the cluster:

```sh
kubeadm token create --print-join-command
```
When you run kubeadm init you will get the join command as follows , you need to run this on all worker nodes.

```sh
kubeadm join 192.168.X.X:6443 --token 85iw5v.ymo1wqcs9mrqmnnf --discovery-token-ca-cert-hash sha256:4710a65a4f0a2be37c03249c83ca9df2377b4433c6564db4e61e9c07f5a213dd
```

### Activate the flannel CNI plugin:

```sh
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml
```

*If you use custom podCIDR (not 10.244.0.0/16) you first need to download the above manifest and modify the network to match your one*


### Reference Links:

- [Containerd](https://github.com/containerd/containerd)

- [etcd](https://github.com/etcd-io/etcd)

- [runc](https://github.com/opencontainers/runc)

- [cni plugins](https://github.com/containernetworking/plugins)

- [cilium](https://docs.cilium.io/en/stable/gettingstarted/k8s-install-default/)