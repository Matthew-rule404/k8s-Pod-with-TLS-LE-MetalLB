
# sp-api-stt12 — エンドツーエンド手順（日本語）
**バージョン:** 2025-09-07  
本書は **最初から最後までの全手順** を詳細にまとめ直したものです。コピペで実行できるように構成しています。

> 目的：**NGINX Ingress** で TLS 終端、**cert‑manager + Let’s Encrypt（HTTP‑01）** を **IPv6** で発行。  
> **Harbor** から **ロボットアカウント**で Pull。永続化は **Deployment（RWX共有）** または **StatefulSet（RWO/Podごと）** を選択。  
> L4 は `loadBalancerClass` の切替で **kube‑VIP / MetalLB** 両対応。

---

## 0) 固定値
- **FQDN**: `sp-api-stt12.phone.omnigrid.jp`
- **IPv6 VIP（AAAA先）**: `2400:4053:2602:e700::12:ffff`
- **ACME 連絡先**: `m.nadayoshi@nn-com.co.jp`
- **IngressClass**: `nginx`
- **Harbor**: `harbor.sp12-k8s.phone.omnigrid.jp`
  - プロジェクト: `sp-api-stt`
  - リポジトリ: `harbor.sp12-k8s.phone.omnigrid.jp/sp-api-stt/pyapi`
- **例で使う Namespace**: `default`

---

## 1) アーキテクチャ概要
```
[ Internet (IPv6) ]
        | 80/443 (HTTP-01 は :80 必須)
        v
[ NGINX Ingress Controller ]  <-- cert-manager によりTLS終端
        | (ClusterIP:80)
        v
[ Service ] ---> [ sp-api (Deployment or StatefulSet) ]
                     |__ /data (PVC)
ローカルIPv4（同一FQDN） -> /etc/hosts -> プライベートIPv4 -> NGINX -> sp-api
```
- インターネット側は **IPv6** で到達。**HTTP‑01** のため **:80** が必須。  
- ローカルは **同一FQDN** を `/etc/hosts` でプライベートIPv4へ。URLは常に **https://FQDN**。

---

## 2) 前提チェックリスト
- Kubernetes ≥ 1.24（可能ならデュアルスタック）  
- **NGINX Ingress Controller** / **cert‑manager（CRD + Controller）**  
- Ingress Controller の Service（80/443）を **kube‑VIP** か **MetalLB** で公開  
- DNS AAAA: `sp-api-stt12.phone.omnigrid.jp` → `2400:4053:2602:e700::12:ffff`  
- 共有書き込みが必要なら **RWX** 対応 StorageClass（NFS/CephFS等）

---

## 3) Harbor — ロボットアカウント

### 3.1 概要（何者か／なぜ使うか）
- **人間ではない** 自動化用アカウント（CI/CD・ビルドノードなど）。  
- **トークン**で認証（作成時に **一度だけ**表示）。  
- **最小権限**（特定プロジェクトの **Pull/Push** 等）に絞れる。  
- 影響範囲が小さく、**ローテーション／失効**が安全に行える。  
- 形式例：`robot$sp-api-stt+ci`。

### 3.2 作成手順（UI）
1. `https://harbor.sp12-k8s.phone.omnigrid.jp/harbor/projects` にログイン。  
2. **Project** → `sp-api-stt` → **Robot Accounts** → **New Robot**。  
3. 次を設定：
   - **Name**: 例 `ci`（ユーザ名は `robot$sp-api-stt+ci` になる）  
   - **Permissions**: 当プロジェクトの **push / pull**  
   - **Expiration**: 定期ローテーション推奨（または無期限）  
4. **Save** → **表示されたトークンを控える（1回限り）**。CIのシークレットに保存。

### 3.3 Podman でログイン／ビルド／Push
> Harbor が独自 CA の場合、ビルドノードに CA を配布：  
> `/etc/containers/certs.d/harbor.sp12-k8s.phone.omnigrid.jp/ca.crt`

```bash
export REG=harbor.sp12-k8s.phone.omnigrid.jp
export PROJ=sp-api-stt
export IMG=pyapi
export TAG=v0.1.0

podman login "$REG"   --username 'robot$sp-api-stt+ci'   --password '<ROBOT_TOKEN>'

# Build
podman build -t "$REG/$PROJ/$IMG:$TAG" .

# Push
podman push "$REG/$PROJ/$IMG:$TAG"

# Harbor UI でタグを確認
```

### 3.4 Kubernetes の Pull Secret を作成
**kubectl（単一 Namespace）**:
```bash
kubectl create secret docker-registry harbor-cred   --docker-server=harbor.sp12-k8s.phone.omnigrid.jp   --docker-username='robot$sp-api-stt+ci'   --docker-password='<ROBOT_TOKEN>'   --docker-email='m.nadayoshi@nn-com.co.jp'   -n default
```
**Ansible（複数 Namespace）**: `extras/ansible/harbor-cred-playbook.yaml` を使用。
```bash
ansible-playbook extras/ansible/harbor-cred-playbook.yaml   -e harbor_password='<ROBOT_TOKEN>'   -e 'namespaces=["default","sp-api"]'
```

---

## 4) NGINX Ingress の公開（kube‑VIP / MetalLB）
`extras/ingress-services/` のどちらかを適用：
```bash
kubectl apply -f extras/ingress-services/ingress-svc-kubevip.yaml
# または
kubectl apply -f extras/ingress-services/ingress-svc-metallb.yaml

kubectl get svc -n ingress-nginx   # IPv6 の EXTERNAL-IP を確認
```
`spec.loadBalancerClass` により **一致する実装のみ** が担当するため、両者の共存も可能です。

---

## 5) values.yaml（永続化・環境切替・ワークロード種別）
`pyapi/values.yaml` の主な項目：
```yaml
# 5.1 ワークロード切替
workload:
  type: "deployment"   # "deployment"=共有RWX / "statefulset"=PodごとRWO

# 5.2 永続化
persistence:
  enabled: true
  # deployment（共有ボリューム用）
  storageClass: "nfs-client"
  accessModes: ["ReadWriteMany"]
  size: "10Gi"
  existingClaim: ""
  mountPath: "/data"
  # statefulset（Podごとボリューム用）
  statefulAccessModes: ["ReadWriteOnce"]
  statefulStorageClass: ""
  statefulSize: "10Gi"

# 5.3 cert-manager の環境切替
certManager:
  enabled: true
  environment: "staging"    # 動作確認後 "production" に変更
  clusterIssuer:
    create: true
    name: ""                # 空なら自動: staging=letsencrypt-staging / production=letsencrypt
    email: m.nadayoshi@nn-com.co.jp

# 5.4 Ingress
ingress:
  enabled: true
  className: nginx
  hostname: sp-api-stt12.phone.omnigrid.jp
  tls:
    enabled: true
    secretName: pyapi-tls
```

**用語（略語）**
- **PVC（PersistentVolumeClaim）**: Pod からのストレージ要求  
- **PV（PersistentVolume）**: 実体ストレージ  
- **RWO（ReadWriteOnce）**: 1ノードのみ書込可（StatefulSet に典型）  
- **RWX（ReadWriteMany）**: 複数ノードから同時書込可（共有用途）  
- **StatefulSet**: 安定ID & PodごとPVC（`volumeClaimTemplates`）

---

## 6) インストール → 検証 → 本番化
```bash
# まずは staging で
helm upgrade --install pyapi ./pyapi -f pyapi/values.yaml

# 発行/経路の確認
kubectl get certificate -A
kubectl describe challenge -A

# 問題なければ production に切替
#  - values の certManager.environment を "production" に変更
helm upgrade --install pyapi ./pyapi -f pyapi/values.yaml
```

**ローカルIPv4アクセス**  
`/etc/hosts` 例：`10.0.12.50 sp-api-stt12.phone.omnigrid.jp`  
アクセスは常に **https://sp-api-stt12.phone.omnigrid.jp**（IP直打ちは不可）。

---

## 7) NetworkPolicy / PDB / ServiceMonitor（任意）
- `networkPolicy.enabled: true` ：同一Namespace + `ingress-nginx` のみ Ingress 許可、DNS への Egress 許可。  
- `pdb.enabled: true` ＆ `minAvailable: 1` ：任意停止時の可用性を確保。  
- `metrics.enabled: true` ：Prometheus Operator の CRD があれば **ServiceMonitor** を自動作成（`/metrics`）。

---

## 8) トラブルシュート
- **HTTP‑01 失敗**：IPv6 の :80/443 到達性、IngressClass のミスマッチ、競合する Ingress ルールを確認。  
- **EXTERNAL‑IP が付与されない**：kube‑VIP / MetalLB のコントローラ状態、`loadBalancerClass` を確認。  
- **Harbor Pull 失敗**：`imagePullSecrets`、ロボットトークン、Harbor CA の信頼を再確認。  
- **PVC Pending**：StorageClass 名・アクセスモード。共有用途なら RWX が必要。  
- **LAN で TLS エラー**：URL に IP を使っていないか。必ず **FQDN** を使用。

---

## 9) ライセンス
このチャートおよびドキュメントは **Apache‑2.0** を推奨します。

