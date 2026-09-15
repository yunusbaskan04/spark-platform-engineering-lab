# 04 --- Jupyter Enterprise Gateway, KernelSpec ve ProcessProxy

## 1. Enterprise Gateway'e neden ihtiyaç duyduk?

Önceki bölümde problem şuydu:

``` text
Jupyter Server
      ↓
Python Kernel
```

Küçük sistemde bu yeterliydi.

Ama ABC Bank'ta kernel'ı:

``` text
Jupyter Server
      ↓
      ?
      ↓
Cluster
      ↓
Python Kernel
```

şeklinde uzakta çalıştırmak istiyoruz.

Enterprise Gateway burada devreye giriyor.

> **Enterprise Gateway, Jupyter kernel'larını uzaktaki yönetilen
> cluster'larda başlatıp yönetmek için kullanılan katmandır.**

Böylece Jupyter ile compute cluster birbirinden ayrılabilir.

------------------------------------------------------------------------

## 2. Büyük resim

``` text
User
 ↓
JupyterLab
 ↓
Jupyter Server
 ↓
Enterprise Gateway
 ↓
KernelSpec
 ↓
ProcessProxy
 ↙        ↘
YARN     Kubernetes
 ↓          ↓
Kernel    Pod
 ↓          ↓
PySpark  Python Kernel
 ↓
Spark
```

Her parçanın görevi farklıdır:

  -----------------------------------------------------------------------
  Parça                               Görevi
  ----------------------------------- -----------------------------------
  JupyterLab                          Kullanıcının notebook arayüzü

  Jupyter Server                      Jupyter tarafındaki sunucu

  Enterprise Gateway                  Remote kernel başlatma/yönetme

  KernelSpec                          Kernel'ın nasıl/nereye
                                      başlatılacağını tarif eder

  ProcessProxy                        Seçilen altyapıya göre kernel
                                      başlatma ve yaşam döngüsünü yönetir

  YARN / Kubernetes                   Cluster kaynaklarını/workload'ları
                                      yönetir

  Kernel                              Python kodunu çalıştırır

  PySpark / Spark                     Veri işleme tarafını yürütür
  -----------------------------------------------------------------------

------------------------------------------------------------------------

## 3. KernelSpec nedir?

KernelSpec'i şimdilik:

> **"Bu kernel nasıl ve nerede başlatılacak?" tarif dosyası**

olarak düşün.

Jupyter'da örneğin:

``` text
kernels/
├── python3/
│   └── kernel.json
├── python_kubernetes/
│   └── kernel.json
└── spark_python_yarn_cluster/
    └── kernel.json
```

gibi farklı kernel tanımları bulunabilir.

Kullanıcı:

``` text
Spark - Python (Kubernetes Mode)
```

seçtiğinde Enterprise Gateway ilgili kernelspec'e bakabilir.

------------------------------------------------------------------------

## 4. Gerçek `kernel.json` bize ne söylüyor?

Kaynakta incelenen Spark + Kubernetes kernelspec'inin ilgili kısmı
kabaca:

``` json
{
  "language": "python",
  "display_name": "Spark - Python (Kubernetes Mode)",
  "metadata": {
    "process_proxy": {
      "class_name": "enterprise_gateway.services.processproxies.k8s.KubernetesProcessProxy",
      "config": {
        "image_name": "elyra/kernel-spark-py:VERSION",
        "executor_image_name": "elyra/kernel-spark-py:VERSION"
      }
    }
  },
  "env": {
    "SPARK_HOME": "/opt/spark",
    "SPARK_OPTS": "..."
  }
}
```

Bunu ezberlemeye gerek yok.

İnsan diline çevirirsek:

> "Ben Python tabanlı bir Spark kernel'ıyım. Kubernetes üzerinde
> çalışacağım. Beni `KubernetesProcessProxy` başlatsın. Şu image'ı
> kullan. Spark için gerekli environment'ı da ayarla."

Kaynakta bu dosyanın `image_name`, `executor_image_name`, `SPARK_HOME`
ve Spark/Kubernetes ayarlarını taşıdığı gösteriliyor.

------------------------------------------------------------------------

## 5. ProcessProxy neden gerekli?

Çünkü Enterprise Gateway'in önünde farklı altyapılar olabilir:

``` text
Enterprise Gateway
       ↓
   "Kernel başlat"
      /         ↓      ↓
   YARN  Kubernetes
```

Fakat YARN ile Kubernetes aynı şekilde çalışmaz.

YARN için:

``` text
YarnClusterProcessProxy
```

Kubernetes için:

``` text
KubernetesProcessProxy
```

kullanılabilir.

ProcessProxy'yi şöyle düşün:

> **"Kernel'ı seçtiğin altyapıda başlat ve yaşam döngüsünü takip et."**

Bu sayede Enterprise Gateway, Jupyter tarafına aynı kernel başlatma
fikrini sunarken alttaki altyapıya göre farklı mekanizma kullanabilir.

------------------------------------------------------------------------

## 6. `run.sh` neden var?

Normal bir Python kernel doğrudan:

``` json
"argv": [
  "python",
  "-m",
  "ipykernel_launcher",
  "-f",
  "{connection_file}"
]
```

gibi başlatılabilir.

Remote Kubernetes senaryosunda ise arada bir launcher gerekir.

Örneğin:

``` json
"argv": [
  "/usr/local/share/jupyter/kernels/spark_python_kubernetes/bin/run.sh",
  "--RemoteProcessProxy.kernel-id",
  "{kernel_id}",
  ...
]
```

Buradaki `run.sh` Python kernel'ın kendisi değildir.

Bir **başlatıcıdır**.

Kaynakta anlatılan mantık:

``` text
kernel.json
    ↓
run.sh
    ↓
Kubernetes üzerinde launch işlemi
    ↓
Remote Kernel
```

------------------------------------------------------------------------

## 7. İki ayrı "launch" olduğunu unutma

Container tabanlı remote kernel yapısında iki farklı başlatma aşaması
vardır:

### 1. Container / Pod'u başlat

``` text
Enterprise Gateway
       ↓
Kubernetes
       ↓
Pod
```

### 2. Pod'un içinde kernel'ı başlat

``` text
Pod
 ↓
Python Kernel
```

Yani:

``` text
Jupyter
 ↓
Enterprise Gateway
 ↓
Kubernetes
 ↓
Pod
 ↓
Python Kernel
```

tek bir "process başlat" adımı değildir.

------------------------------------------------------------------------

## 8. Docker image burada tekrar karşımıza çıkıyor

Şimdi önceki bölümde oluşturduğumuz image'ın neden önemli olduğunu
görebiliriz.

`kernel.json` içerisinde:

``` json
"image_name": "elyra/kernel-spark-py:VERSION"
```

gibi bir bilgi bulunabilir.

Bu şu anlama gelir:

``` text
KernelSpec
    ↓
image_name
    ↓
Spark/Python image
    ↓
Kubernetes Pod
    ↓
Python Kernel
```

Örneğin kendi şirket image'ımızı düşünelim:

``` text
company-pyspark:1.0
```

İçerisinde:

``` text
Python
PySpark
Java
Spark
```

bulunsun.

Kubernetes'teki kernel Pod'u bu image'ı kullanabilir.

İşte burada önceki:

``` text
Docker Image
   ↓
Image Inheritance
```

konumuz gerçek sistemde bir kullanım alanına bağlanıyor.

------------------------------------------------------------------------

## 9. `kernel-pod.yaml.j2` nedir?

Kubernetes'e Pod oluşturmasını söylemek için bir Pod tanımına ihtiyaç
var.

Enterprise Gateway tarafında bunun için bir template bulunabilir:

``` text
kernel-pod.yaml.j2
```

Buradaki `.j2`, Jinja2 template anlamına gelir.

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

gibi bir Pod tanımı üretilebilir.

Mantık:

``` text
kernel-pod.yaml.j2
        ↓
değerleri doldur
        ↓
Pod tanımı
        ↓
Kubernetes API
        ↓
Pod
```

------------------------------------------------------------------------

## 10. Gerçek akışı takip edelim

Kullanıcı JupyterLab'da:

``` text
New → Notebook
```

yapıp:

``` text
Spark - Python (Kubernetes Mode)
```

kernel'ını seçiyor.

Sonra kabaca:

``` text
JupyterLab
    ↓
Jupyter Server
    ↓
Enterprise Gateway
    ↓
kernel.json
    ↓
KubernetesProcessProxy
    ↓
run.sh
    ↓
launch_kubernetes.py
    ↓
kernel-pod.yaml.j2
    ↓
Kubernetes API
    ↓
Pod
    ↓
Docker Image
    ↓
Python Kernel
```

oluşuyor.

Kaynakta gerçek dosyalar üzerinden bu zincirin
`kernel.json → run.sh → launch_kubernetes.py → kernel-pod.yaml.j2 → Kubernetes API → Pod → Python Kernel`
şeklinde takip edildiği gösteriliyor.

------------------------------------------------------------------------

## 11. En önemli nokta: image_name

Burada önceki Spark image bölümümüz doğrudan anlam kazanıyor.

Örneğin:

``` text
company-pyspark:1.0
```

şöyle oluşturulmuş olsun:

``` text
pyspark-notebook
       ↓
company-pyspark
```

ve içinde:

``` text
Python
Java
Spark
PySpark
```

bulunsun.

Kubernetes'e:

> "Bu kernel'ı `company-pyspark:1.0` image'ı ile çalıştır."

dediğimizde:

``` text
Kubernetes
   ↓
Pod
   ↓
company-pyspark:1.0
   ↓
Python Kernel
   ↓
PySpark
   ↓
Spark
```

akışı ortaya çıkar.

Böylece Docker image inheritance artık sadece teorik bir konu değildir;
remote kernel'ın çalışacağı ortamı paketlememize yardımcı olur.

------------------------------------------------------------------------

## 12. YARN ve Kubernetes'te fark nerede?

Enterprise Gateway'in üst tarafı aynı fikri koruyabilir:

``` text
Jupyter
  ↓
Enterprise Gateway
  ↓
"Kernel başlat"
```

Ama aşağıda kullanılan mekanizma değişir.

### Kubernetes

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

### YARN

``` text
YarnClusterProcessProxy
       ↓
YARN
       ↓
Cluster
       ↓
Kernel
```

Kaynakta özellikle bu iki ProcessProxy'nin ilgili backend'lerde kernel
başlatma ve yaşam döngüsü yönetimi için kullanıldığı açıklanıyor.

------------------------------------------------------------------------

## 13. Neden bu mimari faydalı?

ABC Bank yarın:

``` text
YARN → Kubernetes
```

geçişi yaparsa Jupyter kullanıcısının deneyiminin büyük kısmı aynı
kalabilir.

Değişen ana bölüm:

``` text
Enterprise Gateway
        ↓
ProcessProxy
        ↓
Compute backend
```

tarafıdır.

Yani Enterprise Gateway bir **soyutlama katmanı** sağlar.

------------------------------------------------------------------------

## 14. Bu bölümün zihinsel modeli

Şu zinciri anlayabiliyorsan konu oturmuştur:

``` text
Kullanıcı
   ↓
JupyterLab
   ↓
Jupyter Server
   ↓
Enterprise Gateway
   ↓
KernelSpec
   ↓
ProcessProxy
   ↓
Kubernetes / YARN
   ↓
Remote Kernel
   ↓
PySpark
   ↓
Spark
```

Ve Kubernetes özelinde:

``` text
KernelSpec
   ↓
KubernetesProcessProxy
   ↓
run.sh / launcher
   ↓
Kubernetes API
   ↓
Pod
   ↓
Docker Image
   ↓
Python Kernel
```

Bu noktadan sonra artık Kubernetes'in kendisini daha detaylı öğrenmeye
geçebiliriz.
