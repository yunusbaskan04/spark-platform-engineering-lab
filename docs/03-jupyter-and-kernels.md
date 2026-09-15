# 03 --- Jupyter, Jupyter Server ve Kernel

## 1. Önce problemi anlayalım

JupyterLab'da bir notebook açıp:

``` python
print("Merhaba")
```

yazdığımızda kodu gerçekten kim çalıştırıyor?

JupyterLab'ın kendisi değil.

Kodun çalışmasını sağlayan şey **kernel**'dır.

Basit durumda:

``` text
Browser
   ↓
JupyterLab
   ↓
Python Kernel
   ↓
Python kodu
```

Bu küçük bir ortam için yeterlidir. Kernel, Jupyter Server'ın bulunduğu
ortamda çalışabilir.

------------------------------------------------------------------------

## 2. Kernel nedir?

Kernel'ı şimdilik şöyle düşün:

> **Notebook'taki kodu gerçekten çalıştıran süreç.**

Örneğin:

``` python
x = 10
x + 5
```

kodunu yazdığında Jupyter bu kodu kernel'a gönderir.

Kernel:

``` text
x = 10
   ↓
x + 5
   ↓
15
```

sonucunu üretir ve sonucu Jupyter'a geri gönderir.

Bu nedenle Jupyter'ı:

> **Kernel ile konuştuğumuz çalışma arayüzü**

olarak düşünebiliriz.

------------------------------------------------------------------------

## 3. Küçük sistemde neden sorun yok?

Laptopunda:

``` text
Jupyter
   ↓
Python Kernel
```

yeterlidir.

Ama ABC Bank'ın ortamını düşünelim:

``` text
Jupyter Server
8 CPU / 16 GB RAM
```

Senin Spark işin ise:

``` text
128 GB RAM
32 CPU
```

istiyor.

Notebook'un bulunduğu makine ile compute cluster artık aynı kaynak
değil.

İşte problem burada başlıyor.

------------------------------------------------------------------------

## 4. Remote Kernel ihtiyacı

İstediğimiz yapı:

``` text
Jupyter Server
      ↓
      ?
      ↓
Compute Cluster
      ↓
Python Kernel
```

Yani notebook Jupyter tarafında kalacak ama kernel başka bir
makinede/cluster'da çalışacak.

Böylece:

``` text
Jupyter
   ↓
Remote Kernel
   ↓
PySpark
   ↓
Spark
```

haline geçiyoruz.

Bu yapının temel amacı Jupyter arayüzü ile compute kaynaklarını
birbirinden ayırmaktır.

------------------------------------------------------------------------

## 5. Jupyter Server ile JupyterLab aynı şey mi?

Basitleştirerek:

``` text
Browser
   ↓
JupyterLab
   ↓
Jupyter Server
   ↓
Kernel
```

diye düşünebiliriz.

-   **JupyterLab:** Kullanıcının gördüğü notebook çalışma ortamı.
-   **Jupyter Server:** Jupyter tarafındaki sunucu bileşeni.
-   **Kernel:** Python kodunu gerçekten çalıştıran süreç.

Remote kernel kullandığımızda araya Enterprise Gateway girer:

``` text
Browser
   ↓
JupyterLab
   ↓
Jupyter Server
   ↓
Enterprise Gateway
   ↓
Remote Kernel
```

------------------------------------------------------------------------

## 6. Bu neden önemli?

ABC Bank'ta 100 Data Engineer olduğunu düşün.

Herkesin kernel'ını kendi Jupyter Server'ının yanında çalıştırırsak
compute kaynaklarını merkezi bir cluster'dan yönetmek zorlaşır.

Bunun yerine:

``` text
                    Jupyter
                       │
              Enterprise Gateway
                       │
                ┌──────┴──────┐
                ↓             ↓
              YARN       Kubernetes
                │             │
                ↓             ↓
             Kernel          Pod
```

yapısıyla notebook ortamını compute altyapısından ayırabiliriz.

------------------------------------------------------------------------

## 7. Akılda kalacak model

Jupyter dünyasında üç şeyi ayır:

``` text
JupyterLab
   → kullanıcı arayüzü

Jupyter Server
   → Jupyter tarafındaki sunucu

Kernel
   → kodu gerçekten çalıştıran süreç
```

Ve remote kullanımda:

``` text
Jupyter
   ↓
Enterprise Gateway
   ↓
Remote Kernel
```

olur.

Bir sonraki bölümde bu `Enterprise Gateway` katmanının remote kernel'ı
nasıl başlattığını göreceğiz.
