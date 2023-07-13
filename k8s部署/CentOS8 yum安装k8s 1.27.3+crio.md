# CentOS8 yum安装k8s 1.27.3+crio

## 一、机器准备

| 主机名     | IP          | 功能     |
| ---------- | ----------- | -------- |
| k8s-master | 192.168.2.2 | mast节点 |
| k8s-node1  | 192.168.2.3 | node1    |
| k8s-node2  | 192.168.2.4 | node2    |
| k8s-node3  | 192.168.2.5 | node3    |
| k8s-node4  | 192.168.2.6 | node4    |

## 二、环境准备(所有节点)

### 2.1 设置主机名和设置语言环境

```bash
hostnamectl set-hostname k8s-master  #其他节点改成　k8s-node1,k8s-node2,k8s-node3,k8s-node4
echo "export LC_ALL=en_US.UTF8" >> /etc/profile
source /etc/profile
```

### 2.2 修改/etc/hosts

```bash
cat >> /etc/hosts <<EOF
192.168.2.2  k8s-master
192.168.2.3  k8s-node1
192.168.2.4  k8s-node2
192.168.2.5  k8s-node3
192.168.2.6  k8s-node4
EOF
```

### 2.3 关闭SElinux

```bash
sed -i 's/enforcing/disabled/' /etc/selinux/config
setenforce 0
```

### 2.4 关闭swap

```bash
swapoff -a
sed -ri 's/.*swap.*/#&/' /etc/fstab
```

### 2.5 修改yum镜像源，安装基础依赖包

```bash
sed -e 's|^mirrorlist=|#mirrorlist=|g' \
         -e 's|^#baseurl=http://mirror.centos.org/$contentdir|baseurl=https://mirrors.tuna.tsinghua.edu.cn/centos|g' \
         -i.bak \
         /etc/yum.repos.d/CentOS-*.repo
yum update -y && yum -y install wget psmisc vim net-tools nfs-utils telnet yum-utils device-mapper-persistent-data lvm2 git network-scripts tar curl chrony         
```

### 2.6 修改网络配置(不需要)

```bash
cat > /etc/NetworkManager/conf.d/calico.conf << EOF
[keyfile]
unmanaged-devices=interface-name:cali*;interface-name:tunl*
EOF

systemctl restart NetworkManager
```

### 2.7 时间同步

#### (1)k8s-master节点

```bash
cat > /etc/chrony.conf << EOF
pool ntp.aliyun.com iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
allow 192.168.2.0/24
local stratum 10
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd ; systemctl enable chronyd
```

#### (2)node节点

```bash
cat > /etc/chrony.conf << EOF
pool 192.168.2.2 iburst
driftfile /var/lib/chrony/drift
makestep 1.0 3
rtcsync
keyfile /etc/chrony.keys
leapsectz right/UTC
logdir /var/log/chrony
EOF

systemctl restart chronyd ; systemctl enable chronyd
#使用客户端进行验证
chronyc sources -v
```

### 2.8 ulimit修改

```bash
ulimit -SHn 65535
cat >> /etc/security/limits.conf <<EOF
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* seft memlock unlimited
* hard memlock unlimitedd
EOF
```

### 2.9 sysctl参数修改

```bash
cat <<EOF > /etc/sysctl.d/k8s.conf
net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
fs.may_detach_mounts = 1
vm.overcommit_memory=1
vm.panic_on_oom=0
fs.inotify.max_user_watches=89100
fs.file-max=52706963
fs.nr_open=52706963
net.netfilter.nf_conntrack_max=2310720

net.ipv4.tcp_keepalive_time = 600
net.ipv4.tcp_keepalive_probes = 3
net.ipv4.tcp_keepalive_intvl =15
net.ipv4.tcp_max_tw_buckets = 36000
net.ipv4.tcp_tw_reuse = 1
net.ipv4.tcp_max_orphans = 327680
net.ipv4.tcp_orphan_retries = 3
net.ipv4.tcp_syncookies = 1
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.ip_conntrack_max = 65536
net.ipv4.tcp_max_syn_backlog = 16384
net.ipv4.tcp_timestamps = 0
net.core.somaxconn = 16384

net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
net.ipv6.conf.all.forwarding = 1
EOF

sysctl --system
```

### 2.10 ssh免密设置

```bash
ssh-copy-id --help
ssh-keygen -f /root/.ssh/id_rsa  -P ''
export IP="192.168.2.3 192.168.2.4 192.168.2.5 192.168.2.6"
for HOST in $IP;do      ssh-copy-id -p 51293 -o StrictHostKeyChecking=no $HOST; done
```

### 2.11 ipvs模块挂载

```bash
modprobe br_netfilter
lsmod | grep br_netfilter # 验证是否生效
yum -y install ipset ipvsadm
cat > /etc/sysconfig/modules/ipvs.modules <<EOF
modprobe -- ip_vs
modprobe -- ip_vs_rr
modprobe -- ip_vs_wrr
modprobe -- ip_vs_sh
modprobe -- nf_conntrack_ipv4
EOF

chmod +x /etc/sysconfig/modules/ipvs.modules
# 执行脚本
/etc/sysconfig/modules/ipvs.modules
# 验证ipvs模块
lsmod | grep -e ip_vs -e nf_conntrack_ipv4
```



## 三、安装kubernates(所有节点)

### 3.1 添加k8s国内yum源

```bash
cat <<EOF > /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://mirrors.aliyun.com/kubernetes/yum/repos/kubernetes-el7-x86_64
enabled=1
gpgcheck=0
repo_gpgcheck=0
gpgkey=https://mirrors.aliyun.com/kubernetes/yum/doc/yum-key.gpg https://mirrors.aliyun.com/kubernetes/yum/doc/rpm-package-key.gpg
EOF

yum clean all && yum makecache
```

### 3.2 安装cri-o

```bash
export OS=CentOS_8_Stream
export VERSION=1.27
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
dnf install cri-o -y
mkdir -p /var/lib/crio
```

### 3.3 修改/etc/crio/crio.conf，配置国内镜像源

```bash
[crio.image]
pause_image = "registry.aliyuncs.com/google_containers/pause:3.6"
registries = ["4v2510z7.mirror.aliyuncs.com:443/library","docker.io"]
```

### 3.4 启动crio

```bash
systemctl daemon-reload && systemctl enable --now crio && systemctl status crio
```

### 3.5 安装kubernates组件

```bash
dnf -y install kubelet kubeadm kubectl --disableexcludes=kubernetes epel-release
```

### 3.6 在每个节点上预先拉取系统镜像

```bash
kubeadm config images pull --image-repository registry.aliyuncs.com/google_containers
```

### 3.7 集群初始化(master)

#### (1)生成默认初始化文件

```bash
kubeadm config print init-defaults > kubeadm-config.yaml
```

#### (2)修改初始化文件kubeadm-config.yml,注释的地方需要修改

```yaml
apiVersion: kubeadm.k8s.io/v1beta3
bootstrapTokens:
- groups:
  - system:bootstrappers:kubeadm:default-node-token
  token: b574ec.0407860329c7f810
  ttl: 24h0m0s
  usages:
  - signing
  - authentication
kind: InitConfiguration
localAPIEndpoint:
  advertiseAddress: 192.168.2.2			#k8s-master节点IP
  bindPort: 6443
nodeRegistration:
  criSocket: unix:///var/run/crio/crio.sock			#使用crio
  imagePullPolicy: IfNotPresent
  name: k8s-master
  taints: null
---
apiServer:
  timeoutForControlPlane: 4m0s
apiVersion: kubeadm.k8s.io/v1beta3
certificatesDir: /etc/kubernetes/pki
clusterName: kubernetes
controllerManager: {}
dns:
  type: CoreDNS				#设置
etcd:
  local:
    dataDir: /var/lib/etcd
imageRepository: registry.aliyuncs.com/google_containers		#阿里镜像源
kind: ClusterConfiguration
kubernetesVersion: 1.27.3
networking:
  dnsDomain: cluster.local
  podSubnet: 10.85.0.0/16							#pod网段
  serviceSubnet: 10.96.0.0/12
scheduler: {}
```

#### (3)初始化集群

```bash
kubeadm init --config kubeadm-config.yml
```

#### (4)复制/etc/kubernetes/admin.conf到各node节点

```bash
scp -P 51293 /etc/kubernetes/admin.conf k8s-node1:/etc/kubernetes/admin.conf
scp -P 51293 /etc/kubernetes/admin.conf k8s-node2:/etc/kubernetes/admin.conf
scp -P 51293 /etc/kubernetes/admin.conf k8s-node3:/etc/kubernetes/admin.conf
scp -P 51293 /etc/kubernetes/admin.conf k8s-node4:/etc/kubernetes/admin.conf
```

#### (5)各节点加入(在node节点上执行)

```bash
kubeadm join 192.168.2.2:6443 --token b574ec.0407860329c7f810 --discovery-token-ca-cert-hash sha256:cfc560cf10665f2fe9bf30fb6cd0f1846f435f1d64183c1342bd740768ef2d83
```

#### (6)配置所有节点环境变量

```bash
echo "export KUBECONFIG=/etc/kubernetes/admin.conf" >> .bash_profile
source .bash_profile
```

#### (7)编辑kube-flannel.yml文件(可选)

```yaml
---
kind: Namespace
apiVersion: v1
metadata:
  name: kube-flannel
  labels:
    k8s-app: flannel
    pod-security.kubernetes.io/enforce: privileged
---
kind: ClusterRole
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
rules:
- apiGroups:
  - ""
  resources:
  - pods
  verbs:
  - get
- apiGroups:
  - ""
  resources:
  - nodes
  verbs:
  - get
  - list
  - watch
- apiGroups:
  - ""
  resources:
  - nodes/status
  verbs:
  - patch
- apiGroups:
  - networking.k8s.io
  resources:
  - clustercidrs
  verbs:
  - list
  - watch
---
kind: ClusterRoleBinding
apiVersion: rbac.authorization.k8s.io/v1
metadata:
  labels:
    k8s-app: flannel
  name: flannel
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: flannel
subjects:
- kind: ServiceAccount
  name: flannel
  namespace: kube-flannel
---
apiVersion: v1
kind: ServiceAccount
metadata:
  labels:
    k8s-app: flannel
  name: flannel
  namespace: kube-flannel
---
kind: ConfigMap
apiVersion: v1
metadata:
  name: kube-flannel-cfg
  namespace: kube-flannel
  labels:
    tier: node
    k8s-app: flannel
    app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "cniVersion": "0.3.1",
      "plugins": [
        {
          "type": "flannel",
          "delegate": {
            "hairpinMode": true,
            "isDefaultGateway": true
          }
        },
        {
          "type": "portmap",
          "capabilities": {
            "portMappings": true
          }
        }
      ]
    }
  net-conf.json: |
    {
      "Network": "10.85.0.0/16",		#pod网段
      "Backend": {
        "Type": "vxlan"
      }
    }
---
apiVersion: apps/v1
kind: DaemonSet
metadata:
  name: kube-flannel-ds
  namespace: kube-flannel
  labels:
    tier: node
    app: flannel
    k8s-app: flannel
spec:
  selector:
    matchLabels:
      app: flannel
  template:
    metadata:
      labels:
        tier: node
        app: flannel
    spec:
      affinity:
        nodeAffinity:
          requiredDuringSchedulingIgnoredDuringExecution:
            nodeSelectorTerms:
            - matchExpressions:
              - key: kubernetes.io/os
                operator: In
                values:
                - linux
      hostNetwork: true
      priorityClassName: system-node-critical
      tolerations:
      - operator: Exists
        effect: NoSchedule
      serviceAccountName: flannel
      initContainers:
      - name: install-cni-plugin
        image: docker.io/flannel/flannel-cni-plugin:v1.1.2
       #image: docker.io/rancher/mirrored-flannelcni-flannel-cni-plugin:v1.1.2
        command:
        - cp
        args:
        - -f
        - /flannel
        - /opt/cni/bin/flannel
        volumeMounts:
        - name: cni-plugin
          mountPath: /opt/cni/bin
      - name: install-cni
        image: docker.io/flannel/flannel:v0.22.0
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.22.0
        command:
        - cp
        args:
        - -f
        - /etc/kube-flannel/cni-conf.json
        - /etc/cni/net.d/10-flannel.conflist
        volumeMounts:
        - name: cni
          mountPath: /etc/cni/net.d
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
      containers:
      - name: kube-flannel
        image: docker.io/flannel/flannel:v0.22.0
       #image: docker.io/rancher/mirrored-flannelcni-flannel:v0.22.0
        command:
        - /opt/bin/flanneld
        args:
        - --ip-masq
        - --kube-subnet-mgr
        resources:
          requests:
            cpu: "100m"
            memory: "50Mi"
        securityContext:
          privileged: false
          capabilities:
            add: ["NET_ADMIN", "NET_RAW"]
        env:
        - name: POD_NAME
          valueFrom:
            fieldRef:
              fieldPath: metadata.name
        - name: POD_NAMESPACE
          valueFrom:
            fieldRef:
              fieldPath: metadata.namespace
        - name: EVENT_QUEUE_DEPTH
          value: "5000"
        volumeMounts:
        - name: run
          mountPath: /run/flannel
        - name: flannel-cfg
          mountPath: /etc/kube-flannel/
        - name: xtables-lock
          mountPath: /run/xtables.lock
      volumes:
      - name: run
        hostPath:
          path: /run/flannel
      - name: cni-plugin
        hostPath:
          path: /opt/cni/bin
      - name: cni
        hostPath:
          path: /etc/cni/net.d
      - name: flannel-cfg
        configMap:
          name: kube-flannel-cfg
      - name: xtables-lock
        hostPath:
          path: /run/xtables.lock
          type: FileOrCreate
```

#### (8)安装flannel(可选)

```bash
kubectl apply -f kube-flannel.yml
```

#### (9)安装calico网络插件cni(与flannel二选一，实测推荐)

```bash
wget http://manongbiji.oss-cn-beijing.aliyuncs.com/ittailkshow/k8s/download/calico.yaml
kubectl apply -f calico.yaml
```