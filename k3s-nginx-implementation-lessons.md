# K3s + Nginx 実装時の問題点と対応策

## 概要
k3s環境でNginx Ingress Controllerを実装する際に遭遇した問題点と、今後のコスト最適化のための対応策をまとめました。

## 遭遇した主要な問題

### 1. システムアップデート時の対話的プロンプト問題
**問題**: `sudo apt update && sudo apt upgrade -y` 実行時にPAM設定の対話的プロンプトで処理が停止
**影響**: 約5分間の作業時間ロス
**原因**: `-y` フラグがあってもPAM設定変更時は対話的確認が必要

**対応策**:
```bash
# 非対話モードでのシステムアップデート
export DEBIAN_FRONTEND=noninteractive
sudo apt update && sudo apt upgrade -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"
```

### 2. k3s初回インストール時のネットワーク問題
**問題**: デフォルトのflannel設定でkernel moduleエラーによりk3s起動失敗
**影響**: 複数回の再インストールが必要
**原因**: コンテナ環境でbr_netfilter、overlayモジュールが利用不可

**対応策**:
```bash
# 最初からhost-gwバックエンドでインストール
curl -sfL https://get.k3s.io | sh -s - --disable traefik --flannel-backend=host-gw
```

### 3. Helm インストール時のタイムアウト問題
**問題**: ネットワーク問題のあるk3sクラスタでHelm installが3分以上ハング
**影響**: 作業時間の大幅なロス
**原因**: ノードがNotReady状態でPodがPending状態のまま

**対応策**:
- k3sクラスタの状態を事前確認: `kubectl get nodes`
- Podの状態確認: `kubectl get pods -A`
- ノードがReady状態になってからHelm操作を実行

## 効率的な作業フロー

### 1. 事前確認チェックリスト
```bash
# システム状態確認
kubectl get nodes
kubectl get pods -A

# 必要なツールの確認
which k3s && which kubectl && which helm
```

### 2. 推奨インストール手順
```bash
# 1. 非対話モードでシステム更新
export DEBIAN_FRONTEND=noninteractive
sudo apt update && sudo apt upgrade -y -o Dpkg::Options::="--force-confdef" -o Dpkg::Options::="--force-confold"

# 2. k3sを適切なネットワーク設定でインストール
curl -sfL https://get.k3s.io | sh -s - --disable traefik --flannel-backend=host-gw

# 3. kubectl設定
mkdir -p ~/.kube
sudo cp /etc/rancher/k3s/k3s.yaml ~/.kube/config
sudo chown $USER:$USER ~/.kube/config
export KUBECONFIG=~/.kube/config

# 4. クラスタ状態確認（Ready状態まで待機）
kubectl get nodes
kubectl get pods -A

# 5. Helmインストール
curl https://raw.githubusercontent.com/helm/helm/main/scripts/get-helm-3 | bash

# 6. Nginx Ingress Controller デプロイ
helm repo add ingress-nginx https://kubernetes.github.io/ingress-nginx
helm repo update
helm install ingress-nginx ingress-nginx/ingress-nginx \
  --namespace ingress-nginx \
  --create-namespace \
  --set controller.service.type=NodePort \
  --set controller.service.nodePorts.http=30080 \
  --set controller.service.nodePorts.https=30443
```

### 3. 検証手順
```bash
# Ingress Controller確認
kubectl get pods -n ingress-nginx
kubectl get svc -n ingress-nginx

# サンプルアプリケーションデプロイ
kubectl apply -f sample-app.yaml

# アプリケーション確認
kubectl get pods
kubectl get svc
kubectl get ingress

# ローカルDNS設定とアクセステスト
echo "127.0.0.1 sample.local" | sudo tee -a /etc/hosts
curl http://sample.local:30080
```

## コスト最適化のための推奨事項

### 1. 事前準備の徹底
- 環境固有の問題を事前に把握
- 非対話モードでのコマンド実行
- 適切なネットワーク設定での初回インストール

### 2. 効率的なデバッグ手法
- ログ確認: `sudo journalctl -u k3s -n 20`
- 状態確認: `kubectl describe pod <pod-name>`
- 段階的な確認で問題の早期発見

### 3. 再利用可能なスクリプト化
今回の手順をスクリプト化して今後の作業効率化を図る

## 今回の実装結果
- ✅ k3s クラスタ正常動作
- ✅ Nginx Ingress Controller デプロイ完了
- ✅ サンプルアプリケーション動作確認
- ✅ HTTP アクセス正常動作
- ✅ sample-app.yaml ファイル分離完了

## 推定時間短縮効果
今回の対応策により、同様の作業を約15-20分短縮可能と推定されます。
