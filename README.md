# CloudPulse-GNN

### eBPF ve Graf Sinir Ağları ile Kubernetes Ortamlarında Çalışma Zamanı Güvenliği İzleme ve Anomali Tespiti

**CloudPulse-GNN**, Kubernetes ortamlarında çalışan konteynerlerin sistem davranışlarını Linux çekirdeği seviyesinde izlemek, bu davranışlar arasındaki ilişkileri analiz etmek ve şüpheli aktiviteleri graf tabanlı makine öğrenmesi yöntemleriyle tespit etmek amacıyla geliştirilen bir araştırma projesidir.

Proje, Linux çekirdeğinden **eBPF (Extended Berkeley Packet Filter)** aracılığıyla sistem olaylarını toplar. Toplanan olaylar süreçler, dosyalar, ağ bağlantıları ve Kubernetes kaynakları arasındaki ilişkileri gösterecek şekilde bir **Sistem Provenans Grafına (System Provenance Graph)** dönüştürülür.

Oluşturulan graf yapısı, **Graf Sinir Ağları (Graph Neural Networks – GNN)** kullanılarak analiz edilir. Böylece tek başına normal görünen sistem aktivitelerinin bir araya geldiğinde şüpheli bir davranış zinciri oluşturup oluşturmadığı araştırılır.

CloudPulse-GNN'nin temel amacı, geleneksel olay bazlı güvenlik analizinin ötesine geçerek sistem davranışlarını ilişkisel ve zamansal açıdan incelemektir.

**Proje yaklaşımı:**

- eBPF ile sistem olaylarını toplamak.
- Go Agent ile olayları kullanıcı alanına aktarmak.
- Kubernetes kaynaklarıyla olayları ilişkilendirmek.
- Sistem aktivitelerinden zamansal provenans grafı oluşturmak.
- GCN ve GAT modelleriyle şüpheli davranışları analiz etmek.
- Tespit edilen anomalilerin nedenlerini graf ilişkileri üzerinden incelemek.

> **Proje durumu:** CloudPulse-GNN şu anda mimari tasarım ve geliştirme planlama aşamasındadır. Henüz tamamlanmış bir sistem veya doğrulanmış deneysel sonuçlar bulunmamaktadır.

---

## İçindekiler

1. [Proje Hakkında](#1-proje-hakkında)
2. [Projenin Amaçları](#2-projenin-amaçları)
3. [Araştırma Soruları](#3-araştırma-soruları)
4. [Tehdit Modeli](#4-tehdit-modeli)
5. [Projenin Temel Katkıları](#5-projenin-temel-katkıları)
6. [Sistem Mimarisi](#6-sistem-mimarisi)
7. [Temel Bileşenler](#7-temel-bileşenler)
8. [Sistem Provenans Grafı](#8-sistem-provenans-grafı)
9. [Graf Özellikleri ve Veri Modeli](#9-graf-özellikleri-ve-veri-modeli)
10. [GNN Tabanlı Anomali Tespiti](#10-gnn-tabanlı-anomali-tespiti)
11. [Kök Neden Analizi](#11-kök-neden-analizi)
12. [Veri Seti Oluşturma](#12-veri-seti-oluşturma)
13. [Deneysel Ortam](#13-deneysel-ortam)
14. [Performans Değerlendirmesi](#14-performans-değerlendirmesi)
15. [Teknoloji Altyapısı](#15-teknoloji-altyapısı)
16. [Proje Klasör Yapısı](#16-proje-klasör-yapısı)
17. [Geliştirme Yol Haritası](#17-geliştirme-yol-haritası)
18. [Kurulum](#18-kurulum)
19. [Sınırlılıklar ve Teknik Zorluklar](#19-sınırlılıklar-ve-teknik-zorluklar)
20. [Gelecekteki Geliştirmeler](#20-gelecekteki-geliştirmeler)
21. [Güvenlik ve Etik](#21-güvenlik-ve-etik)
22. [Kaynaklar](#22-kaynaklar)
23. [Lisans](#23-lisans)

---

## 1. Proje Hakkında

Kubernetes ortamlarında çalışan uygulamalar, normal çalışma süreçleri boyunca sürekli olarak sistem aktiviteleri gerçekleştirir.

Bu aktiviteler arasında:

- Yeni süreçlerin oluşturulması.
- Programların çalıştırılması.
- Dosyaların okunması ve yazılması.
- Ağ bağlantılarının kurulması.
- Konteynerler arasındaki iletişim.
- Süreçlerin sonlandırılması.

bulunur.

Bu aktivitelerin ayrı ayrı incelenmesi, güvenlik açısından şüpheli davranışların anlaşılmasını zorlaştırabilir.

Örneğin bir konteynerde aşağıdaki olayların gerçekleştiğini düşünelim:

1. Uygulama tarafından beklenmeyen bir shell süreci başlatılır.
2. Bu süreç normalde erişmemesi gereken bir dosyayı okur.
3. Ardından harici bir IP adresine bağlantı kurar.
4. Süreç, dosya ve ağ aktiviteleri arasında şüpheli bir davranış zinciri oluşur.

Bu olaylar tek başlarına değerlendirildiğinde normal sistem aktivitelerine benzeyebilir. Ancak aralarındaki ilişkiler birlikte incelendiğinde güvenlik açısından şüpheli bir durum ortaya çıkabilir.

CloudPulse-GNN, bu tür davranış zincirlerini provenans grafı üzerinde modelleyerek ilişkisel anomali tespiti gerçekleştirmeyi hedefler.

### Genel çalışma akışı

```text
       Kubernetes Ortamı
               |
               v
          Linux Kernel
               |
               v
             eBPF
               |
               v
           Go Agent
               |
               v
        Olay Toplama
               |
               v
          JSONL Akışı
               |
               v
       Python İşleme Katmanı
               |
               v
       Olay Normalizasyonu
               |
               v
      Kimlik İlişkilendirme
               |
               v
       Provenans Grafı
               |
               v
      PyTorch Geometric
               |
               v
          GCN / GAT
               |
               v
       Anomali Tespiti
               |
               v
       Kök Neden Analizi
               |
               v
          SQLite
               |
               v
       Sonuçların İzlenmesi
```

---

## 2. Projenin Amaçları

CloudPulse-GNN dört temel amaç üzerine kuruludur.

### 2.1. Sistem Aktivitelerinin İzlenmesi

eBPF kullanarak Linux çekirdeğinde gerçekleşen sistem aktivitelerini gözlemlemek.

İlk aşamada aşağıdaki olayların toplanması hedeflenmektedir:

- Süreç oluşturma ve çalıştırma.
- Süreç sonlandırma.
- Dosya erişim girişimleri.
- Gelen ve giden ağ bağlantıları.
- Konteyner içerisindeki süreç aktiviteleri.

### 2.2. Sistem Provenans Grafının Oluşturulması

Toplanan olayları düğümler ve yönlü ilişkiler aracılığıyla graf yapısına dönüştürmek.

Bu sayede süreçler, dosyalar, ağ bağlantıları ve Kubernetes kaynakları arasındaki ilişkilerin birlikte incelenmesi sağlanacaktır.

### 2.3. Graf Tabanlı Anomali Tespiti

Oluşturulan graf yapısını GNN modelleriyle analiz ederek normal sistem davranışlarından farklılaşan aktiviteleri belirlemek.

İlk aşamada GCN ve GAT modelleri değerlendirilecek, sonuçlar geleneksel makine öğrenmesi yöntemleriyle karşılaştırılacaktır.

### 2.4. Kök Neden Analizi

Tespit edilen anomalilerin hangi süreçler, dosyalar, ağ bağlantıları ve Kubernetes kaynaklarıyla ilişkili olduğunu belirlemek.

Amaç yalnızca anomali tespit etmek değil, şüpheli davranışın nasıl meydana geldiğini de açıklamaya yardımcı olmaktır.

---

## 3. Araştırma Soruları

Proje kapsamında aşağıdaki araştırma sorularına cevap aranacaktır.

**RQ1:** Sistem provenans grafları, geleneksel özellik tabanlı makine öğrenmesi yöntemlerine kıyasla anomali tespit başarısını artırabilir mi?

**RQ2:** GCN ve GAT modelleri, süreçler, dosyalar ve ağ bağlantıları arasındaki şüpheli ilişkileri tespit edebilir mi?

**RQ3:** Zamansal bilgilerin graf yapısına eklenmesi anomali tespit performansını nasıl etkiler?

**RQ4:** Graf tabanlı analiz, Kubernetes ortamlarındaki şüpheli aktivitelerin kök nedenlerini belirlemeye ne ölçüde yardımcı olabilir?

**RQ5:** eBPF tabanlı olay toplama ve GNN tabanlı analiz, Kubernetes ortamlarında ne kadar CPU, bellek ve işlem süresi tüketir?

---

## 4. Tehdit Modeli

İlk sürümde proje kapsamı belirli sistem davranışlarıyla sınırlandırılmıştır.

### 4.1. Kapsam Dahilindeki Davranışlar

- Beklenmeyen shell veya yardımcı süreç çalıştırılması.
- Şüpheli parent-child süreç ilişkileri.
- Uygulamanın normal davranışından farklı dosya erişimleri.
- Beklenmeyen dış ağ bağlantıları.
- Birden fazla olayın birleşmesiyle oluşan şüpheli davranış zincirleri.

### 4.2. Kapsam Dışındaki Davranışlar

İlk prototip aşağıdaki saldırı türlerinin tamamını tespit etmeyi hedeflememektedir:

- Kernel exploit saldırıları.
- Bellek içi saldırıların tamamı.
- Şifreli ağ trafiğinin içerik analizi.
- Container escape saldırılarının tüm biçimleri.
- Dağıtık saldırıların kesin tespiti.

Bu konular ilerleyen araştırma aşamalarında değerlendirilebilir.

### 4.3. Temel Varsayımlar

- İzleme ajanı yetkili Linux/Kubernetes düğümlerinde çalışır.
- Kernel ve eBPF altyapısının güvenilir olduğu varsayılır.
- İlk deneyler izole test ortamında gerçekleştirilir.
- Dosya içerikleri yerine öncelikle dosya erişim metadata bilgileri toplanır.
- İzleme için gerekli yetkiler mümkün olduğunca sınırlandırılır.

---

## 5. Projenin Temel Katkıları

CloudPulse-GNN kapsamında geliştirilmesi ve deneysel olarak değerlendirilmesi planlanan katkılar şunlardır:

| Katkı | Açıklama |
|---|---|
| Çekirdek Seviyesinde İzleme | eBPF ile sistem olaylarının toplanması |
| Kubernetes İlişkilendirmesi | Olayların Pod, Namespace ve Container kaynaklarıyla eşleştirilmesi |
| Provenans Grafı | Sistem aktivitelerinin ilişkisel olarak modellenmesi |
| Zamansal Analiz | Olayların zaman sırasının korunması |
| Graf Tabanlı Öğrenme | GCN ve GAT modellerinin değerlendirilmesi |
| Açıklanabilir Analiz | Şüpheli davranış zincirlerinin incelenmesi |
| Performans Karşılaştırması | GNN ve geleneksel modellerin karşılaştırılması |

Bu maddeler hedeflenen araştırma katkılarıdır; henüz deneysel olarak doğrulanmış sonuçlar değildir.

---

## 6. Sistem Mimarisi

CloudPulse-GNN modüler bir mimariyle tasarlanmıştır.

Sistem, birbirinden bağımsız ancak birbiriyle ilişkili bileşenlerden oluşur.

### 6.1. Mimari Katmanlar

| Katman | Görevi |
|---|---|
| Kernel İzleme | eBPF ile sistem olaylarını toplamak |
| Olay Toplama | Go Agent ile olayları almak |
| Veri İşleme | Olayları düzenlemek ve doğrulamak |
| Kimlik İlişkilendirme | Süreçleri Kubernetes kaynaklarıyla eşleştirmek |
| Graf Oluşturma | Provenans grafını oluşturmak |
| Graf Öğrenme | GNN modellerini çalıştırmak |
| Anomali Tespiti | Şüpheli aktiviteleri belirlemek |
| Kök Neden Analizi | Anomalilerin ilişkilerini incelemek |
| Veri Saklama | Olayları ve sonuçları kaydetmek |
| Görselleştirme | Sistem metriklerini ve analiz sonuçlarını sunmak |

### 6.2. Mimari Diyagram

```text
                Kubernetes Cluster
                        |
                        v
                   Linux Kernel
                        |
                        v
                    eBPF Probes
                        |
                        v
                     Go Agent
                        |
              +---------+---------+
              |                   |
              v                   v
          Event Buffer       Metadata
              |                   |
              +---------+---------+
                        |
                        v
                   JSONL Events
                        |
                        v
               Python Processing
                        |
              +---------+---------+
              |                   |
              v                   v
        Normalization       Identity Mapping
              |                   |
              +---------+---------+
                        |
                        v
                Provenance Graph
                        |
                        v
               PyTorch Geometric
                        |
              +---------+---------+
              |                   |
              v                   v
         Baseline Models       GCN / GAT
              |                   |
              +---------+---------+
                        |
                        v
                Anomaly Detection
                        |
                        v
                  RCA Engine
                        |
                        v
                 Result Storage
                        |
                        v
                 Visualization
```

---

## 7. Temel Bileşenler

### 7.1. eBPF ile Olay Toplama

eBPF, Linux çekirdeğinde gerçekleşen belirli sistem aktivitelerinin düşük seviyede gözlemlenmesini sağlayan bir teknolojidir.

CloudPulse-GNN kapsamında eBPF programları kullanılarak aşağıdaki olayların toplanması hedeflenmektedir:

| Olay | İzleme Mekanizması | Açıklama |
|---|---|---|
| Süreç Oluşturma | `sched_process_fork` | Yeni süreç oluşturulması |
| Program Çalıştırma | `sched_process_exec` | Bir programın çalıştırılması |
| Süreç Sonlandırma | `sched_process_exit` | Sürecin sonlanması |
| Dosya Açma | `openat` / `openat2` | Dosya açma girişimleri |
| Giden Bağlantı | `connect` | Ağ bağlantısı kurulması |
| Gelen Bağlantı | `accept` / `accept4` | Bağlantının kabul edilmesi |

Kullanılacak hook türleri ve izleme yöntemleri kernel sürümüne göre belirlenecektir. Örneğin dosya ve ağ olaylarında sistem çağrısı izleme veya uygun kernel tracepoint mekanizmaları değerlendirilebilir.

Toplanması hedeflenen temel bilgiler:

- PID ve TGID.
- Parent PID.
- Süreç adı.
- Çalıştırılabilir dosya yolu.
- Dosya yolu ve erişim türü.
- Kaynak ve hedef IP adresleri.
- Kaynak ve hedef portlar.
- Olay türü.
- Zaman damgası.
- Cgroup ID.
- Container ID.

**Kullanılacak teknolojiler:** C ve Go (`cilium/ebpf`).

### 7.2. Go Tabanlı Olay Toplama Ajanı

Go Agent, eBPF programlarından gelen olayları kullanıcı alanında karşılayan bileşendir.

Temel görevleri:

- eBPF programlarını yüklemek ve yönetmek.
- Kernel üzerinden gelen olayları okumak.
- Olayları yapılandırılmış verilere dönüştürmek.
- Süreç ve konteyner metadata bilgilerini ilişkilendirmek.
- JSONL formatında çıktı üretmek.
- Olayları Python işleme katmanına aktarmak.

İlk prototipte olay aktarımı için JSON Lines (JSONL) kullanılacaktır.

Yoğun olay trafiğinde performans ve veri kaybı ölçülecek; gerekirse daha verimli bir aktarım yöntemi değerlendirilecektir.

### 7.3. Kubernetes Kaynaklarıyla İlişkilendirme

Linux çekirdeğinden elde edilen süreç bilgilerinin Kubernetes kaynaklarıyla eşleştirilmesi gerekmektedir.

Bu kapsamda kullanılacak bilgiler:

- Host ID
- PID
- TGID
- Process Start Time
- Cgroup ID
- Container ID
- Pod UID
- Pod Name
- Namespace

Yalnızca PID kullanmak doğru değildir. Çünkü işletim sistemi sonlandırılan bir sürecin PID değerini başka bir süreç için yeniden kullanabilir.

Bu nedenle süreç kimliği aşağıdaki bileşenlerle oluşturulacaktır:

```text
Process Key =
Host ID + PID + Process Start Time
```

Cgroup ve Container ID bilgileri ise sürecin hangi konteyner ve Kubernetes kaynağı içerisinde çalıştığını belirlemeye yardımcı olacaktır.

Container metadata bilgilerinin gerektiğinde container runtime veya Kubernetes API üzerinden tamamlanması planlanmaktadır.

### 7.4. Olay İşleme ve Normalizasyon

Go Agent tarafından üretilen olaylar Python katmanına aktarılır.

Bu aşamada:

- JSON verilerinin okunması.
- Olay formatının doğrulanması.
- Zaman damgalarının standartlaştırılması.
- Tekrarlanan olayların kontrol edilmesi.
- Süreç kimliklerinin eşleştirilmesi.
- Kubernetes metadata bilgilerinin eklenmesi.
- İlişkili olayların bir araya getirilmesi.

işlemleri gerçekleştirilir.

Bu aşamanın çıktısı provenans grafı oluşturmak için kullanılacaktır.

### 7.5. Provenans Grafı Oluşturma

Sistem bileşenleri grafın düğümleri, bu bileşenler arasındaki etkileşimler ise yönlü kenarları olarak temsil edilir.

Örnek düğüm türleri:

- Process
- File
- Network Endpoint
- Container
- Pod
- Namespace

Örnek ilişki türleri:

- Creates
- Executes
- Accesses
- Connects
- Runs In
- Belongs To

Grafın oluşturulması sırasında her ilişkinin olay zamanı ve gerekli metadata bilgileri korunacaktır.

### 7.6. GNN Tabanlı Analiz

Oluşturulan graf yapısı PyTorch Geometric kullanılarak analiz edilecektir.

İlk aşamada Isolation Forest baseline modeli ve temel bir GNN modeli geliştirilecektir.

Sonraki aşamada:

- GCN
- GAT

modelleri uygulanarak karşılaştırılacaktır.

### 7.7. Kök Neden Analizi

Anomali tespitinden sonra ilişkili süreçler, dosyalar, ağ bağlantıları ve Kubernetes kaynakları incelenecektir.

RCA bileşeninin görevleri:

- Şüpheli süreçleri belirlemek.
- Parent-child ilişkilerini takip etmek.
- İlişkili dosya erişimlerini incelemek.
- Ağ bağlantılarını belirlemek.
- İlgili Container ve Pod bilgilerini göstermek.
- Şüpheli davranış zincirini ortaya çıkarmak.

### 7.8. Veri Saklama ve İzleme

İlk prototipte SQLite kullanılacaktır.

Aşağıdaki bilgiler saklanacaktır:

- Toplanan sistem olayları.
- Graf oluşturma bilgileri.
- Anomali skorları.
- Tespit sonuçları.
- Deneysel ölçüm sonuçları.

Grafana ise aşağıdaki sistem metriklerinin izlenmesi için kullanılacaktır:

- Olay toplama hızı.
- Süreç aktivitesi.
- Ağ aktivitesi.
- Tespit edilen anomali sayısı.
- CPU ve bellek kullanımı.
- Olay işleme gecikmesi.
- Model tahmin süresi.

Not: Grafana sistem metriklerini göstermek için kullanılacaktır. Provenans grafındaki düğüm ve kenar ilişkilerinin etkileşimli incelenmesi için ayrıca bir graf görselleştirme bileşeni gereklidir.

---

## 8. Sistem Provenans Grafı

Sistem Provenans Grafı, CloudPulse-GNN'nin temel veri yapısını oluşturur.

Bu graf, sistem bileşenlerinin birbirleriyle nasıl etkileşim kurduğunu gösterir.

Matematiksel olarak:

\[
G=(V,E,X_V,X_E)
\]

Burada:

- \(V\): Graf düğümleri.
- \(E\): Graf kenarları.
- \(X_V\): Düğüm özellikleri.
- \(X_E\): Kenar özellikleri.

### 8.1. Düğüm Türleri

| Düğüm | Açıklama |
|---|---|
| Process | Çalışan süreçler |
| File | Erişilen dosyalar |
| Network Endpoint | Ağ uç noktaları |
| Container | Konteynerler |
| Pod | Kubernetes Pod kaynakları |
| Namespace | Kubernetes Namespace kaynakları |

### 8.2. Kenar Türleri

| İlişki | Kaynak → Hedef | Açıklama |
|---|---|---|
| Creates | Process → Process | Bir sürecin başka bir süreç oluşturması |
| Executes | Process → File/Image | Bir sürecin program çalıştırması |
| Accesses | Process → File | Bir sürecin dosyaya erişmesi |
| Connects | Process → Network Endpoint | Bir sürecin ağ bağlantısı kurması |
| Runs In | Process → Container | Sürecin konteyner içerisinde çalışması |
| Belongs To | Container → Pod | Konteynerin Pod ile ilişkilendirilmesi |
| Belongs To | Pod → Namespace | Pod'un Namespace içerisinde bulunması |

### 8.3. Örnek Provenans Grafı

```text
             Namespace
                 |
                 v
                Pod
                 |
                 v
              Container
                 |
                 v
              Process A
                 |
          +------+------+
          |             |
          v             v
      Process B       File A
          |
     +----+----+
     |         |
     v         v
   File B   Network Endpoint
```

### 8.4. Zamansal Graf Modelleme

Sistem olayları belirli zamanlarda gerçekleştiği için graf ilişkilerinde zaman bilgisi de tutulacaktır.

Örnek olay:

```json
{
  "source": "process_101",
  "target": "file_205",
  "event_type": "FILE_ACCESS",
  "timestamp": 1795521600,
  "process_id": 101,
  "container_id": "container_01",
  "pod": "frontend-pod",
  "namespace": "online-boutique"
}
```

Bu yapı sayesinde olayların hangi sırayla gerçekleştiği korunabilecektir.

İlk aşamada zaman damgaları ve zaman pencereleri kullanılacaktır. Daha sonraki aşamalarda zamansal graf öğrenme yöntemleri değerlendirilebilir.

---

## 9. Graf Özellikleri ve Veri Modeli

Graf oluşturma ve makine öğrenmesi aşamalarında aynı veri yapısının kullanılması önemlidir.

Bu nedenle ilk prototipte aşağıdaki veri modeli kullanılacaktır.

### 9.1. Düğüm Özellikleri

| Düğüm Türü | Örnek Özellikler |
|---|---|
| Process | Süreç adı, çalıştırılabilir dosya, UID, başlangıç zamanı, yaşam süresi |
| File | Dosya yolu, dosya türü, erişim türü |
| Network Endpoint | IP adresi, port, protokol, bağlantı yönü |
| Container | Container ID, runtime, imaj bilgisi |
| Pod | Pod UID, Pod adı, etiketler |
| Namespace | Namespace adı ve UID |

### 9.2. Kenar Özellikleri

| İlişki | Özellikler |
|---|---|
| Creates | Zaman, olay türü |
| Executes | Zaman, çalıştırılan program |
| Accesses | Zaman, erişim türü, dosya bayrakları |
| Connects | Zaman, protokol, bağlantı yönü |
| Runs In | Container kimliği, gözlem zamanı |
| Belongs To | Kubernetes kaynak bilgileri |

Hassas bilgiler ve yüksek sayıda farklı değer üretebilen alanlar gerektiğinde anonimleştirilecek veya normalleştirilecektir.

### 9.3. Süreç Kimliği

PID tek başına güvenilir bir süreç kimliği değildir.

Bu nedenle:

```text
Process Key =
Host ID + PID + Process Start Time
```

yaklaşımı kullanılacaktır.

Böylece PID yeniden kullanıldığında farklı süreçlerin yanlışlıkla aynı düğüm olarak değerlendirilmesi önlenecektir.

### 9.4. Zaman Pencereleri

Grafın sürekli büyümesini önlemek için olaylar belirli zaman aralıklarında gruplandırılacaktır.

Örneğin:

- 30 saniyelik zaman penceresi.
- 60 saniyelik zaman penceresi.
- 5 dakikalık zaman penceresi.

Bunlar başlangıçta değerlendirilecek örnek değerlerdir; uygun pencere uzunluğu deneysel olarak belirlenecektir.

Pencereler arasında aynı süreç ve kaynak kimlikleri tutarlı biçimde korunacaktır.

---

## 10. GNN Tabanlı Anomali Tespiti

CloudPulse-GNN'nin makine öğrenmesi katmanında geleneksel yöntemler ile graf tabanlı öğrenme yöntemleri karşılaştırılacaktır.

### 10.1. Geleneksel Makine Öğrenmesi

**Isolation Forest**

Etiket gerektirmeyen anomali tespiti için baseline olarak kullanılacaktır.

Süreç aktiviteleri, dosya erişim sayıları ve ağ bağlantıları gibi özellikler üzerinden olağan dışı davranışların belirlenmesi hedeflenmektedir.

**Random Forest**

Etiketlenmiş normal ve anormal örnekler üzerinde sınıflandırma yapmak için kullanılacaktır.

Random Forest, denetimli bir yöntem olduğundan eğitim için etiketli veri gerektirir.

Kullanılabilecek özellikler:

- Süreç çalıştırma sayısı.
- Dosya erişim sıklığı.
- Ağ bağlantısı sayısı.
- Alt süreç sayısı.
- Süreç yaşam süresi.
- Gelen ve giden bağlantı sayıları.
- Graf derece bilgileri.
- Zamansal aktivite özellikleri.

### 10.2. Graph Convolutional Network (GCN)

GCN, bir düğümün kendi özelliklerini komşu düğümlerin özellikleriyle birleştirerek yeni bir temsil oluşturmasını sağlar.

Bu sayede bir sürecin yalnızca kendi davranışları değil, etkileşimde bulunduğu dosya ve ağ kaynakları da analiz edilebilir.

### 10.3. Graph Attention Network (GAT)

GAT, graf üzerindeki komşu düğümlerin önemini attention mekanizmasıyla öğrenir.

Bu yaklaşımın farklı süreç, dosya ve ağ ilişkilerinin önemini belirlemede sağlayabileceği avantajlar araştırılacaktır.

### 10.4. Model Karşılaştırması

| Model | Yöntem | Kullanım |
|---|---|---|
| Isolation Forest | Denetimsiz ML | Anomali tespiti |
| Random Forest | Denetimli ML | Sınıflandırma |
| GCN | GNN | Graf temsili öğrenme |
| GAT | Attention tabanlı GNN | İlişkisel özellik öğrenme |

### 10.5. Öğrenme Görevi

GCN ve GAT doğrudan anomali tespit algoritmaları değildir. Bu nedenle model mimarisi ve öğrenme hedefi ayrı tanımlanmalıdır.

İlk prototipte temel görev olarak **kenar seviyesinde anomali sınıflandırması (Edge-Level Anomaly Classification)** kullanılacaktır.

Bu yaklaşımda:

- **Girdi:** Belirli bir zaman penceresinde oluşturulmuş provenans grafı.
- **Özellikler:** Düğüm türleri, kenar türleri, olay zamanları ve sistem metadata bilgileri.
- **Hedef:** Normal veya anormal olarak etiketlenmiş olay ilişkileri.
- **Çıktı:** Her hedef kenar için anomali olasılığı veya skoru.

Graf heterojen yapıda olduğundan farklı düğüm ve kenar türlerinin korunması planlanmaktadır. PyTorch Geometric içerisindeki `HeteroData` yapısı bu amaçla değerlendirilebilir.

### 10.6. Anomali Tespit Süreci

1. Sistem olaylarının toplanması.
2. Olaylardan provenans grafının oluşturulması.
3. Düğüm ve kenar özelliklerinin hazırlanması.
4. PyTorch Geometric veri yapısına dönüştürülmesi.
5. GCN ve GAT modellerinin eğitilmesi.
6. Anomali skorlarının üretilmesi.
7. Eşik değerinin doğrulama verisi üzerinde belirlenmesi.
8. Test verisi üzerinde performans ölçümü.
9. Sonuçların baseline modellerle karşılaştırılması.

İlerleyen aşamalarda düğüm seviyesinde ve graf seviyesinde anomali tespiti de değerlendirilebilir.

---

## 11. Kök Neden Analizi

Kök Neden Analizi (Root Cause Analysis – RCA), tespit edilen şüpheli aktivitelerin hangi sistem bileşenleriyle ilişkili olduğunu incelemeyi amaçlar.

Örnek davranış zinciri:

```text
     Şüpheli Süreç
           |
           v
   Beklenmeyen Dosya Erişimi
           |
           v
  Harici Ağ Bağlantısı
           |
           v
      Anomali Alarmı
           |
           v
    Kök Neden Analizi
```

RCA bileşeni aşağıdaki bilgileri inceleyecektir:

- Şüpheli sürecin kimliği.
- Parent process bilgisi.
- İlişkili alt süreçler.
- Dosya erişimleri.
- Ağ bağlantıları.
- Container ID.
- Pod ve Namespace bilgileri.
- Olayların gerçekleşme sırası.

Amaç, güvenlik analistinin yalnızca bir alarm görmesini değil, alarmın hangi davranış ilişkilerinden kaynaklandığını da anlayabilmesidir.

İlk sürümde RCA, anomaliyle ilişkili düğüm ve kenarları içeren alt graf oluşturma yaklaşımıyla gerçekleştirilecektir.

---

## 12. Veri Seti Oluşturma

Model geliştirme aşamasında normal ve anormal sistem davranışlarından oluşan kontrollü bir veri seti hazırlanacaktır.

### 12.1. Normal Davranışlar

Normal sistem aktiviteleri için Google Online Boutique uygulamasının kullanılması planlanmaktadır.

Örnek aktiviteler:

- Ürün görüntüleme.
- Ürün arama.
- Sepete ürün ekleme.
- Sipariş oluşturma.
- Ödeme işlemleri.
- Servisler arası iletişim.
- Normal süreç çalıştırma.
- Beklenen dosya erişimleri.

Normal trafik oluşturmak için Locust kullanılacaktır.

### 12.2. Anormal Davranışlar

İzole Kubernetes ortamında aşağıdaki kontrollü senaryolar oluşturulacaktır:

- Beklenmeyen shell çalıştırılması.
- Şüpheli alt süreç oluşturulması.
- Olağan dışı dosya erişimi.
- Beklenmeyen dış ağ bağlantısı.
- Normal çalışma profiliyle uyuşmayan süreç aktiviteleri.
- Beklenmeyen servisler arası iletişim.

Bütün senaryolar yalnızca yetkili test ortamında gerçekleştirilecektir.

### 12.3. Veri Etiketleme

Olaylar ve graf örnekleri normal veya anormal olarak etiketlenecektir.

| Etiket | Açıklama |
|---|---|
| 0 | Normal |
| 1 | Anormal |

Olay seviyesindeki etiketlerle davranış zinciri seviyesindeki etiketler birbirinden ayrılacaktır.

Örneğin bir saldırı senaryosunda yalnızca belirli bir olay anormal olarak etiketlenebileceği gibi, bütün davranış zinciri de anormal olarak işaretlenebilir.

Etiketleme politikası deneylerden önce belirlenecek ve bütün modellerde aynı şekilde uygulanacaktır.

### 12.4. Veri Setinin Bölünmesi

Veri seti üç parçaya ayrılacaktır:

- Training Set
- Validation Set
- Test Set

Veri sızıntısını azaltmak için mümkün olduğunca zaman veya senaryo bazlı ayrım kullanılacaktır.

Örneğin aynı saldırı senaryosunun benzer örneklerinin hem eğitim hem test verisinde bulunması engellenmeye çalışılacaktır.

Model parametreleri eğitim verisiyle öğrenilecek, eşik değerleri doğrulama verisiyle belirlenecek ve nihai başarı test verisi üzerinde ölçülecektir.

---

## 13. Deneysel Ortam

Projenin geliştirme ve test aşamalarında aşağıdaki teknolojilerin kullanılması planlanmaktadır.

| Bileşen | Teknoloji |
|---|---|
| İşletim Sistemi | Linux |
| Konteyner Altyapısı | Docker |
| Orkestrasyon | Kubernetes |
| Yerel Küme | Kind / Minikube |
| Test Uygulaması | Google Online Boutique |
| Trafik Üretimi | Locust |
| Sistem İzleme | eBPF |
| Olay Toplama | Go Agent |
| Veri İşleme | Python |
| Graf Öğrenme | PyTorch Geometric |
| Makine Öğrenmesi | PyTorch / Scikit-learn |
| Veri Saklama | SQLite |
| Metrik Görselleştirme | Grafana |

### Test Senaryoları

**Senaryo 1 – Normal İş Yükü**

Online Boutique üzerinde kullanıcı davranışları simüle edilecektir.

**Senaryo 2 – Anormal Süreç Aktiviteleri**

Beklenmeyen süreç oluşturma ve program çalıştırma aktiviteleri kontrollü olarak gerçekleştirilecektir.

**Senaryo 3 – Anormal Dosya ve Ağ Aktiviteleri**

Beklenmeyen dosya erişimleri ve ağ bağlantıları oluşturulacaktır.

Her senaryoda olay toplama doğruluğu, graf oluşturma başarısı ve anomali tespit performansı değerlendirilecektir.

---

## 14. Performans Değerlendirmesi

CloudPulse-GNN'nin başarısı iki temel açıdan değerlendirilecektir:

1. Anomali tespit performansı.
2. Sistem kaynak tüketimi.

### 14.1. Makine Öğrenmesi Metrikleri

| Metrik | Açıklama |
|---|---|
| Accuracy | Genel sınıflandırma başarısı |
| Precision | Tespit edilen anomalilerin ne kadarının doğru olduğu |
| Recall | Gerçek anomalilerin ne kadarının yakalandığı |
| F1-Score | Precision ve Recall dengesi |
| ROC-AUC | Sınıflandırma ayırt etme başarısı |
| PR-AUC | Dengesiz veri setlerinde başarı |
| False Positive Rate | Yanlış alarm oranı |

Normal aktivitelerin anormal aktivitelerden daha fazla olabileceği dikkate alınarak özellikle Precision, Recall, PR-AUC ve False Positive Rate değerleri önemlidir.

### 14.2. Sistem Performans Metrikleri

Sistemin çalışma maliyetini değerlendirmek için aşağıdaki ölçümler yapılacaktır:

- CPU kullanımı.
- Bellek tüketimi.
- Olay toplama gecikmesi.
- Olay işleme gecikmesi.
- Graf oluşturma süresi.
- Model tahmin süresi.
- Saniye başına işlenen olay sayısı.
- Olay kayıp oranı.

### 14.3. Model Karşılaştırması

| Metrik | Isolation Forest | Random Forest | GCN | GAT |
|---|---|---|---|---|
| Accuracy | TBD | TBD | TBD | TBD |
| Precision | TBD | TBD | TBD | TBD |
| Recall | TBD | TBD | TBD | TBD |
| F1-Score | TBD | TBD | TBD | TBD |
| ROC-AUC | TBD | TBD | TBD | TBD |
| PR-AUC | TBD | TBD | TBD | TBD |
| Tahmin Süresi | TBD | TBD | TBD | TBD |

*Not: TBD değerleri deneysel sonuçlar elde edildikten sonra doldurulacaktır.*

Karşılaştırmalar aynı veri bölünmesi, etiketleme politikası ve değerlendirme koşulları altında gerçekleştirilecektir.

### 14.4. Ablation Study

Grafın hangi bileşenlerinin tespit başarısına katkı sağladığını incelemek için ablation deneyleri yapılacaktır.

| Deney | Kullanılan Özellikler | Amaç |
|---|---|---|
| A | Düğüm özellikleri | Düğüm bilgilerinin katkısı |
| B | Düğüm + kenar özellikleri | İlişkilerin katkısı |
| C | Düğüm + kenar + zaman bilgisi | Zamansal bilginin katkısı |
| D | Tam graf modeli | Bütün bileşenlerin birlikte değerlendirilmesi |

Deneylerde aynı veri bölünmesi ve eğitim koşulları kullanılacaktır.

### 14.5. Tekrarlanabilirlik

Deneylerin yeniden gerçekleştirilebilmesi için aşağıdaki bilgiler kaydedilecektir:

- Linux kernel sürümü.
- Kubernetes ve container runtime sürümleri.
- eBPF program sürümü.
- Go ve Python sürümleri.
- PyTorch ve PyTorch Geometric sürümleri.
- Veri seti sürümü.
- Eğitim, doğrulama ve test ayrımları.
- Model parametreleri.
- Rastgelelik tohumu (random seed).
- Donanım bilgileri.
- Performans ölçüm sonuçları.

Ham veriler, işlenmiş veriler ve model çıktıları birbirinden ayrı saklanacaktır.

---

## 15. Teknoloji Altyapısı

| Kategori | Teknoloji |
|---|---|
| İşletim Sistemi | Linux |
| Çekirdek İzleme | eBPF |
| Programlama Dilleri | C, Go, Python |
| eBPF Kütüphanesi | cilium/ebpf |
| Konteyner | Docker |
| Orkestrasyon | Kubernetes |
| Yerel Küme | Kind / Minikube |
| Test Uygulaması | Google Online Boutique |
| Trafik Üretimi | Locust |
| Makine Öğrenmesi | PyTorch |
| Graf Öğrenme | PyTorch Geometric |
| Graf Modelleri | GCN, GAT |
| Baseline Modeller | Isolation Forest, Random Forest |
| Veri İşleme | Python |
| Veri Saklama | SQLite |
| Metrik Görselleştirme | Grafana |
| Graf Görselleştirme | Cytoscape.js (değerlendirilecek) |

---

## 16. Proje Klasör Yapısı

Planlanan proje klasör yapısı aşağıdaki gibidir:

```text
CloudPulse-GNN/
│
├── README.md
├── LICENSE
├── requirements.txt
├── go.mod
├── .gitignore
│
├── ebpf/
│   ├── probes/
│   │   ├── process.c
│   │   ├── file.c
│   │   └── network.c
│   │
│   └── agent/
│       ├── main.go
│       ├── collector.go
│       ├── event.go
│       ├── metadata.go
│       └── exporter.go
│
├── infrastructure/
│   ├── kubernetes/
│   │   ├── namespace.yaml
│   │   ├── deployment.yaml
│   │   └── online-boutique/
│   │
│   └── grafana/
│       └── dashboards/
│
├── src/
│   ├── preprocessing/
│   │   ├── parser.py
│   │   ├── normalizer.py
│   │   └── enrichment.py
│   │
│   ├── graph_builder/
│   │   ├── nodes.py
│   │   ├── edges.py
│   │   ├── provenance.py
│   │   └── temporal.py
│   │
│   ├── models/
│   │   ├── isolation_forest.py
│   │   ├── random_forest.py
│   │   ├── gcn.py
│   │   └── gat.py
│   │
│   ├── detection/
│   │   ├── anomaly_detector.py
│   │   ├── scoring.py
│   │   └── evaluation.py
│   │
│   ├── rca/
│   │   ├── analyzer.py
│   │   └── subgraph.py
│   │
│   └── storage/
│       └── database.py
│
├── dataset/
│   ├── raw/
│   ├── processed/
│   ├── graphs/
│   └── labels/
│
├── attacks/
│   ├── process/
│   ├── file/
│   └── network/
│
├── notebooks/
│   ├── exploratory_analysis.ipynb
│   ├── graph_analysis.ipynb
│   └── model_evaluation.ipynb
│
└── docs/
    ├── architecture.md
    ├── research.md
    ├── data_model.md
    └── evaluation.md
```

*Not: Bu yapı hedeflenen proje mimarisini göstermektedir. Henüz bütün dosyalar oluşturulmuş veya geliştirilmiş değildir.*

---

## 17. Geliştirme Yol Haritası

Proje, birbirini takip eden geliştirme aşamalarıyla gerçekleştirilecektir.

### Aşama 1 – Altyapının Hazırlanması

- [ ] Linux geliştirme ortamının hazırlanması.
- [ ] Kubernetes test kümesinin kurulması.
- [ ] Online Boutique uygulamasının dağıtılması.
- [ ] Locust ile normal trafik oluşturulması.

### Aşama 2 – eBPF Geliştirme

- [ ] Süreç izleme programlarının geliştirilmesi.
- [ ] Dosya erişim izleme programlarının geliştirilmesi.
- [ ] Ağ bağlantısı izleme programlarının geliştirilmesi.
- [ ] Olay toplama mekanizmasının hazırlanması.
- [ ] Olay kayıplarının ölçülmesi.

### Aşama 3 – Go Agent Geliştirme

- [ ] Go tabanlı ajanın hazırlanması.
- [ ] `cilium/ebpf` entegrasyonu.
- [ ] JSONL olay çıktısının oluşturulması.
- [ ] Süreç metadata bilgilerinin alınması.
- [ ] Container metadata ilişkilendirmesi.
- [ ] Python katmanına olay aktarımı.

### Aşama 4 – Olay İşleme

- [ ] JSON olay okuyucusunun geliştirilmesi.
- [ ] Olay normalizasyonu.
- [ ] Zaman damgalarının standartlaştırılması.
- [ ] Süreç kimliklerinin eşleştirilmesi.
- [ ] Kubernetes metadata bilgilerinin eklenmesi.

### Aşama 5 – Provenans Grafı

- [ ] Düğüm yapısının oluşturulması.
- [ ] Kenar yapısının oluşturulması.
- [ ] Süreç ilişkilerinin modellenmesi.
- [ ] Dosya erişim ilişkilerinin modellenmesi.
- [ ] Ağ bağlantılarının modellenmesi.
- [ ] Kubernetes kaynaklarının eklenmesi.
- [ ] Zamansal graf yapısının hazırlanması.

### Aşama 6 – Veri Seti Oluşturma

- [ ] Normal aktivite senaryolarının hazırlanması.
- [ ] Anormal aktivite senaryolarının hazırlanması.
- [ ] Veri etiketleme mekanizmasının geliştirilmesi.
- [ ] Eğitim, doğrulama ve test kümelerinin oluşturulması.
- [ ] PyTorch Geometric veri yapısının hazırlanması.

### Aşama 7 – Makine Öğrenmesi

- [ ] Isolation Forest modelinin geliştirilmesi.
- [ ] Random Forest modelinin geliştirilmesi.
- [ ] GCN modelinin geliştirilmesi.
- [ ] GAT modelinin geliştirilmesi.
- [ ] Anomali skorlamasının hazırlanması.
- [ ] Model karşılaştırmalarının yapılması.

### Aşama 8 – Kök Neden Analizi

- [ ] Anomaliyle ilişkili düğümlerin belirlenmesi.
- [ ] Şüpheli alt graf oluşturulması.
- [ ] Süreç ilişkilerinin incelenmesi.
- [ ] Kubernetes kaynak bilgilerinin gösterilmesi.

### Aşama 9 – Veri Saklama ve Görselleştirme

- [ ] SQLite entegrasyonu.
- [ ] Anomali sonuçlarının kaydedilmesi.
- [ ] Grafana entegrasyonu.
- [ ] Anomali gösterge panelinin oluşturulması.
- [ ] Sistem performans metriklerinin görselleştirilmesi.

### Aşama 10 – Deneysel Değerlendirme

- [ ] Model başarı ölçümlerinin yapılması.
- [ ] Yanlış alarm oranının incelenmesi.
- [ ] Olay toplama gecikmesinin ölçülmesi.
- [ ] CPU ve bellek tüketiminin ölçülmesi.
- [ ] Model tahmin süresinin ölçülmesi.
- [ ] Deneysel sonuçların raporlanması.

---

## 18. Kurulum

CloudPulse-GNN henüz geliştirme aşamasında olduğu için aşağıdaki adımlar hedeflenen geliştirme ortamını tanımlamaktadır.

### 18.1. Gereksinimler

- Linux tabanlı işletim sistemi.
- Docker.
- Kubernetes.
- Kind veya Minikube.
- Go.
- Python 3.
- eBPF destekli Linux kernel.
- Gerekli kernel headers.
- Uygun BTF desteği.
- eBPF programlarını çalıştırmak için gerekli yetkiler.

### 18.2. Depoyu İndirme

```bash
git clone https://github.com/USERNAME/CloudPulse-GNN.git

cd CloudPulse-GNN
```

`USERNAME` alanı GitHub kullanıcı adıyla değiştirilmelidir.

### 18.3. Python Ortamını Hazırlama

```bash
python3 -m venv venv

source venv/bin/activate

pip install -r requirements.txt
```

PyTorch ve PyTorch Geometric kurulumları kullanılacak Python, işletim sistemi ve donanım sürümlerine uygun olarak yapılandırılmalıdır.

### 18.4. Kubernetes Ortamını Hazırlama

Kind veya Minikube kullanılarak yerel bir Kubernetes kümesi oluşturulacaktır.

### 18.5. eBPF Gereksinimleri

eBPF programlarının yüklenebilmesi için uygun kernel desteği ve gerekli yetkiler sağlanmalıdır.

Kullanılacak izleme mekanizmalarına göre kernel sürümü, gerekli yetkiler ve program türleri ayrıca belirlenecektir.

*Not: Çalıştırılabilir kurulum ve başlatma komutları ilgili bileşenler geliştirildikçe eklenecektir.*

---

## 19. Sınırlılıklar ve Teknik Zorluklar

Projenin geliştirilmesi sırasında aşağıdaki teknik sorunların ele alınması gerekmektedir.

### 19.1. Olay Toplama Maliyeti

Yüksek sistem aktivitesi bulunan ortamlarda eBPF olaylarının toplanması CPU ve bellek tüketimini artırabilir.

### 19.2. Olay Kaybı

Kernel ile kullanıcı alanı arasındaki veri aktarımında buffer kapasitesi ve olay yoğunluğu nedeniyle veri kaybı yaşanabilir.

### 19.3. Süreç Kimliklerinin Yönetimi

PID değerlerinin yeniden kullanılması, süreçlerin doğru şekilde eşleştirilmesini zorlaştırabilir.

### 19.4. Grafın Ölçeklenebilirliği

Büyük Kubernetes ortamlarında graf boyutu hızla büyüyebilir.

Bu nedenle zaman pencereleri ve alt graf oluşturma yöntemleri önemlidir.

### 19.5. Sınıf Dengesizliği

Normal olayların anormal olaylardan çok daha fazla olması model performansını etkileyebilir.

### 19.6. Yanlış Alarmlar

Normal ancak nadir gerçekleşen sistem aktiviteleri yanlış alarm oluşturabilir.

### 19.7. Model Açıklanabilirliği

GNN modellerinin ürettiği sonuçların güvenlik analistleri tarafından anlaşılabilir olması gerekir.

Bu nedenle model skorlarının ilişkili olduğu düğüm ve kenarların incelenmesi önemlidir.

---

## 20. Gelecekteki Geliştirmeler

İlk prototipin tamamlanmasının ardından aşağıdaki konular değerlendirilebilir:

- Zamansal Graf Sinir Ağları.
- Dinamik graf öğrenmesi.
- Denetimsiz ve öz denetimli graf öğrenmesi.
- Açıklanabilir GNN yöntemleri.
- Gerçek zamanlı olay işleme.
- Çok düğümlü Kubernetes desteği.
- Gelişmiş süreç ilişki analizi.
- Graf tabanlı tehdit avcılığı.
- Çevrim içi anomali tespiti.
- Dağıtık izleme altyapısı.

Özellikle zamansal graf öğrenmesi ve açıklanabilir GNN yöntemleri, projenin araştırma kapsamını genişletebilecek önemli alanlardır.

---

## 21. Güvenlik ve Etik

CloudPulse-GNN, akademik araştırma, eğitim ve yetkili güvenlik testleri amacıyla geliştirilmektedir.

Proje kapsamında:

- Sistem olayları yalnızca yetkili ortamlarda toplanacaktır.
- Hassas dosya içerikleri gereksiz yere kaydedilmeyecektir.
- Ağ izleme işlemlerinde mümkün olduğunca metadata kullanılacaktır.
- Toplanan verilerin gizliliği korunacaktır.
- Test senaryoları izole Kubernetes ortamında gerçekleştirilecektir.
- İzleme ajanının yetkileri mümkün olduğunca sınırlandırılacaktır.

---

## 22. Kaynaklar

Projenin geliştirilmesi sırasında aşağıdaki teknik dokümantasyonlardan yararlanılacaktır.

| Kaynak | Bağlantı |
|---|---|
| eBPF Documentation | https://docs.ebpf.io/ |
| Cilium eBPF Go Library | https://github.com/cilium/ebpf |
| Kubernetes Documentation | https://kubernetes.io/docs/ |
| PyTorch Documentation | https://pytorch.org/docs/stable/ |
| PyTorch Geometric | https://pytorch-geometric.readthedocs.io/ |
| Kind Documentation | https://kind.sigs.k8s.io/ |
| Google Online Boutique | https://github.com/GoogleCloudPlatform/microservices-demo |
| Locust Documentation | https://docs.locust.io/ |
| Grafana Documentation | https://grafana.com/docs/ |

---

## 23. Lisans

CloudPulse-GNN, MIT Lisansı altında lisanslanacaktır.

Kullanım, değiştirme ve dağıtım koşulları için proje içerisindeki `LICENSE` dosyasına bakınız.

---

<div align="center">

### CloudPulse-GNN

**Çekirdek seviyesinde izle. Graf ilişkileriyle analiz et. Makine öğrenmesiyle anormallikleri tespit et.**

*eBPF · Sistem Provenansı · Graf Sinir Ağları · Kubernetes Güvenliği*

</div>
