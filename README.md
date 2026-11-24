# CloudPulse-GNN

Kubernetes mikroservis ortamlarında meydana gelebilecek anomali ve yetkisiz erişim girişimlerini çekirdek seviyesinde tespit etmek için geliştirilen eBPF ve Graf Sinir Ağları tabanlı güvenlik izleme projesi.

## Projenin Amacı ve Yaklaşımı

Klasik güvenlik araçları genellikle uygulama katmanındaki loglara veya hazır veri setlerine dayanır. Bu projede ise hem veriyi kendimiz üretmeyi hem de sistem çağrılarını doğrudan çekirdek seviyesinde yakalamayı hedefliyoruz:

1. **Kendi Veri Setimizi Üretme:** Dışarıdan hazır veri seti indirmek yerine, lokalde kurduğumuz Kubernetes kümesi üzerinde Locust ile normal kullanıcı trafiği oluşturup, aynı anda belirlediğimiz saldırı senaryolarını çalıştırarak kendi etiketli verimizi elde edeceğiz.
2. **Çekirdek Seviyesi İzleme:** Uygulama logları manipüle edilebildiği için, sistem çağrılarını doğrudan Linux çekirdeği üzerinden **Cilium Tetragon** aracıyla JSON formatında toplayacağız.
3. **Graf Tabanlı Analiz:** Toplanan süreç, ağ ve dosya hareketlerini tekil satırlar olarak değil, aralarındaki ilişkiyi gösteren graflar halinde modelleyip GNN algoritmaları ile sınıflandıracağız.

---

## Sistem Mimarisi

Verinin üretiminden anomali tespitine kadar izlenecek temel akış:

```text
+-----------------------------------------------------------------------+
| 1. Test Ortamı ve Trafik Üretimi                                      |
|    Minikube K8s Cluster -> Google Online Boutique                     |
|    |-- Locust ile Normal Trafik - Etiket: 0                           |
|    |-- Saldırı Senaryoları / Container Escape vb. - Etiket: 1         |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| 2. Çekirdek Seviyesi Loglama                                          |
|    Linux Kernel Syscalls -> JSON Log Akışı                            |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| 3. Veri İşleme ve Graf Oluşturma                                      |
|    Python ile zaman pencerelerine bölme                               |
|    Düğüm ve kenar çıkarımı -> PyTorch Geometric veri seti             |
+-----------------------------------+-----------------------------------+
                                    |
                                    v
+-----------------------------------------------------------------------+
| 4. Model Eğitimi ve Görselleştirme                                    |
|    GNN / GAT Modeli -> Anomali Skoru -> SQLite -> Grafana Paneli      |
+-----------------------------------------------------------------------+
Lisans:
Bu proje akademik ve araştırma amaçlı geliştirilmekte olup MIT Lisansı altında lisanslanmıştır.
