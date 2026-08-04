# DNS Sunucusu Kurulum ve Yapılandırması

---

## Amaç
Bu çalışmanın amacı; **Debian** işletim sistemi üzerinde **BIND9** kullanarak yerel bir DNS sunucusu kurmak, **Ubuntu** ve **Rocky Linux** istemcilerini bu sunucuya bağlamak ve tüm makinelerin IP adresi yerine alan adı (hostname / FQDN) kullanarak haberleşmesini sağlamaktır.

---

##  1. BIND9 DNS Sunucusu Kurulumu (Debian)

Öncelikle BIND9 paketi Debian sunucusuna kurulmuş ve servisin durumu `systemctl` ile doğrulanmıştır:

```bash
nehir@debian:~$ systemctl status bind9
● named.service - BIND Domain Name Server
     Loaded: loaded (/usr/lib/systemd/system/named.service; enabled; preset: enabled)
     Active: active (running) since Thu 2026-07-16 13:35:16 +03; 1min 10s ago
   Main PID: 7770 (named)
     Status: "running"
```

### A. ACL (Güvenilir Ağlar) Yapılandırması
```bind
acl "trusted" {
    127.0.0.0/8;
    10.10.7.0/24;
    10.10.20.0/24;
};
```

### B. Seçenekler (Options) Yapılandırması
```bind
options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { "trusted"; };
    listen-on { 10.10.7.1; 10.10.20.1; };
};
```
---

## DNS Kayıtlarının (A ve PTR) Oluşturulması

### İleri Yönlü İsim Çözümleme (`/etc/bind/db.profelistest.com`)
```bind
$TTL    604800
@       IN      SOA     ns1.profelistest.com. admin.profelistest.com. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      ns1.profelistest.com.
ns1     IN      A       10.10.7.1
ubuntu  IN      A       10.10.7.70
rocky   IN      A       10.10.20.90

ubuntu.profelistest.com. IN A 10.10.7.70
rocky.profelistest.com.  IN A 10.10.20.90
```

### Tersine İsim Çözümleme (PTR Kayıtları)

#### A. Ubuntu Ağı için (`/etc/bind/db.10.10.7`)
```bind
1   IN  PTR ns1.profelistest.com.
70  IN  PTR ubuntu.profelistest.com.
```

#### B. Rocky Ağı için (`/etc/bind/db.10.10.20`)
```bind
90  IN  PTR rocky.profelistest.com.
```

---

## İstemci Ağ ve Hostname Yapılandırması

* **Ubuntu Netplan (`/etc/netplan/01-netcfg.yaml`):**  
  `nameservers.addresses: [10.10.7.1]` eklenip `sudo netplan apply` çalıştırıldı.

* **Rocky (`nmtui`):**  
  Grafik arayüz üzerinden DNS sunucusu `10.10.7.1` olarak atandı.

### Hostname Ayarları
```bash
# Ubuntu üzerinde:
sudo hostnamectl set-hostname ubuntu.profelistest.com

# Rocky üzerinde:
sudo hostnamectl set-hostname rocky.profelistest.com
```

---

## DNS Sorgu Testleri

`nslookup`, `dig` ve `host` komutları kullanılarak DNS sorguları test edildi:

```bash
[nehir@rocky ~]$ nslookup ubuntu.profelistest.com
Server:         10.10.7.1
Address:        10.10.7.1#53

Name:   ubuntu.profelistest.com
Address: 10.10.7.70

[nehir@rocky ~]$ host 10.10.7.70
70.7.10.10.in-addr.arpa domain name pointer ubuntu.profelistest.com.
```

### Alan Adı İle Ping Testi (`rocky` -> `ubuntu`)
```bash
[root@rocky nehir]# ping ubuntu.profelistest.com
PING ubuntu.profelistest.com (10.10.7.70) 56(84) bayt veri.
64 bayt, ubuntu.profelistest.com (10.10.7.70)'den: icmp_seq=1 ttl=63 zaman=0.698 ms

--- ubuntu.profelistest.com ping istatistikleri ---
5 paket iletildi, 5 alındı, 0% packet loss, time 4011ms
```
