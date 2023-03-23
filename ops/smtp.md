# Bina pelayan penghantaran mel SMTP anda sendiri

## mukadimah

SMTP boleh membeli terus perkhidmatan daripada vendor awan, seperti:

* [Amazon SES SMTP](https://docs.aws.amazon.com/ses/latest/dg/send-email-smtp.html)
* [Tolak e-mel awan Ali](https://www.alibabacloud.com/help/directmail/latest/three-mail-sending-methods)

Anda juga boleh membina pelayan mel anda sendiri - penghantaran tanpa had, kos keseluruhan yang rendah.

Di bawah, kami menunjukkan langkah demi langkah cara membina pelayan mel kami sendiri.

## Pemilihan pelayan

Pelayan SMTP yang dihoskan sendiri memerlukan IP awam dengan port 25, 456 dan 587 terbuka.

Awan awam yang biasa digunakan telah menyekat port ini secara lalai, dan mungkin boleh membukanya dengan mengeluarkan perintah kerja, tetapi ia sangat menyusahkan.

Saya mengesyorkan membeli daripada hos yang membuka port ini dan menyokong penyediaan nama domain terbalik.

Di sini, saya cadangkan [Contabo](https://contabo.com) .

Contabo ialah penyedia pengehosan yang berpangkalan di Munich, Jerman, ditubuhkan pada tahun 2003 dengan harga yang sangat kompetitif.

Jika anda memilih Euro sebagai mata wang pembelian, harganya akan lebih murah (pelayan dengan memori 8GB dan 4 CPU berharga kira-kira 529 yuan setahun, dan yuran pemasangan permulaan adalah percuma selama satu tahun).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/UoAQkwY.webp)

Apabila membuat pesanan, komen `prefer AMD` , dan pelayan dengan CPU AMD akan mempunyai prestasi yang lebih baik.

Dalam perkara berikut, saya akan mengambil VPS Contabo sebagai contoh untuk menunjukkan cara membina pelayan mel anda sendiri.

## Konfigurasi sistem Ubuntu

Sistem pengendalian di sini ialah Ubuntu 22.04

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/smpIu1F.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/m7Mwjwr.webp)

Jika pelayan pada ssh memaparkan `Welcome to TinyCore 13!` (seperti yang ditunjukkan dalam rajah di bawah), ia bermakna sistem belum dipasang lagi. Sila putuskan sambungan ssh dan tunggu beberapa minit untuk log masuk semula.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/-qKACz9.webp)

Apabila `Welcome to Ubuntu 22.04.1 LTS` muncul, pemulaan selesai dan anda boleh meneruskan langkah berikut.

### [Pilihan] Mulakan persekitaran pembangunan

Langkah ini adalah pilihan.

Untuk kemudahan, saya meletakkan pemasangan dan konfigurasi sistem perisian ubuntu dalam [github.com/wactax/ops.os/tree/main/ubuntu](https://github.com/wactax/ops.os/tree/main/ubuntu) .

Jalankan arahan berikut untuk memasang dengan satu klik.

```
bash <(curl -s https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

Pengguna Cina, sila gunakan arahan berikut dan bahasa, zon waktu, dsb. akan ditetapkan secara automatik.

```
CN=1 bash <(curl -s https://ghproxy.com/https://raw.githubusercontent.com/wactax/ops.os/main/ubuntu/boot.sh)
```

### Contabo membolehkan IPV6

Dayakan IPV6 supaya SMTP juga boleh menghantar e-mel dengan alamat IPV6.

edit `/etc/sysctl.conf`

Ubah suai atau tambah baris berikut

```
net.ipv6.conf.all.disable_ipv6 = 0
net.ipv6.conf.default.disable_ipv6 = 0
net.ipv6.conf.lo.disable_ipv6 = 0
```

Susulan dengan [tutorial contabo: Menambah sambungan IPv6 pada pelayan anda](https://contabo.com/blog/adding-ipv6-connectivity-to-your-server/)

Edit `/etc/netplan/01-netcfg.yaml` , tambahkan beberapa baris seperti yang ditunjukkan dalam rajah di bawah (Fail konfigurasi lalai Contabo VPS sudah mempunyai baris ini, cuma nyahkomennya).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/5MEi41I.webp)

Kemudian `netplan apply` untuk membuat konfigurasi yang diubah suai berkuat kuasa.

Selepas konfigurasi berjaya, anda boleh menggunakan `curl 6.ipw.cn` untuk melihat alamat ipv6 rangkaian luaran anda.

## Klon ops repositori konfigurasi

```
git clone https://github.com/wactax/ops.soft.git
```

## Hasilkan sijil SSL percuma untuk nama domain anda

Menghantar mel memerlukan sijil SSL untuk penyulitan dan tandatangan.

Kami menggunakan [acme.sh](https://github.com/acmesh-official/acme.sh) untuk menjana sijil.

acme.sh ialah alat tandatangan sijil automatik sumber terbuka,

Masukkan ops.soft gudang konfigurasi, jalankan `./ssl.sh` , dan folder `conf` akan dibuat dalam **direktori atas** .

Cari pembekal DNS anda daripada [acme.sh dnsapi](https://github.com/acmesh-official/acme.sh/wiki/dnsapi) , edit `conf/conf.sh` .

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Qjq1C1i.webp)

Kemudian jalankan `./ssl.sh 123.com` untuk menjana sijil `123.com` dan `*.123.com` untuk nama domain anda.

Larian pertama akan memasang [acme.sh](https://github.com/acmesh-official/acme.sh) secara automatik dan menambah tugas berjadual untuk pembaharuan automatik. Anda boleh melihat `crontab -l` , terdapat baris seperti berikut.

```
52 0 * * * "/mnt/www/.acme.sh"/acme.sh --cron --home "/mnt/www/.acme.sh" > /dev/null
```

Laluan untuk sijil yang dijana adalah seperti `/mnt/www/.acme.sh/123.com_ecc。`

Pembaharuan sijil akan memanggil skrip `conf/reload/123.com.sh` , edit skrip ini, anda boleh menambah arahan seperti `nginx -s reload` untuk menyegarkan cache sijil aplikasi berkaitan.

## Bina pelayan SMTP dengan chasquid

[chasquid](https://github.com/albertito/chasquid) ialah pelayan SMTP sumber terbuka yang ditulis dalam bahasa Go.

Sebagai pengganti program pelayan mel purba seperti Postfix dan Sendmail, chasquid lebih ringkas dan mudah digunakan, dan ia juga lebih mudah untuk pembangunan sekunder.

Jalankan `./chasquid/init.sh 123.com` akan dipasang secara automatik dengan satu klik (ganti 123.com dengan nama domain penghantaran anda).

## Konfigurasikan Tandatangan E-mel DKIM

DKIM digunakan untuk menghantar tandatangan e-mel untuk mengelakkan surat daripada dianggap sebagai spam.

Selepas arahan berjalan dengan jayanya, anda akan digesa untuk menetapkan rekod DKIM (seperti yang ditunjukkan di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/LJWGsmI.webp)

Cuma tambahkan rekod TXT pada DNS anda (seperti yang ditunjukkan di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/0szKWqV.webp)

## Lihat status perkhidmatan & log

 `systemctl status chasquid` Lihat status perkhidmatan.

Keadaan operasi biasa adalah seperti yang ditunjukkan dalam rajah di bawah

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/CAwyY4E.webp)

 `grep chasquid /var/log/syslog` atau `journalctl -xeu chasquid` boleh melihat log ralat.

## Terbalikkan konfigurasi nama domain

Nama domain terbalik adalah untuk membolehkan alamat IP diselesaikan dengan nama domain yang sepadan.

Menetapkan nama domain terbalik boleh menghalang e-mel daripada dikenal pasti sebagai spam.

Apabila mel diterima, pelayan penerima akan melakukan analisis nama domain terbalik pada alamat IP pelayan penghantaran untuk mengesahkan sama ada pelayan penghantar mempunyai nama domain terbalik yang sah.

Jika pelayan penghantar tidak mempunyai nama domain terbalik atau jika nama domain terbalik tidak sepadan dengan alamat IP pelayan penghantar, pelayan penerima mungkin mengenali e-mel sebagai spam atau menolaknya.

Lawati [https://my.contabo.com/rdns](https://my.contabo.com/rdns) dan konfigurasikan seperti yang ditunjukkan di bawah

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/IIPdBk_.webp)

Selepas menetapkan nama domain terbalik, ingat untuk mengkonfigurasi resolusi hadapan nama domain ipv4 dan ipv6 ke pelayan.

## Edit nama hos chasquid.conf

Ubah suai `conf/chasquid/chasquid.conf` kepada nilai nama domain terbalik.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/6Fw4SQi.webp)

Kemudian jalankan `systemctl restart chasquid` untuk memulakan semula perkhidmatan.

## Sandarkan conf ke repositori git

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/Fier9uv.webp)

Sebagai contoh, saya menyandarkan folder conf kepada proses github saya sendiri seperti berikut

Buat gudang persendirian dahulu

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ZSCT1t1.webp)

Masukkan direktori conf dan serahkan ke gudang

```
git init
git add .
git commit -m "init"
git branch -M main
git remote add origin git@github.com:wac-tax-key/conf.git
git push -u origin main
```

## Tambah penghantar

lari

```
chasquid-util user-add i@wac.tax
```

Boleh tambah penghantar

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/khHjLof.webp)

### Sahkan bahawa kata laluan ditetapkan dengan betul

```
chasquid-util authenticate i@wac.tax --password=xxxxxxx
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/e92JHXq.webp)

Selepas menambah pengguna, `chasquid/domains/wac.tax/users` akan dikemas kini, ingat untuk menyerahkannya ke gudang.

## DNS tambah rekod SPF

SPF ( Rangka Kerja Dasar Pengirim ) ialah teknologi pengesahan e-mel yang digunakan untuk mencegah penipuan e-mel.

Ia mengesahkan identiti penghantar mel dengan menyemak sama ada alamat IP penghantar sepadan dengan rekod DNS bagi nama domain yang didakwanya, menghalang penipu daripada menghantar e-mel palsu.

Menambah rekod SPF boleh menghalang e-mel daripada dikenal pasti sebagai spam sebanyak mungkin.

Jika pelayan nama domain anda tidak menyokong jenis SPF, cuma tambah rekod jenis TXT.

Sebagai contoh, SPF `wac.tax` adalah seperti berikut

`v=spf1 a mx include:_spf.wac.tax include:_spf.google.com ~all`

SPF untuk `_spf.wac.tax`

`v=spf1 a:smtp.wac.tax ~all`

Harap maklum bahawa saya telah `include:_spf.google.com` di sini, ini kerana saya akan mengkonfigurasi `i@wac.tax` sebagai alamat penghantaran dalam peti mel Google kemudian.

## Konfigurasi DNS DMARC

DMARC ialah singkatan daripada (Pengesahan, Pelaporan & Pematuhan Mesej berasaskan Domain).

Ia digunakan untuk menangkap lantunan SPF (mungkin disebabkan oleh ralat konfigurasi, atau orang lain berpura-pura menjadi anda untuk menghantar spam).

Tambahkan rekod TXT `_dmarc` ,

Kandungannya adalah seperti berikut

```
v=DMARC1; p=quarantine; fo=1; ruf=mailto:ruf@wac.tax; rua=mailto:rua@wac.tax
```

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/k44P7O3.webp)

Maksud setiap parameter adalah seperti berikut

### p (Dasar)

Menunjukkan cara mengendalikan e-mel yang gagal pengesahan SPF (Rangka Kerja Dasar Penghantar) atau DKIM (Mel Pengenalpastian DomainKeys). Parameter p boleh ditetapkan kepada salah satu daripada tiga nilai:

* tiada: Tiada tindakan diambil, hanya hasil pengesahan dihantar semula kepada pengirim melalui mekanisme pelaporan e-mel.
* Kuarantin: Letakkan mel yang belum lulus pengesahan ke dalam folder spam, tetapi tidak akan menolak mel secara langsung.
* tolak: Tolak terus e-mel yang gagal pengesahan.

### fo (Pilihan Kegagalan)

Menentukan jumlah maklumat yang dikembalikan oleh mekanisme pelaporan. Ia boleh ditetapkan kepada salah satu daripada nilai berikut:

* 0: Laporkan hasil pengesahan untuk semua mesej
* 1: Hanya laporkan mesej yang gagal pengesahan
* d: Hanya laporkan kegagalan pengesahan nama domain
* s: hanya laporkan kegagalan pengesahan SPF
* l: Hanya laporkan kegagalan pengesahan DKIM

### rua & ruf

* rua (URI pelaporan untuk laporan Agregat): Alamat e-mel untuk menerima laporan agregat
* ruf (Melaporkan URI untuk laporan Forensik): alamat e-mel untuk menerima laporan terperinci

## Tambahkan rekod MX untuk memajukan e-mel ke Google Mail

Oleh kerana saya tidak menemui peti mel korporat percuma yang menyokong alamat universal (Catch-All, boleh menerima sebarang e-mel yang dihantar ke nama domain ini, tanpa sekatan pada awalan), saya menggunakan chasquid untuk memajukan semua e-mel ke peti mel Gmail saya.

**Jika anda mempunyai peti mel perniagaan berbayar anda sendiri, sila jangan ubah suai MX dan langkau langkah ini.**

Edit `conf/chasquid/domains/wac.tax/aliases` , tetapkan peti mel pemajuan

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/OBDl2gw.webp)

`*` menunjukkan semua e-mel, `i` ialah awalan alamat e-mel pengguna penghantar yang dibuat di atas. Untuk memajukan mel, setiap pengguna perlu menambah baris.

Kemudian tambah rekod MX (saya menunjuk terus ke alamat nama domain terbalik di sini, seperti yang ditunjukkan dalam baris pertama dalam rajah di bawah).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/7__KrU8.webp)

Selepas konfigurasi selesai, anda boleh menggunakan alamat e-mel lain untuk menghantar e-mel ke `i@wac.tax` dan `any123@wac.tax` untuk melihat sama ada anda boleh menerima e-mel dalam Gmail.

Jika tidak, semak log chasquid ( `grep chasquid /var/log/syslog` ).

## Hantar e-mel ke i@wac.tax dengan Google Mail

Selepas Google Mail menerima mel, saya sememangnya berharap untuk membalas dengan `i@wac.tax` dan bukannya i.wac.tax@gmail.com.

Lawati [https://mail.google.com/mail/u/1/#settings/accounts](https://mail.google.com/mail/u/1/#settings/accounts) dan klik "Tambah alamat e-mel lain".

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/PAvyE3C.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/_OgLsPT.webp)

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/XIUf6Dc.webp)

Kemudian, masukkan kod pengesahan yang diterima oleh e-mel yang telah dimajukan kepada.

Akhir sekali, ia boleh ditetapkan sebagai alamat penghantar lalai (bersama-sama dengan pilihan untuk membalas dengan alamat yang sama).

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/a95dO60.webp)

Dengan cara ini, kami telah menyelesaikan penubuhan pelayan mel SMTP dan pada masa yang sama menggunakan Google Mail untuk menghantar dan menerima e-mel.

## Hantar e-mel ujian untuk menyemak sama ada konfigurasi berjaya

Masukkan `ops/chasquid`

Jalankan `direnv allow` untuk memasang kebergantungan (direnv telah dipasang dalam proses permulaan satu kekunci sebelumnya dan cangkuk telah ditambahkan pada cangkerang)

kemudian lari

```
user=i@wac.tax pass=xxxx to=iuser.link@gmail.com ./sendmail.coffee
```

Maksud parameter adalah seperti berikut

* pengguna: nama pengguna SMTP
* pas: kata laluan SMTP
* kepada: penerima

Anda boleh menghantar e-mel ujian.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/ae1iWyM.webp)

Adalah disyorkan untuk menggunakan Gmail untuk menerima e-mel ujian untuk menyemak sama ada konfigurasi berjaya.

### penyulitan standard TLS

Seperti yang ditunjukkan dalam rajah di bawah, terdapat kunci kecil ini, yang bermaksud bahawa sijil SSL telah berjaya didayakan.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/SrdbAwh.webp)

Kemudian klik "Tunjukkan E-mel Asal"

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/qQQsdxg.webp)

### DKIM

Seperti yang ditunjukkan dalam rajah di bawah, halaman mel asal Gmail memaparkan DKIM, yang bermaksud bahawa konfigurasi DKIM berjaya.

![](https://pub-b8db533c86124200a9d799bf3ba88099.r2.dev/2023/03/an6aXK6.webp)

Semak Diterima dalam pengepala e-mel asal, dan anda boleh melihat bahawa alamat pengirim ialah IPV6, yang bermaksud bahawa IPV6 juga berjaya dikonfigurasikan.
