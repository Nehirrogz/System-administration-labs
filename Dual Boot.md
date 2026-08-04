## Dual Boot İşletim Sistemi Kurma Deneyimi

**Amaç:** Tek bir sanal disk (200 GB) üzerinde Windows 11, Ubuntu ve Fedora işletim sistemlerinin bir arada sorunsuz çalıştığı bir **Triple/Dual Boot** ortamı yapılandırmaktır.

---

### A. Sanal Disk ve Windows 11 Kurulumu
1. VMware üzerinde 200 GB kapasiteli bir sanal disk oluşturuldu.
2. Windows 11 kurulum ekranında disk bölümlendirme yapılarak Windows için **70 GB** alan ayrıldı.
3. Kalan alan diğer Linux dağıtımları için ayrılmamış alan (unallocated space) olarak bırakıldı.

```text
[ Windows 11 (NTFS) ] ---> 70 GB
[ Ubuntu (ext4) ]     ---> 65 GB
[ Fedora (ext4) ]     ---> 60 GB
```

---

### B. Ubuntu ve Fedora Dağıtımlarının Kurulumu

Linux dağıtımları kurulurken diskin `/boot/efi` (EFI Sistem Bölümü) alanı ortak kullanılmış, root (`/`) dizinleri ise ayrı bölümlere kurulmuştur:

* **EFI Bölümü:** `/dev/nvme0n1p1` (`vfat` / 200 MB)
* **Ubuntu Root (`/`):** `/dev/nvme0n1p5` (`ext4` / 65 GB)
* **Fedora Root (`/`):** `/dev/nvme0n1p7` (`ext4` / 60 GB)

#### Disk Bölümlerinin Doğrulanması (`lsblk`)
Fedora Live ortamında ve sistem açılışında disk dizilimi kontrol edildi:

```bash
liveuser@localhost-live:~$ lsblk
NAME        MAJ:MIN RM   SIZE RO TYPE MOUNTPOINTS
loop0         7:0    0   2.2G 1 loop /run/rootfsbase
sr0          11:0    1   2.5G 0 rom  /run/initramfs/live
zram0       251:0    0   3.8G 0 disk [SWAP]
nvme0n1     259:0    0   200G 0 disk 
├─nvme0n1p1 259:1    0   200M 0 part /mnt/sysroot/boot/efi
├─nvme0n1p2 259:2    0    16M 0 part 
├─nvme0n1p3 259:3    0  69.3G 0 part 
├─nvme0n1p4 259:4    0   753M 0 part 
├─nvme0n1p5 259:5    0    65G 0 part 
├─nvme0n1p6 259:6    0   200M 0 part 
├─nvme0n1p7 259:7    0    60G 0 part /mnt/sysroot
└─nvme0n1p8 259:8    0     4G 0 part 
```

---

### C. Önyükleme Sıralaması ve GRUB Güncellemesi

1. Kurulumlar tamamlandıktan sonra sanal makinenin **UEFI Boot Manager** menüsüne girilerek **Ubuntu** önyükleyicisi en üst sıraya alındı.
2. Ubuntu işletim sistemi başlatıldı ve sistemdeki diğer işletim sistemlerinin (Windows 11 ve Fedora) otomatik taranıp GRUB menüsüne eklenmesi sağlandı:

```bash
ubuntu1@ubuntu:~$ sudo update-grub
Sourcing file `/etc/default/grub'
Generating grub configuration file ...
Found linux image: /boot/vmlinuz-7.0.0-28-generic
Found initrd image: /boot/initrd.img-7.0.0-28-generic
Warning: os-prober will be executed to detect other bootable partitions.
Found Windows Boot Manager on /dev/nvme0n1p1@/EFI/Microsoft/Boot/bootmgfw.efi
Found Fedora Linux 44 (Workstation Edition) on /dev/nvme0n1p7
Adding boot menu entry for UEFI Firmware Settings ...
done
```

---

### D. Sonuç ve Önyükleme Menüsü (GNU GRUB)

`update-grub` komutunun ardından sanal makine her başlatıldığında kullanıcıyı karşılayan ve istenilen işletim sisteminin seçilmesine olanak tanıyan ortak açılış menüsü elde edilmiştir:

```text
               GNU GRUB version 2.14
+-------------------------------------------------------------+
|*Ubuntu                                                      |
| Advanced options for Ubuntu                                 |
| Windows Boot Manager (on /dev/nvme0n1p1)                    |
| Fedora Linux 44 (Workstation Edition) (on /dev/nvme0n1p7)   |
| UEFI Firmware Settings                                      |
+-------------------------------------------------------------+
```
