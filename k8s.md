# 安装kubernetes
## 校验mac地址
ip link 或 ifconfig -a
## 禁用交换分区
如果 kubelet 未被正确配置使用交换分区，则你必须禁用交换分区。 例如，sudo swapoff -a 将暂时禁用交换分区。要使此更改在重启后保持不变，请确保在如 /etc/fstab、systemd.swap 等配置文件中禁用交换分区，具体取决于你的系统如何配置。
## 校验端口占用
https://v1-29.docs.kubernetes.io/zh-cn/docs/reference/networking/ports-and-protocols/
## 安装CRIO
https://github.com/cri-o/packaging/blob/main/README.md

swapoff -a 临时关闭交换分区
modprobe br_netfilter  Linux 桥接网络
sysctl -w net.ipv4.ip_forward=1  允许ip4转发
## 注意cgroup驱动，1.29中crio和kubelet默认都为systemd，所以不用管
### 启动master也就是控制平面
```
kubeadm init \
  --apiserver-advertise-address=192.168.64.11 \
  --cri-socket=unix:///var/run/crio/crio.sock \
  --pod-network-cidr=10.244.0.0/16
```
### 将worker节点加入到master中
master开放给worker的端口
```
sudo ufw allow from 192.168.64.12 to any port 2379
sudo ufw allow from 192.168.64.12 to any port 2380
sudo ufw allow from 192.168.64.12 to any port 10250
sudo ufw allow from 192.168.64.12 to any port 10259
sudo ufw allow from 192.168.64.12 to any port 10257
sudo ufw allow from 192.168.64.12 to any port 6443

sudo ufw allow from 192.168.64.13 to any port 2379
sudo ufw allow from 192.168.64.13 to any port 2380
sudo ufw allow from 192.168.64.13 to any port 10250
sudo ufw allow from 192.168.64.13 to any port 10259
sudo ufw allow from 192.168.64.13 to any port 10257
sudo ufw allow from 192.168.64.13 to any port 6443
```
worker开放给master的端口也是10250，然后开放一个30080用于路由，30080可选
```
sudo ufw allow from 192.168.64.11 to any port 10250
sudo ufw allow 30080/tcp
```

随后加入
```
kubeadm join 192.168.64.11:6443 --token f0wc7w.rhcmfrb6ic31l3ru \
  --cri-socket=unix:///var/run/crio/crio.sock \
	--discovery-token-ca-cert-hash sha256:5e571494370b4df5b452ba1066f5a4cfdfc788bc0796bf0179291d369d418053
```
### 安装网络工具
kubectl apply -f https://github.com/flannel-io/flannel/releases/latest/download/kube-flannel.yml

### 安装界面kuboard
https://kuboard.cn/


### 解决问题
- Failed to create pod sandbox: rpc error: code = Unknown desc = failed to create pod network sandbox k8s_kuboard-v3-leader_kuboard_4df35a2b48e3b63501442160ea34bc4a_0(cad1e5157a73c3e67a36392d78207b771d760af989dfdb7859c9b4129c476e45): error adding pod kuboard_kuboard-v3-leader to CNI network "cbr0": plugin type="flannel" failed (add): failed to delegate add: failed to set bridge addr: "cni0" already has an IP address different from 10.244.0.1/24
解决方法
- 删除cni0网卡，自动重建
```
sudo ifconfig cni0 down    
sudo ip link delete cni0
```

### 设置私服
vim /etc/containers/registries.conf
https://github.com/containers/image/blob/main/docs/containers-registries.conf.5.md
```
[[registry]]
prefix = "10.1.88.XX:5000"
insecure = true
location = "10.1.88.XX:5000"
```

### 配置ingress-nginx作为网关

https://kubernetes.github.io/ingress-nginx/deploy/
```
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/controller-v1.10.1/deploy/static/provider/cloud/deploy.yaml
```
### flannel插件需要互相能够访问8472端口udp
对于 master 主机（IP: 192.168.64.17）:
sudo ufw allow from 192.168.64.15 to any port 8472
sudo ufw allow from 192.168.64.16 to any port 8472
对于 worker1 主机（IP: 192.168.64.15）:
sudo ufw allow from 192.168.64.17 to any port 8472
sudo ufw allow from 192.168.64.16 to any port 8472
对于 worker2 主机（IP: 192.168.64.16）:
sudo ufw allow from 192.168.64.17 to any port 8472
sudo ufw allow from 192.168.64.15 to any port 8472


## 安装低版本的kubernetes

```
yum install -y dnf

cat <<EOF | sudo tee /etc/modules-load.d/k8s.conf
overlay
br_netfilter
EOF

sudo modprobe overlay
sudo modprobe br_netfilter

# 设置所需的 sysctl 参数，参数在重新启动后保持不变
cat <<EOF | sudo tee /etc/sysctl.d/k8s.conf
net.bridge.bridge-nf-call-iptables  = 1
net.bridge.bridge-nf-call-ip6tables = 1
net.ipv4.ip_forward                 = 1
EOF

# 应用 sysctl 参数而不重新启动
sudo sysctl --system

OS=CentOS_7
VERSION=1.25
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable.repo https://download.opensuse.org/repositories/devel:/kubic:/libcontainers:/stable/$OS/devel:kubic:libcontainers:stable.repo
curl -L -o /etc/yum.repos.d/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo https://download.opensuse.org/repositories/devel:kubic:libcontainers:stable:cri-o:$VERSION/$OS/devel:kubic:libcontainers:stable:cri-o:$VERSION.repo
dnf install -y cri-o

cat <<EOF | sudo tee /etc/yum.repos.d/kubernetes.repo
[kubernetes]
name=Kubernetes
baseurl=https://packages.cloud.google.com/yum/repos/kubernetes-el7-\$basearch
enabled=1
gpgcheck=1
gpgkey=https://packages.cloud.google.com/yum/doc/rpm-package-key.gpg
exclude=kubelet kubeadm kubectl
EOF

# 将 SELinux 设置为 permissive 模式（相当于将其禁用）
sudo setenforce 0
sudo sed -i 's/^SELINUX=enforcing$/SELINUX=permissive/' /etc/selinux/config

sudo dnf install -y kubelet kubeadm kubectl --disableexcludes=kubernetes

sudo systemctl enable --now kubelet
```
修改镜像
vim /etc/containers/registries.conf
```
[[registry]]
prefix = "registry.k8s.io"
 location = "k8s.dockerproxy.com"
```
systemctl restart crio
systemctl restart kubelet

初始化
```
kubeadm init   --apiserver-advertise-address=10.19.XX.XX   --cri-socket=unix:///var/run/crio/crio.sock   --pod-network-cidr=10.244.0.0/16 --image-repository='registry.cn-hangzhou.aliyuncs.com/google_containers'
```


