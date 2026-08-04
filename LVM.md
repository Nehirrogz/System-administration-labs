# LVM Yapılandırması
---

##  Amaç
LVM (Logical Volume Manager) mimarisini kullanarak dinamik disk yönetimi, Volume Group (VG) genişletme, küçültme ve disk ekleme/çıkarma süreçlerini uygulamak.

---

## LVM Kurulum Aşaması

Sanal makineye (Ubuntu) 3 GB, 4 GB ve 5 GB boyutlarında 3 adet ek sanal disk eklendi.

### A. Physical Volume (PV) Oluşturma
Eklenen bağımsız diskler LVM tarafından yönetilebilmesi için PV olarak yapılandırıldı:

```bash
ubuntu1@ubuntu:~$ sudo pvcreate /dev/nvme0n3 /dev/nvme0n2 /dev/nvme0n4
```

### B. Volume Group (VG) Oluşturma
Hazırlanan bu 3 fiziksel disk `vg0` adı verilen ortak bir havuzda (Volume Group) birleştirildi:

```bash
ubuntu1@ubuntu:~$ sudo vgcreate vg0 /dev/nvme0n3 /dev/nvme0n2 /dev/nvme0n4
Volume group "vg0" successfully created
```

### C. Logical Volume (LV) Oluşturma ve Dosya Sistemi Bağlama
`vg0` havuzundan 5 GB büyüklüğünde `lv0` isimli mantıksal birim ayrıldı. Ardından ext4 dosya sistemiyle biçimlendirilip `/mnt/depo` dizinine bağlandı:

```bash
ubuntu1@ubuntu:~$ sudo lvcreate -n lv0 -L 5G vg0
Logical volume "lv0" created.

ubuntu1@ubuntu:~$ sudo mkfs.ext4 /dev/vg0/lv0
ubuntu1@ubuntu:~$ sudo mkdir -p /mnt/depo
ubuntu1@ubuntu:~$ sudo mount /dev/vg0/lv0 /mnt/depo
```

---

## LVM Yapısını İnceleme

Oluşturulan LVM bileşenleri (PV, VG, LV) durum kontrol komutları ile doğrulandı:

```bash
ubuntu1@ubuntu:~$ sudo pvs
ubuntu1@ubuntu:~$ sudo vgs
ubuntu1@ubuntu:~$ sudo lvs
```

---

## Ek Disk Ekleme ve Volume Group (VG) Genişletme

Sisteme 10 GB'lık yeni bir disk eklendi (`/dev/nvme0n5` veya ortama göre tanınan yeni disk).

### A. Yeni Diski PV Yapma
```bash
ubuntu1@ubuntu:~$ sudo pvcreate /dev/nvme0n3
Physical volume "/dev/nvme0n3" successfully created.
```

### B. Volume Group Genişletme (`vgextend`)
Yeni PV mevcut `vg0` grubuna dahil edildi:

```bash
ubuntu1@ubuntu:~$ sudo vgextend vg0 /dev/nvme0n3
Volume group "vg0" successfully extended
```

### C. Logical Volume Büyütme (`lvextend`)
VG içerisindeki tüm boş alan (`+100%FREE`) `lv0` birimine aktarıldı:

```bash
ubuntu1@ubuntu:~$ sudo lvextend -l +100%FREE /dev/vg0/lv0
Size of logical volume vg0/lv0 changed from 13.98 GiB to 21.98 GiB.
Logical volume vg0/lv0 successfully resized.

ubuntu1@ubuntu:~$ sudo lvs
```

### D. Dosya Sistemini Genişletme (`resize2fs`)
Blok katmanı büyüdükten sonra üzerindeki ext4 dosya sistemi de yeni boyuta genişletildi:

```bash
ubuntu1@ubuntu:~$ sudo resize2fs /dev/vg0/lv0
ubuntu1@ubuntu:~$ df -h /mnt/depo/
```

---

## Diski Veri ile Doldurma ve Küçültme Testi

### A. Test Verisi Oluşturma (`dd`)
Disk çıkarma senaryosunu simüle etmek için `/dev/urandom` üzerinden rastgele veri yazılarak depo dolduruldu:

```bash
ubuntu1@ubuntu:~$ sudo dd if=/dev/urandom of=/mnt/depo/dosya1 bs=1M count=3000
ubuntu1@ubuntu:~$ sudo dd if=/dev/urandom of=/mnt/depo/dosya2 bs=1M count=3000
ubuntu1@ubuntu:~$ df -h /mnt/depo/
```

### B. Alan Açma (Gereksiz Veri Silme)
Diski guvenle çıkarabilmek için taşınacak verinin kapladığı alan temizlendi:

```bash
ubuntu1@ubuntu:~$ sudo rm /mnt/depo/dosya1
ubuntu1@ubuntu:~$ df -h /mnt/depo/
```

---

## LV ve Dosya Sistemini Küçültme (Disk Çıkarma Hazırlığı)

### A. Unmount ve Disk Kontrolü (`e2fsck`)
Küçültme öncesinde dosya sistemi unmount edildi ve bütünlük kontrolü yapıldı:

```bash
ubuntu1@ubuntu:~$ sudo umount /mnt/depo
ubuntu1@ubuntu:~$ sudo e2fsck -f /dev/vg0/lv0
```

### B. Dosya Sistemi ve LV Boyutunu Düşürme
Önce dosya sistemi 9 GB seviyesine çekildi, ardından LV katmanı 9 GB'a küçültülüp tekrar mount edildi:

```bash
# 1. Dosya sistemini küçült:
ubuntu1@ubuntu:~$ sudo resize2fs /dev/vg0/lv0 9G

# 2. LV katmanını küçült:
ubuntu1@ubuntu:~$ sudo lvreduce -L 9G /dev/vg0/lv0

# 3. Tekrar mount et ve durum kontrolü yap:
ubuntu1@ubuntu:~$ sudo mount /dev/vg0/lv0 /mnt/depo/
ubuntu1@ubuntu:~$ sudo vgs
ubuntu1@ubuntu:~$ sudo pvs
```

---

## Disk Çıkarma ve VG'den Temizleme

### A. Verileri Başka Disklere Taşıma (`pvmove`)
Çıkarılacak diskin (`/dev/nvme0n5`) üzerindeki aktif bloklar diğer disk alanlarına kaydırıldı:

```bash
ubuntu1@ubuntu:~$ sudo pvmove /dev/nvme0n5
/dev/nvme0n5: Moved: 100.00%
```

### B. Diski VG'den Ayırma (`vgreduce`)
Boşaltılan disk Volume Group yapısından çıkarıldı:

```bash
ubuntu1@ubuntu:~$ sudo vgreduce vg0 /dev/nvme0n5
Removed "/dev/nvme0n5" from volume group "vg0"
```

### C. Son Durum Doğrulaması
Disk fiziksel olarak sanal makineden silinmeden önce LVM durumu kontrol edildi:

```bash
ubuntu1@ubuntu:~$ sudo pvs
ubuntu1@ubuntu:~$ sudo vgs
ubuntu1@ubuntu:~$ sudo lvs
```
