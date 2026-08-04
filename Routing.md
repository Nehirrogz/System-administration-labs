## Linux Ağ ve Yönlendirme Raporu
> **Debian Router İle İzole Ağlar Arası Katman-3 Yönlendirme Yapılandırması**

---
## Amaç ve Genel Mimari

> **Özet Görev:** VMware ortamında oluşturulan iki tamamen izole ağın (**VMnet3** ve **VMnet4**), çift ağ kartı tanımlanmış bir **Debian Linux Sunucusu** üzerinden haberleştirilmesi ve IP Forwarding ile yönlendirici olarak çalıştırılmasıdır.

### Ağ Topolojisi ve IP Planlaması
Ubuntu VM ve Rocky VM farklı subnet'lerde yer almakta olup, aralarındaki tek geçiş noktası Debian Router'dır:

```text
[ Ubuntu VM ]  <--->  (10.10.7.0/24)  <--->  [ Debian Router ]  <--->  (10.10.20.0/24)  <--->  [ Rocky VM ]
10.10.7.70                                     10.10.7.1 | 10.10.20.1                                10.10.20.90

```markdown
## İstemci Sunucuların Yapılandırılması

### 1. Ubuntu Sunucu
* **Ağ:** VMnet3
* **IP Adresi:** `10.10.7.70/24`
* **Gateway:** `10.10.7.1`
* **Arayüz:** `enp2s0`


#### Netplan Yapılandırması (`/etc/netplan/50-cloud-init.yaml`)
```yaml
network:
  version: 2
  ethernets:
    enp2s0:
      match:
      dhcp4: no
      addresses:
        - 10.10.7.70/24
      routes:
        - to: default
          via: 10.10.7.1

nehir@ubuntu:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc pfifo_fast state UP
    link/ether 00:0c:29:7b:9a:80 brd ff:ff:ff:ff:ff:ff
    inet 10.10.7.70/24 brd 10.10.7.255 scope global enp2s0


### 1. Rocky Sunucu
* **Ağ:** VMnet4
* **IP Adresi:** `10.10.20.90/24`
* **Gateway:** `10.10.20.1`
* **Arayüz:** `enp2s0`

[nehir@localhost ~]$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
2: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 00:0c:29:7a:c2:af brd ff:ff:ff:ff:ff:ff
    inet 10.10.20.90/24 brd 10.10.20.255 scope global noprefixroute enp2s0

---

#  Router Yapılandırması ve Testler

```markdown
## Debian Router ve IP Forwarding

Debian sunucusu her iki ağa da fiziksel  olarak bağlıdır. Paketlerin bir ağ arayüzünden girip diğerinden çıkabilmesi için Linux Kernel IP Forwarding parametresi aktif edilmiştir.

```bash
root@debian:~# sysctl -w net.ipv4.ip_forward=1
net.ipv4.ip_forward = 1

root@debian:~# echo "net.ipv4.ip_forward=1" >> /etc/sysctl.conf

nehir@debian:~$ ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN
    inet 127.0.0.1/8 scope host lo
2: enp2s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 00:0c:29:a4:85:a2 brd ff:ff:ff:ff:ff:ff
    inet 10.10.7.1/24 brd 10.10.7.255 scope global enp2s0
3: enp26s0: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc fq_codel state UP
    link/ether 00:0c:29:a4:85:ac brd ff:ff:ff:ff:ff:ff
    inet 10.10.20.1/24 brd 10.10.20.255 scope global enp26s0

nehir@ubuntu:~$ ping 10.10.20.90
PING 10.10.20.90 (10.10.20.90) 56(84) bayt veri.
64 bayt, 10.10.20.90'den: icmp_seq=1 ttl=63 zaman=1.16 ms
64 bayt, 10.10.20.90'den: icmp_seq=2 ttl=63 zaman=1.02 ms
64 bayt, 10.10.20.90'den: icmp_seq=3 ttl=63 zaman=0.906 ms

--- 10.10.20.90 ping istatistikleri ---
6 paket iletildi, 6 alındı, 0% packet loss, time 5010ms
rtt min/avg/max/mdev = 0.894/1.001/1.162/0.091 ms
