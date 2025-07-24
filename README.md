# k3s-nginx

Ubuntu上でK3sとNginxを実現する方法について説明します。

## K3sのインストール

### 1. システム要件の確認
```bash
# Ubuntu 20.04/22.04 LTS推奨
# メモリ: 最小512MB（推奨1GB以上）
# CPU: 1コア以上

# システムアップデート
sudo apt update && sudo apt upgrade -y
```

### 2. K3sのインストール
```bash
# K3sインストールスクリプトの実行
curl -sfL https://get.k3s.io | sh -

# インストール確認
sudo k3s kubectl get nodes

# kubeconfigの設定（一般ユーザーで使用する場合）
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config
```

## Nginx Ingress Controllerのデプロイ

### 方法1: K3s標準のTraefikを無効化してNginxを使用

```bash
# K3sをTraefik無効でインストール（新規インストールの場合）
curl -sfL https://get.k3s.io | sh -s - --disable traefik

# 既存のK3sでTraefikを無効化
sudo systemctl stop k3s
sudo vim /etc/systemd/system/k3s.service
# ExecStartに --disable traefik を追加
sudo systemctl daemon-reload
sudo systemctl start k3s
```

### 方法2: Nginx Ingress Controllerのインストール

```bash
# Helmのインストール（まだの場合）
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# Nginx Ingress Controllerのデプロイ
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update

# NodePort構成でインストール（K3s向け）
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

## サンプルアプリケーションのデプロイ### アプリケーションのデプロイ

```bash
# サンプルアプリケーションをデプロイ
kubectl apply -f sample-app.yaml

# デプロイ状況の確認
kubectl get pods
kubectl get svc
kubectl get ingress
```

## アクセス確認

```bash
# /etc/hostsに追加
echo "127.0.0.1 sample.local" | sudo tee -a /etc/hosts

# Nginx Ingress ControllerのNodePortでアクセス
curl http://sample.local:30080

# または直接NodeのIPでアクセス
NODE_IP=$(kubectl get nodes -o jsonpath='{.items[0].status.addresses[?(@.type=="InternalIP")].address}')
curl http://$NODE_IP:30080 -H "Host: sample.local"
```

## 本番環境向けの考慮事項

### 1. LoadBalancerタイプの使用（AWS EKS環境）
```yaml
# AWS環境では LoadBalancer タイプを使用
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=LoadBalancer
```

### 2. SSL/TLS証明書の設定
```bash
# cert-managerのインストール
kubectl apply -f https://github.com/cert-manager/cert-manager/releases/download/v1.13.0/cert-manager.yaml

# Let's Encrypt証明書の自動取得設定
```

### 3. リソース制限の設定
```yaml
resources:
  limits:
    cpu: 200m
    memory: 256Mi
  requests:
    cpu: 100m
    memory: 128Mi
```

## トラブルシューティング

```bash
# K3sのログ確認
sudo journalctl -u k3s -f

# Nginx Ingress Controllerのログ
kubectl logs -n ingress-nginx -l app.kubernetes.io/name=ingress-nginx

# Podの詳細確認
kubectl describe pod <pod-name>
```

この構成により、UbuntuでK3sクラスタを構築し、Nginx Ingress Controllerを使用してアプリケーションを公開できます。開発環境ではこの構成で十分ですが、本番環境ではAWS EKSへの移行を検討してください。

```
---
# sample-app.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: sample-app
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: sample-app
  template:
    metadata:
      labels:
        app: sample-app
    spec:
      containers:
      - name: nginx
        image: nginx:alpine
        ports:
        - containerPort: 80
        volumeMounts:
        - name: html
          mountPath: /usr/share/nginx/html
      volumes:
      - name: html
        configMap:
          name: sample-html
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: sample-html
  namespace: default
data:
  index.html: |
    <!DOCTYPE html>
    <html>
    <head>
        <title>K3s + Nginx Sample</title>
    </head>
    <body>
        <h1>Welcome to K3s with Nginx!</h1>
        <p>This is running on K3s cluster with Nginx Ingress Controller</p>
    </body>
    </html>
---
apiVersion: v1
kind: Service
metadata:
  name: sample-app-service
  namespace: default
spec:
  selector:
    app: sample-app
  ports:
  - port: 80
    targetPort: 80
---
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: sample-app-ingress
  namespace: default
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  ingressClassName: nginx
  rules:
  - host: sample.local
    http:
      paths:
      - path: /
        pathType: Prefix
        backend:
          service:
            name: sample-app-service
            port:
              number: 80
```
