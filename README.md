# Kubernetes The Hard Way on Azure

```sh
$ az group create -n kubernetes -l eastus2

$ az resource list -g kubernetes --query='[].{name:name}' -o table
```

### Networking
 Kubernetes Networking Model: flat "IP-per-pod" model
  - pods on a node can communicate with all pods on all nodes without NAT
  - agents on a node (e.g. system daemons, kubelet) can communicate with all pods on that node
  - pods in the host network of a node can communicate with all pods on all nodes without NAT

10.240.0.0/24 IP address range = 254 compute instances

```sh
$ az network vnet create -g kubernetes \
  -n kubernetes-vnet \
  --address-prefix 10.240.0.0/24 \
  --subnet-name kubernetes-subnet

$ az network nsg create -g kubernetes -n kubernetes-nsg

$ az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --network-security-group kubernetes-nsg

$ az network nsg rule create -g kubernetes \
  -n kubernetes-allow-ssh \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 22 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1000

$ az network nsg rule create -g kubernetes \
  -n kubernetes-allow-api-server \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range 6443 \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1001

$ az network nsg rule list -g kubernetes --nsg-name kubernetes-nsg --query "[].{Name:name, \
  Direction:direction, Priority:priority, Port:destinationPortRange}" -o table

$ az network lb create -g kubernetes \
  -n kubernetes-lb \
  --backend-pool-name kubernetes-lb-pool \
  --public-ip-address kubernetes-pip \
  --public-ip-address-allocation static

$ az network public-ip list --query="[?name=='kubernetes-pip'].{ResourceGroup:resourceGroup, \
  Region:location,Allocation:publicIpAllocationMethod,IP:ipAddress}" -o table
```



### Compute

```sh

$ az vm image list --location eastus2 --publisher Canonical --offer UbuntuServer --sku 18.04-LTS --all -o table

$ UBUNTULTS="Canonical:UbuntuServer:18.04-LTS:18.04.202101290"

$ az vm availability-set create -g kubernetes -n controller-as

$ for i in 0 1 2; do
    echo "[Controller ${i}] Creating public IP..."
    az network public-ip create -n controller-${i}-pip -g kubernetes > /dev/null

    echo "[Controller ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n controller-${i}-nic \
        --private-ip-address 10.240.0.1${i} \
        --public-ip-address controller-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding \
        --lb-name kubernetes-lb \
        --lb-address-pools kubernetes-lb-pool > /dev/null

    echo "[Controller ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n controller-${i} \
        --image ${UBUNTULTS} \
        --nics controller-${i}-nic \
        --availability-set controller-as \
        --nsg '' \
        --admin-username 'kuberoot' \
        --generate-ssh-keys > /dev/null
done


$ az vm availability-set create -g kubernetes -n worker-as

$ for i in 0 1; do
    echo "[Worker ${i}] Creating public IP..."
    az network public-ip create -n worker-${i}-pip -g kubernetes > /dev/null

    echo "[Worker ${i}] Creating NIC..."
    az network nic create -g kubernetes \
        -n worker-${i}-nic \
        --private-ip-address 10.240.0.2${i} \
        --public-ip-address worker-${i}-pip \
        --vnet kubernetes-vnet \
        --subnet kubernetes-subnet \
        --ip-forwarding > /dev/null

    echo "[Worker ${i}] Creating VM..."
    az vm create -g kubernetes \
        -n worker-${i} \
        --image ${UBUNTULTS} \
        --nics worker-${i}-nic \
        --tags pod-cidr=10.200.${i}.0/24 \
        --availability-set worker-as \
        --nsg '' \
        --generate-ssh-keys \
        --admin-username 'kuberoot' > /dev/null
done

$ az vm list -d -g kubernetes -o table
```

### TLS Certificates
Generate Certificate Authority
```sh
$ cfssl gencert -initca ca-csr.json | cfssljson -bare ca
```

Generate Admin Client Certificates
```sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  admin-csr.json | cfssljson -bare admin
```

Control-plan consists of 5 components:
  - etcd (database)
  - kube-apiserver (API and core of cluster)
  - kube-controller-manager (performs operations on k8s resources)
  - kube-scheduler (main scheduler)
  - kubelets (starts containers on nodes)

Generate Kubelet Client Certificates
```sh
$ EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -o tsv)

$ for instance in worker-0 worker-1; do
INTERNAL_IP=$(az vm show -d -n ${instance} -g kubernetes --query privateIps -o tsv)

cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=${instance},${EXTERNAL_IP},${INTERNAL_IP} \
  -profile=kubernetes \
  ${instance}-csr.json | cfssljson -bare ${instance}
done
```

Generate Controller Manager Client Certificate
```sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-controller-manager-csr.json | cfssljson -bare kube-controller-manager
```

Generate Kube Proxy Client Certificate
```sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-proxy-csr.json | cfssljson -bare kube-proxy
```

Generate Scheduler Client Certificate
```sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  kube-scheduler-csr.json | cfssljson -bare kube-scheduler
```

Generate Kubernetes API Server Certificate
```sh
$ KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query "ipAddress" -o tsv)

$ KUBERNETES_HOSTNAMES=kubernetes,kubernetes.default,kubernetes.default.svc,kubernetes.default.svc.cluster,kubernetes.svc.cluster.local

$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -hostname=10.32.0.1,10.240.0.10,10.240.0.11,10.240.0.12,${KUBERNETES_PUBLIC_ADDRESS},127.0.0.1,${KUBERNETES_HOSTNAMES} \
  -profile=kubernetes \
  kubernetes-csr.json | cfssljson -bare kubernetes
```

Generate Service Account Key Pair
```sh
$ cfssl gencert \
  -ca=ca.pem \
  -ca-key=ca-key.pem \
  -config=ca-config.json \
  -profile=kubernetes \
  service-account-csr.json | cfssljson -bare service-account
```

Distribute Certificates and Keys to Workers
```sh
$ for instance in worker-0 worker-1; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -o tsv)

  scp -o StrictHostKeyChecking=no ca.pem ${instance}-key.pem ${instance}.pem kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```
Distribute Certificates and Keys to Controllers

```sh
$ for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -o tsv)

  scp -o StrictHostKeyChecking=no ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

### Kubeconfig Files
Generate kubeconfig files for:
- controller manager
- kubelet
- kube-proxy
- scheduler clients
- admin user

```sh
$ KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query "ipAddress" -otsv)
```
Generate kubelet kubeconfig file
```sh
$ for instance in worker-0 worker-1; do
  kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-credentials system:node:${instance} \
    --client-certificate=${instance}.pem \
    --client-key=${instance}-key.pem \
    --embed-certs=true \
    --kubeconfig=${instance}.kubeconfig

  kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:node:${instance} \
    --kubeconfig=${instance}.kubeconfig

  kubectl config use-context default --kubeconfig=${instance}.kubeconfig
done
```

Generate kube-proxy kubeconfig file
```sh
$ kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443 \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-credentials kube-proxy \
  --client-certificate=kube-proxy.pem \
  --client-key=kube-proxy-key.pem \
  --embed-certs=true \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config set-context default \
  --cluster=kubernetes-the-hard-way \
  --user=kube-proxy \
  --kubeconfig=kube-proxy.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-proxy.kubeconfig
```

Generate kube-controller-manager kubeconfig file
```sh
$ kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-credentials system:kube-controller-manager \
    --client-certificate=kube-controller-manager.pem \
    --client-key=kube-controller-manager-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-controller-manager \
    --kubeconfig=kube-controller-manager.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-controller-manager.kubeconfig
```

Generate kube-scheduler kubeconfig file
```sh
$ kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-credentials system:kube-scheduler \
    --client-certificate=kube-scheduler.pem \
    --client-key=kube-scheduler-key.pem \
    --embed-certs=true \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=system:kube-scheduler \
    --kubeconfig=kube-scheduler.kubeconfig

$ kubectl config use-context default --kubeconfig=kube-scheduler.kubeconfig
```

Generate admin user kubeconfig file
```sh
$ kubectl config set-cluster kubernetes-the-hard-way \
    --certificate-authority=ca.pem \
    --embed-certs=true \
    --server=https://127.0.0.1:6443 \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-credentials admin \
    --client-certificate=admin.pem \
    --client-key=admin-key.pem \
    --embed-certs=true \
    --kubeconfig=admin.kubeconfig

$ kubectl config set-context default \
    --cluster=kubernetes-the-hard-way \
    --user=admin \
    --kubeconfig=admin.kubeconfig

$ kubectl config use-context default --kubeconfig=admin.kubeconfig
```

Distribute kubelet/kube-proxy kubeconfig files to Workers
```sh
$ for instance in worker-0 worker-1; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp ${instance}.kubeconfig kube-proxy.kubeconfig kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

Distribute kube-controller-manager/kube-scheduler kubeconfig files to Controllers
```sh
for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp admin.kubeconfig kube-controller-manager.kubeconfig kube-scheduler.kubeconfig kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

### Data Encryption Config and Key
Distribute encryption config to Controllers
```sh
$ ENCRYPTION_KEY=$(head -c 32 /dev/urandom | base64)

$ for instance in controller-0 controller-1 controller-2; do
  PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
    -n ${instance}-pip --query "ipAddress" -otsv)

  scp -o StrictHostKeyChecking=no encryption-config.yaml kuberoot@${PUBLIC_IP_ADDRESS}:~/
done
```

### Bootstrap etcd cluster
Kubernetes components are stateless and store cluster state in etcd.

SSH into each controller instance
```sh
$ CONTROLLER="controller-0"

$ PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

$ ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

Download, install, and configure etcd release binaries
```sh
$ wget -q --show-progress --https-only --timestamping \
  "https://github.com/etcd-io/etcd/releases/download/v3.4.14/etcd-v3.4.14-linux-amd64.tar.gz"

$ tar -xvf etcd-v3.4.14-linux-amd64.tar.gz

$ sudo mv etcd-v3.4.14-linux-amd64/etcd* /usr/local/bin/

$ sudo mkdir -p /etc/etcd /var/lib/etcd

$ sudo cp ca.pem kubernetes-key.pem kubernetes.pem /etc/etcd/
```

Internal IP address will be used to serve client requests and communicate with etcd cluster peers.
```sh
$ INTERNAL_IP=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
```

Set etcd member name to unique hostname of VM
```sh
$ ETCD_NAME=$(hostname -s)
```

Start etcd service
```sh
$ cat > etcd.service <<EOF
[Unit]
Description=etcd
Documentation=https://github.com/coreos

[Service]
Type=notify
ExecStart=/usr/local/bin/etcd \\
  --name ${ETCD_NAME} \\
  --cert-file=/etc/etcd/kubernetes.pem \\
  --key-file=/etc/etcd/kubernetes-key.pem \\
  --peer-cert-file=/etc/etcd/kubernetes.pem \\
  --peer-key-file=/etc/etcd/kubernetes-key.pem \\
  --trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-trusted-ca-file=/etc/etcd/ca.pem \\
  --peer-client-cert-auth \\
  --client-cert-auth \\
  --initial-advertise-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-peer-urls https://${INTERNAL_IP}:2380 \\
  --listen-client-urls https://${INTERNAL_IP}:2379,https://127.0.0.1:2379 \\
  --advertise-client-urls https://${INTERNAL_IP}:2379 \\
  --initial-cluster-token etcd-cluster-0 \\
  --initial-cluster controller-0=https://10.240.0.10:2380,controller-1=https://10.240.0.11:2380,controller-2=https://10.240.0.12:2380 \\
  --initial-cluster-state new \\
  --data-dir=/var/lib/etcd
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

$ sudo mv etcd.service /etc/systemd/system/

$ sudo systemctl daemon-reload
$ sudo systemctl enable etcd
$ sudo systemctl start etcd
```

Verify and list etcd cluster members
```sh
$ sudo ETCDCTL_API=3 etcdctl member list \
  --endpoints=https://${INTERNAL_IP}:2379 \
  --cacert=/etc/etcd/ca.pem \
  --cert=/etc/etcd/kubernetes.pem \
  --key=/etc/etcd/kubernetes-key.pem
```

### Bootstrap Kubernetes Control Plane

SSH into each controller instance
```sh
$ CONTROLLER="controller-0"

$ PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${CONTROLLER}-pip --query "ipAddress" -otsv)

$ ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

Download, install, and configure Kubernetes Controller binaries
```sh
$ wget -q --show-progress --https-only --timestamping \
  "https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kube-apiserver" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kube-controller-manager" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kube-scheduler" \
  "https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kubectl"

$ chmod +x kube-apiserver kube-controller-manager kube-scheduler kubectl

$ sudo mv kube-apiserver kube-controller-manager kube-scheduler kubectl /usr/local/bin/

$ sudo mkdir -p /etc/kubernetes/config
```

Configure Kubernetes API Server
```sh
$ sudo mkdir -p /var/lib/kubernetes/

$ sudo mv ca.pem ca-key.pem kubernetes-key.pem kubernetes.pem \
    service-account-key.pem service-account.pem \
    encryption-config.yaml /var/lib/kubernetes/
```

Internal IP used to advertise API server to cluster members
```sh
INTERNAL_IP=$(ip addr show eth0 | grep -oP '(?<=inet\s)\d+(\.\d+){3}')
```

Create kube-apiserver service systemd unit file
```sh
cat <<EOF | sudo tee /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-apiserver \\
  --advertise-address=${INTERNAL_IP} \\
  --allow-privileged=true \\
  --apiserver-count=2 \\
  --audit-log-maxage=30 \\
  --audit-log-maxbackup=3 \\
  --audit-log-maxsize=100 \\
  --audit-log-path=/var/log/audit.log \\
  --authorization-mode=Node,RBAC \\
  --bind-address=0.0.0.0 \\
  --client-ca-file=/var/lib/kubernetes/ca.pem \\
  --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,TaintNodesByCondition,Priority,DefaultTolerationSeconds,DefaultStorageClass,PersistentVolumeClaimResize,MutatingAdmissionWebhook,ValidatingAdmissionWebhook,ResourceQuota \\
  --enable-swagger-ui=true \\
  --etcd-cafile=/var/lib/kubernetes/ca.pem \\
  --etcd-certfile=/var/lib/kubernetes/kubernetes.pem \\
  --etcd-keyfile=/var/lib/kubernetes/kubernetes-key.pem \\
  --etcd-servers=https://10.240.0.10:2379,https://10.240.0.11:2379,https://10.240.0.12:2379 \\
  --event-ttl=1h \\
  --experimental-encryption-provider-config=/var/lib/kubernetes/encryption-config.yaml \\
  --kubelet-certificate-authority=/var/lib/kubernetes/ca.pem \\
  --kubelet-client-certificate=/var/lib/kubernetes/kubernetes.pem \\
  --kubelet-client-key=/var/lib/kubernetes/kubernetes-key.pem \\
  --kubelet-https=true \\
  --runtime-config=api/all=true \\
  --service-account-key-file=/var/lib/kubernetes/service-account.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --service-node-port-range=30000-32767 \\
  --tls-cert-file=/var/lib/kubernetes/kubernetes.pem \\
  --tls-private-key-file=/var/lib/kubernetes/kubernetes-key.pem \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF

```

Configure Kubernetes Controller Manager
```sh
$ sudo mv kube-controller-manager.kubeconfig /var/lib/kubernetes/
```

Create kube-controller-manager service systemd unit file
```sh
$ cat <<EOF | sudo tee /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-controller-manager \\
  --address=0.0.0.0 \\
  --allocate-node-cidrs=true \\
  --cluster-cidr=10.200.0.0/16 \\
  --cluster-name=kubernetes \\
  --cluster-signing-cert-file=/var/lib/kubernetes/ca.pem \\
  --cluster-signing-key-file=/var/lib/kubernetes/ca-key.pem \\
  --kubeconfig=/var/lib/kubernetes/kube-controller-manager.kubeconfig \\
  --leader-elect=true \\
  --root-ca-file=/var/lib/kubernetes/ca.pem \\
  --service-account-private-key-file=/var/lib/kubernetes/service-account-key.pem \\
  --service-cluster-ip-range=10.32.0.0/24 \\
  --use-service-account-credentials=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Configure Kubernetes Scheduler
```sh
$ sudo mv kube-scheduler.kubeconfig /var/lib/kubernetes/
```

Create kube-scheduler service systemd unit file
```sh
$ cat <<EOF | sudo tee /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-scheduler \\
  --config=/etc/kubernetes/config/kube-scheduler.yaml \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Create kube-scheduler configuration file
```sh
$ cat <<EOF | sudo tee /etc/kubernetes/config/kube-scheduler.yaml
apiVersion: kubescheduler.config.k8s.io/v1beta1
kind: KubeSchedulerConfiguration
clientConnection:
  kubeconfig: "/var/lib/kubernetes/kube-scheduler.kubeconfig"
leaderElection:
  leaderElect: true
EOF
```

Start Controller services
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl enable kube-apiserver kube-controller-manager kube-scheduler
$ sudo systemctl start kube-apiserver kube-controller-manager kube-scheduler
```

Verfication
```sh
$ kubectl get componentstatuses
```

Configure API server with RBAC Authz permissions to Kubelet for retrieving metrics, logs, and executing commands in pods

Create ClusterRole with permissions to access Kubelet API
```sh
$ cat <<EOF | kubectl apply -f -
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
EOF
```

Kubernetes API Server authenticates to Kubelet as 'kubernetes' user,
using client certificate defined by the '--kubelet-client-certificate' flag

```sh
$ cat <<EOF | kubectl apply -f -
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
EOF
```

Provision external load balancer to front Kubernetes API Server
```sh
$ az network lb probe create -g kubernetes \
  --lb-name kubernetes-lb \
  --name kubernetes-apiserver-probe \
  --port 6443 \
  --protocol tcp

$ az network lb rule create -g kubernetes \
  -n kubernetes-apiserver-rule \
  --protocol tcp \
  --lb-name kubernetes-lb \
  --frontend-ip-name LoadBalancerFrontEnd \
  --frontend-port 6443 \
  --backend-pool-name kubernetes-lb-pool \
  --backend-port 6443 \
  --probe-name kubernetes-apiserver-probe
```

Verification
```sh
$ KUBERNETES_PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -otsv)

$ curl --cacert ca.pem https://$KUBERNETES_PUBLIC_IP_ADDRESS:6443/version
```

### Bootstrap Kubernetes Worker Nodes

SSH into each worker instance
```sh
$ WORKER="worker-0"

$ PUBLIC_IP_ADDRESS=$(az network public-ip show -g kubernetes \
  -n ${WORKER}-pip --query "ipAddress" -otsv)

$ ssh kuberoot@${PUBLIC_IP_ADDRESS}
```

Install OS dependencies
socat binary enables support for `$ kubectl port-forward` command
```sh
$ sudo apt-get update
$ sudo apt-get -y install socat conntrack ipset
```

Download, configure, and install Worker binaries
```sh
$ wget -q --show-progress --https-only --timestamping \
  https://github.com/kubernetes-sigs/cri-tools/releases/download/v1.19.0/crictl-v1.19.0-linux-amd64.tar.gz \
  https://storage.googleapis.com/gvisor/releases/nightly/latest/runsc \
  https://github.com/opencontainers/runc/releases/download/v1.0.0-rc10/runc.amd64\
  https://github.com/containernetworking/plugins/releases/download/v0.8.5/cni-plugins-linux-amd64-v0.8.5.tgz \
  https://github.com/containerd/containerd/releases/download/v1.3.2/containerd-1.3.2.linux-amd64.tar.gz \
  https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kubectl \
  https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kube-proxy \
  https://storage.googleapis.com/kubernetes-release/release/v1.19.7/bin/linux/amd64/kubelet

$ sudo mkdir -p \
  /etc/cni/net.d \
  /opt/cni/bin \
  /var/lib/kubelet \
  /var/lib/kube-proxy \
  /var/lib/kubernetes \
  /var/run/kubernetes

$ sudo mv runc.amd64 runc
$ chmod +x kubectl kube-proxy kubelet runc runsc
$ sudo mv kubectl kube-proxy kubelet runc runsc /usr/local/bin/
$ sudo tar -xvf crictl-v1.19.0-linux-amd64.tar.gz -C /usr/local/bin/
$ sudo tar -xvf cni-plugins-linux-amd64-v0.8.5.tgz -C /opt/cni/bin/
$ sudo tar -xvf containerd-1.3.2.linux-amd64.tar.gz -C /
```

Configure CNI Networking
```sh
$ POD_CIDR="$(echo $(curl --silent -H Metadata:true "http://169.254.169.254/metadata/instance/compute/tags?api-version=2017-08-01&format=text") | cut -d : -f2)"
```

Create bridge network configuration file
```sh
$ cat <<EOF | sudo tee /etc/cni/net.d/10-bridge.conf
{
    "cniVersion": "0.3.0",
    "name": "bridge",
    "type": "bridge",
    "bridge": "cnio0",
    "isGateway": true,
    "ipMasq": true,
    "ipam": {
        "type": "host-local",
        "ranges": [
          [{"subnet": "${POD_CIDR}"}]
        ],
        "routes": [{"dst": "0.0.0.0/0"}]
    }
}
EOF
```

Create loopback network configuration file
```sh
$ cat <<EOF | sudo tee /etc/cni/net.d/99-loopback.conf
{
    "cniVersion": "0.3.0",
    "name": "lo",
    "type": "loopback"
}
EOF
```

Create containerd configuration file
```sh
$ sudo mkdir -p /etc/containerd/

$ cat << EOF | sudo tee /etc/containerd/config.toml
[plugins]
  [plugins.cri.containerd]
    snapshotter = "overlayfs"
    [plugins.cri.containerd.default_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runc"
      runtime_root = ""
    [plugins.cri.containerd.untrusted_workload_runtime]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
    [plugins.cri.containerd.gvisor]
      runtime_type = "io.containerd.runtime.v1.linux"
      runtime_engine = "/usr/local/bin/runsc"
      runtime_root = "/run/containerd/runsc"
EOF
```

Create containerd service systemd unit file
```sh
$ cat <<EOF | sudo tee /etc/systemd/system/containerd.service
[Unit]
Description=containerd container runtime
Documentation=https://containerd.io
After=network.target

[Service]
ExecStartPre=/sbin/modprobe overlay
ExecStart=/bin/containerd

Delegate=yes
KillMode=process
# Having non-zero Limit*s causes performance problems due to accounting overhead
# in the kernel. We recommend using cgroups to do container-local accounting.
LimitNPROC=infinity
LimitCORE=infinity
LimitNOFILE=infinity
# Comment TasksMax if your systemd version does not supports it.
# Only systemd 226 and above support this version.
TasksMax=infinity

[Install]
WantedBy=multi-user.target
EOF
```

Configure kubelet
```sh
$ sudo mv ${HOSTNAME}-key.pem ${HOSTNAME}.pem /var/lib/kubelet/
$ sudo mv ${HOSTNAME}.kubeconfig /var/lib/kubelet/kubeconfig
$ sudo mv ca.pem /var/lib/kubernetes/
```

Create kubelet config
```sh
$ cat <<EOF | sudo tee /var/lib/kubelet/kubelet-config.yaml
kind: KubeletConfiguration
apiVersion: kubelet.config.k8s.io/v1beta1
authentication:
  anonymous:
    enabled: false
  webhook:
    enabled: true
  x509:
    clientCAFile: "/var/lib/kubernetes/ca.pem"
authorization:
  mode: Webhook
clusterDomain: "cluster.local"
clusterDNS:
  - "10.32.0.10"
podCIDR: "${POD_CIDR}"
resolvConf: "/run/systemd/resolve/resolv.conf"
runtimeRequestTimeout: "15m"
tlsCertFile: "/var/lib/kubelet/${HOSTNAME}.pem"
tlsPrivateKeyFile: "/var/lib/kubelet/${HOSTNAME}-key.pem"
EOF
```

Create kubelet service systemd unit file
```sh
$ cat <<EOF | sudo tee /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/usr/local/bin/kubelet \\
  --config=/var/lib/kubelet/kubelet-config.yaml \\
  --container-runtime=remote \\
  --container-runtime-endpoint=unix:///var/run/containerd/containerd.sock \\
  --image-pull-progress-deadline=2m \\
  --kubeconfig=/var/lib/kubelet/kubeconfig \\
  --network-plugin=cni \\
  --register-node=true \\
  --v=2
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Configure kubernetes proxy
```sh
$ sudo mv kube-proxy.kubeconfig /var/lib/kube-proxy/kubeconfig
```

Create kubernetes proxy config
```sh
$ cat <<EOF | sudo tee /var/lib/kube-proxy/kube-proxy-config.yaml
kind: KubeProxyConfiguration
apiVersion: kubeproxy.config.k8s.io/v1alpha1
clientConnection:
  kubeconfig: "/var/lib/kube-proxy/kubeconfig"
mode: "iptables"
clusterCIDR: "10.200.0.0/16"
EOF
```

Create kube-proxy service systemd unit file
```sh
$ cat <<EOF | sudo tee /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes

[Service]
ExecStart=/usr/local/bin/kube-proxy \\
  --config=/var/lib/kube-proxy/kube-proxy-config.yaml
Restart=on-failure
RestartSec=5

[Install]
WantedBy=multi-user.target
EOF
```

Start Worker services
```sh
$ sudo systemctl daemon-reload
$ sudo systemctl enable containerd kubelet kube-proxy
$ sudo systemctl start containerd kubelet kube-proxy
```

Verification from Controller nodes
```sh
$ kubectl get nodes
```

### Configure kubectl for Remote Access

```sh
$ KUBERNETES_PUBLIC_ADDRESS=$(az network public-ip show -g kubernetes \
  -n kubernetes-pip --query ipAddress -otsv)

$ kubectl config set-cluster kubernetes-the-hard-way \
  --certificate-authority=ca.pem \
  --embed-certs=true \
  --server=https://${KUBERNETES_PUBLIC_ADDRESS}:6443

$ kubectl config set-credentials admin \
  --client-certificate=admin.pem \
  --client-key=admin-key.pem

$ kubectl config set-context kubernetes-the-hard-way \
  --cluster=kubernetes-the-hard-way \
  --user=admin

$ kubectl config use-context kubernetes-the-hard-way

$ kubectl get componentstatuses
$ kubectl get nodes
```

### Pod Networking Routes
Pods scheduled to a node receive an IP address from the node's Pod CIDR range.
At this point pods can not communicate with other pods running on different nodes due to missing network routes.

```sh
$ for instance in worker-0 worker-1; do
  PRIVATE_IP_ADDRESS=$(az vm show -d -g kubernetes -n ${instance} --query "privateIps" -otsv)
  POD_CIDR=$(az vm show -g kubernetes --name ${instance} --query "tags" -o tsv)
  echo $PRIVATE_IP_ADDRESS $POD_CIDR
done
```

Create network routes for each worker instance
```sh
$ az network route-table create -g kubernetes -n kubernetes-routes

$ az network vnet subnet update -g kubernetes \
  -n kubernetes-subnet \
  --vnet-name kubernetes-vnet \
  --route-table kubernetes-routes

$ for i in 0 1; do
  az network route-table route create -g kubernetes \
  -n kubernetes-route-10-200-${i}-0-24 \
  --route-table-name kubernetes-routes \
  --address-prefix 10.200.${i}.0/24 \
  --next-hop-ip-address 10.240.0.2${i} \
  --next-hop-type VirtualAppliance
done

$ az network route-table route list -g kubernetes --route-table-name kubernetes-routes -o table
```


### DNS Cluster Add-on with CoreDNS

```sh
$ kubectl apply -f https://raw.githubusercontent.com/ivanfioravanti/kubernetes-the-hard-way-on-azure/master/deployments/coredns.yaml
$ kubectl apply -f coredns.yaml

OR

$ helm repo add coredns https://coredns.github.io/helm
$ helm install coredns/
```

Verification
```sh
$ kubectl run networkbox --image=praqma/network-multitool --command -- sleep 3600
$ POD_NAME=$(kubectl get pods -l run=busybox -o jsonpath="{.items[0].metadata.name}")
$ kubectl exec -ti $POD_NAME -- nslookup kubernetes
```


### Smoke Test

```sh
$ kubectl create deployment nginx --image=nginx
```
- validate `port-forward`
- validate `kubectl logs`
- validate `kubectl exec`

Expose over NodePort
```sh
$ kubectl expose deployment nginx --port 80 --type NodePort

$ NODE_PORT=$(kubectl get svc nginx \
  --output=jsonpath='{range .spec.ports[0]}{.nodePort}')

$ az network nsg rule create -g kubernetes \
  -n kubernetes-allow-nginx \
  --access allow \
  --destination-address-prefix '*' \
  --destination-port-range ${NODE_PORT} \
  --direction inbound \
  --nsg-name kubernetes-nsg \
  --protocol tcp \
  --source-address-prefix '*' \
  --source-port-range '*' \
  --priority 1002

$ EXTERNAL_IP=$(az network public-ip show -g kubernetes \
  -n worker-0-pip --query "ipAddress" -otsv)

$ curl -I http://$EXTERNAL_IP:$NODE_PORT
```

Deploy metrics server
```sh
$ kubectl apply -f https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.4.2/components.yaml

```

#### References
- https://github.com/ivanfioravanti/kubernetes-the-hard-way-on-azure
