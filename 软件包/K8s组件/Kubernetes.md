Bmw@baesar

# 环境描述

## 主机规划

| 主机名          | 地址           | 角色         |
| --------------- | -------------- | ------------ |
| test-nec-k8s-m1 | 172.28.104.112 | Master，Etcd |
| test-nec-k8s-m2 | 172.28.142.192 | Master，Etcd |
| test-nec-k8s-m3 | 172.28.142.196 | Master，Etcd |
| test-nec-k8s-lb | 172.28.142.159 | vip          |
| test-nec-k8s-n1 | 172.28.104.113 | Node         |
| test-nec-k8s-n2 | 172.28.104.114 | Node         |
| test-nec-k8s-n3 | 172.28.142.219 | Node         |
| test-nec-harbor | 172.28.142.226 | Harbor       |

## 地址规划

| 作用           | 地址          |
| -------------- | ------------- |
| SVC            | 10.250.0.0/16 |
| POD            | 10.244.0.0/16 |
| Kubernetes svc | 10.250.0.1    |
| kube-dns       | 10.250.0.10   |

## 版本规划

| 软件           | 版本   | 兼容性文档                                                   |
| -------------- | ------ | ------------------------------------------------------------ |
| Kubernetes     | 1.23.6 | -                                                            |
| Containerd     | 1.4.11 | https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md |
| Etcd           | 3.5.1  | https://github.com/kubernetes/kubernetes/blob/master/CHANGELOG/CHANGELOG-1.23.md |
| Calico         | 3.22.2 | https://projectcalico.docs.tigera.io/archive/v3.21/getting-started/kubernetes/requirements |
| CoreDNS        | 1.8.6  | https://github.com/coredns/deployment/blob/master/kubernetes/CoreDNS-k8s_version.md |
| Metrics-server | 0.5.2  | https://github.com/kubernetes-sigs/metrics-server            |
| Ingress        | v1.1.2 | https://github.com/kubernetes/ingress-nginx/                 |

# 1、集群环境配置

**以下操作在所有机器中都需要做**

## 1.1、修改hosts文件

```bash
vi /etc/hosts
172.28.104.112 bmw-eda-k8s-m1
172.28.142.192 test-nec-k8s-m2
172.28.142.196 test-nec-k8s-m3
172.28.104.113 bmw-eda-k8s-n1
172.28.104.114 bmw-eda-k8s-n2
172.28.142.219 test-nec-k8s-n3
172.28.142.159 test-nec-k8s-lb
172.28.142.226 test-harbor.nec-k8s.com.cn

```

## 1.2、关闭Selinux，Firewalld等

```bash
systemctl disable --now firewalld
systemctl disable --now dnsmasq
systemctl disable --now NetworkManager
sed -i "s/SELINUX=enforcing/SELINUX=disabled/" /etc/selinux/config
setenforce 0
```

## 1.3、关闭swap分区

```bash
swapoff -a
sysctl -w vm.swappiness=0
vi /etc/fstab
#/dev/mapper/cl-swap     swap                    swap    defaults        0 0
```

## 1.4、安装基本软件包

```bash
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup

wget -O /etc/yum.repos.d/epel.repo http://mirrors.aliyun.com/repo/epel-7.repo
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-7.repo

sed -i -e '/mirrors.cloud.aliyuncs.com/d' -e '/mirrors.aliyuncs.com/d' /etc/yum.repos.d/CentOS-Base.repo


# 导入内核公钥
rpm --import https://www.elrepo.org/RPM-GPG-KEY-elrepo.org
# #下载并安装elrepo仓库
rpm -Uvh http://www.elrepo.org/elrepo-release-7.0-2.el7.elrepo.noarch.rpm
yum makecache
yum install -y \
wget jq psmisc vim net-tools yum-utils device-mapper-persistent-data lvm2 conntrack-tools ipset ipvsadm sysstat libseccomp nfs-utils socat  bash-completion createrepo keepalived haproxy
```

## 1.5、内核升级

```bash
# 升级最新版内核
yum  --enablerepo=elrepo-kernel install -y kernel-ml kernel-ml-devel
# 查看可用内核
sudo awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
# 设置默认启动内核
grub2-set-default 0
# 生成配置文件
grub2-mkconfig -o /etc/grub2.cfg

grubby --args="user_namespace.enable=1" --update-kernel="$(grubby --default-kernel)"

reboot
```

## 1.6、内核参数设置

```bash
vi /etc/modules-load.d/ipvs.conf
ip_vs
ip_vs_lc
ip_vs_wlc
ip_vs_rr
ip_vs_wrr
ip_vs_lblc
ip_vs_lblcr
ip_vs_dh
ip_vs_sh
ip_vs_fo
ip_vs_nq
ip_vs_sed
ip_vs_ftp
ip_vs_sh
nf_conntrack
ip_tables
ip_set
xt_set
ipt_set
ipt_rpfilter
ipt_REJECT
ipip

# k8s优化配置
vi  /etc/sysctl.d/k8s.conf

net.ipv4.ip_forward = 1
net.bridge.bridge-nf-call-iptables = 1
net.bridge.bridge-nf-call-ip6tables = 1
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
#net.ipv4.vs.conn_reuse_mode=0

sysctl --system
systemctl enable --now systemd-modules-load.service
```

## 1.7、Limit配置

```bash
vi /etc/security/limits.conf
# 末尾添加如下内容
* soft nofile 655360
* hard nofile 131072
* soft nproc 655350
* hard nproc 655350
* soft memlock unlimited
* hard memlock unlimited


ulimit -SHn 65535

```





# 2、生成证书



## 2.1、证书工具准备

```bash
mkdir /opt/kube/ssl -p 
mkdir /opt/kube/bin
wget "https://pkg.cfssl.org/R1.2/cfssl_linux-amd64" -O /opt/kube/bin/cfssl
wget "https://pkg.cfssl.org/R1.2/cfssljson_linux-amd64" -O /opt/kube/bin/cfssljson
chmod +x /opt/kube/bin/cfssl /opt/kube/bin/cfssljson
echo "export PATH=$PATH:/opt/kube/bin" >> /etc/profile
source /etc/profile

```

## 2.2、生成CA证书

`**仅需要在Master01中配置**`

```bash
vi /opt/kube/ssl/ca-config.json
{
  "signing": {
    "default": {
      "expiry": "438000h"
    },
    "profiles": {
      "kubernetes": {
        "usages": [
            "signing",
            "key encipherment",
            "server auth",
            "client auth"
        ],
        "expiry": "438000h"
      }
    }
  }
}

vi /opt/kube/ssl/ca-csr.json
{
  "CN": "kubernetes",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ],
  "ca": {
    "expiry": "876000h"
  }
}

cfssl gencert -initca /opt/kube/ssl/ca-csr.json | cfssljson -bare /opt/kube/ssl/ca
```

## 2.3、生成Etcd证书

```bash
vi /opt/kube/ssl/etcd-csr.json
{
  "CN": "etcd",
  "hosts": [
    "192.168.253.128", # 需要吧etcd所有地址写在这里
    "127.0.0.1"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert \
-ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json \
-profile=kubernetes /opt/kube/ssl/etcd-csr.json | \
cfssljson -bare /opt/kube/ssl/etcd

```

## 2.4、生成API证书

```bash
vi /opt/kube/ssl/kubernetes-csr.json
{
  "CN": "kubernetes",
  "hosts": [
    "127.0.0.1",
    "10.250.0.1",
    "192.168.253.128",
    "kubernetes",
    "kubernetes.default",
    "kubernetes.default.svc",
    "kubernetes.default.svc.cluster",
    "kubernetes.default.svc.cluster.local"
  ],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert \
-ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json \
-profile=kubernetes /opt/kube/ssl/kubernetes-csr.json | \
cfssljson -bare /opt/kube/ssl/kubernetes
```

## 2.5、生成聚合证书

```bash
vi /opt/kube/ssl/aggregator-proxy-csr.json
{
  "CN": "aggregator",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert \
-ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json  \
-profile=kubernetes /opt/kube/ssl/aggregator-proxy-csr.json | \
cfssljson -bare /opt/kube/ssl/aggregator-proxy
```

## 2.6、生成kube-controller-manager证书

```bash
vi /opt/kube/ssl/controller-manager-csr.json
{
  "CN": "system:kube-controller-manager",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert \
-ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json  \
-profile=kubernetes /opt/kube/ssl/controller-manager-csr.json | \
cfssljson -bare /opt/kube/ssl/controller-manager
```

## 2.7、生成scheduler证书

```bash
vi /opt/kube/ssl/scheduler-csr.json
{
  "CN": "system:kube-scheduler",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert \
-ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json  \
-profile=kubernetes /opt/kube/ssl/scheduler-csr.json | \
cfssljson -bare /opt/kube/ssl/scheduler
```

## 2.8、生成kube-proxy证书

```bash
vi /opt/kube/ssl/kube-proxy-csr.json
{
  "CN": "system:kube-proxy",
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "k8s",
      "OU": "System"
    }
  ]
}

cfssl gencert -ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json  \
-profile=kubernetes /opt/kube/ssl/kube-proxy-csr.json | \
cfssljson -bare /opt/kube/ssl/kube-proxy
```

## 2.9、生成管理员证书

```bash
vi /opt/kube/ssl/admin-csr.json
{
  "CN": "admin",
  "hosts": [],
  "key": {
    "algo": "rsa",
    "size": 2048
  },
  "names": [
    {
      "C": "CN",
      "ST": "HangZhou",
      "L": "XS",
      "O": "system:masters",
      "OU": "System"
    }
  ]
}

cfssl gencert -ca=/opt/kube/ssl/ca.pem \
-ca-key=/opt/kube/ssl/ca-key.pem \
-config=/opt/kube/ssl/ca-config.json \
-profile=kubernetes /opt/kube/ssl/admin-csr.json | \
cfssljson -bare /opt/kube/ssl/admin
```

## 2.10、生成ServiceAount Key

```bash
openssl genrsa -out /opt/kube/ssl/sa.key 2048
openssl rsa -in /opt/kube/ssl/sa.key -pubout -out /opt/kube/ssl/sa.pub
```

## 2.11、分发证书

```bash

[root@test-nec-k8s-m1 bin]# for i in 113 114 ;do scp /opt/kube/ssl/* root@172.28.140.$i:/opt/kube/ssl/ ;done

```



# 3、安装Containerd

相关文档：

https://kubernetes.io/zh/docs/setup/production-environment/container-runtimes/#containerd

https://github.com/easzlab/kubeasz/blob/master/docs/setup/03-container_runtime.md

https://github.com/containerd/cri/blob/master/docs/installation.md

## 3.1、分区

```bash
fdisk /dev/sda
	n --- p --- 1 --- Enter --- Enter --- t --- 8e --- w

partprobe
pvcreate /dev/sdb1
vgcreate data /dev/sdb1
lvcreate -n containerd data -L +49G
lvs
  LV         VG     Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
  root       centos -wi-ao---- <69.51g                     
  containerd data   -wi-a-----  49.00g 
  
mkfs.xfs /dev/data/containerd
mkdir /var/lib/containerd
echo "/dev/data/containerd /var/lib/containerd xfs defaults 0 0 ">> /etc/fstab
mount -a
df -h |grep contai
/dev/mapper/data-containerd   99G   33M   99G    1% /var/lib/containerd
```

## 3.2、安装

```bash
wget https://github.com/containerd/containerd/releases/download/v1.4.11/cri-containerd-cni-1.4.11-linux-amd64.tar.gz

tar --no-overwrite-dir -C / -xzf cri-containerd-cni-1.4.11-linux-amd64.tar.gz
rm -rf /etc/cni/net.d/10-containerd-net.conflist
cat <<EOF | sudo tee /etc/modules-load.d/containerd.conf
overlay
br_netfilter
EOF

mkdir /etc/containerd
containerd config default |  tee /etc/containerd/config.toml
```

## 3.3、配置

```bash
##配置SystemdCgroup
vi /etc/containerd/config.toml
sandbox_image = "test-harbor.nec-k8s.com.cn/kube-system/pause-amd64:3.1"
...
[plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc]
  ...
  [plugins."io.containerd.grpc.v1.cri".containerd.runtimes.runc.options]
    SystemdCgroup = true
 
##配置连接Harbor证书
##证书从Harbor服务器中拷贝--在 Harbor /opt/harbor/ssl/ca.crt
cp /opt/software/harbor-ca.crt /etc/pki/ca-trust/source/anchors/
update-ca-trust
systemctl enable --now containerd

##拉取测试
##如果没有镜像需要手动去公网下载该镜像并导入到本地Harbor
crictl pull  harbor.dev-dms-nec.com/kube-system/pause-amd64:3.1
```

# 4、下载Kubernetes 二进制包，并安装客户端工具

## 4.1、下载二进制包

```bash
cd /opt/software/
wget https://dl.k8s.io/v1.23.6/kubernetes-server-linux-amd64.tar.gz -O kubernetes-server-v1.23.6-linux-amd64.tar.gz
tar zxf kubernetes-server-v1.23.6-linux-amd64.tar.gz
cd kubernetes/server/bin/

##仅Master节点需要 kube-apiserver kube-scheduler kube-controller-manager
cp kube-apiserver kube-scheduler kube-controller-manager kube-proxy kubelet kubectl /opt/kube/bin/
vim /etc/profile
	export PATH=$PATH:/opt/kube/bin
source  /etc/profile
```

## 4.2、命令补齐

```bash
##命令补全
vim ~/.bash_profile
	source /usr/share/bash-completion/bash_completion
	source <(kubectl completion bash)

source ~/.bash_profile
```

# 5、生成Kubeconfig文件

## 5.1、创建所需目录

```bash
mkdir /etc/kubernetes/
```

后面的仅需在一台机器中操作即可

## 5.2、kube-controller-manage

```bash
kubectl config set-cluster kubernetes \
 --certificate-authority=/opt/kube/ssl/ca.pem \
 --embed-certs=true \
 --server=https://192.168.253.128:6443 \
 --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

kubectl config set-credentials  system:kube-controller-manager \
--client-certificate=/opt/kube/ssl/controller-manager.pem \
--client-key=/opt/kube/ssl/controller-manager-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

kubectl config set-context system:kube-controller-manager@kubernetes \
--cluster=kubernetes  \
--user=system:kube-controller-manager \
--kubeconfig=/etc/kubernetes/controller-manager.kubeconfig

kubectl config use-context system:kube-controller-manager@kubernetes \
--kubeconfig=/etc/kubernetes/controller-manager.kubeconfig
```

## 5.3  、scheduler

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kube/ssl/ca.pem \
--embed-certs=true \
--server=https://192.168.253.128:6443 \
--kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-credentials system:kube-scheduler \
--client-certificate=/opt/kube/ssl/scheduler.pem \
--client-key=/opt/kube/ssl/scheduler-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config set-context system:kube-scheduler@kubernetes \
--cluster=kubernetes \
--user=system:kube-scheduler \
--kubeconfig=/etc/kubernetes/scheduler.kubeconfig

kubectl config use-context system:kube-scheduler@kubernetes \
--kubeconfig=/etc/kubernetes/scheduler.kubeconfig
```

## 5.4、kube-proxy

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kube/ssl/ca.pem \
--embed-certs=true \
--server=https://192.168.253.128:6443 \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-credentials system:kube-proxy \
--client-certificate=/opt/kube/ssl/kube-proxy.pem \
--client-key=/opt/kube/ssl/kube-proxy-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config set-context system:kube-proxy@kubernetes \
--cluster=kubernetes \
--user=system:kube-proxy \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig

kubectl config use-context system:kube-proxy@kubernetes \
--kubeconfig=/etc/kubernetes/kube-proxy.kubeconfig
```

## 5.5、管理员用户

```bash
kubectl config set-cluster kubernetes \
--certificate-authority=/opt/kube/ssl/ca.pem \
--embed-certs=true \
--server=https://192.168.253.128:6443 \
--kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-credentials admin \
--client-certificate=/opt/kube/ssl/admin.pem \
--client-key=/opt/kube/ssl/admin-key.pem \
--embed-certs=true \
--kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config set-context admin@kubernetes \
--cluster=kubernetes \
--user=admin \
--kubeconfig=/etc/kubernetes/admin.kubeconfig

kubectl config use-context admin@kubernetes \
--kubeconfig=/etc/kubernetes/admin.kubeconfig
```

## 5.6、分发Kubeconfig文件

```bash
[root@test-nec-k8s-m1 bin]# for i in  192 196 202 209 219 ;do scp -r /etc/kubernetes/ root@172.28.142.$i:/etc/ ;done
```



# 6、安装Etcd

**仅需要在Master中安装**

## 6.1、下载Etcd安装包

```bash
##在 三台 Master中安装
cd /opt/software/
wget https://github.com/etcd-io/etcd/releases/download/v3.5.1/etcd-v3.5.1-linux-amd64.tar.gz
mkdir /etc/etcd/
mkdir -p  /var/lib/etcd/wal
cd /opt/software
tar zxf etcd-v3.5.1-linux-amd64.tar.gz
cp etcd-v3.5.1-linux-amd64/etcd* /opt/kube/bin/
```

## 6.2、安装Etcd

```bash
------------------------
## 各节点配置文件
## MASTER-1
vi /etc/etcd/etcd.config.yml
name: 'master'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://192.168.253.128:2380'
listen-client-urls: 'https://192.168.253.128:2379,https://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://192.168.253.128:2380'
advertise-client-urls: 'https://192.168.253.128:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'master=https://192.168.253.128:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false


## MASTER-2
vi /etc/etcd/etcd.config.yml
name: 'test-nec-k8s-m2'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://172.28.142.192:2380'
listen-client-urls: 'https://172.28.142.192:2379,https://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://172.28.142.192:2380'
advertise-client-urls: 'https://172.28.142.192:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'test-nec-k8s-m1=https://172.28.104.112:2380,test-nec-k8s-m2=https://172.28.142.192:2380,test-nec-k8s-m3=https://172.28.142.196:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false


## MASTER-3
vi /etc/etcd/etcd.config.yml
name: 'test-nec-k8s-m3'
data-dir: /var/lib/etcd
wal-dir: /var/lib/etcd/wal
snapshot-count: 5000
heartbeat-interval: 100
election-timeout: 1000
quota-backend-bytes: 0
listen-peer-urls: 'https://172.28.142.196:2380'
listen-client-urls: 'https://172.28.142.196:2379,https://127.0.0.1:2379'
max-snapshots: 3
max-wals: 5
cors:
initial-advertise-peer-urls: 'https://172.28.142.196:2380'
advertise-client-urls: 'https://172.28.142.196:2379'
discovery:
discovery-fallback: 'proxy'
discovery-proxy:
discovery-srv:
initial-cluster: 'test-nec-k8s-m1=https://172.28.104.112:2380,test-nec-k8s-m2=https://172.28.142.192:2380,test-nec-k8s-m3=https://172.28.142.196:2380'
initial-cluster-token: 'etcd-k8s-cluster'
initial-cluster-state: 'new'
strict-reconfig-check: false
enable-v2: true
enable-pprof: true
proxy: 'off'
proxy-failure-wait: 5000
proxy-refresh-interval: 30000
proxy-dial-timeout: 1000
proxy-write-timeout: 5000
proxy-read-timeout: 0
client-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
peer-transport-security:
  cert-file: '/opt/kube/ssl/etcd.pem'
  key-file: '/opt/kube/ssl/etcd-key.pem'
  peer-client-cert-auth: true
  trusted-ca-file: '/opt/kube/ssl/ca.pem'
  auto-tls: true
debug: false
log-package-levels:
log-outputs: [default]
force-new-cluster: false

-------------------------------------------------
vi /usr/lib/systemd/system/etcd.service

[Unit]
Description=Etcd Service
Documentation=https://coreos.com/etcd/docs/latest/
After=network.target

[Service]
Type=notify
ExecStart=/opt/kube/bin/etcd --config-file=/etc/etcd/etcd.config.yml
Restart=on-failure
RestartSec=10
LimitNOFILE=65536

[Install]
WantedBy=multi-user.target
Alias=etcd3.service




chmod  700 /var/lib/etcd  -R
systemctl daemon-reload
systemctl enable --now etcd



##查看状态
etcdctl --endpoints=192.168.217.128:2379 \
--cacert=/opt/kube/ssl/ca.pem \
--cert=/opt/kube/ssl/etcd.pem \
--key=/opt/kube/ssl/etcd-key.pem  \
endpoint status --write-out=table
```



# 7、高可用软件安装

## 7.1、安装Haproxy

**仅需要在Master中安装**

```bash
yum -y install haproxy
vi /etc/haproxy/haproxy.cfg
global
        log /dev/log    local0
        log /dev/log    local1 notice
        chroot /var/lib/haproxy
        stats socket /run/admin.sock mode 660 level admin
        stats timeout 30s
        user root
        group root
        daemon
        nbproc 1

defaults
        log     global
        timeout connect 5000
        timeout client  10m
        timeout server  10m

listen kube-api
        bind 0.0.0.0:8443
        mode tcp
        option tcplog
        balance roundrobin
        server test-nec-k8s-m1 172.28.104.112:6443 check inter 2000 fall 2 rise 2 weight 1
        server test-nec-k8s-m1 172.28.142.192:6443 check inter 2000 fall 2 rise 2 weight 1
        server test-nec-k8s-m1 172.28.142.196:6443 check inter 2000 fall 2 rise 2 weight 1


systemctl enable --now haproxy
netstat  -utpln |grep 8443
tcp        0      0 0.0.0.0:8443            0.0.0.0:*               LISTEN      12787/haproxy
```



## 7.2、Keepalived



```bash

yum -y install keepalved

--------------------
# Master-1
vi /etc/keepalived/keepalived.conf
global_defs {
    router_id lb-master-172.28.104.112
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -60
}

vrrp_instance VI-kube-master {
    state MASTER
    priority 120
    unicast_src_ip 172.28.104.112
    unicast_peer {
        172.28.142.192
        172.28.142.196
    }
    dont_track_primary
    interface ens192
    virtual_router_id 110
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        172.28.142.159
    }
}

# Master-2
vi /etc/keepalived/keepalived.conf
global_defs {
    router_id lb-backup-172.28.142.192
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -60
}

vrrp_instance VI-kube-master {
    state BACKUP
    priority 119
    unicast_src_ip 172.28.142.192
    unicast_peer {
        172.28.104.112
        172.28.142.196
    }
    dont_track_primary
    interface ens192
    virtual_router_id 110
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        172.28.142.159
    }
}

# Master-3
vi /etc/keepalived/keepalived.conf
global_defs {
    router_id lb-backup-172.28.142.196
}

vrrp_script check-haproxy {
    script "killall -0 haproxy"
    interval 5
    weight -60
}

vrrp_instance VI-kube-master {
    state BACKUP
    priority 114
    unicast_src_ip 172.28.142.196
    unicast_peer {
        172.28.104.112
        172.28.142.192
    }
    dont_track_primary
    interface ens192
    virtual_router_id 110
    advert_int 3
    track_script {
        check-haproxy
    }
    virtual_ipaddress {
        172.28.142.159
    }
}


-------------------------------

systemctl enable --now keepalived


```



# 8、安装Kubernetes组件

## 8.1、kube-apiserver

**仅需要在Master中安装**

```bash
ls /opt/kube/bin/kube-apiserver
mkdir /var/log/kubernetes

------------------------------------------
# Master-1
vi /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=192.168.217.128 \
      --service-cluster-ip-range=10.250.0.0/16  \
      --service-node-port-range=30000-40000  \
      --etcd-servers=https://192.168.217.128:2379 \
      --etcd-cafile=/opt/kube/ssl/ca.pem  \
      --etcd-certfile=/opt/kube/ssl/etcd.pem  \
      --etcd-keyfile=/opt/kube/ssl/etcd-key.pem  \
      --client-ca-file=/opt/kube/ssl/ca.pem  \
      --tls-cert-file=/opt/kube/ssl/kubernetes.pem  \
      --tls-private-key-file=/opt/kube/ssl/kubernetes-key.pem  \
      --kubelet-client-certificate=/opt/kube/ssl/kubernetes.pem  \
      --kubelet-client-key=/opt/kube/ssl/kubernetes-key.pem  \
      --service-account-key-file=/opt/kube/ssl/sa.pub  \
      --service-account-signing-key-file=/opt/kube/ssl/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/opt/kube/ssl/ca.pem  \
      --proxy-client-cert-file=/opt/kube/ssl/aggregator-proxy.pem  \
      --proxy-client-key-file=/opt/kube/ssl/aggregator-proxy-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target


# Master-2

vi /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=172.28.142.192 \
      --service-cluster-ip-range=10.250.0.0/16  \
      --service-node-port-range=30000-40000  \
      --etcd-servers=https://172.28.104.112:2379,https://172.28.142.192:2379,https://172.28.142.196:2379 \
      --etcd-cafile=/opt/kube/ssl/ca.pem  \
      --etcd-certfile=/opt/kube/ssl/etcd.pem  \
      --etcd-keyfile=/opt/kube/ssl/etcd-key.pem  \
      --client-ca-file=/opt/kube/ssl/ca.pem  \
      --tls-cert-file=/opt/kube/ssl/kubernetes.pem  \
      --tls-private-key-file=/opt/kube/ssl/kubernetes-key.pem  \
      --kubelet-client-certificate=/opt/kube/ssl/kubernetes.pem  \
      --kubelet-client-key=/opt/kube/ssl/kubernetes-key.pem  \
      --service-account-key-file=/opt/kube/ssl/sa.pub  \
      --service-account-signing-key-file=/opt/kube/ssl/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/opt/kube/ssl/ca.pem  \
      --proxy-client-cert-file=/opt/kube/ssl/aggregator-proxy.pem  \
      --proxy-client-key-file=/opt/kube/ssl/aggregator-proxy-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

# Master-3

vi /etc/systemd/system/kube-apiserver.service
[Unit]
Description=Kubernetes API Server
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-apiserver \
      --v=2  \
      --logtostderr=true  \
      --allow-privileged=true  \
      --bind-address=0.0.0.0  \
      --secure-port=6443  \
      --insecure-port=0  \
      --advertise-address=172.28.142.196 \
      --service-cluster-ip-range=10.250.0.0/16  \
      --service-node-port-range=30000-40000  \
      --etcd-servers=https://172.28.104.112:2379,https://172.28.142.192:2379,https://172.28.142.196:2379 \
      --etcd-cafile=/opt/kube/ssl/ca.pem  \
      --etcd-certfile=/opt/kube/ssl/etcd.pem  \
      --etcd-keyfile=/opt/kube/ssl/etcd-key.pem  \
      --client-ca-file=/opt/kube/ssl/ca.pem  \
      --tls-cert-file=/opt/kube/ssl/kubernetes.pem  \
      --tls-private-key-file=/opt/kube/ssl/kubernetes-key.pem  \
      --kubelet-client-certificate=/opt/kube/ssl/kubernetes.pem  \
      --kubelet-client-key=/opt/kube/ssl/kubernetes-key.pem  \
      --service-account-key-file=/opt/kube/ssl/sa.pub  \
      --service-account-signing-key-file=/opt/kube/ssl/sa.key  \
      --service-account-issuer=https://kubernetes.default.svc.cluster.local \
      --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname  \
      --enable-admission-plugins=NamespaceLifecycle,LimitRanger,ServiceAccount,DefaultStorageClass,DefaultTolerationSeconds,NodeRestriction,ResourceQuota  \
      --authorization-mode=Node,RBAC  \
      --enable-bootstrap-token-auth=true  \
      --requestheader-client-ca-file=/opt/kube/ssl/ca.pem  \
      --proxy-client-cert-file=/opt/kube/ssl/aggregator-proxy.pem  \
      --proxy-client-key-file=/opt/kube/ssl/aggregator-proxy-key.pem  \
      --requestheader-allowed-names=aggregator  \
      --requestheader-group-headers=X-Remote-Group  \
      --requestheader-extra-headers-prefix=X-Remote-Extra-  \
      --requestheader-username-headers=X-Remote-User

Restart=on-failure
RestartSec=10s
LimitNOFILE=65535

[Install]
WantedBy=multi-user.target

-----------------------------------
systemctl enable --now kube-apiserver

```

## 8.2、kube-controller-manager

**仅需要在Master中安装**

```bash
ls /opt/kube/bin/kube-controller-manager

vi /etc/systemd/system/kube-controller-manager.service
[Unit]
Description=Kubernetes Controller Manager
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-controller-manager \
      --v=2 \
      --logtostderr=true \
      --address=127.0.0.1 \
      --root-ca-file=/opt/kube/ssl/ca.pem \
      --cluster-signing-cert-file=/opt/kube/ssl/ca.pem \
      --cluster-signing-key-file=/opt/kube/ssl/ca-key.pem \
      --service-account-private-key-file=/opt/kube/ssl/sa.key \
      --kubeconfig=/etc/kubernetes/controller-manager.kubeconfig \
      --leader-elect=true \
      --use-service-account-credentials=true \
      --node-monitor-grace-period=40s \
      --node-monitor-period=5s \
      --pod-eviction-timeout=2m0s \
      --controllers=*,bootstrapsigner,tokencleaner \
      --allocate-node-cidrs=true \
      --cluster-cidr=10.244.0.0/16 \
      --requestheader-client-ca-file=/opt/kube/ssl/aggregator-proxy.pem \
      --node-cidr-mask-size=24

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

systemctl enable --now kube-controller-manager
```

## 8.3、kube-scheduler

**仅需要在Master中安装**

```bash
ls /opt/kube/bin/kube-scheduler

vi /etc/systemd/system/kube-scheduler.service
[Unit]
Description=Kubernetes Scheduler
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-scheduler \
      --v=2 \
      --logtostderr=true \
      --address=127.0.0.1 \
      --leader-elect=true \
      --kubeconfig=/etc/kubernetes/scheduler.kubeconfig

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

systemctl enable --now kube-scheduler
```

## 8.4、kubectl

**所有节点安装**

```bash
##仅需将上面生成的管理员用户的kubeconfig文件拷贝至不同主机的~/.kube/config 即可
mkdir  ~/.kube
cp /etc/kubernetes/admin.kubeconfig ~/.kube/config

kubectl  get cs
NAME                 STATUS    MESSAGE             ERROR
scheduler            Healthy   ok
controller-manager   Healthy   ok
etcd-0               Healthy   {"health":"true"}
```

## 8.5、TLS Bootstrapping

**仅需在Master中找一台节点做即可**

```bash
kubectl config set-cluster Kubernetes \
--certificate-authority=/opt/kube/ssl/ca.pem \
--embed-certs=true \
--server=https://192.168.217.128:6443 \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-credentials tls-bootstrap-token-user \
--token=c8ad9c.2e4d610cf3e7426e \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config set-context tls-bootstrap-token-user@kubernetes \
--cluster=Kubernetes \
--user=tls-bootstrap-token-user \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig

kubectl config use-context tls-bootstrap-token-user@kubernetes \
--kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig


##将生成的文件复制到所有节点
for i in 192 196 202 209 219 ;do scp  /etc/kubernetes/bootstrap-kubelet.kubeconfig root@172.28.142.$i:/etc/kubernetes/ ;done

```

```bash
vi /etc/kubernetes/bootstrap.secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: bootstrap-token-c8ad9c
  namespace: kube-system
type: bootstrap.kubernetes.io/token
stringData:
  description: "The default bootstrap token generated by 'kubelet '."
  token-id: c8ad9c
  token-secret: 2e4d610cf3e7426e
  usage-bootstrap-authentication: "true"
  usage-bootstrap-signing: "true"
  auth-extra-groups:  system:bootstrappers:default-node-token,system:bootstrappers:worker,system:bootstrappers:ingress

---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: kubelet-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:node-bootstrapper
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-bootstrap
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:nodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:bootstrappers:default-node-token
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: node-autoapprove-certificate-rotation
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:certificates.k8s.io:certificatesigningrequests:selfnodeclient
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: Group
  name: system:nodes
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
  name: kube-apiserver
---
apiVersion: rbac.authorization.k8s.io/v1
kind: ClusterRoleBinding
metadata:
  name: system:kubernetes
  namespace: ""
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: system:kube-apiserver-to-kubelet
subjects:
- apiGroup: rbac.authorization.k8s.io
  kind: User
  name: kubernetes


kubectl  create -f /etc/kubernetes/bootstrap.secret.yaml
```



## 8.6、kubelet

**所有节点安装**

```bash
ls /opt/kube/bin/kubelet
mkdir -p /var/lib/kubelet /var/log/kubernetes \
/etc/systemd/system/kubelet.service.d /etc/kubernetes/manifests/

vi /etc/systemd/system/kubelet.service
[Unit]
Description=Kubernetes Kubelet
Documentation=https://github.com/kubernetes/kubernetes
After=containerd.service
Requires=containerd.service

[Service]
ExecStart=/opt/kube/bin/kubelet

Restart=always
StartLimitInterval=0
RestartSec=10

[Install]
WantedBy=multi-user.target




vi  /etc/systemd/system/kubelet.service.d/10-kubelet.conf
[Service]
Environment="KUBELET_KUBECONFIG_ARGS=--bootstrap-kubeconfig=/etc/kubernetes/bootstrap-kubelet.kubeconfig --kubeconfig=/etc/kubernetes/kubelet.kubeconfig"
Environment="KUBELET_SYSTEM_ARGS=--network-plugin=cni --cni-conf-dir=/etc/cni/net.d --cni-bin-dir=/opt/cni/bin"
Environment="KUBELET_CONFIG_ARGS=--config=/etc/kubernetes/kubelet-conf.yml --pod-infra-container-image=harbor.dev-dms-nec.com/kube-system/pause-amd64:3.1"
Environment="KUBELET_CONTAINERD_ARGS=--container-runtime=remote  --container-runtime-endpoint=unix:///run/containerd/containerd.sock"
Environment="KUBELET_EXTRA_ARGS=--node-labels=node.kubernetes.io/node='' --tls-cipher-suites=TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256,TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384  --image-pull-progress-deadline=30m "
ExecStart=
ExecStart=/opt/kube/bin/kubelet $KUBELET_KUBECONFIG_ARGS $KUBELET_CONFIG_ARGS $KUBELET_SYSTEM_ARGS $KUBELET_EXTRA_ARGS $KUBELET_CONTAINERD_ARGS

vi /etc/kubernetes/kubelet-conf.yml
apiVersion: kubelet.config.k8s.io/v1beta1
kind: KubeletConfiguration
address: 0.0.0.0
port: 10250
readOnlyPort: 10255
authentication:
  anonymous:
    enabled: false
  webhook:
    cacheTTL: 2m0s
    enabled: true
  x509:
    clientCAFile: /opt/kube/ssl/ca.pem
authorization:
  mode: Webhook
  webhook:
    cacheAuthorizedTTL: 5m0s
    cacheUnauthorizedTTL: 30s
cgroupDriver: systemd
cgroupsPerQOS: true
clusterDNS:
- 10.250.0.10
clusterDomain: cluster.local
containerLogMaxFiles: 5
containerLogMaxSize: 10Mi
contentType: application/vnd.kubernetes.protobuf
cpuCFSQuota: true
cpuManagerPolicy: none
cpuManagerReconcilePeriod: 10s
enableControllerAttachDetach: true
enableDebuggingHandlers: true
enforceNodeAllocatable:
- pods
eventBurst: 10
eventRecordQPS: 5
evictionHard:
  imagefs.available: 15%
  memory.available: 100Mi
  nodefs.available: 10%
  nodefs.inodesFree: 5%
evictionPressureTransitionPeriod: 5m0s
failSwapOn: true
fileCheckFrequency: 20s
hairpinMode: promiscuous-bridge
healthzBindAddress: 127.0.0.1
healthzPort: 10248
httpCheckFrequency: 20s
imageGCHighThresholdPercent: 85
imageGCLowThresholdPercent: 80
imageMinimumGCAge: 2m0s
iptablesDropBit: 15
iptablesMasqueradeBit: 14
kubeAPIBurst: 10
kubeAPIQPS: 5
makeIPTablesUtilChains: true
maxOpenFiles: 1000000
maxPods: 110
nodeStatusUpdateFrequency: 10s
oomScoreAdj: -999
podPidsLimit: -1
registryBurst: 10
registryPullQPS: 5
resolvConf: /etc/resolv.conf
rotateCertificates: true
runtimeRequestTimeout: 2m0s
serializeImagePulls: true
staticPodPath: /etc/kubernetes/manifests
streamingConnectionIdleTimeout: 4h0m0s
syncFrequency: 1m0s
volumeStatsAggPeriod: 1m0s
allowedUnsafeSysctls:
  - "net.core*"
  - "net.ipv4.*"
kubeReserved:
  cpu: "1000m"
  memory: "1024m"
systemReserved:
  cpu: "1000m"
  memory: "1024m"
```

```bash
##在Master中将所需证书分发至所有节点的对应目录下
/opt/kube/ssl/aggregator-proxy*
/opt/kube/ssl/kube-proxy*
/opt/kube/ssl/ca*

/etc/kubernetes/bootstrap-kubelet.kubeconfig
/etc/kubernetes/kube-proxy.kubeconfig
```

```bash
systemctl enable --now kubelet
kubectl  get node 
NAME              STATUS     ROLES    AGE   VERSION
test-nec-k8s-m1   NotReady   <none>   35s   v1.23.6
test-nec-k8s-m2   NotReady   <none>   36s   v1.23.6
test-nec-k8s-m3   NotReady   <none>   35s   v1.23.6
test-nec-k8s-n1   NotReady   <none>   35s   v1.23.6
test-nec-k8s-n2   NotReady   <none>   35s   v1.23.6
test-nec-k8s-n3   NotReady   <none>   35s   v1.23.6
```

## 8.7、kube-proxy

```bash
ls /opt/kube/bin/kube-proxy

vi /etc/kubernetes/kube-proxy.conf
apiVersion: kubeproxy.config.k8s.io/v1alpha1
bindAddress: 0.0.0.0
clientConnection:
  acceptContentTypes: ""
  burst: 10
  contentType: application/vnd.kubernetes.protobuf
  kubeconfig: /etc/kubernetes/kube-proxy.kubeconfig
  qps: 5
clusterCIDR: 10.244.0.0/16
configSyncPeriod: 15m0s
conntrack:
  max: null
  maxPerCore: 32768
  min: 131072
  tcpCloseWaitTimeout: 1h0m0s
  tcpEstablishedTimeout: 24h0m0s
enableProfiling: false
healthzBindAddress: 0.0.0.0:10256
hostnameOverride: ""
iptables:
  masqueradeAll: false
  masqueradeBit: 14
  minSyncPeriod: 0s
  syncPeriod: 30s
ipvs:
  masqueradeAll: true
  minSyncPeriod: 5s
  scheduler: "rr"
  syncPeriod: 30s
kind: KubeProxyConfiguration
metricsBindAddress: 127.0.0.1:10249
mode: "ipvs"
nodePortAddresses: null
oomScoreAdj: -999
portRange: ""
udpIdleTimeout: 250ms

vi /etc/systemd/system/kube-proxy.service
[Unit]
Description=Kubernetes Kube Proxy
Documentation=https://github.com/kubernetes/kubernetes
After=network.target

[Service]
ExecStart=/opt/kube/bin/kube-proxy \
  --config=/etc/kubernetes/kube-proxy.conf \
  --v=2

Restart=always
RestartSec=10s

[Install]
WantedBy=multi-user.target

systemctl enable --now kube-proxy
```

## 8.8、Calico

**tmci-k8s-client 这台机器**

```bash
mkdir /opt/software/
cd /opt/software/
wget https://github.com/projectcalico/calico/releases/download/v3.22.2/release-v3.22.2.tgz
tar zxf release-v3.22.2.tgz
cp -r /opt/software/release-v3.22.2/bin/calicoctl /usr/bin/
cd release-v3.22.2/k8s-manifests
cp ../bin/calicoctl-linux-amd64  /opt/kube/bin/calicoctl
vim calico-typha.yaml
.....
            - name: CALICO_IPV4POOL_CIDR
              value: "10.244.0.0/16"
.....

kubectl  create -f calico-typha.yaml
kubectl  get nodes
NAME              STATUS   ROLES    AGE   VERSION
test-nec-k8s-m1   Ready    <none>   13m   v1.23.6
test-nec-k8s-m2   Ready    <none>   13m   v1.23.6
test-nec-k8s-m3   Ready    <none>   13m   v1.23.6
test-nec-k8s-n1   Ready    <none>   13m   v1.23.6
test-nec-k8s-n2   Ready    <none>   13m   v1.23.6
test-nec-k8s-n3   Ready    <none>   13m   v1.23.6


mkdir /etc/calico/
vi /etc/calico/calicoctl.cfg
apiVersion: projectcalico.org/v3
kind: CalicoAPIConfig
metadata:
spec:
  datastoreType: "kubernetes"
  kubeconfig: "/root/.kube/config"
  
calicoctl get ippool
NAME                  CIDR            SELECTOR
default-ipv4-ippool   10.244.0.0/16   all()

calicoctl  get nodes -owide
NAME              ASN       IPV4                IPV6
test-nec-k8s-m1   (64512)   172.28.104.112/25
test-nec-k8s-m2   (64512)   172.28.142.192/25
test-nec-k8s-m3   (64512)   172.28.142.196/25
test-nec-k8s-n1   (64512)   172.28.104.113/25
test-nec-k8s-n2   (64512)   172.28.104.114/25
test-nec-k8s-n3   (64512)   172.28.142.219/25


##如果需要将Calico的镜像导入到Harbor中，需要进入到 /opt/software/release-v3.22.2/images 下手动导入然后推送，修改calico-typha.yaml替换image
##需要先在Harbor中创建一个名为calico的项目
================================================
cd /opt/software/release-v3.22.2/images/
for i in `ls` ;do docker load -i $i ;done

for i in `docker images |grep calico |awk '{print $1":"$2}'`;do \
docker tag $i test-harbor.nec-k8s.com.cn/$i && \
docker push test-harbor.nec-k8s.com.cn/$i ;done

##修改对应image地址
vi /opt/software/release-v3.22.2/k8s-manifests/calico.yaml
:%s/image: docker.io/image: test-harbor.nec-k8s.com.cn/g
:wq


kubectl  create -f /opt/software/release-v3.22.2/k8s-manifests/calico.yaml

================================================

```

## 8.9、CoreDNS

```bash
cd /opt/software/
git clone https://github.com/coredns/deployment.git
cd deployment/kubernetes
./deploy.sh -s -i 10.250.0.10 > coredns.yaml
vi coredns.yaml	
	image: coredns/coredns:1.8.6
	
kubectl  create -f coredns.yaml
kubectl  -n kube-system get pod  |grep coredns
coredns-7b6c7877df-5rb5p                   1/1     Running   0          15s
```

## 8.10、Metrics-Server

```bash
cd /opt/software/
wget https://github.com/kubernetes-sigs/metrics-server/releases/download/v0.5.2/components.yaml
vi components.yaml
##修改成如下
...
      - args:
        - --cert-dir=/tmp
        - --secure-port=4443
        - --kubelet-preferred-address-types=InternalIP,ExternalIP,Hostname
        - --kubelet-use-node-status-port
        - --metric-resolution=15s
        - --kubelet-insecure-tls
        - --requestheader-client-ca-file=/opt/kube/ssl/aggregator-proxy.pem
        - --requestheader-username-headers=X-Remote-User
        - --requestheader-group-headers=X-Remote-Group
        - --requestheader-extra-headers-prefix=X-Remote-Extra-
        image: registry.cn-hangzhou.aliyuncs.com/meetfate/metrics-server:v0.5.2
...
        volumeMounts:
        - mountPath: /tmp
          name: tmp-dir
        - name: ca-ssl
          mountPath: /opt/kube/ssl
...
      volumes:
      - emptyDir: {}
        name: tmp-dir
      - name: ca-ssl
        hostPath:
          path: /opt/kube/ssl
          
kubectl  create -f components.yaml
kubectl  -n kube-system get pod  |grep metrics
metrics-server-7c49b77795-2vchc            1/1     Running   0          42s

kubectl  get apiservices.apiregistration.k8s.io  |grep metrics
v1beta1.metrics.k8s.io                 kube-system/metrics-server   True        5m1s

kubectl  top nodes
NAME              CPU(cores)   CPU%   MEMORY(bytes)   MEMORY%
test-nec-k8s-m1   332m         8%     1460Mi          18%
test-nec-k8s-m2   148m         3%     1581Mi          20%
test-nec-k8s-m3   159m         3%     1429Mi          18%
test-nec-k8s-n1   63m          1%     1090Mi          13%
test-nec-k8s-n2   129m         3%     692Mi           8%
test-nec-k8s-n3   66m          1%     743Mi           9%


kubectl  top pods -n kube-system
calico-kube-controllers-7d4687698-sgtqh   8m           29Mi
calico-node-2lh47                         30m          165Mi
calico-node-2ms8k                         51m          167Mi
calico-node-5twlh                         30m          167Mi
calico-node-6zgbl                         20m          229Mi
calico-node-9lww8                         63m          174Mi
calico-node-hn9zs                         23m          168Mi
calico-typha-7c9b585485-pfx4q             10m          37Mi
coredns-75995546d-dblck                   2m           20Mi
metrics-server-7f7f67cc84-kb75x           3m           24Mi
```

## 8.11、Ingress

```bash
cd /opt/software/
wget https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.1.2/deploy/static/provider/baremetal/deploy.yaml

vi deploy.yaml
#修改镜像地址
registry.cn-hangzhou.aliyuncs.com/meetfate/ingress-nginx-controller:v1.1.2
registry.cn-hangzhou.aliyuncs.com/meetfate/ingress-nginx-kube-webhook-certgen:v1.1.1

#修改运行模式
apiVersion: apps/v1
kind: DaemonSet ##原 Deployment 修改为DaemonSet
metadata:
  name: ingress-nginx-controller
  namespace: ingress-nginx
spec:
  selector:
    matchLabels:
      ...
  template:
    metadata:
      labels:
		...
    spec:
      hostNetwork: true  ##新增
		...
	      dnsPolicy: ClusterFirst
      nodeSelector:
        ingress: "true" ## 原  kubernetes.io/os: linux 修改为 ingress: "true"

kubectl  label nodes test-nec-k8s-n1 ingress=true
kubectl  label nodes test-nec-k8s-n2 ingress=true
kubectl  label nodes test-nec-k8s-n3 ingress=true

kubectl  create -f deploy.yaml
kubectl  -n ingress-nginx  get pod
NAME                                   READY   STATUS      RESTARTS   AGE
ingress-nginx-admission-create-b98qg   0/1     Completed   0          46s
ingress-nginx-admission-patch-t2dsr    0/1     Completed   1          46s
ingress-nginx-controller-2r484         1/1     Running     0          19s
ingress-nginx-controller-gkszt         1/1     Running     0          23s
ingress-nginx-controller-svqd6         1/1     Running     0          21s

```

```bash
#设置Master中的Haproxy
vim /etc/haproxy/haproxy.cfg
...
listen ingress-http
        bind 0.0.0.0:80
        mode tcp
        option tcplog
        balance roundrobin
        server test-nec-k8s-n1 172.28.104.113:80 check inter 2000 fall 2 rise 2 weight 1
        server test-nec-k8s-n1 172.28.104.114:80 check inter 2000 fall 2 rise 2 weight 1
        server test-nec-k8s-n1 172.28.142.219:80 check inter 2000 fall 2 rise 2 weight 1

listen ingress-https
        bind 0.0.0.0:443
        mode tcp
        option tcplog
        balance roundrobin
        server tmci-k8s-node-01 172.28.104.113:443 check inter 2000 fall 2 rise 2 weight 1
        server tmci-k8s-node-01 172.28.104.114:443 check inter 2000 fall 2 rise 2 weight 1
        server tmci-k8s-node-01 172.28.142.219:443 check inter 2000 fall 2 rise 2 weight 1

systemctl  reload haproxy


```

## 8.12 设置节点标签

```bash
kubectl label nodes test-nec-k8s-m1 kubernetes.io/role=master
kubectl label nodes test-nec-k8s-m2 kubernetes.io/role=master
kubectl label nodes test-nec-k8s-m3 kubernetes.io/role=master
kubectl label nodes test-nec-k8s-n1 kubernetes.io/role=node
kubectl label nodes test-nec-k8s-n2 kubernetes.io/role=node
kubectl label nodes test-nec-k8s-n3 kubernetes.io/role=node

kubectl  get nodes
NAME              STATUS   ROLES    AGE   VERSION
test-nec-k8s-m1   Ready    master   46h   v1.23.6
test-nec-k8s-m2   Ready    master   46h   v1.23.6
test-nec-k8s-m3   Ready    master   46h   v1.23.6
test-nec-k8s-n1   Ready    node     46h   v1.23.6
test-nec-k8s-n2   Ready    node     46h   v1.23.6
test-nec-k8s-n3   Ready    node     46h   v1.23.6

```



# 9、设置NFS存储类



```bash
# 需先安装NFS集群！！
# 在Master 1 中执行
cd /opt/software/
git clone https://github.com/kubernetes-sigs/nfs-subdir-external-provisioner.git
cd nfs-subdir-external-provisioner/deploy
vim deployment.yaml
...
	spec:
      serviceAccountName: nfs-client-provisioner
      containers:
        - name: nfs-client-provisioner
          image: registry.cn-hangzhou.aliyuncs.com/meetfate/nfs-subdir-external-provisioner:v4.0.2
...
          env:
            - name: PROVISIONER_NAME
              value: k8s-sigs.io/nfs-subdir-external-provisioner
            - name: NFS_SERVER
              value: 172.28.142.131
            - name: NFS_PATH
              value: /data
      volumes:
        - name: nfs-client-root
          nfs:
            server: 172.28.142.131
            path: /data
            
vim class.yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  annotations:
    storageclass.kubernetes.io/is-default-class: "true"
  name: nfs
provisioner: k8s-sigs.io/nfs-subdir-external-provisioner # or choose another name, must match deployment's env PROVISIONER_NAME'
parameters:
  archiveOnDelete: "false"
  reclaimPolicy: Delete


kubectl  create -f rbac.yaml
kubectl  create -f class.yaml
kubectl  create -f deployment.yaml

kubectl  get pod  |grep nfs
nfs-client-provisioner-5c96fc5594-77mxs   1/1     Running   0          5s

vim test-claim.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test-claim
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 1Mi

kubectl  create -f test-claim.yaml

kubectl  get pvc
NAME         STATUS   VOLUME                                     CAPACITY   ACCESS MODES   STORAGECLASS   AGE
test-claim   Bound    pvc-1a43089b-d4a1-4551-95bf-b8e0749776c8   1Mi        RWX            nfs            <invalid>


```



# 10、测试

```bash
![微信图片_20220306222202](C:\Users\xmtao\Desktop\微信图片_20220306222202.png)vim test.yaml
kind: PersistentVolumeClaim
apiVersion: v1
metadata:
  name: test
spec:
  storageClassName: nfs
  accessModes:
    - ReadWriteMany
  resources:
    requests:
      storage: 50Mi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  labels:
    app: test
  name: test
spec:
  replicas: 1
  selector:
    matchLabels:
      app: test
  template:
    metadata:
      labels:
        app: test
    spec:
      affinity:
        podAntiAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
          - podAffinityTerm:
              labelSelector:
                matchExpressions:
                - key: app
                  operator: In
                  values:
                  - test
              topologyKey: kubernetes.io/hostname
            weight: 100
      containers:
      - image: test-harbor.nec-k8s.com.cn/public/nginx:alpine
        imagePullPolicy: Always
        name: test
        ports:
        - containerPort: 80
          name: test
          protocol: TCP
        resources:
          limits:
            cpu: "1"
            memory: 1Gi
          requests:
            cpu: "100m"
            memory: 300Mi
        volumeMounts:
        - mountPath: /etc/localtime
          name: zoneinfo
          readOnly: true
        - mountPath: /usr/share/nginx/html/
          name: test
      volumes:
      - hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
          type: ""
        name: zoneinfo
      - name: test
        persistentVolumeClaim:
          claimName: test
      nodeSelector:
        kubernetes.io/role: node
---
apiVersion: v1
kind: Service
metadata:
  name: test
spec:
  ports:
  - port: 80
    targetPort: 80
  selector:
    app: test
  type: NodePort

kubectl  create -f test.yaml

kubectl  get pod
NAME                                      READY   STATUS    RESTARTS   AGE
nfs-client-provisioner-5c96fc5594-77mxs   1/1     Running   0          6m9s
test-6b7c56497f-7m8b9                     1/1     Running   0          2m28s



kubectl  get svc
NAME         TYPE        CLUSTER-IP      EXTERNAL-IP   PORT(S)        AGE
kubernetes   ClusterIP   10.250.0.1      <none>        443/TCP        46h
test         NodePort    10.250.178.41   <none>        80:34107/TCP   2m41s


vim index.html
Hello Word!

kubectl  cp ./index.html test-6b7c56497f-7m8b9:/usr/share/nginx/html/index.html


curl 172.28.142.159:34107
Hello Word!

vim test-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: test
  namespace: default
spec:
  ingressClassName: nginx
  rules:
  - host: test.apps.nec-k8s.com.cn
    http:
      paths:
      - backend:
          service:
            name: test
            port:
              number: 80
        path: /
        pathType: ImplementationSpecific

kubectl  create -f test-ingress.yaml

kubectl  get ingress
NAME   CLASS   HOSTS                      ADDRESS   PORTS   AGE
test   nginx   test.apps.nec-k8s.com.cn             80      12s

echo "172.28.142.159 test.apps.nec-k8s.com.cn" >>/etc/hosts

curl test.apps.nec-k8s.com.cn
Hello Word!
```
