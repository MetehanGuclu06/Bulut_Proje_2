# 🍽️ Ember & Oak — Restoran Rezervasyon Sistemi

> Docker ve Kubernetes ile containerize edilmiş, GKE üzerinde çalışan full-stack web uygulaması.

---

## 📋 İçindekiler

- [Proje Hakkında](#proje-hakkında)
- [Uygulama Mimarisi](#uygulama-mimarisi)
- [Kubernetes Mimarisi](#kubernetes-mimarisi)
- [CI/CD Pipeline](#cicd-pipeline)
- [Kurulum ve Çalıştırma](#kurulum-ve-çalıştırma)
- [Kubernetes Komutları](#kubernetes-komutları)
- [Dosya Yapısı](#dosya-yapısı)

---

## Proje Hakkında

Ember & Oak, bir restoranın rezervasyon yönetim sistemidir. Kullanıcılar web arayüzü üzerinden masa rezervasyonu yapabilir, admin panelinden tüm rezervasyonlar yönetilebilir.

**Kullanılan Teknolojiler:**

| Katman | Teknoloji |
|--------|-----------|
| Backend | Python / Flask |
| Veritabanı | PostgreSQL |
| Containerization | Docker |
| Orchestration | Kubernetes (GKE) |
| CI/CD | Google Cloud Build |
| Image Registry | Google Container Registry |

---

## Uygulama Mimarisi

```
┌─────────────────────────────────────────────┐
│                  Kullanıcı                   │
│              (Web Tarayıcı)                  │
└───────────────────┬─────────────────────────┘
                    │ HTTP
                    ▼
┌─────────────────────────────────────────────┐
│           Kubernetes Service                 │
│         (NodePort / LoadBalancer)            │
└───────────────────┬─────────────────────────┘
                    │
        ┌───────────┴───────────┐
        ▼                       ▼
┌──────────────┐        ┌──────────────┐
│  Flask Pod 1 │        │  Flask Pod 2 │   ← 2 replika
│  (app:v1)    │        │  (app:v1)    │
└──────┬───────┘        └──────┬───────┘
        └───────────┬──────────┘
                    │ ClusterIP (internal)
                    ▼
┌─────────────────────────────────────────────┐
│            PostgreSQL Pod                    │
│         (postgres-service:5432)              │
└───────────────────┬─────────────────────────┘
                    │
┌─────────────────────────────────────────────┐
│         PersistentVolumeClaim                │
│         (1Gi disk — veriler kalıcı)          │
└─────────────────────────────────────────────┘
```

---

## Kubernetes Mimarisi

### Bileşenler ve Görevleri

**Deployment (`deployment.yaml`)**
- Flask uygulamasının 2 kopyasını çalıştırır
- RollingUpdate stratejisi ile sıfır kesinti güncellemesi sağlar
- Liveness ve Readiness probe ile sağlık kontrolü yapar

**Service (`restoran-service`)**
- Dış dünyadan uygulamaya erişim sağlar
- Gelen trafiği sağlıklı pod'lara dağıtır (load balancing)
- GKE'de LoadBalancer, Minikube'de NodePort tipi kullanılır

**PostgreSQL Deployment (`postgres.yaml`)**
- Veritabanı sunucusunu çalıştırır
- ClusterIP service ile sadece iç ağdan erişilebilir
- PVC üzerinden kalıcı depolama kullanır

**PersistentVolume + PVC (`pv-pvc.yaml`)**
- PostgreSQL verilerinin pod silinse bile kaybolmamasını sağlar
- 1GB disk alanı tahsis eder

**NetworkPolicy (`networkpolicy.yaml`)**
- PostgreSQL'e sadece Flask pod'larının erişmesine izin verir
- Diğer tüm erişimler engellenir
- Güvenlik katmanı oluşturur

---

## CI/CD Pipeline

```
GitHub'a push yap
        │
        ▼
Cloud Build tetiklenir
        │
        ├── Adım 1: Docker image build
        │         docker build -t gcr.io/$PROJECT_ID/restoran-app:$SHA
        │
        ├── Adım 2: Image'ı GCR'a push
        │         docker push gcr.io/$PROJECT_ID/restoran-app:$SHA
        │
        ├── Adım 3: GKE'ye bağlan
        │         gcloud container clusters get-credentials
        │
        ├── Adım 4: Rolling Update deploy
        │         kubectl set image deployment/restoran-app ...
        │
        └── Adım 5: Deploy durumunu kontrol et
                  kubectl rollout status deployment/restoran-app
```

Her `git push` otomatik olarak yeni versiyonu production'a deploy eder.

---

## Kurulum ve Çalıştırma

### Ön Gereksinimler
- Docker Desktop
- Minikube
- kubectl
- (GKE için) Google Cloud CLI

### Minikube ile Kurulum

```bash
# 1. Minikube'u başlat
minikube start --driver=docker --cpus=2 --memory=4096

# 2. Minikube'un Docker ortamını kullan
eval $(minikube docker-env)

# 3. Docker image'ı build et
docker build -t restoran-app:latest ./app

# 4. Kubernetes kaynaklarını oluştur
kubectl apply -f k8s/pv-pvc.yaml
kubectl apply -f k8s/postgres.yaml
kubectl apply -f k8s/networkpolicy.yaml
kubectl apply -f k8s/deployment.yaml

# 5. Pod'ların hazır olmasını bekle
kubectl get pods -w

# 6. Uygulamaya erişim URL'ini al
minikube service restoran-service --url
```

### GKE ile Kurulum

```bash
# 1. GKE cluster oluştur
gcloud container clusters create restoran-cluster \
  --num-nodes=2 \
  --region=us-central1

# 2. Cluster'a bağlan
gcloud container clusters get-credentials restoran-cluster --region=us-central1

# 3. Docker image build ve push
docker build -t gcr.io/$PROJECT_ID/restoran-app:v1 ./app
docker push gcr.io/$PROJECT_ID/restoran-app:v1

# 4. deployment.yaml'daki image adresini güncelle, sonra:
kubectl apply -f k8s/

# 5. Dış IP'yi bekle
kubectl get service restoran-service -w
```

---

## Kubernetes Komutları

### Durum Kontrol

```bash
# Tüm pod'ları listele
kubectl get pods

# Tüm servisleri listele
kubectl get services

# Pod loglarını gör
kubectl logs -l app=restoran-app

# Deployment detayı
kubectl describe deployment restoran-app
```

### Scaling — Ölçekleme

```bash
# 5 kopyaya çıkar
kubectl scale deployment restoran-app --replicas=5

# Tekrar 2'ye indir
kubectl scale deployment restoran-app --replicas=2

# Otomatik ölçekleme (HPA)
kubectl autoscale deployment restoran-app --min=2 --max=10 --cpu-percent=70
```

### Rolling Update — Güncelleme

```bash
# Yeni versiyon deploy et
kubectl set image deployment/restoran-app restoran-app=restoran-app:v2

# Güncelleme durumunu izle
kubectl rollout status deployment/restoran-app

# Güncelleme geçmişini gör
kubectl rollout history deployment/restoran-app
```

### Rollback — Geri Alma

```bash
# Bir önceki versiyona dön
kubectl rollout undo deployment/restoran-app

# Belirli bir versiyona dön
kubectl rollout undo deployment/restoran-app --to-revision=1
```

---

## Dosya Yapısı

```
restoran-proje/
├── app/
│   ├── app.py                  # Flask uygulaması
│   ├── requirements.txt        # Python bağımlılıkları
│   └── templates/
│       ├── index.html          # Ana sayfa
│       ├── randevu.html        # Rezervasyon formu
│       ├── basarili.html       # Başarı sayfası
│       └── admin.html          # Admin paneli
├── k8s/
│   ├── deployment.yaml         # Flask Deployment + Service
│   ├── postgres.yaml           # PostgreSQL Deployment + Service
│   ├── pv-pvc.yaml             # PersistentVolume + PVC
│   └── networkpolicy.yaml      # Ağ güvenlik politikası
├── Dockerfile                  # Docker image tarifi
├── cloudbuild.yaml             # CI/CD pipeline (GKE için)
└── README.md                   # Bu dosya
```

---

## Sunum Akışı (7 Dakika)

1. **Uygulama Tanıtımı** (1 dk) — Siteyi tarayıcıda aç, randevu al
2. **Kubernetes Mimarisi** (1 dk) — `kubectl get pods,services` çalıştır
3. **PV/PVC Gösterimi** (1 dk) — `kubectl get pv,pvc` çalıştır
4. **NetworkPolicy** (30 sn) — `kubectl describe networkpolicy` çalıştır
5. **Scaling** (1 dk) — `kubectl scale --replicas=5` canlı göster
6. **Rolling Update** (1 dk) — Kod değiştir, push et, otomatik deploy izle
7. **Rollback** (30 sn) — `kubectl rollout undo` çalıştır
8. **CI/CD Pipeline** (1 dk) — Cloud Build dashboard göster
