# 在K3S环境上安装Rancher

节点信息：

系统：Ubuntu22.04LTS

主机名：k3s

IP: 172.16.10.56

## 安装 k3s

```bash
# 设置主机名称
hostnamectl set-hostname k3s

# 安装k3s，指定版本v1.27.10+k3s1
curl -sfL https://rancher-mirror.rancher.cn/k3s/k3s-install.sh | INSTALL_K3S_VERSION="v1.27.10+k3s1" INSTALL_K3S_MIRROR=cn sh -s - server --cluster-init

mkdir .kube
cp /etc/rancher/k3s/k3s.yaml .kube/config
chmod 600 .kube/config
```

## 安装helm

```bash
curl -fsSL https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash -
```

## 安装cert-manager 

```bash
# 添加仓库
helm repo add jetstack https://charts.jetstack.io

# 更新仓库
helm repo update

# 安装cert-manager crds资源清单
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.4/cert-manager.crds.yaml

# 安装cert-manager
helm install cert-manager \
jetstack/cert-manager \
--namespace cert-manager \
--create-namespace
```

## 安装rancher

```bash
# 添加仓库
helm repo add rancher-latest https://releases.rancher.com/server-charts/latest

# 更新仓库
helm repo update

# 安装rancher
helm install rancher rancher-latest/rancher \
--namespace cattle-system \
--create-namespace \
--set hostname=rancher.axbsec.com \
--set service.type=NodePort \
--set replicas=1 \
--set bootstrapPassword=12bitpassword
```



##  修改svc/rancher服务类型

```bash
# 编辑rancher服务，修改Type为NodePort
kubectl edit svc rancher -n cattle-system

# 获取rancher服务的NodePort
kubectl get svc/rancher -n cattle-system
NAME      TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
rancher   NodePort   10.43.195.230   <none>        80:32212/TCP,443:31766/TCP   50m
```

## 添加nginx反向代理



```bash
# rancher.axbsec.com.conf内容

map $http_upgrade $connection_upgrade {
    default Upgrade;
    '' close;
}

server {

    listen 80;

    if ($scheme = "http") {
        return 301 https://$host$request_uri;
    }

    listen 443 ssl;
    server_name  rancher.axbsec.com;

    ssl_certificate      /etc/nginx/ssl_cert/axbsec.com.pem;
    ssl_certificate_key  /etc/nginx/ssl_cert/axbsec.com.key;
    ssl_session_cache    shared:SSL:1m;
    ssl_session_timeout  10m;
    ssl_protocols	TLSv1.2 TLSv1.3;
    ssl_ciphers  HIGH:!aNULL:!MD5;
    ssl_prefer_server_ciphers   on;

    location / {
        proxy_set_header Host $host;
        proxy_set_header X-Forwarded-Proto $scheme;
        proxy_set_header X-Forwarded-Port $server_port;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_pass http://172.16.10.56:32212;
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection $connection_upgrade;
        # 此项允许执行的 shell 窗口保持开启，最长可达15分钟。不使用此参数的话，默认1分钟后自动关闭。
        proxy_read_timeout 900s;
        proxy_buffering off;
    }

}
```

## 访问

在浏览器使用[https://rancher.axbsec.com](https://rancher.axbsec.com)访问

用户名：admin

初始密码：安装rancher时使用--set bootstrapPassword指定的初始密码