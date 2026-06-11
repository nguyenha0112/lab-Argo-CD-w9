# W9 Lab Evidence

## 1. GitOps

Da tao va sync ArgoCD app-of-apps:

- Root app: `w9-root`
- Platform app: `w9-mini-platform`
- Rollout app: `w9-rollout`

Lenh da chay:

```powershell
kubectl get applications -n argocd
```

Ket qua:

```text
NAME               SYNC STATUS   HEALTH STATUS
w9-mini-platform   Synced        Healthy
w9-rollout         Synced        Healthy
w9-root            Synced        Healthy
```

Repo da push commit moi:

```text
af7abff Fix w9 canary analysis evidence
```

## 2. Platform App

Da deploy namespace, frontend, backend va service trong `mini-platform`.

Lenh da chay:

```powershell
kubectl apply -f cloud\w9\lab\manifests
kubectl get pods -n mini-platform -o wide
kubectl get svc,endpoints -n mini-platform
```

Ket qua:

```text
web-5675fd79c9-2v9pd         1/1 Running
web-5675fd79c9-lqlzx         1/1 Running
xbrain-api-d69bbcb4b-2lw8p   1/1 Running

service/web          80/TCP
service/xbrain-api   8080/TCP
endpoints/web        10.244.0.91:80,10.244.0.90:80
endpoints/xbrain-api 10.244.0.82:8080
```

Kiem tra frontend:

```powershell
kubectl port-forward svc/web -n mini-platform 18080:80
curl.exe -s http://localhost:18080
```

Ket qua co:

```text
<title>XBrain Company Form</title>
<h1>XBrain Company Intake</h1>
```

Kiem tra backend qua Nginx proxy:

```powershell
curl.exe --% -s -X POST http://localhost:18080/api/xbrain-company -H "content-type: application/json" -d "{""company"":""XBrain"",""email"":""hello@xbrain.local"",""message"":""GitOps evidence test""}"
```

Ket qua co response tu backend pod:

```text
"hostname": "xbrain-api-d69bbcb4b-2lw8p"
"company": "XBrain"
"email": "hello@xbrain.local"
```

## 3. Observability

Da deploy OpenTelemetry Collector, PrometheusRule va fake Prometheus local.

Lenh da chay:

```powershell
kubectl apply -f cloud\w9\W9-D2_Observability_SLO_OTel\otel\collector.yaml
kubectl apply -f cloud\w9\W9-D2_Observability_SLO_OTel\alert-rules\slo-burn-rate.yaml
kubectl get pods -n observability
kubectl get prometheusrule -n observability
```

Ket qua:

```text
fake-prometheus-fd7468447-wbfs6   1/1 Running
otel-collector-5fb9587786-kz5dc   1/1 Running

NAME                AGE
web-slo-burn-rate   20h
```

Fake Prometheus dang tra metric tot:

```powershell
kubectl get configmap fake-prometheus-content -n observability -o yaml
```

Ket qua:

```text
value [0,"0"]
```

## 4. Canary Rollout

Da deploy Argo Rollouts resource trong `cloud/w9/lab/rollout`.

Lenh da chay:

```powershell
kubectl apply -f cloud\w9\lab\rollout
kubectl get rollout web -n mini-platform
kubectl get analysisrun -n mini-platform --sort-by=.metadata.creationTimestamp
```

Ket qua rollout:

```text
NAME   DESIRED   CURRENT   UP-TO-DATE   AVAILABLE
web    2         2         2            2
```

AnalysisRun moi thanh cong:

```text
web-5675fd79c9-15-2   Successful
web-5675fd79c9-15-5   Successful
```

Dieu kien analysis dang dung:

```powershell
kubectl get analysistemplate web-error-rate -n mini-platform -o jsonpath="{.spec.metrics[0].successCondition}"
```

Ket qua:

```text
result[0] <= 0.01
```

## 5. Bad Canary Abort

Da co evidence canary fail va rollback ve stable.

Lenh da chay:

```powershell
kubectl get analysisrun -n mini-platform --sort-by=.metadata.creationTimestamp
kubectl describe rollout web -n mini-platform
```

Ket qua fail:

```text
web-5675fd79c9-13-2   Failed
```

Trong rollout event co:

```text
Rollout aborted update to revision 13
Metric "error-rate" assessed Failed
Rollback to stable ReplicaSets
```

Trang thai cuoi sau rollback va sync lai Git:

```text
Rollout web: Healthy
ArgoCD apps: Synced / Healthy
AnalysisRun revision 15: Successful
```

## 6. Link Evidence De Chup

- Web app: `http://localhost:18080`
- ArgoCD UI: `https://localhost:8080`
- ArgoCD user: `admin`
- ArgoCD password: `DdgYNc3Bm7921R97`
