# K3s集群部署指南

## 1. 服务器准备
- k3s-master (主节点)
- k3s-node1
- k3s-node2
- k3s-node3

建议服务器配置：2cpu/3GIB/20G

需要确保服务器之间可以互相访问。

## 2. 配置k3s镜像
使用root账号分别登录所有服务器，并执行以下脚本：
```
apt install -y iptables
mkdir -p /etc/rancher/k3s/
cat >> /etc/rancher/k3s/registries.yaml << EOF
mirrors:
  docker.io:
    endpoint:
      - "https://docker.m.daocloud.io"

  gcr.io:
    endpoint:
      - "https://gcr.m.daocloud.io"

  quay.io:
    endpoint:
      - "https://quay.m.daocloud.io"

  registry.k8s.io:
    endpoint:
      - "https://k8s.m.daocloud.io"
EOF
```

## 3.部署主节点
使用root账号登录k3s-master, 执行以下脚本:
```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_EXEC="--disable=traefik" INSTALL_K3S_MIRROR=cn sh -
```

执行以下脚本获取集群令牌：
```
cat /var/lib/rancher/k3s/server/node-token
```
输出结果：
```
K105ea972e2d8ac1a0d07aa914399f5426ccc76cbfee014784a6b96be04a3f616e7::server:9a16639e03addec8ee029634dd7e40ea
```

准备Agent节点部署脚本：
```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://{主节点IP或域名}:6443 K3S_TOKEN={集群令牌} sh -
```
## 4.部署Agent节点
使用root账号分别登录k3s-node1、k3s-node2、k3s-node3, 并执行以下脚本:
```
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_MIRROR=cn K3S_URL=https://{主节点IP或域名}:6443 K3S_TOKEN={集群令牌} sh -
```

## 5.验证集群节点
使用root账号登录k3s-master, 并执行以下脚本:
```
kubectl get nodes -A -o wide
```

## 6. 授予普通用户访问集群权限
使用root账号登录k3s-master, 并执行以下脚本：
```
touch /etc/rancher/k3s/config.yaml

echo 'write-kubeconfig-mode: "0644"' >> /etc/rancher/k3s/config.yaml

systemctl restart k3s
```

## 7. 卸载内置Traefik Addon (跳过)
登录k3s-master, 执行以下脚本:
```
sudo echo "disable: traefik" >> /etc/rancher/k3s/config.yaml
sudo systemctl daemon-reload
sudo systemctl restart k3s
```
执行以下脚本，确认traefik addon 已经被卸载：
```
kubectl get addon -A
```

## 8.安装helm
登录k3s-master, 执行以下脚本:
```
curl -fsSL -o get_helm.sh https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3
chmod 700 get_helm.sh
./get_helm.sh

mkdir -p ~/.kube
ln -s /etc/rancher/k3s/k3s.yaml ~/.kube/config
```

## 9. 部署traefik
登录k3s-master, 按顺序执行以下操作.

添加helm仓库
```
helm repo add traefik https://traefik.github.io/charts
helm repo update
```
创建命名空间
```
kubectl create namespace traefik
```
创建自签名证书
```
openssl req -x509 -nodes -days 365 -newkey rsa:2048 \
  -keyout tls.key -out tls.crt \
  -subj "/CN=*.k3s.local"

kubectl create secret tls traefik-gateway-tls \
  --cert=tls.crt --key=tls.key \
  --namespace traefik
```
创建values.yaml配置文件
```

# Configure Network Ports and EntryPoints
# EntryPoints are the network listeners for incoming traffic.
ports:
  # Defines the HTTP entry point named 'web'
  web:
    port: 80
    nodePort: 30000
    # Instructs this entry point to redirect all traffic to the 'websecure' entry point
    http:
      redirections:
        entryPoint:
          to: websecure
          scheme: https
          permanent: true

  # Defines the HTTPS entry point named 'websecure'
  websecure:
    port: 443
    nodePort: 30001

# Enables the dashboard in Secure Mode
api:
  dashboard: true
  insecure: false

ingressRoute:
  dashboard:
    enabled: true
    matchRule: Host(`traefik.k3s.local`)
    entryPoints:
      - websecure

# We will route with Gateway API instead.
ingressClass:
  enabled: true

# Enable Gateway API Provider & Disables the KubernetesIngress provider
# Providers tell Traefik where to find routing configuration.
providers:
  kubernetesIngress:
    enabled: true
  kubernetesGateway:
    enabled: true

## Gateway Listeners
gateway:
  listeners:
    web:           # HTTP listener that matches entryPoint `web`
      port: 80
      protocol: HTTP
      namespacePolicy:
        from: All

    websecure:         # HTTPS listener that matches entryPoint `websecure`
      port: 443
      protocol: HTTPS  # TLS terminates inside Traefik
      namespacePolicy:
        from: All
      mode: Terminate
      certificateRefs:    
        - kind: Secret
          name: traefik-gateway-tls  # the Secret we created before the installation
          group: ""

# Enable Observability
log:
  level: INFO
# This enables access logs, outputting them to Traefik's standard output by default. The [Access Logs Documentation](https://doc.traefik.io/traefik/observability/access-logs/) covers formatting, filtering, and output options.
accessLog:
  enabled: true

# Enables Prometheus for Metrics
metrics:
  prometheus:
    enabled: true
```

部署traefik
```
kubectl apply -f https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.5.1/standard-install.yaml

helm install traefik traefik/traefik \
  --namespace traefik \
  --values values.yaml
```

可以使用游览器打开 https://dashboard.k3s.local/

## 10. 部署NFS Server
登录k3s-master, 执行以下脚本：
```
# 安装服务
sudo apt install nfs-kernel-server -y
# 创建共享目录
sudo mkdir -p /mnt/k3s-nfs
sudo chown nobody:nogroup /mnt/k3s-nfs
sudo chmod 770 /mnt/k3s-nfs
```
编辑 /etc/exports 文件, 添加下面内容：
```
/mnt/k3s-nfs  192.168.100.0/24(rw,sync,no_subtree_check,no_root_squash)
```

启动NFS Server, 执行以下脚本：
```
sudo exportfs -ra
sudo systemctl enable --now nfs-kernel-server
```

## 11. 安装NFS依赖包
登录所有服务器，执行以下脚本：
```
sudo apt install nfs-common
```

## 12. 部署nfs-subdir-external-provisioner
登录k3s-master, 执行以下脚本：
```
helm repo add nfs-subdir-external-provisioner https://kubernetes-sigs.github.io/nfs-subdir-external-provisioner/
helm repo update

helm install nfs-provisioner nfs-subdir-external-provisioner/nfs-subdir-external-provisioner \
  --namespace kube-system \
  --set nfs.server=k3s-master \
  --set nfs.path=/mnt/k3s-nfs \
  --set storageClass.name=managed-nfs-storage \
  --set storageClass.defaultClass=true

kubectl patch storageclass local-path -p '{"metadata": {"annotations":{"storageclass.kubernetes.io/is-default-class":"false"}}}'
```

## 13. 部署Portainer
登录k3s-master, 执行以下脚本：
```
helm repo add portainer https://portainer.github.io/k8s/
helm repo update

helm upgrade --install --create-namespace -n portainer portainer portainer/portainer \
    --set service.type=ClusterIP \
    --set image.tag=lts \
    --set ingress.enabled=true \
    --set ingress.ingressClassName=traefik \
    --set ingress.annotations."traefik\.ingress\.kubernetes\.io/router\.entrypoints"=websecure \
    --set ingress.annotations."traefik\.ingress\.kubernetes\.io/service\.serversscheme"=https \
    --set ingress.hosts[0].host=portainer.k3s.local \
    --set ingress.hosts[0].paths[0].path="/"
```

获取Setup Token, 执行以下脚本：
```
kubectl logs pod/portainer-8b7c7d986-qp2w9 --namespace portainer
```
从输出内容找到类似内容：
```
setup_token=75d8601fa2b1332e801cc955ee51952b06507e31f6e8e4b5e18152d79be3e645
```
访问https://portainer.k3s.local 设置admin账号密码