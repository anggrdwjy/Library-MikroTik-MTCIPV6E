# MTCIPv6E — Panduan Implementasi dan Operasi

## Modul 1 - Introduction to IPv6

### Konsep Kunci

- **IPv4 dan IPv6:** IPv4 memiliki panjang 32 bit dengan jumlah alamat sekitar 4,3 miliar. IPv6 memiliki panjang 128 bit dengan jumlah alamat yang sangat besar, sehingga pendekatan perancangan tidak lagi menghemat alamat, melainkan membaginya secara terstruktur.
- **Notasi:** Notasi menggunakan heksadesimal yang dipisahkan oleh titik dua. Nol di depan setiap grup dapat dihilangkan. Satu rangkaian nol berturut-turut dapat disingkat menjadi `::` dan hanya boleh digunakan satu kali dalam satu alamat. Contoh: `2001:0db8:0000:0000:0000:0000:0000:0001` menjadi `2001:db8::1`.
- **Jenis alamat:** Global Unicast (dapat di-routing di internet, awalan `2000::/3`), Link-Local (`fe80::/10`, hanya berlaku pada satu link dan terbentuk secara otomatis), Unique Local (`fc00::/7`, setara dengan alamat private IPv4), Multicast (`ff00::/8`), Anycast (alamat yang sama digunakan oleh banyak node, paket dikirim ke node terdekat), Loopback `::1`, dan Unspecified `::`.
- **EUI-64:** Mekanisme pembentukan 64 bit terakhir alamat dari MAC address. MAC address sepanjang 48 bit disisipi `fffe` di bagian tengah, kemudian bit ke-7 dibalik (bit Universal/Local). Mekanisme ini memungkinkan SLAAC beroperasi.

### Perbandingan RouterOS v6 dan v7 — Modul 1

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| Sintaks dasar | `/ipv6 address add ...` (spasi) | `/ipv6/address/add ...` (slash) | Fungsi sama, penulisan berbeda. Skrip v6 harus disesuaikan sebelum dijalankan di v7. |
| Flag advertise | `advertise=yes/no` pada `/ipv6 address` | `advertise=yes/no` pada `/ipv6/address` | Pada v7, flag ini lebih konsisten dengan NDP. Pastikan `advertise=yes` untuk LAN klien. |
| EUI-64 | `eui-64=yes` didukung | `eui-64=yes` didukung | Pada v7, pembentukan alamat dari MAC lebih stabil dan tercatat pada `dynamic` address. |
| Link-Local | Terbentuk otomatis `fe80::/10`, dapat ditambah manual | Terbentuk otomatis `fe80::/10`, dapat ditambah manual | Pada v7, link-local manual lebih mudah dibedakan pada `/ipv6/address/print`. |
| Duplicate Address Detection (DAD) | Berjalan, status terlihat pada `address print` | Berjalan, status lebih detail pada `address print` | Pada v7, alamat yang gagal DAD ditandai lebih jelas. |
| ULA | Didukung melalui `/ipv6 address` | Didukung melalui `/ipv6/address` | Tidak ada perbedaan fungsional. |

#### Konfigurasi RouterOS v6 — Modul 1

```routeros
# v6: penulisan dengan spasi
/ipv6 address add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="LAN-Core-GW"
/ipv6 address add address=2001:db8:abcd:20::1/64 interface=ether3 advertise=yes comment="DMZ-GW"
/ipv6 address add address=fd00:abcd:10::1/64 interface=ether2 advertise=no comment="ULA-Mgmt"
/ipv6 address add address=2001:db8:abcd:10::/64 eui-64=yes interface=ether2 advertise=no comment="EUI64-Example"
/ipv6 address add address=fe80::1/64 interface=ether2 advertise=no comment="LL-Core"

# Verifikasi v6
/ipv6 address print detail
```

#### Konfigurasi RouterOS v7 — Modul 1

```routeros
# v7: penulisan dengan slash
/ipv6/address/add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="LAN-Core-GW"
/ipv6/address/add address=2001:db8:abcd:20::1/64 interface=ether3 advertise=yes comment="DMZ-GW"
/ipv6/address/add address=fd00:abcd:10::1/64 interface=ether2 advertise=no comment="ULA-Mgmt"
/ipv6/address/add address=2001:db8:abcd:10::/64 eui-64=yes interface=ether2 advertise=no comment="EUI64-Example"
/ipv6/address/add address=fe80::1/64 interface=ether2 advertise=no comment="LL-Core"

# Verifikasi v7
/ipv6/address/print detail
/ipv6/address/print where address~"fe80"
```

> **Rekomendasi:** Gunakan konfigurasi v7 untuk laboratorium baru. Apabila masih mengoperasikan v6, konversikan spasi menjadi slash sebelum migrasi, dan uji kembali `advertise` serta DAD setelah migrasi.

### Studi Kasus 1 - Kantor Cabang di Surabaya

Sebuah perusahaan di Surabaya memperoleh prefix `2001:db8:abcd::/48` dari ISP. Tim IT harus membagi prefix tersebut untuk tiga lokasi: Core, DMZ, dan Branch. Tim IT sebelumnya terbiasa melakukan subnetting IPv4 /24 dan berencana menggunakan /120 untuk LAN IPv6 agar hemat. Pendekatan tersebut merupakan kesalahan umum. Pada IPv6, penghematan tidak dilakukan dengan cara tersebut.

**Solusi:** Gunakan /64 untuk setiap LAN. Dari /48, tersedia 16 bit untuk subnet (64 dikurangi 48). Dengan demikian, tersedia 65.536 subnet /64. Jumlah tersebut memadai untuk seluruh cabang. Pembagiannya adalah sebagai berikut: 10 untuk Core, 20 untuk DMZ, dan 30 untuk Branch. Sisa prefix disimpan untuk pengembangan selanjutnya.

### Implementasi - R1 Edge

```routeros
# 1. Pasang alamat Global pada LAN Core - statis
/ipv6/address/add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="LAN-Core-GW"

# 2. Pasang alamat pada DMZ
/ipv6/address/add address=2001:db8:abcd:20::1/64 interface=ether3 advertise=yes comment="DMZ-GW"

# 3. Pasang alamat ULA untuk manajemen internal
/ipv6/address/add address=fd00:abcd:10::1/64 interface=ether2 advertise=no comment="ULA-Mgmt"

# 4. Contoh EUI-64 - biarkan RouterOS membentuk host-part dari MAC address
/ipv6/address/add address=2001:db8:abcd:10::/64 eui-64=yes interface=ether2 advertise=no comment="EUI64-Example"

# 5. Pasang Link-Local manual agar gateway mudah diingat dan stabil
/ipv6/address/add address=fe80::1/64 interface=ether2 advertise=no comment="LL-Core"
```

### Implementasi - R2 Branch

```routeros
# R2: LAN Branch
/ipv6/address/add address=2001:db8:abcd:30::1/64 interface=ether2 advertise=yes comment="LAN-Branch-GW"
/ipv6/address/add address=fe80::2/64 interface=ether2 advertise=no comment="LL-Branch"
```

### Operasi dan Verifikasi

```routeros
# Tampilkan seluruh alamat, pastikan advertise=yes untuk LAN klien
/ipv6/address/print detail

# Periksa alamat link-local yang terbentuk secara otomatis
/ipv6/address/print where address~"fe80"

# Dari PC klien: periksa apakah alamat global dan gateway telah diperoleh
# Windows: ipconfig /all
# Linux: ip -6 addr show ; ip -6 route show

# Ping gateway dari klien
ping 2001:db8:abcd:10::1
ping fe80::1%ether2
```

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **Klien tidak memperoleh alamat global:** Periksa `advertise=yes` dan pastikan NDP / RA aktif. Lihat Modul 2.
- **Alamat tampil tetapi tidak dapat melakukan ping:** Periksa firewall pada jalur tersebut dan pastikan prefix yang digunakan adalah /64, bukan /120 atau /126.
- **Kesulitan membaca alamat:** Latih kompresi alamat. Perlu diingat bahwa `::` hanya boleh digunakan satu kali dalam satu alamat.

> **Luaran Modul 1:** Tabel addressing plan yang telah diisi dan diterapkan pada R1 dan R2, serta keluaran `/ipv6/address/print` dari kedua router.

## Modul 2 - IPv6 Protocol

**Tujuan:** Mampu membuat klien memperoleh alamat secara otomatis, memahami cara kerja Neighbor Discovery, dan mampu melakukan routing IPv6 dasar.

### Konsep Kunci

- **SLAAC (Stateless Address Autoconfiguration):** Router mengirim Router Advertisement (RA). Klien membentuk alamat sendiri dari prefix pada RA ditambah EUI-64 atau privacy extension. Router tidak mencatat alamat yang diperoleh oleh masing-masing klien.
- **DHCPv6 Stateless:** Klien memperoleh alamat dari SLAAC, sedangkan informasi DNS dan informasi lainnya diperoleh dari DHCPv6 server.
- **DHCPv6 Stateful:** Klien memperoleh alamat lengkap dari DHCPv6 server, serupa dengan DHCP pada IPv4. Router tetap mengirim RA, tetapi flag pada RA menyatakan bahwa DHCPv6 harus digunakan.
- **NDP:** Pengganti ARP. Terdiri atas Router Solicitation (RS), Router Advertisement (RA), Neighbor Solicitation (NS), Neighbor Advertisement (NA), dan Redirect. NS juga digunakan untuk Duplicate Address Detection (DAD).
- **Routing IPv6:** Konsepnya sama dengan IPv4, tetapi next-hop sering menggunakan alamat link-local `fe80::`. Oleh karena itu, routing table yang berisi `fe80::` merupakan hal yang wajar.

### Perbandingan RouterOS v6 dan v7 — Modul 2

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| NDP / RA | `/ipv6 nd set`, `/ipv6 nd prefix` | `/ipv6/nd/set`, `/ipv6/nd/prefix` | Pada v7, parameter `ra-interval`, `ra-lifetime`, dan flag M/O lebih konsisten. |
| Flag M (Managed) / O (Other) | `managed-address-configuration`, `other-configuration` | `managed-address-configuration`, `other-configuration` | Fungsi sama. Pada v7, perubahan flag langsung terlihat pada RA berikutnya. |
| DHCPv6 Server | `/ipv6 dhcp-server` | `/ipv6/dhcp-server` | Pada v7, pool dan binding lebih stabil, dan `rapid-commit` didukung lebih baik. |
| DHCPv6 Client | `/ipv6 dhcp-client` terbatas | `/ipv6/dhcp-client` dengan `request=address/prefix` | Pada v6, DHCPv6 client untuk prefix delegation terbatas. Pada v7, lebih andal. |
| Neighbor | `/ipv6 neighbor print` | `/ipv6/neighbor/print` | Pada v7, status neighbor (stale, reachable, delay) lebih detail. |
| Routing dasar | `/ipv6 route add dst-address=... gateway=...` | `/ipv6/route/add dst-address=... gateway=... routing-table=main` | Pada v7, wajib memperhatikan `routing-table`. Tanpa parameter tersebut, route dapat masuk ke tabel yang tidak diinginkan. |

#### Konfigurasi RouterOS v6 — Modul 2

```routeros
# v6: NDP SLAAC
/ipv6 nd set [find default=yes] disabled=no ra-interval=20s-60s ra-lifetime=30m managed-address-configuration=no other-configuration=yes
/ipv6 nd prefix add prefix=2001:db8:abcd:10::/64 interface=ether2 autonomous=yes on-link=yes

# v6: DHCPv6 Stateful Branch
/ipv6 pool add name=pool-branch prefix=2001:db8:abcd:30::/64 prefix-length=64
/ipv6 dhcp-server add name=dhcpv6-branch interface=ether2 address-pool=pool-branch lease-time=2h
/ipv6 nd set [find interface=ether2] managed-address-configuration=yes other-configuration=yes

# v6: Routing - tanpa routing-table
/ipv6 route add dst-address=2001:db8:abcd:30::/64 gateway=fe80::2%ether1 comment="to-Branch-via-LL"
/ipv6 route add dst-address=::/0 gateway=fe80::1%ether1 comment="default-to-R1"
```

#### Konfigurasi RouterOS v7 — Modul 2

```routeros
# v7: NDP SLAAC
/ipv6/nd/set [find default=yes] disabled=no ra-interval=20s-60s ra-lifetime=30m hop-limit=64 managed-address-configuration=no other-configuration=yes
/ipv6/nd/prefix/add prefix=2001:db8:abcd:10::/64 interface=ether2 autonomous=yes on-link=yes valid-lifetime=1d preferred-lifetime=1h

# v7: DHCPv6 Stateful Branch
/ipv6/pool/add name=pool-branch prefix=2001:db8:abcd:30::/64 prefix-length=64
/ipv6/dhcp-server/add name=dhcpv6-branch interface=ether2 address-pool=pool-branch lease-time=2h
/ipv6/nd/set [find interface=ether2] managed-address-configuration=yes other-configuration=yes

# v7: DHCPv6 Stateless DNS
/ipv6/dhcp-server/add name=dhcpv6-stateless interface=ether2 address-pool=pool-dns lease-time=1h rapid-commit=yes
/ipv6/dhcp-server/option/add name=dns-opt code=23 value="'2001:db8:abcd:10::1'"

# v7: Routing - dengan routing-table=main
/ipv6/route/add dst-address=2001:db8:abcd:30::/64 gateway=fe80::2%ether1 routing-table=main comment="to-Branch-via-LL"
/ipv6/route/add dst-address=::/0 gateway=fe80::1%ether1 routing-table=main comment="default-to-R1"
```

> **Rekomendasi:** Pada v7, selalu cantumkan `routing-table=main` pada route IPv6. Pada v6, periksa kembali lease DHCPv6 setelah reboot, karena binding dapat hilang apabila tidak disimpan sebagai static binding.

### Studi Kasus 2 - WISP dengan Dua Tipe Pelanggan

Sebuah WISP memiliki dua tipe pelanggan. Pelanggan rumahan menginginkan layanan plug and play tanpa konfigurasi tambahan. Pelanggan korporat menginginkan alamat yang tercatat secara rapi dan DNS internal. Solusinya tidak memaksakan satu metode untuk seluruh pelanggan. Gunakan SLAAC untuk pelanggan rumahan, serta DHCPv6 Stateful dan DHCP PD untuk pelanggan korporat. Kedua metode tersebut dapat berjalan pada interface yang berbeda.

### Implementasi - SLAAC untuk LAN Core (R1)

```routeros
# Aktifkan NDP pada interface LAN
/ipv6/nd/set [find default=yes] disabled=no ra-interval=20s-60s ra-lifetime=30m hop-limit=64 managed-address-configuration=no other-configuration=yes

# Atur prefix yang diiklankan pada RA
/ipv6/nd/prefix/add prefix=2001:db8:abcd:10::/64 interface=ether2 autonomous=yes on-link=yes valid-lifetime=1d preferred-lifetime=1h

# Periksa NDP
/ipv6/nd/print
/ipv6/nd/prefix/print
/ipv6/neighbor/print
```

### Implementasi - DHCPv6 Stateless untuk DNS

```routeros
# Pool untuk informasi DNS saja, bukan untuk alamat
/ipv6/pool/add name=pool-dns prefix=2001:db8:abcd:10::/64 prefix-length=64

# DHCPv6 server stateless - hanya mengirim DNS dan domain
/ipv6/dhcp-server/add name=dhcpv6-stateless interface=ether2 address-pool=pool-dns lease-time=1h rapid-commit=yes use-radius=no

# DHCPv6 option untuk DNS
/ipv6/dhcp-server/option/add name=dns-opt code=23 value="'2001:db8:abcd:10::1'"
/ipv6/dhcp-server/option/add name=domain-opt code=24 value="'surabaya.lan'"

# Pada NDP, aktifkan Other-Configuration agar klien mengetahui bahwa DHCPv6 harus ditanyakan
/ipv6/nd/set [find default=yes] other-configuration=yes
```

### Implementasi - DHCPv6 Stateful untuk Branch (R2)

```routeros
# Pool alamat untuk Branch - dari /64 Branch
/ipv6/pool/add name=pool-branch prefix=2001:db8:abcd:30::/64 prefix-length=64

# DHCPv6 server stateful
/ipv6/dhcp-server/add name=dhcpv6-branch interface=ether2 address-pool=pool-branch lease-time=2h

# Pada NDP Branch, aktifkan Managed flag agar klien menggunakan DHCPv6 untuk alamat
/ipv6/nd/set [find interface=ether2] managed-address-configuration=yes other-configuration=yes

# Binding statis untuk server penting agar alamatnya tetap
/ipv6/dhcp-server/binding/add address=2001:db8:abcd:30::10 duid="00010001..." server=dhcpv6-branch comment="Server-Branch"
```

### Implementasi - Routing Dasar

```routeros
# R1: Route ke Branch melalui R2, next-hop menggunakan link-local R2
/ipv6/route/add dst-address=2001:db8:abcd:30::/64 gateway=fe80::2%ether1 comment="to-Branch-via-LL"

# R2: Default route ke R1 melalui link-local R1
/ipv6/route/add dst-address=::/0 gateway=fe80::1%ether1 comment="default-to-R1"

# Alternatif: menggunakan global address sebagai gateway
/ipv6/route/add dst-address=2001:db8:abcd:30::/64 gateway=2001:db8:abcd:1::2

# Periksa routing table
/ipv6/route/print
/ipv6/route/print where dst-address=2001:db8:abcd:30::/64
```

### Operasi dan Verifikasi

```routeros
# Pada R1/R2: lihat neighbor yang telah terdeteksi
/ipv6/neighbor/print

# Lihat klien DHCPv6 yang memperoleh lease
/ipv6/dhcp-server/lease/print
/ipv6/dhcp-server/binding/print

# Dari klien: periksa RA yang diterima
# Linux: rdisc6 ether2 atau tcpdump -i eth0 icmp6
# Windows: netsh interface ipv6 show neighbors

# Ping antar LAN
ping 2001:db8:abcd:30::1
ping 2001:db8:abcd:10::1

# Traceroute untuk melihat jalur
/tool/traceroute address=2001:db8:abcd:30::10
```

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **Klien memperoleh alamat tetapi tidak memperoleh DNS:** Periksa `other-configuration=yes` dan pastikan DHCPv6 option DNS telah dikonfigurasi secara benar.
- **Klien tidak memperoleh alamat sama sekali pada mode Stateful:** Periksa `managed-address-configuration=yes` pada NDP. Apabila flag tersebut bernilai no, klien akan menggunakan SLAAC saja.
- **Routing tidak berfungsi:** Periksa gateway yang menggunakan `fe80::...%interface`. Link-local wajib disertai nama interface, karena `fe80::` dapat terdapat pada banyak interface.
- **Duplicate Address:** Periksa log DAD. Apabila terdapat duplikasi, ganti alamat statis yang mengalami konflik.

> **Luaran Modul 2:** Satu LAN beroperasi dengan SLAAC, satu LAN beroperasi dengan DHCPv6 Stateful, serta routing R1 ke R2 dalam dua arah dapat melakukan ping.


## Modul 3 - IPv6 Packet

**Tujuan:** Mampu membaca header IPv6, memahami cara kerja extension header, fragmentasi, dan Path MTU Discovery. Modul ini meningkatkan kemampuan troubleshooting.

### Konsep Kunci

- **Header IPv6 sepanjang 40 byte dan bersifat fixed:** Version, Traffic Class, Flow Label, Payload Length, Next Header, Hop Limit, Source, dan Destination. Tidak terdapat checksum header sebagaimana pada IPv4, dan tidak terdapat field fragmentasi pada header utama.
- **Next Header dan Daisy Chaining:** Next Header menunjuk ke header berikutnya. Header dapat berantai sebagai berikut: IPv6 - Hop-by-Hop - Routing - Fragment - Destination - TCP/UDP. Rangkaian ini disebut extension header.
- **Fragmentasi hanya dilakukan oleh pengirim:** Router tidak memecah paket. Apabila paket terlalu besar, router akan membuang paket tersebut dan mengirim ICMPv6 Packet Too Big. Pengirim yang harus memecah atau memperkecil ukuran paket.
- **Path MTU Discovery (PMTUD):** Pengirim mengirim paket berukuran besar, menerima ICMPv6 Packet Too Big, kemudian menurunkan ukuran paket hingga dapat diteruskan. Apabila ICMPv6 Packet Too Big diblokir oleh firewall, PMTUD akan gagal dan situs web akan terasa memuat sebagian atau mengalami timeout.

### Perbandingan RouterOS v6 dan v7 — Modul 3

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| MTU interface | `/interface set mtu=...` | `/interface/set mtu=...` | Fungsi sama. Pada v7, perubahan MTU pada tunnel langsung memengaruhi PMTUD. |
| IPv6 Settings | `/ipv6 settings set forward=yes` | `/ipv6/settings/set forward=yes accept-router-advertisements=yes` | Pada v7, parameter `accept-router-advertisements` lebih eksplisit. |
| Ping dengan do-not-fragment | `/tool ping address=... size=...` (opsi terbatas) | `/tool/ping address=... size=... do-not-fragment=yes` | Pada v7, opsi `do-not-fragment` didukung penuh untuk pengujian PMTUD. |
| Traceroute | `/tool traceroute` | `/tool/traceroute` | Pada v7, keluaran lebih detail untuk hop IPv6. |
| Sniffer / Torch | `/tool sniffer`, `/tool torch` | `/tool/sniffer`, `/tool/torch` | Pada v7, filter `ip-protocol=ipv6` lebih akurat. |
| Path MTU Discovery | Bergantung pada ICMPv6 Packet Too Big | Bergantung pada ICMPv6 Packet Too Big, penanganan lebih baik | Pada kedua versi, ICMPv6 type 2 tidak boleh diblokir. |

#### Konfigurasi RouterOS v6 — Modul 3

```routeros
# v6: MTU dan pengujian
/interface set [find name=gre6-tunnel] mtu=1400
/ipv6 settings print

# v6: ping bertahap (tanpa do-not-fragment eksplisit pada beberapa build)
/tool ping address=2001:db8:abcd:30::1 size=1400 count=5
/tool ping address=2001:db8:abcd:30::1 size=1280 count=5
/tool traceroute address=2001:db8:abcd:30::10
/tool torch interface=ether1
```

#### Konfigurasi RouterOS v7 — Modul 3

```routeros
# v7: MTU dan pengujian PMTUD
/interface/gre6/set [find name=gre6-tunnel] mtu=1400
/interface/eoipv6/set [find name=eoip6-tunnel] mtu=1400
/ipv6/settings/print

# v7: ping dengan do-not-fragment untuk PMTUD
/tool/ping address=2001:db8:abcd:30::1 size=1500 do-not-fragment=yes count=5
/tool/ping address=2001:db8:abcd:30::1 size=1400 do-not-fragment=yes count=5
/tool/ping address=2001:db8:abcd:30::1 size=1280 do-not-fragment=yes count=5
/tool/traceroute address=2001:db8:abcd:30::10

# v7: sniffer dan torch dengan filter IPv6
/tool/sniffer/set filter-interface=ether1 filter-ip-protocol=ipv6
/tool/torch interface=ether1 ip-protocol=ipv6
```

> **Rekomendasi:** Gunakan v7 untuk pengujian PMTUD karena opsi `do-not-fragment` tersedia secara eksplisit. Pada v6, dokumentasikan MTU tunnel secara manual, karena pengujian tidak seakurat v7.

### Studi Kasus 3 - Situs Web Lambat, Ping Lancar

Pelanggan melaporkan bahwa ping ke server berjalan lancar, tetapi pembukaan situs web berukuran besar sering gagal atau hanya memuat sebagian. Pada IPv4, kondisi tersebut tidak terjadi. Pada IPv6, terdapat tunnel dengan MTU 1400, tetapi firewall memblokir ICMPv6. Akibatnya, pengirim tidak pernah mengetahui bahwa paket harus diperkecil. PMTUD gagal. Solusinya bukan menaikkan MTU, melainkan mengizinkan ICMPv6 Packet Too Big dan mengatur MTU tunnel secara benar.

### Implementasi - Pemeriksaan dan Pengaturan MTU

```routeros
# Tampilkan MTU interface
/interface/print where name~"ether"

# Atur MTU untuk tunnel (contoh GRE6 / EoIPv6) agar lebih kecil dari 1500
/interface/gre6/set [find name=gre6-tunnel] mtu=1400
/interface/eoipv6/set [find name=eoip6-tunnel] mtu=1400

# Pada RouterOS, periksa IPv6 settings
/ipv6/settings/print

# Uji MTU dari klien / router
# Ping dengan ukuran bertahap, tanpa fragmentasi
/tool/ping address=2001:db8:abcd:30::1 size=1500 do-not-fragment=yes count=5
/tool/ping address=2001:db8:abcd:30::1 size=1400 do-not-fragment=yes count=5
/tool/ping address=2001:db8:abcd:30::1 size=1280 do-not-fragment=yes count=5
```

### Implementasi - Melihat Header dan Jalur

```routeros
# Traceroute IPv6 untuk melihat hop dan MTU per hop
/tool/traceroute address=2001:db8:abcd:30::10

# Sniffer untuk melihat header IPv6 (gunakan filter)
/tool/sniffer/set filter-interface=ether1 filter-ip-protocol=ipv6
/tool/sniffer/start
/tool/sniffer/stop
/tool/sniffer/packet/print detail

# Torch untuk melihat traffic IPv6 secara real-time
/tool/torch interface=ether1 ip-protocol=ipv6
```

### Operasi dan Verifikasi

- Ping berukuran 1280 byte harus berhasil. Nilai 1280 merupakan MTU minimum IPv6. Apabila ping 1280 byte saja gagal, terdapat permasalahan pada link atau firewall.
- Ping berukuran 1500 byte dengan do-not-fragment gagal, tetapi ping 1400 byte berhasil: hal tersebut menunjukkan terdapat link dengan MTU 1400 di tengah jalur. Kondisi tersebut wajar untuk tunnel. Catat nilai MTU pada dokumentasi.
- Periksa firewall: pastikan ICMPv6 type 2 (Packet Too Big) tidak di-drop. Lihat Modul 4 untuk aturan firewall lengkap.

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **Situs web hanya memuat sebagian:** Periksa MTU tunnel dan ICMPv6 Packet Too Big.
- **Ping berukuran besar gagal, tetapi ping berukuran kecil berhasil:** Gejala tersebut menunjukkan permasalahan MTU, bukan permasalahan routing.
- **Fragmentasi muncul secara terus-menerus:** Periksa aplikasi yang mengirim paket berukuran besar. Sebaiknya aplikasi menggunakan ukuran yang sesuai dengan MTU, bukan mengandalkan fragmentasi.

> **Luaran Modul 3:** Laporan MTU setiap link (WAN, tunnel, LAN) dan hasil ping 1280, 1400, dan 1500 byte dengan do-not-fragment.


## Modul 4 - IPv6 Security

**Tujuan:** Mampu mengamankan NDP, ICMPv6, dan traffic IPv6 dengan firewall yang benar, serta memahami IPsec untuk IPv6.

### Konsep Kunci

- **ICMPv6 bersifat wajib, bukan opsional.** Pada IPv4, pemblokiran seluruh ICMP sering dilakukan. Pada IPv6, tindakan tersebut berakibat fatal, karena NDP, PMTUD, dan error reporting seluruhnya menggunakan ICMPv6.
- **NDP rentan terhadap spoofing:** Setiap pihak dapat mengirim RA palsu dan mengaku sebagai gateway. Serangan tersebut disebut Rogue RA. Mitigasinya adalah dengan memfilter RA pada port access, menggunakan RA Guard pada switch apabila tersedia, dan mengizinkan firewall RA hanya dari router yang resmi.
- **Detail Neighbor Discovery:** RS (klien mencari router), RA (router mengiklankan prefix), NS (mencari tetangga / DAD), NA (menjawab NS), Redirect (router memberikan informasi jalur yang lebih baik), dan MLD (manajemen multicast).
- **Temporary Address / Privacy Extension:** Klien mengganti alamat host-part secara berkala agar tidak mudah dilacak dari MAC address. Server sebaiknya menggunakan alamat statis, sedangkan klien dapat menggunakan temporary address.
- **IPsec:** AH (Authentication Header) untuk autentikasi dan integritas header, ESP (Encapsulating Security Payload) untuk enkripsi payload, dan AH ditambah ESP untuk keduanya. Pada IPv6, IPsec merupakan bagian dari desain awal, tetapi pada RouterOS konfigurasinya dilakukan melalui menu IPsec yang sama sebagaimana pada IPv4.

### Perbandingan RouterOS v6 dan v7 — Modul 4

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| Firewall IPv6 | `/ipv6 firewall filter` | `/ipv6/firewall/filter` | Sintaks filter sama, penulisan berbeda. Pada v7, `print stats` lebih detail. |
| ICMPv6 filter | `protocol=icmpv6 icmp-options=...` | `protocol=icmpv6 icmp-options=...` | Pada kedua versi, ICMPv6 wajib tidak boleh diblokir seluruhnya. |
| Connection-state | `connection-state=established,related` | `connection-state=established,related` | Fungsi sama. |
| Logging | `log=yes log-prefix=...` | `log=yes log-prefix=...` | Pada v7, log drop lebih mudah difilter pada `/log/print`. |
| IPsec Profile / Proposal | `/ip ipsec profile`, `/ip ipsec proposal` | `/ip/ipsec/profile`, `/ip/ipsec/proposal` | Pada v7, `hash-algorithm`, `enc-algorithm`, dan `dh-group` wajib konsisten. |
| IPsec Peer / Identity | `/ip ipsec peer`, `/ip ipsec identity` | `/ip/ipsec/peer`, `/ip/ipsec/identity` | Pada v7, IKEv2 lebih stabil untuk IPv6. Pada v6, IKEv1 masih sering digunakan. |
| IPsec Policy | `/ip ipsec policy` dengan `sa-src-address`, `sa-dst-address` | `/ip/ipsec/policy` dengan `src-address`, `dst-address`, `tunnel=yes` | Pada v7, penulisan policy lebih sederhana dan terintegrasi dengan peer. |
| RA Guard | Tidak tersedia secara native, mengandalkan firewall | Tidak tersedia secara native, mengandalkan firewall + switch | Pada kedua versi, mitigasi Rogue RA tetap mengandalkan firewall dan switch. |

#### Konfigurasi RouterOS v6 — Modul 4

```routeros
# v6: Firewall IPv6 - penulisan spasi
/ipv6 firewall filter add chain=input connection-state=established,related action=accept comment="Allow Established"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=2:0-255 action=accept comment="Allow Packet Too Big"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=133:0-255 action=accept comment="Allow RS"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=134:0-255 action=accept comment="Allow RA"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=135:0-255 action=accept comment="Allow NS"
/ipv6 firewall filter add chain=input protocol=icmpv6 icmp-options=136:0-255 action=accept comment="Allow NA"
/ipv6 firewall filter add chain=forward in-interface=ether2 protocol=icmpv6 icmp-options=134:0-255 src-address=!fe80::1 action=drop comment="Drop Rogue RA" log=yes
/ipv6 firewall filter add chain=input action=drop comment="Drop Other Input"

# v6: IPsec - penulisan spasi
/ip ipsec profile add name=ipv6-profile hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048
/ip ipsec proposal add name=ipv6-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=1h
/ip ipsec peer add name=peer-R2 address=2001:db8:abcd:1::2/128 exchange-mode=ike2 profile=ipv6-profile
/ip ipsec identity add peer=peer-R2 auth-method=pre-shared-key secret="GantiDenganPasswordKuat123!"
/ip ipsec policy add src-address=2001:db8:abcd:10::/64 dst-address=2001:db8:abcd:30::/64 peer=peer-R2 tunnel=yes proposal=ipv6-proposal
```

#### Konfigurasi RouterOS v7 — Modul 4

```routeros
# v7: Firewall IPv6 - penulisan slash
/ipv6/firewall/filter/add chain=input connection-state=established,related action=accept comment="Allow Established"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=2:0-255 action=accept comment="Allow Packet Too Big - PMTUD"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=128:0-255 action=accept comment="Allow Ping Request"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=129:0-255 action=accept comment="Allow Ping Reply"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=133:0-255 action=accept comment="Allow RS"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=134:0-255 action=accept comment="Allow RA"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=135:0-255 action=accept comment="Allow NS"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=136:0-255 action=accept comment="Allow NA"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=137:0-255 action=accept comment="Allow Redirect"
/ipv6/firewall/filter/add chain=forward in-interface=ether2 protocol=icmpv6 icmp-options=134:0-255 src-address=!fe80::1 action=drop comment="Drop Rogue RA" log=yes
/ipv6/firewall/filter/add chain=input action=drop comment="Drop Other Input" log=yes log-prefix="IPV6-INPUT-DROP"

# v7: IPsec - penulisan slash, IKEv2 direkomendasikan
/ip/ipsec/profile/add name=ipv6-profile hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048 lifetime=8h
/ip/ipsec/proposal/add name=ipv6-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=1h pfs-group=modp2048
/ip/ipsec/peer/add name=peer-R2 address=2001:db8:abcd:1::2/128 exchange-mode=ike2 profile=ipv6-profile
/ip/ipsec/identity/add peer=peer-R2 auth-method=pre-shared-key secret="GantiDenganPasswordKuat123!"
/ip/ipsec/policy/add src-address=2001:db8:abcd:10::/64 dst-address=2001:db8:abcd:30::/64 peer=peer-R2 tunnel=yes proposal=ipv6-proposal

# Verifikasi v7
/ipv6/firewall/filter/print stats
/ip/ipsec/active-peers/print
/ip/ipsec/installed-sa/print
```

> **Rekomendasi:** Gunakan v7 untuk IPsec IPv6 karena IKEv2 lebih stabil. Pada v6, uji kembali SA setelah reboot, karena policy dapat memerlukan penyesuaian `sa-src-address` dan `sa-dst-address`.

### Studi Kasus 4 - Rogue RA di Kantor

Di sebuah kantor, terdapat pengguna yang memasang router WiFi rumahan pada LAN. Router tersebut mengirim RA dengan prefix palsu. Seluruh PC pada LAN tersebut tiba-tiba memperoleh gateway palsu dan akses internet terputus, atau traffic dialihkan melalui router tersebut. Tim IT mengalami kebingungan karena ping ke gateway asli masih berhasil, tetapi penjelajahan web tidak berfungsi. Kondisi tersebut merupakan gejala klasik Rogue RA.

**Solusi:** Pada R1, filter RA yang masuk dari interface access. Hanya RA dari R1 dan R2 yang diizinkan. Pada switch access, aktifkan RA Guard apabila mendukung. Pada firewall, drop RA dari interface yang bukan merupakan uplink router resmi.

### Implementasi - Firewall IPv6 yang Benar

Urutan firewall sangat penting: terima established/related terlebih dahulu, kemudian terima ICMPv6 yang wajib, kemudian terima service yang memang dibuka, dan terakhir drop sisanya. Jangan menempatkan drop all di bagian atas.

```routeros
# === INPUT CHAIN - untuk traffic menuju router itu sendiri ===
# 1. Terima established, related
/ipv6/firewall/filter/add chain=input connection-state=established,related action=accept comment="Allow Established"

# 2. Terima ICMPv6 yang wajib - JANGAN memblokir seluruh ICMPv6
# Echo Request (128), Echo Reply (129), Packet Too Big (2), Parameter Problem (4)
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=128:0-255 action=accept comment="Allow Ping Request"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=129:0-255 action=accept comment="Allow Ping Reply"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=2:0-255 action=accept comment="Allow Packet Too Big - PMTUD"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=4:0-255 action=accept comment="Allow Parameter Problem"

# 3. NDP - RS (133), RA (134), NS (135), NA (136), Redirect (137)
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=133:0-255 action=accept comment="Allow RS"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=134:0-255 action=accept comment="Allow RA - batasi di forward"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=135:0-255 action=accept comment="Allow NS"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=136:0-255 action=accept comment="Allow NA"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=137:0-255 action=accept comment="Allow Redirect"

# 4. MLD - 130, 131, 132
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=130:0-255 action=accept comment="Allow MLD Query"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=131:0-255 action=accept comment="Allow MLD Report"
/ipv6/firewall/filter/add chain=input protocol=icmpv6 icmp-options=132:0-255 action=accept comment="Allow MLD Done"

# 5. Service yang dibuka ke router - contoh Winbox, SSH, DNS dari LAN saja
/ipv6/firewall/filter/add chain=input src-address=2001:db8:abcd:10::/64 protocol=tcp dst-port=8291 action=accept comment="Winbox dari LAN Core"
/ipv6/firewall/filter/add chain=input src-address=2001:db8:abcd:10::/64 protocol=tcp dst-port=22 action=accept comment="SSH dari LAN Core"
/ipv6/firewall/filter/add chain=input src-address=2001:db8:abcd:10::/64 protocol=udp dst-port=53 action=accept comment="DNS dari LAN"

# 6. Drop sisanya menuju router
/ipv6/firewall/filter/add chain=input action=drop comment="Drop Other Input" log=yes log-prefix="IPV6-INPUT-DROP"

# === FORWARD CHAIN - untuk traffic yang melewati router ===
/ipv6/firewall/filter/add chain=forward connection-state=established,related action=accept comment="Forward Established"
/ipv6/firewall/filter/add chain=forward protocol=icmpv6 icmp-options=2:0-255 action=accept comment="Forward Packet Too Big"

# Blokir RA palsu dari LAN access - hanya RA dari R1 yang diizinkan
/ipv6/firewall/filter/add chain=forward in-interface=ether2 protocol=icmpv6 icmp-options=134:0-255 src-address=!fe80::1 action=drop comment="Drop Rogue RA" log=yes

# Contoh: izinkan HTTP/HTTPS dari LAN ke internet
/ipv6/firewall/filter/add chain=forward src-address=2001:db8:abcd:10::/64 protocol=tcp dst-port=80,443 action=accept comment="Allow Web LAN Core"
/ipv6/firewall/filter/add chain=forward src-address=2001:db8:abcd:30::/64 protocol=tcp dst-port=80,443 action=accept comment="Allow Web Branch"

# Drop sisanya
/ipv6/firewall/filter/add chain=forward action=drop comment="Drop Other Forward" log=yes log-prefix="IPV6-FWD-DROP"
```

### Implementasi - IPsec IPv6 R1 ke R2 (Site to Site)

```routeros
# R1 dan R2: Buat Profile IPsec
/ip/ipsec/profile/add name=ipv6-profile hash-algorithm=sha256 enc-algorithm=aes-256 dh-group=modp2048 lifetime=8h

# Proposal
/ip/ipsec/proposal/add name=ipv6-proposal auth-algorithms=sha256 enc-algorithms=aes-256-cbc lifetime=1h pfs-group=modp2048

# Peer - gunakan alamat Global Loopback / WAN IPv6
# R1:
/ip/ipsec/peer/add name=peer-R2 address=2001:db8:abcd:1::2/128 exchange-mode=ike2 profile=ipv6-profile
# R2:
/ip/ipsec/peer/add name=peer-R1 address=2001:db8:abcd:1::1/128 exchange-mode=ike2 profile=ipv6-profile

# Identity - gunakan pre-shared-key untuk laboratorium, sertifikat untuk produksi
/ip/ipsec/identity/add peer=peer-R2 auth-method=pre-shared-key secret="GantiDenganPasswordKuat123!"

# Policy - enkripsi traffic LAN Core ke LAN Branch
# R1:
/ip/ipsec/policy/add src-address=2001:db8:abcd:10::/64 dst-address=2001:db8:abcd:30::/64 peer=peer-R2 tunnel=yes proposal=ipv6-proposal
# R2:
/ip/ipsec/policy/add src-address=2001:db8:abcd:30::/64 dst-address=2001:db8:abcd:10::/64 peer=peer-R1 tunnel=yes proposal=ipv6-proposal

# Periksa status
/ip/ipsec/active-peers/print
/ip/ipsec/installed-sa/print
```

### Operasi dan Verifikasi

```routeros
# Periksa firewall hit count - rule mana yang terkena
/ipv6/firewall/filter/print stats

# Periksa log drop Rogue RA
/log/print where message~"Rogue"

# Uji ping tetap berfungsi setelah firewall aktif
ping 2001:db8:abcd:30::1
ping 2001:db8:abcd:10::1

# Uji IPsec - ping harus berjalan melalui SA, periksa bytes terenkripsi yang meningkat
/ip/ipsec/installed-sa/print
```

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **Setelah pemasangan firewall, akses internet terputus sepenuhnya:** Periksa apakah ICMPv6 Packet Too Big dan NDP ter-drop. Kondisi tersebut merupakan penyebab yang paling sering terjadi.
- **IPsec tidak terbentuk:** Periksa peer address, pastikan pre-shared-key sama pada kedua sisi, dan pastikan firewall tidak memblokir UDP 500 dan 4500 untuk IPv6.
- **Rogue RA masih terjadi:** Periksa rule drop RA yang terdapat pada chain forward dan pada interface yang benar. Periksa pula apakah switch access telah mengaktifkan RA Guard.

> **Luaran Modul 4:** Firewall IPv6 aktif pada R1 dan R2 dengan log drop, serta IPsec R1-R2 dalam keadaan established dan traffic LAN terenkripsi.


## Modul 5 - Transition Mechanisms

**Tujuan:** Mampu membuat IPv4 dan IPv6 berjalan secara bersamaan, serta mampu menghubungkan pulau IPv6 melalui jaringan IPv4, atau sebaliknya.

### Konsep Kunci

- **Dual Stack:** Menjalankan IPv4 dan IPv6 secara bersamaan pada seluruh interface, routing, DNS, dan firewall. Metode ini merupakan cara yang paling aman dan direkomendasikan oleh RIPE. Seluruh perangkat modern mendukung metode ini.
- **6to4:** Tunnel otomatis dengan prefix `2002::/16`, alamat IPv6 dibentuk dari IPv4 publik. Metode ini cepat untuk laboratorium, tetapi tidak direkomendasikan untuk produksi karena tidak stabil dan sering dinonaktifkan.
- **6RD (IPv6 Rapid Deployment):** Versi 6to4 yang dikendalikan oleh ISP. ISP menentukan prefix dan relay sendiri, sehingga lebih stabil dan dapat dikelola.
- **Teredo:** Tunnel melalui NAT dengan menggunakan UDP. Metode ini digunakan untuk klien di belakang NAT yang tidak dapat menggunakan tunnel lain. Kinerjanya tidak optimal, tetapi berguna sebagai alternatif terakhir.
- **DS-Lite:** Pelanggan hanya memperoleh IPv6 dari ISP. Traffic IPv4 dibungkus di dalam IPv6 menuju AFTR, kemudian di-NAT di sisi tersebut. Metode ini sesuai untuk ISP yang kehabisan IPv4 publik.

### Perbandingan RouterOS v6 dan v7 — Modul 5

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| Dual Stack | Didukung, IPv4 dan IPv6 berjalan bersamaan | Didukung, IPv4 dan IPv6 berjalan bersamaan | Pada kedua versi, firewall harus dibuat untuk kedua protokol. |
| DNS Dual Stack | `/ip dns set servers=...` | `/ip/dns/set servers=...` | Pada v7, DNS cache dan `allow-remote-requests` lebih stabil. |
| 6to4 | `/interface 6to4` tersedia | `/interface/6to4` tersedia tetapi dianggap deprecated | Pada kedua versi, 6to4 tidak direkomendasikan untuk produksi. |
| 6RD | Dikonfigurasi manual melalui tunnel | Dikonfigurasi manual melalui tunnel | Tidak terdapat wizard khusus pada kedua versi. |
| GRE6 | `/interface gre6` | `/interface/gre6` | Pada v7, parameter `mtu` dan `local-address` lebih eksplisit. |
| IPIPv6 | `/interface ipipv6` | `/interface/ipipv6` | Fungsi sama. |
| Teredo | Tidak didukung native pada RouterOS, melalui klien | Tidak didukung native pada RouterOS, melalui klien | Teredo dipahami sebagai konsep, implementasi dilakukan pada endpoint. |
| DS-Lite | Konseptual, AFTR harus dibangun manual | Konseptual, AFTR lebih mudah diintegrasikan dengan tunnel v7 | Pada kedua versi, DS-Lite memerlukan AFTR khusus. |

#### Konfigurasi RouterOS v6 — Modul 5

```routeros
# v6: Dual Stack
/ip address add address=192.168.10.1/24 interface=ether2 comment="IPv4 LAN Core"
/ipv6 address add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="IPv6 LAN Core"
/ip dns set servers=8.8.8.8,2001:4860:4860::8888 allow-remote-requests=yes

# v6: 6to4 Lab
/interface 6to4 add name=6to4-tunnel local-address=203.0.113.1 remote-address=192.88.99.1 comment="6to4 Lab"
/ipv6 address add address=2002:cb00:7101::1/64 interface=6to4-tunnel advertise=no
/ipv6 route add dst-address=2002::/16 gateway=6to4-tunnel

# v6: GRE6
/interface gre6 add name=gre6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 mtu=1400
/ipv6 address add address=2001:db8:abcd:99::1/64 interface=gre6-R1R2 advertise=no
```

#### Konfigurasi RouterOS v7 — Modul 5

```routeros
# v7: Dual Stack
/ip/address/add address=192.168.10.1/24 interface=ether2 comment="IPv4 LAN Core"
/ipv6/address/add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="IPv6 LAN Core"
/ip/dns/set servers=8.8.8.8,2001:4860:4860::8888 allow-remote-requests=yes
/ip/dns/static/add name=server.lan address=192.168.10.10
/ip/dns/static/add name=server.lan address=2001:db8:abcd:10::10

# v7: 6to4 Lab (hanya untuk laboratorium)
/interface/6to4/add name=6to4-tunnel local-address=203.0.113.1 remote-address=192.88.99.1 comment="6to4 Lab"
/ipv6/address/add address=2002:cb00:7101::1/64 interface=6to4-tunnel advertise=no
/ipv6/route/add dst-address=2002::/16 gateway=6to4-tunnel routing-table=main

# v7: GRE6 dan IPIPv6
/interface/gre6/add name=gre6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 mtu=1400
/ipv6/address/add address=2001:db8:abcd:99::1/64 interface=gre6-R1R2 advertise=no
/interface/ipipv6/add name=ipipv6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2
```

> **Rekomendasi:** Gunakan Dual Stack sebagai strategi utama pada kedua versi. Gunakan tunnel manual (GRE6 / IPIPv6) untuk produksi, dan batasi 6to4 hanya untuk laboratorium.

### Studi Kasus 5 - ISP Kehabisan IPv4 Publik

Sebuah ISP di Jawa Timur tidak lagi memperoleh IPv4 publik baru. Pelanggan baru tetap memerlukan akses ke situs web IPv4 lama. ISP tidak dapat menonaktifkan IPv4 secara langsung. Solusi yang digunakan adalah Dual Stack untuk pelanggan lama yang masih memperoleh IPv4, dan DS-Lite untuk pelanggan baru. Pelanggan baru hanya memperoleh IPv6, tetapi tetap dapat membuka situs IPv4 melalui AFTR. Migrasi berjalan tanpa memutus layanan.

### Implementasi - Dual Stack (Wajib, Paling Penting)

```routeros
# Dual Stack berarti IPv4 tetap berjalan, IPv6 ditambahkan. Jangan menonaktifkan IPv4.
# Contoh pada R1 ether2 - IPv4 dan IPv6 berjalan bersamaan
/ip/address/add address=192.168.10.1/24 interface=ether2 comment="IPv4 LAN Core"
/ipv6/address/add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes comment="IPv6 LAN Core"

# DNS harus menjawab dua versi - A dan AAAA
/ip/dns/set servers=8.8.8.8,2001:4860:4860::8888 allow-remote-requests=yes
/ip/dns/static/add name=server.lan address=192.168.10.10
/ip/dns/static/add name=server.lan address=2001:db8:abcd:10::10

# Firewall: jangan hanya mengamankan IPv4. IPv6 juga harus memiliki firewall (lihat Modul 4)
# Periksa keduanya aktif
/ip/address/print
/ipv6/address/print
```

### Implementasi - Tunnel 6to4 (Laboratorium)

```routeros
# 6to4: Local IPv4 publik R1 = 203.0.113.1, Relay umum 192.88.99.1
# Prefix 6to4 untuk 203.0.113.1 adalah 2002:cb00:7101::/48
# (CB = 203, 00 = 0, 71 = 113, 01 = 1 dalam heksadesimal)

# Pada R1:
/interface/6to4/add name=6to4-tunnel local-address=203.0.113.1 remote-address=192.88.99.1 comment="6to4 Lab"

# Berikan alamat 6to4 pada tunnel
/ipv6/address/add address=2002:cb00:7101::1/64 interface=6to4-tunnel advertise=no

# Route IPv6 melalui 6to4 untuk prefix 2002::/16
/ipv6/route/add dst-address=2002::/16 gateway=6to4-tunnel

# Periksa
/interface/6to4/print
ping 2002:cb00:7101::1
ping 2001:db8::1
```

### Implementasi - Tunnel Manual GRE6 dan IPIPv6 (Lebih Stabil)

```routeros
# GRE6 - R1 ke R2 melalui IPv6
# R1:
/interface/gre6/add name=gre6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 mtu=1400
/ipv6/address/add address=2001:db8:abcd:99::1/64 interface=gre6-R1R2 advertise=no

# R2:
/interface/gre6/add name=gre6-R2R1 local-address=2001:db8:abcd:1::2 remote-address=2001:db8:abcd:1::1 mtu=1400
/ipv6/address/add address=2001:db8:abcd:99::2/64 interface=gre6-R2R1 advertise=no

# IPIPv6 - serupa, tetapi tanpa header GRE
# R1:
/interface/ipipv6/add name=ipipv6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2
# R2:
/interface/ipipv6/add name=ipipv6-R2R1 local-address=2001:db8:abcd:1::2 remote-address=2001:db8:abcd:1::1

# Periksa tunnel
/interface/gre6/print
/interface/ipipv6/print
ping 2001:db8:abcd:99::2
```

### Implementasi - DS-Lite Konsep (Untuk ISP)

DS-Lite memiliki dua sisi: B4 pada CPE pelanggan, dan AFTR pada ISP. Pada laboratorium, pahami alurnya terlebih dahulu sebelum implementasi penuh pada jaringan produksi, karena diperlukan AFTR yang khusus.

```text
Alur DS-Lite:
Pelanggan (B4) hanya memiliki IPv6: 2001:db8:abcd:40::2
AFTR ISP: 2001:db8:abcd::100 (IPv6) + NAT IPv4 di belakangnya
Traffic IPv4 pelanggan dibungkus dalam IPv6 menuju AFTR, pada AFTR dibuka dan di-NAT ke internet IPv4

Pada CPE (B4) - contoh konseptual tunnel IPv4 dalam IPv6:
# /interface/ipipv6/add name=ds-lite-b4 local-address=2001:db8:abcd:40::2 remote-address=2001:db8:abcd::100
Pada AFTR - terima tunnel, NAT IPv4 keluar
# /ip/firewall/nat/add chain=srcnat out-interface=ether1 action=masquerade

Verifikasi: klien di belakang CPE dapat melakukan ping ke 8.8.8.8 (IPv4) meskipun CPE tidak memiliki IPv4 publik
```

### Operasi dan Verifikasi

```routeros
# Dual Stack: periksa klien memperoleh IPv4 DAN IPv6
# Windows: ipconfig (harus terdapat IPv4 dan IPv6)
# Linux: ip addr (harus terdapat inet dan inet6)

# Uji akses situs dual-stack
ping 8.8.8.8
ping 2001:4860:4860::8888

# Periksa tunnel dalam keadaan up
/interface/print where running=yes
/interface/6to4/print
/interface/gre6/print
```

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **Dual Stack berjalan pada IPv4 tetapi IPv6 tidak berfungsi:** Periksa RA dan firewall IPv6, bukan firewall IPv4.
- **6to4 tidak stabil:** Kondisi tersebut wajar, karena 6to4 memang tidak ditujukan untuk produksi. Pindahkan ke tunnel manual GRE6 atau 6RD.
- **DS-Lite lambat:** Periksa MTU dan periksa AFTR yang mengalami overload. DS-Lite menambah overhead dan NAT terpusat.

> **Luaran Modul 5:** R1 dan R2 beroperasi dengan Dual Stack, serta satu tunnel (GRE6 atau 6to4) yang dapat melakukan ping IPv6 melalui tunnel.


## Modul 6 - Interoperability

**Tujuan:** Mampu membuat seluruh layanan berjalan pada IPv6, mulai dari DHCP Prefix Delegation, tunnel, DNS, PPP, hingga tools monitoring. Modul ini merupakan modul penutup yang menggabungkan seluruh modul sebelumnya.

### Konsep Kunci

- **DHCP Prefix Delegation (PD):** ISP memberikan prefix kepada CPE, bukan hanya satu alamat. CPE kemudian membagi prefix tersebut ke LAN yang berada di belakangnya. Cara ini merupakan mekanisme ISP dalam mendelegasikan /56 atau /48 kepada pelanggan.
- **IP Version Agnostic:** Layanan seperti DNS, NTP, dan PPP harus berjalan tanpa bergantung pada IPv4 atau IPv6. DNS harus dapat menjawab A dan AAAA, NTP harus dapat menggunakan server IPv6, dan PPP harus dapat membawa IPv6.
- **Tunnel:** IPIPv6, EoIPv6, dan GRE6 merupakan cara membungkus satu protokol di dalam protokol lain. EoIPv6 dapat membawa Ethernet (Layer 2) melalui IPv6, sehingga dapat melakukan bridging jarak jauh.
- **Tools:** Ping, Traceroute, Torch, Traffic Generator, Netwatch, Email, dan Traffic Flow seluruhnya memiliki versi IPv6. Gunakan tools tersebut untuk operasi harian.

### Perbandingan RouterOS v6 dan v7 — Modul 6

| Aspek | RouterOS v6 | RouterOS v7 | Catatan Operasi |
|---|---|---|---|
| DHCP PD Server | `/ipv6 dhcp-server`, pool PD terbatas | `/ipv6/dhcp-server`, pool PD lebih stabil | Pada v6, PD sering memerlukan restart service. Pada v7, PD lebih andal. |
| DHCP PD Client | `/ipv6 dhcp-client request=prefix` terbatas | `/ipv6/dhcp-client request=prefix pool-prefix-length=56` | Pada v7, parameter `pool-prefix-length`, `add-default-route`, dan `use-peer-dns` didukung penuh. |
| DHCPv6 Client biasa | `/ipv6 dhcp-client request=address` | `/ipv6/dhcp-client request=address` | Fungsi sama. |
| EoIPv6 | `/interface eoipv6` | `/interface/eoipv6` | Pada v7, `tunnel-id` dan MTU lebih eksplisit. |
| DNS | `/ip dns`, `/ip dns static` | `/ip/dns`, `/ip/dns/static` | Pada v7, record AAAA dan PTR `ip6.arpa` lebih konsisten. |
| NTP | `/system ntp client` | `/system/ntp/client` | Pada v7, perintah menjadi `/system/ntp/client/set enabled=yes servers=...` |
| PPP IPv6 | `/ppp profile use-ipv6=yes` | `/ppp/profile use-ipv6=yes` | Pada v7, pool IPv6 untuk PPP harus didefinisikan secara eksplisit pada `/ipv6/pool`. |
| Tools: Ping / Traceroute / Torch | `/tool ping`, `/tool traceroute`, `/tool torch` | `/tool/ping`, `/tool/traceroute`, `/tool/torch` | Fungsi sama, penulisan berbeda. |
| Tools: Netwatch / Traffic Flow | `/tool netwatch`, `/ip traffic-flow` | `/tool/netwatch`, `/ip/traffic-flow` | Pada v7, Netwatch mendukung `up-script` dan `down-script` dengan lebih baik. |
| Routing Global vs Link-Local | `/ipv6 route` tanpa routing-table | `/ipv6/route` dengan `routing-table=main` | Pada v7, selalu cantumkan `routing-table`. |

#### Konfigurasi RouterOS v6 — Modul 6

```routeros
# v6: DHCP PD Server
/ipv6 pool add name=pool-pd prefix=2001:db8:abcd::/48 prefix-length=56 comment="Pool PD"
/ipv6 dhcp-server add name=pd-server interface=ether1 address-pool=pool-pd lease-time=1d comment="DHCP PD Server"

# v6: DHCP PD Client (terbatas, periksa hasil)
/ipv6 dhcp-client add interface=ether1 pool-name=pool-client request=prefix pool-prefix-length=56 add-default-route=yes use-peer-dns=yes

# v6: EoIPv6
/interface eoipv6 add name=eoip6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 tunnel-id=100 mtu=1400

# v6: DNS dan NTP
/ip dns set servers=2001:4860:4860::8888,8.8.8.8 allow-remote-requests=yes
/system ntp client set enabled=yes primary-ntp=2001:db8::1

# v6: PPP
/ppp profile add name=ppp-ipv6 use-ipv6=yes local-address=2001:db8:abcd:10::1 remote-address=pool-ppp
/ipv6 pool add name=pool-ppp prefix=2001:db8:abcd:50::/64 prefix-length=64
```

#### Konfigurasi RouterOS v7 — Modul 6

```routeros
# v7: DHCP PD Server
/ipv6/pool/add name=pool-pd prefix=2001:db8:abcd::/48 prefix-length=56 comment="Pool PD untuk pelanggan"
/ipv6/dhcp-server/add name=pd-server interface=ether1 address-pool=pool-pd lease-time=1d comment="DHCP PD Server"
/ipv6/dhcp-server/binding/print

# v7: DHCP PD Client
/ipv6/dhcp-client/add interface=ether1 pool-name=pool-client request=prefix pool-prefix-length=56 add-default-route=yes use-peer-dns=yes comment="DHCP PD Client"
/ipv6/dhcp-client/print
/ipv6/address/add address=2001:db8:abcd:40::1/64 interface=ether2 advertise=yes comment="LAN dari PD"
/ipv6/nd/prefix/add prefix=2001:db8:abcd:40::/64 interface=ether2 autonomous=yes on-link=yes

# v7: EoIPv6 + Bridge
/interface/eoipv6/add name=eoip6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 tunnel-id=100 mtu=1400
/interface/bridge/add name=bridge-eoip
/interface/bridge/port/add bridge=bridge-eoip interface=eoip6-R1R2
/interface/bridge/port/add bridge=bridge-eoip interface=ether2

# v7: DNS, Reverse DNS, NTP
/ip/dns/set servers=2001:4860:4860::8888,2001:4860:4860::8844,8.8.8.8 allow-remote-requests=yes cache-size=2048KiB
/ip/dns/static/add name=server.surabaya.lan address=2001:db8:abcd:10::10 type=AAAA
/system/ntp/client/set enabled=yes servers=2001:db8::1,2001:4860:4860::8888
/system/clock/set time-zone-name=Asia/Jakarta

# v7: PPP IPv6
/ppp/profile/add name=ppp-ipv6 use-ipv6=yes use-encryption=yes local-address=2001:db8:abcd:10::1 remote-address=pool-ppp
/ipv6/pool/add name=pool-ppp prefix=2001:db8:abcd:50::/64 prefix-length=64
/interface/pppoe-server/server/add service-name=pppoe-ipv6 interface=ether2 default-profile=ppp-ipv6 authentication=pap,chap

# v7: Tools
/tool/ping address=2001:db8:abcd:30::1 count=5
/tool/traceroute address=2001:db8:abcd:30::1
/tool/torch interface=ether1 ip-protocol=ipv6
/tool/netwatch/add host=2001:db8:abcd:30::10 interval=30s up-script="/log info \\"Branch UP\\"" down-script="/log info \\"Branch DOWN - Periksa IPv6\\""
/ip/traffic-flow/set enabled=yes interfaces=ether1,ether2 cache-entries=4k active-flow-timeout=30m
/ip/traffic-flow/target/add address=2001:db8:abcd:10::100 port=2055 version=9
```

> **Rekomendasi:** Gunakan v7 untuk implementasi DHCP PD dan PPP IPv6 pada jaringan produksi, karena stabilitas dan dukungan parameter lebih lengkap. Pada v6, lakukan pengujian PD secara menyeluruh sebelum diterapkan kepada banyak pelanggan.

### Studi Kasus 6 - CPE Pelanggan WISP

Sebuah WISP akan memberikan IPv6 kepada 50 pelanggan rumahan. Apabila tim IT harus melakukan konfigurasi manual satu per satu, pekerjaan tersebut tidak akan selesai tepat waktu. Solusinya adalah DHCP PD. R1 sebagai PD Server memberikan /56 kepada setiap CPE. CPE (R3) sebagai PD Client menerima prefix, kemudian secara otomatis membagikannya ke LAN pelanggan. Pelanggan tidak perlu melakukan konfigurasi tambahan, cukup menghubungkan perangkat dan memperoleh IPv6. Tim IT hanya perlu memonitor pool PD, bukan melakukan konfigurasi per pelanggan.

### Implementasi - DHCP PD Server pada R1

```routeros
# Pool PD - dari /48 utama, delegasikan /56 kepada setiap pelanggan
/ipv6/pool/add name=pool-pd prefix=2001:db8:abcd::/48 prefix-length=56 comment="Pool PD untuk pelanggan"

# DHCPv6 Server untuk PD - pada interface menuju pelanggan (misal ether1 menuju R2/R3)
/ipv6/dhcp-server/add name=pd-server interface=ether1 address-pool=pool-pd lease-time=1d comment="DHCP PD Server"

# Periksa binding PD
/ipv6/dhcp-server/binding/print
/ipv6/pool/print
```

### Implementasi - DHCP PD Client pada R3 (CPE)

```routeros
# R3: Meminta prefix dari ISP / R1
/ipv6/dhcp-client/add interface=ether1 pool-name=pool-client request=prefix pool-prefix-length=56 add-default-route=yes use-peer-dns=yes comment="DHCP PD Client"

# Periksa prefix yang diperoleh
/ipv6/dhcp-client/print
/ipv6/address/print where interface=ether2

# Setelah PD diperoleh, R3 secara otomatis dapat membagikannya ke LAN
# Tambahkan alamat LAN dari prefix yang diperoleh - contoh manual apabila diperlukan:
/ipv6/address/add address=2001:db8:abcd:40::1/64 interface=ether2 advertise=yes comment="LAN dari PD"

# NDP pada LAN pelanggan
/ipv6/nd/prefix/add prefix=2001:db8:abcd:40::/64 interface=ether2 autonomous=yes on-link=yes
```

### Implementasi - DHCPv6 Client Biasa (Bukan PD)

```routeros
# Apabila hanya memerlukan satu alamat untuk router, bukan prefix
/ipv6/dhcp-client/add interface=ether1 pool-name=pool-wan request=address add-default-route=yes

# Periksa
/ipv6/dhcp-client/print
/ipv6/address/print where dynamic=yes
```

### Implementasi - Tunnel EoIPv6 dan IPIPv6 untuk Interoperabilitas

```routeros
# EoIPv6 - bridging Layer 2 melalui IPv6, sesuai untuk menyamakan LAN dua lokasi
# R1:
/interface/eoipv6/add name=eoip6-R1R2 local-address=2001:db8:abcd:1::1 remote-address=2001:db8:abcd:1::2 tunnel-id=100 mtu=1400
# R2:
/interface/eoipv6/add name=eoip6-R2R1 local-address=2001:db8:abcd:1::2 remote-address=2001:db8:abcd:1::1 tunnel-id=100 mtu=1400

# Masukkan EoIPv6 ke bridge agar LAN terhubung
/interface/bridge/add name=bridge-eoip
/interface/bridge/port/add bridge=bridge-eoip interface=eoip6-R1R2
/interface/bridge/port/add bridge=bridge-eoip interface=ether2

# Periksa
/interface/eoipv6/print
/interface/bridge/port/print
```

### Implementasi - DNS, Reverse DNS, NTP yang IP Version Agnostic

```routeros
# DNS Server - dapat menerima query IPv4 dan IPv6, menjawab A dan AAAA
/ip/dns/set servers=2001:4860:4860::8888,2001:4860:4860::8844,8.8.8.8 allow-remote-requests=yes cache-size=2048KiB

# DNS Statis untuk server internal - buat record AAAA
/ip/dns/static/add name=server.surabaya.lan address=2001:db8:abcd:10::10 type=AAAA
/ip/dns/static/add name=server.surabaya.lan address=192.168.10.10 type=A

# Reverse DNS - PTR untuk IPv6 (format ip6.arpa)
/ip/dns/static/add name="1.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.0.ip6.arpa" address=2001:db8:abcd:10::1

# NTP - gunakan server IPv6
/system/ntp/client/set enabled=yes servers=2001:db8::1,2001:4860:4860::8888
/system/clock/set time-zone-name=Asia/Jakarta

# Periksa
/ip/dns/print
/ip/dns/cache/print
/system/ntp/client/print
```

### Implementasi - PPP dengan IPv6

```routeros
# PPP Profile yang mendukung IPv6
/ppp/profile/add name=ppp-ipv6 use-ipv6=yes use-encryption=yes local-address=2001:db8:abcd:10::1 remote-address=pool-ppp

# Pool untuk PPP
/ipv6/pool/add name=pool-ppp prefix=2001:db8:abcd:50::/64 prefix-length=64

# PPPoE Server (contoh)
/interface/pppoe-server/server/add service-name=pppoe-ipv6 interface=ether2 default-profile=ppp-ipv6 authentication=pap,chap

# Secret pelanggan
/ppp/secret/add name=pelanggan01 password="PasswordKuat" profile=ppp-ipv6 service=pppoe

# Periksa
/ppp/active/print
/interface/pppoe-server/server/print
```

### Implementasi - Tools Operasi IPv6

```routeros
# 1. Ping dan Traceroute IPv6
/tool/ping address=2001:db8:abcd:30::1 count=5
/tool/traceroute address=2001:db8:abcd:30::1

# 2. Torch - melihat traffic IPv6 secara real-time
/tool/torch interface=ether1 ip-protocol=ipv6

# 3. Traffic Generator - menguji throughput IPv6
/tool/traffic-generator/packet-template/add name=ipv6-test header-stack=ipv6,udp
/tool/traffic-generator/stream/add name=stream1 packet-template=ipv6-test tx-template=ipv6-test

# 4. Netwatch - monitoring up/down host IPv6, mengirim notifikasi
/tool/netwatch/add host=2001:db8:abcd:30::10 interval=30s up-script="/log info \"Branch UP\"" down-script="/log info \"Branch DOWN - Periksa IPv6\""

# 5. Email - notifikasi melalui IPv6 (pastikan DNS dan routing IPv6 berfungsi)
/tool/e-mail/set address-from=netwatch@surabaya.lan server=2001:db8:abcd:10::25 port=25 start-tls=yes

# 6. Traffic Flow - mengirim flow IPv6 ke collector
/ip/traffic-flow/set enabled=yes interfaces=ether1,ether2 cache-entries=4k active-flow-timeout=30m
/ip/traffic-flow/target/add address=2001:db8:abcd:10::100 port=2055 version=9

# Periksa tools
/tool/netwatch/print
/ip/traffic-flow/print
```

### Implementasi - Routing dengan Global dan Link-Local

```routeros
# Routing menggunakan Global - serupa dengan IPv4, mudah dibaca
/ipv6/route/add dst-address=2001:db8:abcd:40::/64 gateway=2001:db8:abcd:1::2 comment="via Global"

# Routing menggunakan Link-Local - lebih umum pada IPv6, next-hop stabil meskipun global berubah
/ipv6/route/add dst-address=2001:db8:abcd:40::/64 gateway=fe80::2%ether1 comment="via Link-Local"

# Periksa rute yang aktif
/ipv6/route/print where dst-address=2001:db8:abcd:40::/64
```

### Operasi dan Verifikasi Modul 6

```routeros
# PD Server: pool berkurang saat pelanggan mengambil prefix
/ipv6/pool/used/print

# PD Client: prefix diperoleh dan LAN memperoleh alamat
/ipv6/dhcp-client/print
/ipv6/address/print

# DNS: query AAAA harus dijawab
:put [/ip/dns/resolve google.com]
:put [/ip/dns/resolve server.surabaya.lan]

# PPP: pelanggan aktif dan memperoleh IPv6
/ppp/active/print

# Tools: Netwatch harus berstatus UP, Traffic Flow harus mengirim data
/tool/netwatch/print
```

### Apabila Terjadi Kesalahan, Periksa Hal Berikut

- **PD tidak memperoleh prefix:** Periksa sisa pool PD yang masih tersedia, dan periksa interface DHCPv6 server telah benar (interface menuju pelanggan, bukan LAN).
- **DNS tidak menjawab AAAA:** Periksa server DNS yang digunakan telah mendukung IPv6, dan periksa firewall tidak memblokir UDP/TCP 53 untuk IPv6.
- **PPP IPv6 tidak memperoleh alamat:** Periksa profile `use-ipv6=yes` dan pastikan pool IPv6 telah dibuat.
- **Traffic Flow tidak sampai ke collector:** Periksa collector dapat dijangkau melalui IPv6, dan periksa firewall tidak memblokir port 2055.

> **Luaran Modul 6:** R3 sebagai CPE memperoleh prefix melalui PD, LAN pelanggan memperoleh IPv6 secara otomatis, DNS dan NTP berjalan pada IPv6, dan Netwatch monitoring aktif.


## Appendix A - Matriks Troubleshooting Cepat

| Gejala | Kemungkinan Penyebab | Pemeriksaan Pertama |
|---|---|---|
| Klien tidak memperoleh IPv6 | RA tidak berjalan, advertise=no | `/ipv6/nd/print`, `/ipv6/address/print` |
| Memperoleh IPv6 tetapi tidak dapat mengakses internet | Default route hilang, firewall memblokir | `/ipv6/route/print`, `/ipv6/firewall/filter/print` |
| Ping lancar, web gagal | MTU / PMTUD, ICMPv6 Packet Too Big diblokir | Ping ukuran bertahap, firewall ICMPv6 |
| Internet terputus tiba-tiba | Rogue RA | Log RA, firewall drop RA |
| DHCPv6 tidak memberikan alamat | Flag M / O pada NDP tidak sesuai | `/ipv6/nd/print`, managed/other flag |
| PD tidak memperoleh prefix | Pool habis, interface tidak sesuai | `/ipv6/pool/used/print` |
| IPsec tidak terbentuk | PSK berbeda, UDP 500/4500 diblokir | `/ip/ipsec/active-peers/print` |
| Tunnel down | Remote address berubah, MTU | `/interface/gre6/print`, `/interface/eoipv6/print` |


## Appendix B - Cheat Sheet

```routeros
# Alamat
/ipv6/address/print
/ipv6/address/add address=2001:db8:abcd:10::1/64 interface=ether2 advertise=yes

# NDP / Neighbor
/ipv6/nd/print
/ipv6/neighbor/print
/ipv6/nd/prefix/print

# DHCPv6
[STRIPPED 91 bytes]

# Routing
/ipv6/route/print
/ipv6/route/add dst-address=::/0 gateway=fe80::1%ether1

# Firewall
/ipv6/firewall/filter/print stats

# IPsec
/ip/ipsec/active-peers/print
/ip/ipsec/installed-sa/print

# Tools
/tool/ping address=2001:db8:abcd:10::1
/tool/traceroute address=2001:db8:abcd:10::1
/tool/torch interface=ether1
/tool/netwatch/print

# Backup
/export file=backup-ipv6
/system/backup/save name=backup-ipv6
```
