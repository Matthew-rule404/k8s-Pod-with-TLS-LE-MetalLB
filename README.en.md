
# sp-api-stt12 — End-to-End Guide (EN)
**Version:** 2025-09-07  
This guide rebuilds the whole setup *from scratch*, with detailed, copy‑paste ready steps.

> Goal: Deploy a Python 3.13 API behind **NGINX Ingress** with TLS terminated by **cert‑manager + Let’s Encrypt (HTTP‑01)** via **IPv6**, pull images from **Harbor** using **Robot Accounts**, support **Deployment (RWX)** or **StatefulSet (RWO)** persistence, and be switchable between **kube‑VIP** and **MetalLB** by `loadBalancerClass`.

---

## 0) Fixed values in this environment
- **FQDN**: `sp-api-stt12.phone.omnigrid.jp`
- **IPv6 VIP (AAAA target)**: `2400:4053:2602:e700::12:ffff`
- **ACME contact email**: `m.nadayoshi@nn-com.co.jp`
- **IngressClass**: `nginx` (change if your installation differs)
- **Harbor**: `harbor.sp12-k8s.phone.omnigrid.jp`
  - Project: `sp-api-stt`
  - Repository path: `harbor.sp12-k8s.phone.omnigrid.jp/sp-api-stt/pyapi`
- **Namespace used in examples**: `default`

---

## 1) Architecture overview
```
[ Internet (IPv6) ]
        | 80/443
        v
[ NGINX Ingress Controller ]  <-- TLS termination (cert-manager)
        | (ClusterIP:80)
        v
[ Service ] ---> [ sp-api (Deployment or StatefulSet) ]
                     |__ /data (PVC)
Local IPv4 (same FQDN) -> /etc/hosts -> private IPv4 -> NGINX -> sp-api
```
- **HTTP‑01** requires **port 80** reachable over the Internet (IPv6 in this design).
- **Local IPv4** path must use the **same FQDN** via `/etc/hosts`; **never** use raw IPs in URLs with TLS.

---

## 2) Prerequisites checklist
- Kubernetes ≥ 1.24, dual‑stack capable network path
- **NGINX Ingress Controller**
- **cert‑manager** (CRDs + controller)
- L4 exposure of the Ingress Controller Service (80/443) with either:
  - **kube‑VIP** (simple L2 VIP) **or**
  - **MetalLB** (L2 or BGP; BGP allows ECMP scale‑out)
- DNS: AAAA for `sp-api-stt12.phone.omnigrid.jp` → `2400:4053:2602:e700::12:ffff`
- Storage: RWX class (NFS/CephFS) if you plan **shared** writes with multiple replicas

---

## 3) Harbor — Robot Accounts (what/why/how)

### 3.1 What is a Robot Account?
- A **non‑human** account used by automation (CI/CD, build nodes).  
- Uses a **token** (shown **once** at creation) instead of a password.  
- Can be limited to a **single project** and minimal permissions (**pull/push**).  
- Easy to rotate/revoke without impacting human SSO users.  
- Name looks like: `robot$<project>+<name>` e.g. `robot$sp-api-stt+ci`.

### 3.2 Create a Robot Account in Harbor (UI)
1. Login: `https://harbor.sp12-k8s.phone.omnigrid.jp/harbor/projects`  
2. Open **Project** → `sp-api-stt` → **Robot Accounts** → **New Robot**.  
3. Fill in:
   - **Name**: e.g. `ci` (username becomes `robot$sp-api-stt+ci`).  
   - **Permissions**: at least **push** and **pull** for this project.  
   - **Expiration**: set a reasonable rotation schedule (or never).  
4. **Save** and **copy the token** (one‑time). Store it in your CI secret manager.

### 3.3 Login / Build / Push with Podman
> If Harbor uses a private CA, install it first on build nodes:  
> `sudo mkdir -p /etc/containers/certs.d/harbor.sp12-k8s.phone.omnigrid.jp && sudo cp ca.crt /etc/containers/certs.d/harbor.sp12-k8s.phone.omnigrid.jp/ca.crt`

```bash
# Variables
export REG=harbor.sp12-k8s.phone.omnigrid.jp
export PROJ=sp-api-stt
export IMG=pyapi
export TAG=v0.1.0

# Login with Robot Account
podman login "$REG"   --username 'robot$sp-api-stt+ci'   --password '<ROBOT_TOKEN>'

# Build
podman build -t "$REG/$PROJ/$IMG:$TAG" .

# (Optional) Multi-arch with buildx via docker (if needed). For Podman, use build --arch and manifests.

# Push
podman push "$REG/$PROJ/$IMG:$TAG"

# Verify in Harbor UI: project -> repositories -> sp-api-stt/pyapi -> tags
```

### 3.4 Create Kubernetes pull Secret (kubectl or Ansible)
**kubectl (one namespace)**:
```bash
kubectl create secret docker-registry harbor-cred   --docker-server=harbor.sp12-k8s.phone.omnigrid.jp   --docker-username='robot$sp-api-stt+ci'   --docker-password='<ROBOT_TOKEN>'   --docker-email='m.nadayoshi@nn-com.co.jp'   -n default
```
**Ansible (multiple namespaces)**: use `extras/ansible/harbor-cred-playbook.yaml`:
```bash
ansible-playbook extras/ansible/harbor-cred-playbook.yaml   -e harbor_password='<ROBOT_TOKEN>'   -e 'namespaces=["default","sp-api"]'
```

---

## 4) Expose NGINX Ingress (kube‑VIP or MetalLB)
Choose **one** Service manifest from `extras/ingress-services/` and apply:
```bash
# kube-VIP
kubectl apply -f extras/ingress-services/ingress-svc-kubevip.yaml
# or MetalLB
kubectl apply -f extras/ingress-services/ingress-svc-metallb.yaml

kubectl get svc -n ingress-nginx   # confirm EXTERNAL-IP (IPv6)
```
> The key field `spec.loadBalancerClass` ensures only the matching LB implementation handles this Service, so both can co‑exist.

---

## 5) values.yaml (persistence, staging/prod, workload type)
Edit `pyapi/values.yaml` (already included). Important knobs:
```yaml
# 5.1 Switch workload
workload:
  type: "deployment"        # "deployment" (shared RWX PVC) or "statefulset" (per-pod RWO PVC)

# 5.2 Persistence for app
persistence:
  enabled: true
  # For deployment (shared volume):
  storageClass: "nfs-client"
  accessModes: ["ReadWriteMany"]
  size: "10Gi"
  existingClaim: ""         # if set, reuse an existing PVC
  mountPath: "/data"
  # For statefulset (per-pod):
  statefulAccessModes: ["ReadWriteOnce"]
  statefulStorageClass: ""
  statefulSize: "10Gi"

# 5.3 cert-manager environment switch
certManager:
  enabled: true
  environment: "staging"    # "staging" -> untrusted test certs; switch to "production" later
  clusterIssuer:
    create: true
    name: ""                # auto: staging->letsencrypt-staging / production->letsencrypt
    email: m.nadayoshi@nn-com.co.jp

# 5.4 Ingress & hostname
ingress:
  enabled: true
  className: nginx
  hostname: sp-api-stt12.phone.omnigrid.jp
  tls:
    enabled: true
    secretName: pyapi-tls
```

**Glossary**
- **PVC** (PersistentVolumeClaim): a storage request by a pod.  
- **PV** (PersistentVolume): actual storage resource.  
- **RWO** (ReadWriteOnce): single‑node writable; typical for StatefulSet per‑pod disks.  
- **RWX** (ReadWriteMany): multi‑node writable; required for shared data with multiple replicas.  
- **StatefulSet**: gives stable identities and `volumeClaimTemplates` (per‑pod PVCs).

---

## 6) Install / verify / promote
```bash
# Install (staging first)
helm upgrade --install pyapi ./pyapi -f pyapi/values.yaml

# Verify issuance and routing
kubectl get certificate -A
kubectl describe challenge -A

# Promote to production
#  - set certManager.environment: "production" in values
helm upgrade --install pyapi ./pyapi -f pyapi/values.yaml
```

**Local IPv4 access**  
Add to `/etc/hosts` (example):  
`10.0.12.50 sp-api-stt12.phone.omnigrid.jp`  
Access **https://sp-api-stt12.phone.omnigrid.jp** (FQDN only; never raw IP).

---

## 7) Optional: NetworkPolicy, PDB, ServiceMonitor
- `networkPolicy.enabled: true` → only same namespace + `ingress-nginx` can reach the app; DNS egress allowed.  
- `pdb.enabled: true` with `minAvailable: 1` protects against voluntary disruptions.  
- `metrics.enabled: true` → creates **ServiceMonitor** if Prometheus Operator CRDs are present (`/metrics` by default).

---

## 8) Troubleshooting
- **HTTP‑01 fails**: confirm IPv6 :80/443 reachability, IngressClass, and no conflicting Ingress rules.  
- **No EXTERNAL‑IP**: check kube‑VIP/MetalLB controller health and `loadBalancerClass`.  
- **Harbor pull fails**: validate `imagePullSecrets`, Robot token, Harbor CA trust.  
- **PVC Pending**: wrong StorageClass or access mode; RWX needed for shared volumes.  
- **TLS errors on LAN**: using IP in URL? Must use FQDN.

---

## 9) License
Use **Apache‑2.0** for this chart and docs.

