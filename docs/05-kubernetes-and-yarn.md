# 05 --- Kubernetes ve YARN: Kernel Gerçekte Nerede Çalışıyor?

Bu bölümün amacı Kubernetes veya YARN'ı baştan sona öğretmek değil.

Önce şu soruyu tamamen netleştirmek:

> **Jupyter'da bir notebook açtım. Kernel olarak Kubernetes'i seçtim.
> Yazdığım Python kodu gerçekte nereye gidiyor, orada ne çalışıyor ve
> Docker image bunun neresinde?**

Bunu anladıktan sonra `Pod`, `Container`, `Image`, `Launcher`,
`kernel-pod.yaml.j2` ve `Kubernetes API` gibi isimler anlamlı hale
geliyor.

------------------------------------------------------------------------

## 1. Kubernetes neden var?

ABC Bank'ın veri platformunda 10 güçlü sunucu olduğunu düşünelim:

``` text
Server 1
Server 2
Server 3
...
Server 10
```

Bunlar ortak bilgisayar kaynaklarıdır.

Aynı anda:

``` text
Yunus  → Spark işi
Ahmet  → Python analizi
Ayşe   → ML işi
Mehmet → başka bir Spark işi
```

çalıştırmak istiyoruz.

Doğal problem:

> Bu işler hangi sunucuda çalışacak?

Bunu insanların elle yönetmesini istemeyiz. Cluster üzerinde çalışan
uygulamaların ve kaynakların yönetilmesi gerekir.

Kubernetes bu tür container/workload yönetimi için kullanılan
platformlardan biridir.

------------------------------------------------------------------------

## 2. Kubernetes'i şimdilik nasıl düşünmeliyiz?

Çok basitleştirirsek:

> **Kubernetes, birden fazla makineden oluşan bir cluster üzerinde
> uygulamaları/container'ları çalıştıran ve yöneten sistemdir.**

Kabaca:

``` text
              Kubernetes Cluster
                     │
          ┌──────────┼──────────┐
          ↓          ↓          ↓
       Node 1     Node 2     Node 3
          │          │          │
       Container  Container  Container
```

Buradaki **Node**, cluster'daki makinelerden biridir.

Şimdilik:

``` text
Node = üzerinde workload çalıştırabileceğimiz makine
```

demek yeterli.

------------------------------------------------------------------------

## 3. Pod nedir?

Kubernetes'te karşımıza çıkan temel kavramlardan biri **Pod**'dur.

İlk aşamada:

> **Pod, Kubernetes'in çalıştırdığı temel workload birimidir.**

Şöyle düşün:

``` text
Kubernetes
    ↓
   Pod
    ↓
Container
```

Bizim senaryomuzda bir Pod'un içinde Python kernel'ın çalıştığı
container bulunabilir:

``` text
Pod
└── Container
    └── Python Kernel
```

Pod ile container aynı şey değildir.

-   **Pod:** Kubernetes'in yönettiği çalışma birimi.
-   **Container:** Uygulamanın gerçekten çalıştığı ortam.

Bu projede çoğunlukla:

``` text
1 Pod
└── 1 Container
```

gibi basit bir yapı göreceğiz.

------------------------------------------------------------------------

## 4. Docker Image burada nerede?

Önceki bölümden bildiğimiz Docker image'ı hatırlayalım.

Örneğin:

``` text
company-pyspark:1.0
```

image'ımız olsun.

İçerisinde:

``` text
Python
PySpark
Java
Spark
pandas
numpy
```

gibi gerekli yazılımlar bulunabilir.

Image çalışan bir process değildir.

Kabaca:

``` text
Image
  │
  │ çalıştır
  ↓
Container
```

Kubernetes'te ise bunu:

``` text
Pod
└── Container
    └── company-pyspark:1.0 image'ından oluşturulan çalışma ortamı
```

şeklinde düşünebiliriz.

Daha doğru ifadeyle Kubernetes, Pod içindeki container için hangi
image'ın kullanılacağını belirtir:

``` yaml
containers:
  - name: kernel
    image: company-pyspark:1.0
```

Yani image'ın görevi:

> **Container'ın çalışacağı yazılım ortamını sağlamak.**

------------------------------------------------------------------------

## 5. Peki Jupyter'da yazdığım kod nereye gidiyor?

Asıl önemli nokta burası.

Sen JupyterLab'da:

``` python
df = spark.read.parquet("transactions.parquet")
```

yazdın.

Remote Kubernetes kernel kullanıyorsan kodu çalıştıran Python process'i
JupyterLab'ın kendisi değildir.

Kabaca:

``` text
Sen
 ↓
JupyterLab
 ↓
Jupyter Server
 ↓
Enterprise Gateway
 ↓
Kubernetes
 ↓
Pod
 ↓
Container
 ↓
Python Kernel
```

Kodun çalıştığı yer:

``` text
Pod
└── Container
    └── Python Kernel
```

Yani:

> **Notebook'ta yazdığın Python kodu remote kernel'a gönderilir ve
> Python kernel tarafından Kubernetes tarafında çalıştırılır.**

JupyterLab burada kullanıcının notebook ile çalıştığı arayüzdür.

------------------------------------------------------------------------

## 6. `import pyspark` nerede çalışıyor?

Notebook'ta:

``` python
from pyspark.sql import SparkSession
```

yazdığını düşün.

Bu import'u yapan Python:

``` text
Remote Python Kernel
```

olur.

Dolayısıyla `pyspark` paketinin kernel'ın çalıştığı environment'ta
bulunması gerekir.

Örneğin:

``` text
Kubernetes Pod
└── Container
    └── company-pyspark:1.0
         ├── Python
         ├── PySpark
         ├── Java
         └── Spark
```

varsa:

``` python
import pyspark
```

çalışabilir.

------------------------------------------------------------------------

## 7. Jupyter Server'ın içinde PySpark olmak zorunda mı?

**Remote kernel kullanıyorsak, hayır.**

Çünkü:

``` text
Jupyter Server
```

kodu çalıştıran Python process değildir.

Kodu çalıştıran:

``` text
Remote Python Kernel
```

dır.

Dolayısıyla şöyle bir yapı gayet mümkündür:

``` text
Jupyter Server
├── JupyterLab
└── Jupyter bileşenleri

Kubernetes
└── Pod
    └── Container
        └── company-pyspark:1.0
             ├── Python
             ├── PySpark
             ├── Spark
             └── Java
```

Notebook'ta:

``` python
import pyspark
```

dediğinde import remote kernel'ın environment'ında aranır.

------------------------------------------------------------------------

## 8. `pandas` için de aynı şey geçerli

Notebook'ta:

``` python
import pandas as pd
```

yazıyorsan `pandas` paketinin de **kodu çalıştıran kernel'ın
environment'ında** bulunması gerekir.

Örneğin image:

``` text
company-pyspark:1.0
├── Python
├── pandas
├── PySpark
├── Spark
└── Java
```

içeriyorsa import çalışabilir.

Yoksa:

``` text
ModuleNotFoundError
```

alırsın.

Bu yüzden image'ın amacı sadece Spark kurmak değildir.

Asıl amaç:

> **Kernel'ın ihtiyaç duyduğu çalışma ortamını paketlemek.**

------------------------------------------------------------------------

## 9. O zaman Docker image neden önemli?

Her yeni kernel açıldığında:

``` text
PySpark kur
pandas kur
Java kur
Spark kur
ayarları yap
```

demek istemeyiz.

Bunun yerine:

``` text
company-pyspark:1.0
```

gibi bir image oluştururuz.

Bu image gerekli ortamı içerir.

Sonra Kubernetes:

``` text
Pod
└── Container
    └── company-pyspark:1.0
```

şeklinde bu ortamı kullanarak kernel'ı çalıştırabilir.

Böylece her kernel aynı tanımlı environment ile başlayabilir.

İşte önceki **Docker Image Inheritance** konumuz burada gerçek bir
kullanım alanına dönüşüyor:

``` text
base-notebook
      ↓
data-science
      ↓
pyspark
      ↓
company-pyspark
      ↓
Kubernetes Pod
      ↓
Remote Kernel
```

------------------------------------------------------------------------

## 10. Kubernetes API nedir?

Enterprise Gateway'in Kubernetes'e:

> "Şu Pod'u oluştur."

demesi gerekiyor.

Bunu Kubernetes'in **API'si** üzerinden yapabilir.

Basit düşün:

``` text
Enterprise Gateway
        │
        │ API isteği
        ↓
Kubernetes API
        ↓
Kubernetes
        ↓
Pod oluştur
```

Yani Kubernetes API, Kubernetes'e programatik olarak komut vermemizi
sağlayan arayüzdür.

------------------------------------------------------------------------

## 11. `kernel-pod.yaml.j2` nedir?

Bu dosya adı ilk bakışta karmaşık görünüyor.

Üç parçaya ayıralım:

``` text
kernel-pod
yaml
.j2
```

`kernel-pod`:

> Kernel için oluşturulacak Pod'u tarif ediyor.

`yaml`:

> Bu tarif YAML formatında.

`.j2`:

> Dosyanın Jinja2 template'i olduğunu gösteriyor.

Dolayısıyla:

> **`kernel-pod.yaml.j2`, Kubernetes'e gönderilecek Pod tanımının
> şablonudur.**

Örneğin kavramsal olarak:

``` yaml
apiVersion: v1
kind: Pod

metadata:
  name: {{ kernel_id }}

spec:
  containers:
    - name: kernel
      image: {{ image_name }}
```

Buradaki:

``` text
{{ kernel_id }}
{{ image_name }}
```

değişkenlerini düşün.

Enterprise Gateway gerekli değerleri koyar.

Örneğin:

``` text
kernel_id  = yunus-kernel
image_name = company-pyspark:1.0
```

olursa ortaya kabaca:

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

çıkar.

Sonra bu Pod tanımı Kubernetes API'ye gönderilir.

Kaynakta da `kernel-pod.yaml.j2` dosyasının Pod tanımının template'i
olduğu ve image bilgisinin bu tanımda yer aldığı gösteriliyor.


------------------------------------------------------------------------

## 12. Launcher nedir?

**Launcher = başlatıcı.**

Bizim senaryomuzda remote kernel'ı çalıştırmak için birden fazla adım
gerekiyor.

Bu nedenle arada kernel'ı başlatma sürecini yönlendiren script/program
bulunuyor.

Örneğin:

``` text
run.sh
```

bir launcher olabilir.

Bunu:

> **"Remote kernel'ı başlatma sürecini tetikleyen script"**

olarak düşün.

`run.sh` Python kernel'ın kendisi değildir.

Kaynakta remote container yapısında iki ayrı launch aşaması özellikle
belirtiliyor:

``` text
1. Pod / container'ı başlat
2. Container'ın içinde gerçek kernel'ı başlat
```

------------------------------------------------------------------------

## 13. `run.sh` ve `launch_kubernetes.py` neden var?

Burada dosya isimlerini ezberlemiyoruz.

Sadece zinciri anlamaya çalışıyoruz:

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
    ↓
Pod
```

Kabaca:

-   `kernel.json` → hangi kernel'ın nasıl başlatılacağını tarif eder.
-   `run.sh` → başlatma sürecini tetikler.
-   `launch_kubernetes.py` → Kubernetes tarafındaki launch işlemini
    gerçekleştiren kodun parçasıdır.
-   `kernel-pod.yaml.j2` → oluşturulacak Pod'un şablonudur.
-   Kubernetes API → Pod'un oluşturulmasını Kubernetes'e iletir.

Kaynakta gerçek dosyaların bu zincir üzerinden takip edildiği
gösteriliyor. 

------------------------------------------------------------------------

## 14. Şimdi bütün akışı gerçekten anlayarak okuyalım

Kullanıcı JupyterLab'da:

``` text
New → Notebook
```

diyor.

Kernel olarak:

``` text
Spark - Python (Kubernetes Mode)
```

seçiyor.

Sonra:

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
run.sh / launcher
   ↓
launch_kubernetes.py
   ↓
kernel-pod.yaml.j2
   ↓
Kubernetes API
   ↓
Pod
   ↓
Container
   ↓
company-pyspark:1.0
   ↓
Python Kernel
```

oluşuyor.

Artık kullanıcı notebook'a:

``` python
import pandas as pd
from pyspark.sql import SparkSession
```

yazdığında:

``` text
Python Kernel
     ↓
Python environment
     ├── pandas
     ├── PySpark
     ├── Spark
     └── Java
```

üzerinden kod çalışıyor.

------------------------------------------------------------------------

## 15. YARN bu resmin neresinde?

YARN'ı burada Kubernetes'in alternatifi olarak düşünmek yeterli.

Enterprise Gateway'in arkasında farklı compute altyapıları olabilir:

``` text
                 Enterprise Gateway
                    /                             ↓            ↓
             Kubernetes        YARN
```

Kubernetes tarafında:

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

YARN tarafında ise:

``` text
YarnClusterProcessProxy
       ↓
YARN
       ↓
Cluster
       ↓
Kernel
```

kullanılabilir.

Kaynakta YARN için `YarnClusterProcessProxy`, Kubernetes için
`KubernetesProcessProxy` kullanıldığı ve ilgili backend'lerde kernel
başlatma/yönetme görevini üstlendikleri anlatılıyor.


Bu nedenle:

``` text
Kubernetes ≠ YARN
```

ama ikisi de remote kernel'ın çalışacağı altyapı olabilir.

------------------------------------------------------------------------

# 16. En önemli ayrım

Bu projede artık şu tabloyu net bil:

  -----------------------------------------------------------------------
  Parça                               Görevi
  ----------------------------------- -----------------------------------
  **JupyterLab**                      Notebook arayüzü

  **Jupyter Server**                  Jupyter tarafındaki sunucu

  **Kernel**                          Python kodunu gerçekten çalıştıran
                                      process

  **Enterprise Gateway**              Remote kernel'ın
                                      başlatılmasını/yönetilmesini sağlar

  **KernelSpec**                      Kernel'ın nasıl başlatılacağını
                                      tarif eder

  **ProcessProxy**                    Kernel'ı seçilen altyapıda başlatma
                                      mekanizması

  **Launcher / `run.sh`**             Başlatma sürecini tetikler

  **Kubernetes**                      Container/workload'ları cluster
                                      üzerinde çalıştırır ve yönetir

  **Pod**                             Kubernetes'in workload çalışma
                                      birimi

  **Container**                       Uygulamanın gerçekten çalıştığı
                                      ortam

  **Docker Image**                    Container'ın ihtiyaç duyduğu
                                      yazılım ortamının paketlenmiş hali

  **PySpark**                         Python ile Spark kullanmamızı
                                      sağlar

  **Spark**                           Dağıtık veri işleme motoru
  -----------------------------------------------------------------------

------------------------------------------------------------------------

# 17. Son bir örnek

Notebook'a:

``` python
import pandas as pd
from pyspark.sql import SparkSession

spark = SparkSession.builder.getOrCreate()
```

yazdık.

Kernel:

``` text
Spark - Python (Kubernetes Mode)
```

olsun.

O zaman:

``` text
JupyterLab
    ↓
Jupyter Server
    ↓
Enterprise Gateway
    ↓
KubernetesProcessProxy
    ↓
Kubernetes
    ↓
Pod
    ↓
Container
    ↓
company-pyspark:1.0
    ↓
Python Kernel
    ↓
import pandas
import pyspark
```

Burada:

``` text
JupyterLab
```

kodu çalıştırmıyor.

``` text
Pod
```

tek başına kodu çalıştırmıyor.

``` text
Image
```

de kodu çalıştırmıyor.

**Kodu gerçekten çalıştıran şey Pod'un içindeki container'da çalışan
Python kernel'dır.**

Image ise o kernel'ın ihtiyaç duyduğu ortamı sağlar.

------------------------------------------------------------------------

# 18. Bu bölümün zihinsel modeli

``` text
JUPYTER TARAFI
────────────────────────

User
 ↓
JupyterLab
 ↓
Jupyter Server
 ↓
Enterprise Gateway
        │
        ↓
────────────────────────
KUBERNETES TARAFI

Kubernetes API
 ↓
Pod
 ↓
Container
 ↓
company-pyspark:1.0
 ↓
Python Kernel
 ↓
Notebook'taki Python kodu
```

Ve image'ın rolü:

``` text
Dockerfile
    ↓
Docker Image
    ↓
Container
    ↓
Python + PySpark + Spark + diğer kütüphaneler
    ↓
Python Kernel
```

**Tek cümlelik özet:**

> JupyterLab'da yazdığın kod, remote kernel kullanıyorsan
> Kubernetes'teki Pod'un içindeki container'da çalışan Python kernel
> tarafından çalıştırılır; Docker image ise bu kernel'ın çalışacağı
> Python/PySpark/Spark ve diğer bağımlılıkları içeren çalışma ortamını
> sağlar.

Bu ayrımı oturttuğumuzda `Pod`, `Container`, `Image`, `Kernel`,
`Launcher` ve `kernel-pod.yaml.j2` artık birbirine karışmaz.
