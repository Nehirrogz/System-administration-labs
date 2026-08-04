# 1. Hafta Ödevi: Linux Ağ ve Yönlendirme Raporu
> **Debian Router İle İzole Ağlar Arası Katman-3 Yönlendirme Yapılandırması**

* **Hazırlayan:** Nehir Oğuz
* **Tarih:** 07.07.2026

---

##  Bölüm 1: Amaç ve Genel Mimari

> **Özet Görev:** VMware ortamında oluşturulan iki tamamen izole ağın (**VMnet3** ve **VMnet4**), çift ağ kartı tanımlanmış bir **Debian Linux Sunucusu** üzerinden haberleştirilmesi ve IP Forwarding ile yönlendirici olarak çalıştırılmasıdır.

### Ağ Topolojisi ve IP Planlaması
Ubuntu VM ve Rocky VM farklı subnet'lerde yer almakta olup, aralarındaki tek geçiş noktası Debian Router'dır:

```text
[ Ubuntu VM ]  <--->  (10.10.7.0/24)  <--->  [ Debian Router ]  <--->  (10.10.20.0/24)  <--->  [ Rocky VM ]
10.10.7.70                                     10.10.7.1 | 10.10.20.1                                10.10.20.90

# 📄 SAYFA 2: İstemci Sunucuların Yapılandırılması

```markdown
## 🖥️ Bölüm 3: İstemci Sunucuların Yapılandırılması

### 1. Ubuntu Sunucu
* **Ağ:** VMnet3
* **IP Adresi:** `10.10.7.70/24`
* **Gateway:** `10.10.7.1`
* **Arayüz:** `enp2s0`

Netplan (YAML) yapılandırmasında kartın bulunamaması sorunu, MAC adresi tanımı (`match`) eklenerek çözülmüştür[cite: 1].

#### Netplan Yapılandırması (`/etc/netplan/50-cloud-init.yaml`)
```yaml
network:
  version: 2
  ethernets:
    enp2s0:
      match:
        macaddress: 00:0c:29:7b:9a:80
      dhcp4: no
      addresses:
        - 10.10.7.70/24
      routes:
        - to: default
          via: 10.10.7.1
