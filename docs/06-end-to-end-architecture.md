# 06 --- End-to-End: Jupyter → Enterprise Gateway → Kubernetes → Spark

## 1. Artık bütün parçaları birleştirelim

Şimdiye kadar ayrı ayrı gördüğümüz parçalar:

``` text
Docker
Image inheritance
Spark image
Jupyter
Kernel
Enterprise Gateway
KernelSpec
ProcessProxy
Kubernetes
YARN
```

tek bir sistemin farklı katmanları olarak karşımıza çıkıyor.

ABC Bank senaryosunda hedefimiz:

> **Data Engineer'ın Jupyter üzerinden PySpark kodu yazabilmesi, fakat
> kernel'ın büyük compute cluster'ında çalışması.**

------------------------------------------------------------------------

# 2. ABC Bank'ın ortamı

Şirketin:

``` text
Jupyter Server'ları
        +
Büyük compute cluster
```

olduğunu düşünelim.

Compute tarafında Kubernetes kullanalım.

Şirket ayrıca Spark için kendi image'ını hazırlamış olsun:

``` text
company-pyspark:1.0
```

Bu image'ın içinde:

``` text
Python
Java
Spark
PySpark
```

bulunsun.

Image daha önceki inheritance yaklaşımıyla oluşturulmuş olabilir:

``` text
pyspark-notebook
       ↓
company-pyspark:1.0
```

------------------------------------------------------------------------

# 3. Kullanıcı notebook açıyor

Data Engineer browser'dan:

``` text
JupyterLab
```

açıyor.

Sonra:

``` text
New
  ↓
Notebook
  ↓
Spark - Python (Kubernetes Mode)
```

kernel'ını seçiyor.

Henüz Spark kodu çalışmadı.

Sadece:

> **"Ben bu notebook için şu remote kernel'ı istiyorum."**

demiş oldu.

------------------------------------------------------------------------

# 4. Jupyter Server devreye giriyor

Akış:

``` text
Browser
   ↓
JupyterLab
   ↓
Jupyter Server
```

Jupyter Server kernel'ın başlatılması için Enterprise Gateway'e gider.

Artık kernel lokal bir Python process'i olmak zorunda değildir.

------------------------------------------------------------------------

# 5. Enterprise Gateway kernel bilgisini buluyor

Enterprise Gateway:

``` text
KernelSpec
```

üzerinden seçilen kernel'ın tarifine bakar.

Örneğin:

``` text
Spark - Python (Kubernetes Mode)
```

için:

``` json
{
  "metadata": {
    "process_proxy": {
      "class_name":
        "enterprise_gateway.services.processproxies.k8s.KubernetesProcessProxy",
      "config": {
        "image_name": "company-pyspark:1.0"
      }
    }
  }
}
```

gibi bilgiler bulunduğunu düşün.

Enterprise Gateway artık şunu biliyor:

``` text
Kernel tipi → Python / Spark
Backend → Kubernetes
ProcessProxy → KubernetesProcessProxy
Image → company-pyspark:1.0
```

------------------------------------------------------------------------

# 6. ProcessProxy devreye giriyor

Enterprise Gateway:

> "Bu kernel Kubernetes üzerinde çalışacak."

diyor.

Bu yüzden:

``` text
KubernetesProcessProxy
```

kullanılıyor.

ProcessProxy'nin görevi kabaca:

> **Kernel'ı Kubernetes üzerinde başlatmak ve yaşam döngüsünü takip
> etmek.**

Yani:

``` text
Enterprise Gateway
       ↓
KubernetesProcessProxy
       ↓
Kubernetes
```

------------------------------------------------------------------------

# 7. Launcher ne yapıyor?

Remote container tabanlı yapıda arada launcher bulunuyor.

Kaynakta incelenen zincir:

``` text
kernel.json
    ↓
run.sh
    ↓
launch_kubernetes.py
    ↓
kernel-pod.yaml.j2
    ↓
Kubernetes API
```

Buradaki mantığı ezberlemiyoruz.

Sadece:

> **"Kubernetes'te kernel çalıştırmak için gerekli başlatma adımları
> bunlar."**

diye düşünüyoruz.

------------------------------------------------------------------------

# 8. Pod oluşturuluyor

Template üzerinden Kubernetes'e bir Pod tanımı gönderiliyor.

Kavramsal olarak:

``` yaml
apiVersion: v1
kind: Pod
metadata:
  name: yunus-kernel
spec:
  containers:
    - name: kernel
      image: company-pyspark:1.0
```

Kubernetes:

> "Tamam, bu Pod'u cluster üzerinde çalıştıracağım."

diyor.

Artık:

``` text
Kubernetes Cluster
   │
   ├── Node 1
   ├── Node 2
   │    └── yunus-kernel Pod
   │          └── company-pyspark:1.0
   │
   ├── Node 3
   └── Node 4
```

gibi bir yapı oluşabilir.

------------------------------------------------------------------------

# 9. Image'ın burada ne işi var?

Pod'un içinde:

``` text
company-pyspark:1.0
```

çalışıyor.

Bu image:

``` text
Python
Java
Spark
PySpark
```

gibi gerekli ortamı içeriyor.

Dolayısıyla remote kernel'ın çalışma ortamı baştan tanımlı.

Burada önceki Docker konumuz doğrudan devreye giriyor:

``` text
Dockerfile
   ↓
Image
   ↓
Kubernetes Pod
   ↓
Remote Kernel
```

Image inheritance ise image'ın nasıl üretildiğini açıklıyor:

``` text
pyspark-notebook
       ↓
company-pyspark
       ↓
Kubernetes Pod
```

------------------------------------------------------------------------

# 10. Kernel gerçekten çalışmaya başlıyor

Pod çalıştıktan sonra container'ın içinde Python kernel başlıyor.

Artık:

``` text
Jupyter Server
```

ile:

``` text
Remote Python Kernel
```

ayrı yerlerde olabilir.

Büyük resim:

``` text
USER
  ↓
JupyterLab
  ↓
Jupyter Server
  ↓
Enterprise Gateway
  ↓
KubernetesProcessProxy
  ↓
Kubernetes API
  ↓
Pod
  ↓
company-pyspark:1.0
  ↓
Python Kernel
```

------------------------------------------------------------------------

# 11. Şimdi notebook'ta kod yazalım

Data Engineer:

``` python
from pyspark.sql import SparkSession

spark = SparkSession.builder     .appName("FraudDetection")     .getOrCreate()

df = spark.read.parquet("s3://bank-data/transactions")

result = df.groupBy("customer_id").count()

result.show()
```

yazıyor.

Bu kodun çalıştığı Python process:

``` text
Jupyter Server'ın yanında
```

olmak zorunda değil.

Remote kernel çalışıyorsa:

``` text
Kubernetes Pod
   ↓
Python Kernel
   ↓
PySpark
```

tarafında çalışır.

------------------------------------------------------------------------

# 12. Spark burada nerede?

Burada bir başka önemli ayrım var.

``` text
Jupyter
```

kullanıcının çalışma ortamıdır.

``` text
Enterprise Gateway
```

kernel'ın uzakta başlatılmasını yönetir.

``` text
Kubernetes
```

Pod/container'ın çalıştığı cluster altyapısını yönetir.

``` text
Python Kernel
```

Python kodunu çalıştırır.

``` text
PySpark
```

Python tarafından Spark ile konuşmayı sağlar.

``` text
Spark
```

büyük veri işleme tarafıdır.

Dolayısıyla:

``` text
Jupyter ≠ Enterprise Gateway
≠ Kubernetes
≠ Spark
```

------------------------------------------------------------------------

# 13. YARN kullansaydık?

Aynı başlangıç:

``` text
User
 ↓
JupyterLab
 ↓
Jupyter Server
 ↓
Enterprise Gateway
```

olarak kalabilir.

Fakat sonrasında:

``` text
YarnClusterProcessProxy
       ↓
YARN
       ↓
Cluster
       ↓
Kernel
```

akışı kullanılır.

Kubernetes'te:

``` text
KubernetesProcessProxy
       ↓
Kubernetes API
       ↓
Pod
       ↓
Container
       ↓
Kernel
```

kullanılıyordu.

Enterprise Gateway'in soyutlama sağlamasının önemli noktalarından biri
burada görülüyor.

------------------------------------------------------------------------

# 14. Bütün sistemin tek resmi

``` text
                         USER
                           │
                           ↓
                      JupyterLab
                           │
                           ↓
                     Jupyter Server
                           │
                           ↓
                 ┌───────────────────┐
                 │ Enterprise Gateway│
                 └─────────┬─────────┘
                           │
                      KernelSpec
                           │
                      ProcessProxy
                     /                                 ↓               ↓
             Kubernetes            YARN
                    │               │
               Kubernetes        YARN Cluster
                    │               │
                   Pod             Kernel
                    │
                Container
                    │
          company-pyspark:1.0
                    │
              Python Kernel
                    │
                 PySpark
                    │
                  Spark
                    │
                Big Data
```

------------------------------------------------------------------------

# 15. Projenin parçaları artık nasıl bağlanıyor?

Bu noktada projenin başlangıcındaki konuların neden seçildiği de ortaya
çıkıyor.

### Docker Image Inheritance

Spark çalışma ortamını tekrar kullanılabilir image'lar halinde
paketlememizi sağlar.

### Spark Image

Spark'ın çalışacağı runtime ortamını temsil eder.

### Jupyter Image

Kullanıcıya notebook tabanlı çalışma ortamı sağlar.

### Kernel

Notebook kodunu gerçekten çalıştıran süreçtir.

### Enterprise Gateway

Kernel'ı Jupyter'ın bulunduğu yerden bağımsız olarak uzaktaki compute
altyapısında başlatmayı/yönetmeyi sağlar.

### Kubernetes / YARN

Kernel'ın çalışacağı cluster altyapısını yönetir.

### Spark

Kernel içerisinden başlatılan büyük veri işleme tarafıdır.

------------------------------------------------------------------------

# 16. Şu anki proje zihinsel modelimiz

Bütün projeyi tek cümlede:

> **Kullanıcı Jupyter üzerinden kod yazar; Enterprise Gateway seçilen
> kernel'ı Kubernetes/YARN gibi bir cluster üzerinde başlatır; kernel
> gerekli Docker image'ı kullanarak çalışır ve PySpark üzerinden Spark
> işlerini yürütür.**

şeklinde özetleyebiliriz.

Ana akış:

``` text
Jupyter
   ↓
Remote Kernel
   ↓
Cluster
   ↓
Spark
```

Bunun nasıl gerçekleştirildiği ise:

``` text
Jupyter
   ↓
Enterprise Gateway
   ↓
KernelSpec
   ↓
ProcessProxy
   ↓
Kubernetes / YARN
   ↓
Kernel
```

ve Kubernetes özelinde:

``` text
ProcessProxy
   ↓
Launcher
   ↓
Pod
   ↓
Docker Image
   ↓
Kernel
```

şeklindedir.

------------------------------------------------------------------------

## Bundan sonra ne öğreniyoruz?

Bu dört dokümanla teorik resmi tamamladık.

Bundan sonraki adım artık **uygulama** tarafı olabilir:

``` text
01 Docker & Image Inheritance
02 Spark Images
03 Jupyter & Kernels
04 Enterprise Gateway
05 Kubernetes & YARN
06 End-to-End Architecture
             ↓
      IMPLEMENTATION
```

Özellikle Kubernetes tarafında artık:

``` text
Pod
Node
Container
Kubernetes API
Deployment
Service
```

gibi kavramları sıfırdan öğrenip, ardından bu projedeki gerçek
`kernel.json`, launcher ve Pod tanımlarını uygulamalı olarak
kurabiliriz.
