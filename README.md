# CloudPulse-GNN: eBPF & Graph Neural Network Based Cloud-Native Anomaly Detection Framework

CloudPulse-GNN, bulut-yeril Kubernetes mikroservis mimarilerinde 0. gün tehditlerini ve yanal sızma hareketlerini tespit etmek amacıyla geliştirilmiş, eBPF tabanlı çekirdek telemetrisi ile Graf Sinir Ağlarını birleştiren uçtan uca bir güvenlik ve anomali tespit altyapısıdır.

## Öne Çıkan Özgün Değer

Geleneksel IDS/IPS ve SIEM çözümleri, statik imzalara dayanır veya eskimiş, sentetik kamu veri setleriyle eğitilir. CloudPulse-GNN bu yaklaşımı tamamen reddeder:

1. **Sıfır Bağımlılıkla Mülk Veri Üretimi:** Dışarıdan hazır veri seti indirilmez. Canlı Kubernetes kümesinde Locust ile sentetik kullanıcı trafiği ve MITRE ATT&CK for Containers matrisine uygun saldırı senaryoları eşzamanlı koşularak veri seti sıfırdan üretilir.
2. **Kaçırılamaz Çekirdek Seviyesi Telemetri:** Uygulama seviyesindeki log filtrelerini veya yetkisi değiştirilmiş logları atlatabilen gelişmiş saldırılar, Cilium Tetragon sayesinde doğrudan Linux Kernel seviyesinde izlenir.
3. **Bağlamsal İlişki Analizi:** Sistem çağrıları tekil zaman serileri olarak değil, süreçler, IP'ler ve dosyalar arasındaki zamansal ilişki ağını kuran Provenans Grafları biçiminde modellenir.

---

## Sistem Mimarisi ve Veri Akışı

Projenin temel çalışma prensibi aşağıdaki 4 aşamalı veri işleme hattına dayanır:

```text
+-----------------------------------------------------------------------------------+
| 1. CANLI KÜME VE BASTIRMA                                                         |
|    Minikube K8s Cluster <--> Google Online Boutique (Microservices)               |
|    |--> Locust Traffic Gen (Label: 0 - Normal Behavior)                           |
|    |--> MITRE ATT&CK Scripts (Label: 1 - Container Escape / Privilege Escalation) |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
| 2. ÇEKİRDEK İZLEME VE TELEMETRİ (eBPF / TETRAGON)                                 |
|    Linux Kernel Hooking -> Executive / Network / File Syscalls                    |
|    JSON Structured Streaming Logs (Raw Observability Data)                        |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
| 3. VERİ MÜHENDİSLİĞİ VE GRAF DÖNÜŞÜMÜ (GRAPH PIPELINE)                            |
|    Python Preprocessing -> Temporal Sliding Windows (e.g., 10s intervals)         |
|    Node/Edge Feature Extraction -> PyTorch Geometric (.pt) Graph Objects          |
+------------------------------------+----------------------------------------------+
                                     |
                                     v
+-----------------------------------------------------------------------------------+
| 4. YAPAY ZEKA VE GÖZLEMLENEBİLİRLİK (INFERENCE & DASHBOARD)                       |
|    GNN Model (GCN / Graph Attention Networks - GAT)                               |
|    Anomaly Scoring (<50ms latency) -> SQLite DB -> Real-time Grafana Alerts        |
+-----------------------------------------------------------------------------------+

Projenin Amacı Tehdit Tespiti: Mikroservis konteynerlerinden kaçış, yetki yükseltme ve zararlı komut çalıştırma davranışlarını çekirdek seviyesinde tespit etmek.
Yüksek Doğruluk: Yapısal graf verileri üzerinde çalışan GNN/GAT modelleri ile yanlış alarm oranını minimuma indirmek.
Gerçek Zamanlılık: Tespit edilen tehditleri <50ms gecikme süresi ile operatör paneline yansıtarak canlı anomali haritası oluşturmak.

Teknoloji Yığını:
| Kategori | Kullanılan Teknolojiler ve Araçlar |
| :--- | :--- |
| Altyapı & Orkestrasyon | Linux , Minikube , Kubernetes, Docker, Helm |
| Çekirdek İzleme  | Cilium Tetragon, eBPF Sensor Agent |
| Simülasyon & Hedef | Google Online Boutique, Locust (Python), MITRE ATT&CK for Containers |
| Veri Mühendisliği | Python 3.10+, Pandas, NetworkX, NumPy |
| Yapay Zeka / GNN | PyTorch 2.0+, PyTorch Geometric , CUDA |
| Gözlemlenebilirlik | Grafana, Prometheus / SQLite Engine |


12 Haftalık Uygulama Yol Haritası :

Faz 1: Altyapı ve Test Ortamı Kurulumu (Hafta 1-2):
[ ] Minikube yerel Kubernetes kümesinin yapılandırılması.
[ ] Google Online Boutique mikroservis uygulamasının dağıtımı.
[ ] Cilium Tetragon eBPF pod'larının ve güvenlik politikalarının aktifleştirilmesi.
Faz 2: Mülk Veri Üretimi ve Simülasyon (Hafta 3-4):
[ ] Locust ile sentetik kullanıcı trafiği senaryolarının koşturulması (Label = 0).
[ ] kubectl exec, yetki yükseltme ve yanal sızma ataklarının simüle edilmesi (Label = 1).
[ ] Ham Tetragon JSON loglarının disk üzerinde yapılandırılması.
Faz 3: Veri Mühendisliği ve Graf Dönüşümü (Hafta 5-6):
[ ] Sistem çağrılarını düğüm ve kenar ilişkilerine dönüştüren Python modülünün yazılması.
[ ] Zaman pencereleme tekniği ile verilerin PyTorch Geometric veri setine dönüştürülmesi.
Faz 4: GNN Model Tasarımı ve Eğitimi (Hafta 7-9):
[ ] GCN ve GAT mimarilerinin PyG ile kodlanması.
[ ] Model eğitimi, hiperparametre optimizasyonu ve başarım metriklerinin değerlendirilmesi.
Faz 5: Gözlemlenebilirlik ve Grafana Entegrasyonu (Hafta 10-11):
[ ] Model tahmin skorlarını anlık olarak veri tabanına aktaran köprü servisin geliştirilmesi.
[ ] Grafana üzerinde kırmızı alarm bölgesine sahip canlı Pod güvenlik paneli tasarımı.
Faz 6: Raporlama ve Savunma (Hafta 12):
[ ] Akademik proje raporunun hazırlanması ve COMP-SIF standartlarında literatür karşılaştırması.
[ ] Jüri sunum ve canlı demo hazırlıklarının tamamlanması. 

Kilometre Taşları:
KM-1 [Altyapı] :Minikube üzerinde Tetragon ile eBPF loglarının JSON olarak eksiksiz toplanması.
KM-2 [Veri Seti] :Kendi simülasyon ortamımızdan elde edilen PyG .pt formatındaki veri setinin hazır olması.
KM-3 [Model] :GNN modelinin doğrulama setinde yüksek F1 skoru (%80+) yakalaması.
KM-4 [Entegrasyon] :Tehdit tespit anında Grafana panelinde ilgili Pod'un anomali uyarısı vermesi.
KM-5 [Final] :Uçtan uca çalışan demo ve akademik rapor teslimi.

Lisans:
Bu proje akademik ve araştırma amaçlı geliştirilmekte olup MIT Lisansı altında lisanslanmıştır.
