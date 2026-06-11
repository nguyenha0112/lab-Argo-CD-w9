# W9 Lab Evidence

## Scope

Lab nay tong hop 3 phan W9:

- GitOps voi ArgoCD va app-of-apps.
- Observability voi OpenTelemetry Collector, PrometheusRule/SLO va fake Prometheus local.
- Progressive delivery voi Argo Rollouts canary va AnalysisTemplate auto-abort.

## 1. GitOps

Root app-of-apps:

- `cloud/w9/lab/argocd/root-app.yaml`
- Application name: `w9-root`
- Source path: `cloud/w9/lab/argocd`
- Vai tro: root quan ly cac Application con trong thu muc `argocd/`.

Application con:

- `w9-mini-platform`: sync path `cloud/w9/lab/manifests`.
- `w9-rollout`: sync path `cloud/w9/lab/rollout`.

Lenh kiem tra:

```powershell
kubectl get applications -n argocd
kubectl describe application w9-mini-platform -n argocd
kubectl describe application w9-rollout -n argocd
```

Ket qua can dat:

```text
NAME               SYNC STATUS   HEALTH STATUS
w9-root           Synced        Healthy
w9-mini-platform  Synced        Healthy
w9-rollout        Synced        Healthy
```

Giai thich: Git la source of truth. Khi sua manifest va push len Git, ArgoCD sync cluster ve dung trang thai trong repo. Rollback dung `git revert`, khong sua tay resource trong cluster.

## 2. Platform App

Resource trong `cloud/w9/lab/manifests`:

- Namespace `mini-platform`.
- ConfigMap `xbrain-frontend`: HTML form `XBrain Company Intake`.
- ConfigMap `xbrain-nginx`: Nginx reverse proxy `/api/` sang backend.
- Deployment `xbrain-api`: backend echo service.
- Service `xbrain-api`: port `8080`.
- Service `web`: port `80`, selector `app=web` va `xbrain.io/component=frontend`.
- Fake Prometheus trong namespace `observability` de AnalysisTemplate co endpoint Prometheus khi chay local.

Lenh kiem tra:

```powershell
kubectl get all -n mini-platform
kubectl port-forward svc/web -n mini-platform 18080:80
curl.exe http://localhost:18080
curl.exe -X POST http://localhost:18080/api/xbrain-company -H "content-type: application/json" -d "{\"company\":\"XBrain\",\"email\":\"hello@xbrain.local\",\"message\":\"GitOps test\"}"
```

Ket qua can dat:

- GET `/` tra HTML co title `XBrain Company Form`.
- POST `/api/xbrain-company` tra response tu pod backend `xbrain-api`.

## 3. Observability

OpenTelemetry Collector:

```powershell
kubectl create namespace observability
kubectl apply -f cloud\w9\W9-D2_Observability_SLO_OTel\otel\collector.yaml
kubectl -n observability rollout status deploy/otel-collector --timeout=180s
```

SLO alert rule:

```powershell
kubectl apply -f cloud\w9\W9-D2_Observability_SLO_OTel\alert-rules\slo-burn-rate.yaml
kubectl get prometheusrule -n observability
```

Neu gap loi `no matches for kind "PrometheusRule"`, cluster chua co Prometheus Operator CRD. Cai CRD/monitoring stack truoc, hoac ghi chu rang local stack chua evaluate rule.

Fake Prometheus local:

```powershell
kubectl apply -f cloud\w9\lab\manifests\03-fake-prometheus.yaml
kubectl get svc prometheus-operated -n observability
```

File fake Prometheus tra error-rate `0`, dung de canary tot pass trong local lab.

## 4. Canary Rollout

Resource trong `cloud/w9/lab/rollout`:

- `AnalysisTemplate/web-error-rate`: query Prometheus va pass khi `result[0] <= 0.01`.
- `Rollout/web`: chay `nginx:1.28`, mount frontend ConfigMap, canary theo buoc 20% -> analysis -> 50% -> analysis.

Lenh kiem tra:

```powershell
kubectl get rollout -n mini-platform
kubectl get analysistemplate -n mini-platform
kubectl get analysisrun -n mini-platform
kubectl describe rollout web -n mini-platform
```

Ket qua can dat:

```text
Rollout web: Healthy
AnalysisRun: Successful
```

Ly do pass: fake Prometheus tra error-rate `0`, thoa `result[0] <= 0.01`.

## 5. Bad Canary Auto-Abort

Cach tao bad canary co kiem soat:

1. Sua fake Prometheus trong `03-fake-prometheus.yaml` de tra `"1"` thay vi `"0"`.
2. Tao mot revision rollout moi, vi du doi annotation `xbrain.io/restarted-at` hoac image tag.
3. Push Git va de ArgoCD sync.

Lenh quan sat:

```powershell
kubectl get analysisrun -n mini-platform
kubectl describe rollout web -n mini-platform
```

Ket qua can dat:

```text
AnalysisRun: Failed
Rollout: Degraded
Abort: true
Metric "error-rate" assessed Failed
```

Sau khi ghi evidence, revert commit xau:

```powershell
git revert <bad_commit>
git push
```

ArgoCD se sync ve fake Prometheus tra `0` va rollout tot.

## 6. Checklist

| Yeu cau | Trang thai mong muon | Evidence |
|---|---|---|
| GitOps app-of-apps | Dat | `w9-root` quan ly app con |
| ArgoCD auto sync | Dat | `w9-mini-platform`, `w9-rollout` Synced/Healthy |
| Web frontend | Dat | `svc/web` tra HTML form |
| Backend API | Dat | `/api/xbrain-company` proxy sang `xbrain-api` |
| Observability | Dat | OTel Collector va PrometheusRule co manifest |
| Canary healthy | Dat | Analysis pass khi error-rate `0` |
| Bad canary abort | Dat | Analysis fail khi error-rate `1` |
| Rollback GitOps | Dat | Dung `git revert` de dua cluster ve state tot |

## 7. Ket Luan

Lab dap ung yeu cau trong mau huong dan W9:

- Moi thay doi ung dung di qua Git va ArgoCD.
- Co root app-of-apps de quan ly cac app con.
- Co observability/SLO manifest va fake Prometheus phuc vu local analysis.
- Canary tot duoc promote khi metric dat nguong.
- Canary xau co the bi auto-abort dua tren AnalysisTemplate.
- Rollback dung Git revert de giu dung tinh than GitOps.
