## 2. BÖLÜM: Nginx Web Server Kurulumu ve DNS Entegrasyonu

Debian sunucusuna Nginx kurulmuş ve DNS zone dosyası üzerinden `[www.profelistest.com](https://www.profelistest.com)` adresi Nginx sunucusunun IP adresine yönlendirilmiştir.

### Nginx Kurulumu
```bash
nehir@debian:~$ sudo apt install nginx -y
```

### DNS Seçeneklerinin Güncellenmesi (`/etc/bind/named.conf.options`)
DNS sunucusunun yeni ağ segmentlerinden ve IP adreslerinden gelen isteklere yanıt verebilmesi için config güncellendi:

```bind
acl "trusted" {
    127.0.0.0/8;
    10.10.7.0/24;
    10.10.20.0/24;
    192.168.111.0/24;
};

options {
    directory "/var/cache/bind";
    recursion yes;
    allow-query { "trusted"; };
    listen-on { 10.10.7.1; 10.10.20.1; 192.168.111.81; };
};
```

### SOA Serial Artırımı ve `www` A Kaydı Ekleme (`/etc/bind/db.profelistest.com`)
Her zone güncellemesinde ikincil DNS'lerin yeni bilgiyi çekebilmesi için SOA Serial numarası **1 artırıldı (3 -> 4)**

```bind
; Serial numarası 3 -> 4 olarak güncellendi
@       IN      SOA     ns1.profelistest.com. admin.profelistest.com. (
                              4         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

www     IN      A       192.168.111.81
```

### DNS ve Web Sunucu Testleri (Ubuntu & Rocky)
İstemci makineler üzerinden `curl` ve `lynx` komutlarıyla `[www.profelistest.com](https://www.profelistest.com)` adresine erişilmiş, DNS çözümlemesi ile Nginx karşılama sayfası doğrulanmıştır.

```bash
[nehir@rocky ~]$ curl http://www.profelistest.com
<!DOCTYPE html>
<html>
<head><title>Welcome to nginx!</title></head>
<body>
<h1>Welcome to nginx!</h1>
</body>
</html>
```

---

## Özel VirtualHost Yapılandırması (`nehir.profelistest.com`)

Aynı web sunucusu üzerinde ikinci bir alan adı ve özel bir web dizini tanımlanmıştır.

### DNS SOA Artırımı ve `nehir` A Kaydı (`/etc/bind/db.profelistest.com`)
SOA Serial numarası **4 -> 5** olarak yükseltilmiş ve yeni A kaydı eklenmiştir:

```bind
@       IN      SOA     ns1.profelistest.com. admin.profelistest.com. (
                              5         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL

www     IN      A       192.168.111.81
nehir   IN      A       192.168.111.81
```

### Web Dizini ve HTML Sayfası Oluşturma
```bash
nehir@debian:~$ sudo mkdir -p /var/www/nehir
nehir@debian:~$ sudo nano /var/www/nehir/index.html
```

**İçerik:**
```html
<h1>Welcome to Nehir's server page!</h1>
```

### Nginx VirtualHost Yapılandırması (`/etc/nginx/sites-available/nehir.profelistest.com`)
```nginx
server {
    listen 80;
    server_name nehir.profelistest.com;

    root /var/www/nehir;
    index index.html index.htm;

    location / {
        try_files $uri $uri/ =404;
    }
}
```

### Symlink Oluşturma ve Yayına Alma
Hazırlanan VirtualHost dosyasının Nginx tarafından okunabilmesi için `sites-enabled` dizinine symlink atanmıştır:

```bash
nehir@debian:~$ sudo ln -s /etc/nginx/sites-available/nehir.profelistest.com /etc/nginx/sites-enabled/
```
