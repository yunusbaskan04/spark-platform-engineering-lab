# 02 --- Spark Image Yapısı

## 1. Önce görseli okuyalım

Bu yapıyı anlamanın en kolay yolu, her katmanda şu soruyu sormaktır:

> **"Bu katman bir üst katmana ne ekliyor?"**

``` text
eclipse-temurin:jre (LTS)
              ↓
         spark-base
              ↓
            spark
           ↙    ↘
      spark-py   spark-r
```

Bu bir Docker **image inheritance** ağacıdır.

Yani alttaki image, üstteki image'ı temel alarak oluşturulur ve kendi
ihtiyacı olan parçaları ekler.

------------------------------------------------------------------------

# 2. En altta neden `eclipse-temurin:jre` var?

Spark'ın temel çalışma ortamı JVM dünyasındadır. Bu nedenle Spark
image'ının en altında Java runtime bulunması mantıklıdır.

``` text
eclipse-temurin:jre
        ↓
     Java Runtime
```

Burada henüz Spark yok.

Sadece:

> **"Spark'ın çalışabilmesi için gerekli Java ortamını hazırla."**

diyoruz.

Bunu bir arabanın motorunun çalışması için gereken temel mekanizma gibi
düşünebiliriz. Henüz araba yok; sadece temel çalışma altyapısı var.

------------------------------------------------------------------------

# 3. `spark-base` ne yapıyor?

Java'nın üzerine Spark için ortak altyapıyı ekliyoruz:

``` text
eclipse-temurin:jre
          ↓
     spark-base
```

Buradaki fikir:

> **Spark kullanan farklı image'ların ortak ihtiyaçlarını tek yerde
> hazırlamak.**

Örneğin Spark'ın dağıtımını image içine koymak ve ortak çalışma ortamını
hazırlamak gibi işler burada bulunabilir.

Böylece `spark-py` ve `spark-r` ayrı ayrı aynı Spark altyapısını kurmak
zorunda kalmaz.

------------------------------------------------------------------------

# 4. `spark` katmanına geldiğimizde ne değişiyor?

Artık:

``` text
eclipse-temurin:jre
          ↓
     spark-base
          ↓
        spark
```

noktasındayız.

Burada elimizde temel anlamda:

> **"Çalıştırılabilir bir Spark ortamı"**

bulunuyor.

Kabaca şöyle düşünebiliriz:

``` text
spark image

/opt/spark/
├── bin/
├── jars/
├── python/
├── examples/
└── ...
```

Buradaki önemli nokta:

> **Spark'ın Jupyter'a ihtiyacı yoktur.**

Örneğin bir Spark job'ı:

``` text
spark-submit
     ↓
Spark
```

şeklinde çalıştırılabilir.

Dolayısıyla bu image'ı sadece notebook çalıştırmak için düşünmemeliyiz.

------------------------------------------------------------------------

# 5. Asıl önemli nokta: Neden iki dala ayrılıyor?

Şimdi görselin en önemli kısmına geldik:

``` text
             spark
            /               ↓      ↓
      spark-py   spark-r
```

Burada `spark-py` ile `spark-r` birbirinin üzerine kurulmuş değildir.

İkisi de:

``` text
spark
```

image'ından türeyen **kardeş image'lardır.**

Sebep şu:

Spark'ın ortak runtime'ı aynıdır fakat Spark'ı kullanacağımız dil tarafı
farklı olabilir.

### Python tarafı

``` text
spark
  ↓
Python / PySpark desteği
  ↓
spark-py
```

### R tarafı

``` text
spark
  ↓
R / SparkR desteği
  ↓
spark-r
```

Böylece ortak olan şeyi tekrar kurmuyoruz.

------------------------------------------------------------------------

# 6. Peki `spark-py` tam olarak ne?

İsmi biraz yanıltıcı olabilir.

`spark-py`:

> **"Spark'ın Python üzerinden kullanılabileceği bir Spark container
> ortamı"**

olarak düşünülebilir.

Yani kabaca:

``` text
spark-py
   │
   ├── Java
   ├── Spark
   ├── Spark runtime
   └── Python / PySpark desteği
```

bulunur.

Örneğin container içerisinde Python ile bir Spark application
çalıştırabiliriz:

``` python
from pyspark.sql import SparkSession

spark = SparkSession.builder     .appName("TransactionAnalysis")     .getOrCreate()

df = spark.read.parquet("/data/transactions")

df.groupBy("customer_id").count().show()
```

Burada Jupyter olmak zorunda değildir.

Bu kod:

``` text
Spark application
       ↓
spark-py container
       ↓
Spark
```

şeklinde çalışabilir.

------------------------------------------------------------------------

# 7. `spark-r` ne?

Aynı mantığın R tarafındaki karşılığıdır:

``` text
spark
  ↓
R / SparkR desteği
  ↓
spark-r
```

Yani ortak Spark altyapısını tekrar kurmak yerine `spark` image'ının
üzerine R tarafındaki ihtiyaçları ekliyoruz.

Buradaki amaç:

``` text
Ortak Spark altyapısı
        ↓
   ┌────┴────┐
   ↓         ↓
 Python      R
   ↓         ↓
spark-py   spark-r
```

şeklinde farklı kullanım ortamları oluşturmaktır.

------------------------------------------------------------------------

# 8. "Ama `pip install pyspark` yapınca da Spark kullanabiliyoruz?"

Evet. Burada kafa karıştıran nokta bu.

``` bash
pip install pyspark
```

ile Python ortamında PySpark kullanabiliriz.

Fakat iki farklı soruyu ayırmalıyız.

### Kullanıcı açısından

> "Python'dan Spark kullanabilir miyim?"

Evet, PySpark ile kullanabilirsin.

### Platform açısından

> "Spark'ın çalışacağı ortamı standart bir container olarak
> paketleyebilir miyim?"

Evet, Spark image'ı bunun için kullanılabilir.

Yani:

``` text
PySpark
   ↓
Python API / binding
```

ile:

``` text
Spark runtime
   ↓
Spark'ın çalışma ortamı
```

aynı kavram değildir.

Bunu basitçe şöyle düşünebiliriz:

``` text
Python kodu
     ↓
   PySpark
     ↓
Spark Runtime
     ↓
Spark Engine
```

------------------------------------------------------------------------

# 9. Neden Spark'ı Jupyter'ın içine koyup geçmiyoruz?

Çünkü Spark sadece Jupyter üzerinden kullanılmaz.

ABC Bank örneğinde aynı Spark ortamının farklı yerlerde kullanıldığını
düşünelim.

### Geliştirme

``` text
Developer
   ↓
JupyterLab
   ↓
PySpark
   ↓
Spark
```

### Production

``` text
Airflow / başka orchestrator
          ↓
     spark-submit
          ↓
     Spark application
```

### Kubernetes

``` text
Kubernetes
     ↓
Spark application
     ↓
Spark runtime
```

Bu senaryoların bazılarında Jupyter hiç yok.

Bu nedenle Spark runtime'ını Jupyter'dan bağımsız bir image ailesi
olarak düşünmek mantıklıdır.

------------------------------------------------------------------------

# 10. Jupyter image ile Spark image arasındaki ilişki

Burada iki farklı ihtiyacı birbirinden ayırmak yeterli.

### Spark image ailesi

``` text
eclipse-temurin:jre
          ↓
     spark-base
          ↓
        spark
       ↙    ↘
  spark-py  spark-r
```

Temel soru:

> **"Spark'ın çalışacağı ortamı nasıl paketleriz?"**

### Jupyter image ailesi

``` text
foundation
    ↓
base-notebook
    ↓
minimal-notebook
    ↓
scipy-notebook
    ↓
pyspark-notebook
```

Temel soru:

> **"İnsanların notebook üzerinden çalışacağı ortamı nasıl
> hazırlarız?"**

Dolayısıyla:

``` text
Spark image
    → runtime

Jupyter image
    → çalışma arayüzü / notebook ortamı
```

şeklinde düşünmek yeterlidir.

------------------------------------------------------------------------

# 11. Önemli ayrım: Kullanım ilişkisi ile inheritance aynı şey değil

Şunu görmek mümkün:

``` text
JupyterLab
    ↓
Notebook
    ↓
PySpark
    ↓
Spark
```

Bu bize çalışma sırasında bileşenlerin ilişkisini anlatır.

Ama şu:

``` dockerfile
FROM spark-py
```

Docker image inheritance ilişkisidir.

Yani bir diyagramda iki bileşenin birbirine bağlı olması, otomatik
olarak birinin Docker'da diğerinin parent image'ı olduğu anlamına
gelmez.

Kaynakta incelenen güncel `pyspark-notebook` yapısında da image
`scipy-notebook` üzerinden ilerleyip Java ve Spark bileşenlerini kendi
build sürecinde kurmaktadır. Bu yüzden iki image ailesini önce
**amaçlarına göre** ayırmak gerekir.

------------------------------------------------------------------------

# 12. ABC Bank için bütün resmi görelim

ABC Bank'ın Spark platformunda temel düşünce şöyle olabilir:

``` text
                 SPARK
                   │
            Ortak runtime
                   │
          ┌────────┴────────┐
          ↓                 ↓
      Python               R
          ↓                 ↓
     spark-py            spark-r
```

Bunlar Spark'ın farklı kullanım biçimleri için hazırlanmış runtime
image'larıdır.

Jupyter ise kullanıcı tarafında ayrıca düşünülebilir:

``` text
              Developer
                  ↓
              JupyterLab
                  ↓
               Notebook
                  ↓
               PySpark
                  ↓
              Spark
```

Yani Jupyter:

> **"Spark'ı kullanmak için insanın oturduğu çalışma masası"**

gibi düşünülebilir.

Spark image ise:

> **"Spark'ın çalıştığı ortam"**

gibi düşünülebilir.

------------------------------------------------------------------------

# 13. Akılda kalması gereken en önemli model

Görseli artık şöyle okuyabilmelisin:

``` text
eclipse-temurin:jre
        │
        │ Java runtime
        ↓
   spark-base
        │
        │ ortak Spark altyapısı
        ↓
      spark
        │
        │ Spark runtime
        ├──────────────┐
        ↓              ↓
    spark-py        spark-r
        │              │
     Python             R
     /PySpark         /SparkR
```

**Mantık tek cümlede:**

> Ortak Spark altyapısını bir kere hazırlıyoruz, sonra bu ortamın
> üzerine farklı dil/çalışma ihtiyaçlarını ekleyerek ayrı image'lar
> oluşturuyoruz.

Bu yüzden inheritance burada sadece "Docker'da katman oluşturmak" için
değil, **ortak altyapıyı tekrar kullanarak farklı Spark çalışma
ortamlarını düzenli şekilde üretmek** için kullanılıyor.

------------------------------------------------------------------------

## Bir sonraki konu

Artık Spark image'ın **ne olduğunu ve neden bu şekilde katmanlandığını**
biliyoruz.

Şimdi daha ilginç bir soru geliyor:

> **Jupyter'da bir notebook açtım ve `spark.read(...)` çalıştırdım. Bu
> kod gerçekten nerede çalışıyor?**

İşte burada:

``` text
Jupyter
   ↓
Kernel
   ↓
Enterprise Gateway
   ↓
Kubernetes / YARN
   ↓
Remote Kernel
   ↓
Spark
```

yapısını anlamaya başlayacağız.
