# 使用 GKE 測試 KEDA (CNCF Project)

目標：在GKE使用KEDA追蹤 Cloud Monitoring Load Balancer Request Count 進行 Deployment 擴縮

## 前置需求

- 使用開啟 Workload Identity 的 GKE

## 設定環境變數

```sh
export PROJECT_ID=<PROJECT_ID>
```

## 設定 GKE Workload Identity

```sh
gcloud iam service-accounts create stackdriver-keda-sa --description="workload identity keda sa" --display-name="stackdriver-keda"

gcloud projects add-iam-policy-binding ${PROJECT_ID} \
  --member "serviceAccount:stackdriver-keda-sa@${PROJECT_ID}.iam.gserviceaccount.com" \
  --role "roles/monitoring.viewer"

gcloud iam service-accounts add-iam-policy-binding stackdriver-keda-sa@${PROJECT_ID}.iam.gserviceaccount.com \
  --role roles/iam.workloadIdentityUser \
  --member "serviceAccount:${PROJECT_ID}.svc.id.goog[keda/keda-operator]"
```

## 建立 Cloud Armor Policy 和 Role

設定允許來源只有本機IP方便測試

```sh
gcloud compute security-policies create my-policy --type=CLOUD_ARMOR --description="policy description"

gcloud compute security-policies rules create 1000 --action=allow --security-policy=my-policy \
  --description="allow my ip" --src-ip-ranges=$(curl ifconfig.me)

gcloud compute security-policies rules update 2147483647 --security-policy=my-policy --action="deny-403"
```

## 保留 Load Balancer 外部IP

```sh
gcloud compute addresses create nginx-ip --global --ip-version IPV4
```

## 驗證 Workload Identity

1. 建立 gcloud.yaml 檔案：

```yaml
apiVersion: v1
kind: Pod
metadata:
  labels:
    run: gcloud
  name: gcloud
  namespace: keda
spec:
  containers:
  - name: gcloud
    image: gcr.io/google.com/cloudsdktool/cloud-sdk:latest
    command:
      - sleep
      - "86400"
  serviceAccountName: keda-operator
```

2. 部署 Pod：

```sh
kubectl apply -f gcloud.yaml
```

3. 執行以下命令檢查 Workload Identity：

```sh
kubectl -n keda exec -it gcloud -- curl -H "Metadata-Flavor: Google" http://169.254.169.254/computeMetadata/v1/instance/service-accounts/
# 執行結果應該要看到 GCP Service Account (stackdriver-keda-sa@${PROJECT_ID}.iam.gserviceaccount.com)
```


## 部署測試用服務

```sh
kubectl apply -f deployment.yaml
kubectl apply -f backendconfig.yaml
kubectl apply -f service.yaml
kubectl apply -f managedcertificate.yaml # 需修改 domain
kubectl apply -f ingress.yaml 
kubectl apply -f keda.yaml # 需修改 projectId
```

## 設定 DNS A Record

## 檢視狀態

```sh
kubectl get po,svc,ing,mcrt,hpa,so
```

## 使用 curl 進行測試

```sh
for i in {0..1000}; do curl -s https://demo.poiblacktea.com > /dev/null ; done
kubectl get po,svc,ing,mcrt,hpa,so
```
