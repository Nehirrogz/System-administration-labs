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

```markdown
acl "trusted" {
    127.0.0.0/8;
    10.10.7.0/24;
    10.10.20.0/24;
};

options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { "trusted"; };
    listen-on { 10.10.7.1; 10.10.20.1; };
};
```
