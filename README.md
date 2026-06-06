# Firmware Analiz Raporu
**Analiz edilen dosyalar:** `new-firmware.z1`, `udp-server.z1`, `udp-client.z1`  
**Platform:** Zolertia Z1 / MSP430F2617  
**İşletim Sistemi:** Contiki-NG v4.8

---

## 1. Binary Kimlik Analizi

### Hedef Platform
`msp430-objdump` çıktısı: `file format elf32-msp430`. `.z1` uzantısı Zolertia Z1 mote platformunu hedefler. Z1, Texas Instruments MSP430F2617 işlemcisi kullanır. Cooja simülatöründe MSPSim emülatörü ile tam donanım emülasyonu yapılır.

| Platform | Mimari | Hedef |
|---|---|---|
| `.z1` | MSP430X | Zolertia Z1 donanımı |
| `.sky` | MSP430 | TelosB/Sky mote |
| `cooja-native` | x86/x64 | Cooja simülatörü |
| `ARM M4F (CC1352R)` | ARM Cortex-M4F | TI üretim SoC |

### MSP430 Mimari Tipi
`Flags: 0x10000001` — `0x10000000` biti **MSP430X genişletilmiş mimari**yi gösterir. 20-bit adres uzayı destekler; `calla`, `.far.text` bölümünün `0x10000` adresinde bulunması (64KB sınırı ötesi) bunu doğrular.

### ELF Format Bilgisi
```
Class: ELF32 | Type: EXEC | OS/ABI: Standalone App | Sections: 21 | Program Headers: 6
```
Tam bağlanmış, yeniden konumlandırılamaz, bare-metal çalıştırılabilir dosya.

### Endianness
`Data: 2's complement, little endian`. MSP430 little-endian mimarisidir. `0x1234ABCD` değeri bellekte `CD AB 34 12` sırasıyla saklanır. OTA aktarımda bayt sırası değiştirilmeden yazılabilir.

### Entry Point
`0x3100` — `.text` bölümünün başlangıcı. C startup kodu (`crt0.S:118`) buradan çalışır: stack başlatma → `.data` kopyalama → `.bss` sıfırlama → `main()` çağrısı.

### ABI
`OS/ABI: Standalone App, Version: 0`. Sistem çağrısı, dinamik bağlama yok. **mspgcc ABI:** `r12–r15` argüman ve dönüş registerleri, `r4–r11` callee-saved, `r12–r15` caller-saved.

### Compiler İzi ve Toolchain
```
GCC: (GNU) 4.7.2 20120920 (mspgcc dev 20120911)
Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty
```
`-dirty` son eki, derleme sırasında kaynak depoda uncommitted değişiklikler bulunduğunu gösterir.

### Optimization Level Tahmini
`pushm.a #8, r11` (8 register tek komutta), kompakt `calla` zinciri: **`-Os -g`** kombinasyonu. Boyut odaklı optimizasyon + debug sembolü.

### Debug Symbol Durumu
8 DWARF debug bölümü mevcut: `.debug_info`, `.debug_line`, `.debug_frame`, `.debug_loc`, `.debug_str`, `.debug_aranges`, `.debug_abbrev`, `.debug_ranges`. GDB ile kaynak satırı düzeyinde hata ayıklama yapılabilir. Bu bölümler hedefe yüklenmez.

---

## 2. Bellek Kullanım Analizi

### Flash ve RAM Kavramları
- **Flash (text+data):** Kalıcı, güç kesilince silinmez. Kod ve sabit veriler burada saklanır.
- **RAM (data+bss):** Uçucu, güç kesilince kaybolur. Çalışma zamanı değişkenleri burada yaşar.
- **Stack:** RAM içinde fonksiyon çağrı zincirine ayrılan alan; derleyici tarafından otomatik yönetilir.
- **Heap:** Dinamik bellek (malloc/free). Kısıtlı gömülü sistemlerde genellikle kullanılmaz.

### Boyut Özeti (new-firmware.z1)

| Bölüm | Boyut | Yer | Açıklama |
|---|---|---|---|
| `.text` | 38.766 B | Flash | Ana kod |
| `.far.text` | 19.064 B | Flash (>64KB) | MSP430X uzak kod |
| `.rodata` | 13.821 B | Flash | Sabit veriler, string'ler |
| `.data` | 336 B | Flash→RAM | Boot'ta RAM'e kopyalanır |
| `.bss` | 5.704 B | RAM | Sıfır başlangıçlı değişkenler |
| `.noinit` | 2 B | RAM | Reset'ten sonra da korunan |
| `.vectors` | 64 B | Flash | Kesme vektör tablosu |
| **Flash toplamı** | **72.051 B** | | text+rodata+data+vectors+far.text |
| **RAM toplamı** | **6.042 B** | | data+bss |

### Program Segmentleri (msp430-readelf -l)

| Segment | VirtAddr | Boyut | İzin | İçerik |
|---|---|---|---|---|
| 0 | 0x300C | 39.010 B | R-X | .text (kod) |
| 1 | 0xC870 | 13.821 B | R-- | .rodata |
| 2 | 0x1100 | 6.040 B | RW- | .data + .bss |
| 3 | 0x2898 | 2 B | RW- | .noinit |
| 4 | 0xFFC0 | 64 B | R-X | .vectors |
| 5 | 0x10000 | 19.064 B | R-X | .far.text |

### Stack Kullanım Tahmini
`stack_check_process` ve `stack_top.2424` sembollerinin varlığı, firmware'in kendi stack izleme mekanizmasını içerdiğini gösterir. Stack, RAM'de BSS bölümünün üzerinde büyür. MSP430F2617'nin 10KB RAM'i göz önüne alındığında kullanılabilir stack alanı yaklaşık 3–4KB olarak tahmin edilebilir.

### Heap Analizi
`malloc`, `free` veya heap yönetim sembollerine rastlanmamıştır. Firmware **heap kullanmıyor**; tüm bellek statik olarak tahsis edilmiş. Bu Contiki-NG'nin kaynak kısıtlı cihazlar için benimsediği tipik yaklaşımdır.

### En Büyük Veri Yapıları (BSS)
```
0x1b08 → 0x1b18  _rpl_neighbors_mem     (RPL komşu tablosu)
0x1af8 → 0x1b08  bufmem_memb_mem        (paket tamponu)
0x15a8 → 0x1af8  buframmem_memb_mem     (RAM tamponu havuzu)
0x1350 → 0x13a0  _link_stats_mem        (bağlantı istatistikleri)
0x263a → 0x274a  uip_ds6_if             (IPv6 arayüz yapısı, 270B)
0x27be → 0x284a  uip_aligned_buf        (IPv6 paket tamponu, 140B)
```

---

## 3. Symbol / Fonksiyon Analizi

### Toplam Semboller
Kod bölümünde (`T/t`) **385 sembol** tespit edilmiştir.

### Contiki-NG Process Giriş Noktaları

| Process | Adres | Görev |
|---|---|---|
| `accmeter_process` | 0x1100 | İvmeölçer sensörü |
| `cc2420_process` | 0x110C | CC2420 radyo sürücüsü |
| `ctimer_process` | 0x1130 | Callback timer yönetimi |
| `etimer_process` | 0x113C | Event timer yönetimi |
| `hello_world_process` | 0x114A | Uygulama katmanı |
| `sensors_process` | 0x11D6 | Genel sensör yönetimi |
| `stack_check_process` | 0x11E2 | Stack taşma denetimi |
| `tcpip_process` | 0x11EE | TCP/IP yığını |

### ISR (Kesme Servisi) Fonksiyonları

| ISR No | Fonksiyon | Görev |
|---|---|---|
| 16 | `i2c_tx_interrupt` | I2C veri gönderimi |
| 17 | `i2c_rx_interrupt` | I2C veri alımı |
| 18 | `port1_isr` | GPIO Port 1 kesme |
| 19 | `irq_p2` | GPIO Port 2 kesme |
| 23 | `uart0_rx_interrupt` | UART alım kesmesi |
| 24 | `timera1` | Timer A1 |
| 25 | `timera0` | Timer A0 |
| 26 | `watchdog_interrupt` | Watchdog timer |
| 28 | `cc2420_timerb1_interrupt` | Radyo timer B |

Tanımsız (varsayılan) ISR'ler `0x3376` adresine yönlendirilmiştir (`__isr_0` – `__isr_30` weak semboller).

### Radyo Sürücüsü Fonksiyonları
`cc2420_arch_init`, `cc2420_on`, `cc2420_off`, `cc2420_set_channel`, `cc2420_set_pan_addr`, `cc2420_interrupt`, `cc2420_set_txpower`, `cc2420_get_txpower`, `cc2420_rssi`, `cc2420_init`

### Matematik / Yazılım Kütüphanesi
MSP430'un donanım çarpma/bölme desteği sınırlı olduğundan derleyici yazılım rutinleri eklemiştir:
`__mulsi3`, `__udivhi3`, `__divhi3`, `__udivsi3`, `__divsi3`, `__modsi3`, `__udivdi3`, `__umoddi3`

### Networking / RPL Fonksiyonları
`rpl_dag_root_start`, `rpl_process_dio`, `rpl_process_dao`, `rpl_icmp6_dio_output`, `rpl_icmp6_dao_output`, `rpl_neighbor_select_best`, `rpl_dag_update_state`, `rpl_timers_*` serisi — tam bir RPL Lite yönlendirme yığını mevcuttur.

---

## 4. String ve Metadata Analizi

### Firmware Kimliği
```
Starting Contiki-NG-release/v4.8-625-g8518cbaff-dirty
```

### Bağlı Sensörler
```
ADXL345 sensor   → İvmeölçer (SPI üzerinden)
TMP102 sensor    → Sıcaklık sensörü (I2C üzerinden)
```

### Ağ Protokol Katmanları
Tespit edilen string'ler:
- `RPL Lite` — RPL yönlendirme protokolü
- `6LoWPAN` — IPv6 sıkıştırma katmanı
- `IPv6 Nbr / Route / DS / NDP / SR` — IPv6 altyapısı
- `TSCH` bileşeni string olarak bulunamamış → firmware **CSMA MAC** kullanıyor

### Log Formatı
```
[%-4s: %-10s]  → Modül adı + log mesajı formatı
```
Bu format Contiki-NG'nin standart LOG_INFO/LOG_ERR makrolarından gelir.

### Debug Mesajları
```
Neighbor queue full
IPv6 cache full, dropping DIO
created a new RPL DAG / failed to create a new RPL DAG
SRH node not found, skip SRH insertion
rpl_icmp6_dao_output: no preferred parent, skip sending DAO
output: routing protocol extension header update error
```
Ağ sorunlarını debug etmeye yarayan detaylı log mesajları. Üretim firmware'lerinde genellikle kaldırılır.

### Hardcoded Konfigürasyon
`mac_pan_id` (0x1148) — ağ PAN kimliği statik olarak tanımlı.

---

## 5. Assembly / Instruction Analizi

### Fonksiyon Prologue/Epilogue
```asm
pushm.a  #8, r11     ; r4–r11 tek komutta stack'e kaydedildi
add      #-14, r1    ; 14 byte stack frame açıldı
...
popm.a   #8, r11     ; r4–r11 tek komutta geri yüklendi
reta                 ; 20-bit dönüş
```
`pushm.a` / `popm.a` MSP430X'e özgü çok-register işlemleridir; `-Os` optimizasyonunun açık göstergesidir.

### main() Başlangıç Zinciri
`main()` (0x313E) içinde art arda `calla` ile çağrılan başlatma fonksiyonları:
```asm
calla #0x06ae4  → clock_init()
calla #0x0453a  → rtimer_arch_init()
calla #0x0aada  → rtimer_init()
calla #0x06d7c  → process_init()
calla #0x06e8e  → process_start()  (etimer_process)
calla #0x05034  → ctimer_init()
calla #0x0512a  → etimer_set()
calla #0x06af6  → netstack_init()
```
Sistem başlatma sırası: saat → timer → process scheduler → ağ yığını.

### Dallanma ve Döngü Yapıları
`jnz`, `jz`, `jge`, `jl`, `jmp` komutları yoğun kullanılmaktadır. `bra` (branch absolute) komutu 20-bit dallanma için kullanılır.

### Register Kullanımı
`r15` dönüş değeri / son argüman, `r14` ikinci argüman, `r12-r13` ek argümanlar, `r4-r11` yerel değişkenler (callee-saved).

---

## 6. Source-Level Mapping Analizi

### Debug Build Durumu
8 DWARF bölümü mevcut (bkz. Bölüm 1.10). Kaynak seviyesi eşleme aktif.

### Entry Point Çözümlemesi
`msp430-addr2line -e new-firmware.z1 0x3100` çıktısı:
```
/home/user/tmp/gcc-4.7.2-msp430/.../crt0.S:118
```
Entry point `0x3100`, C startup dosyasının (`crt0.S`) 118. satırına karşılık gelir. Bu, `main()`'den önce çalışan başlatma kodudur.

### Inline Fonksiyon Tespiti
Debug sembollerinin varlığına karşın bazı küçük yardımcı fonksiyonlar inline edilmiş olabilir (özellikle `-Os` ile). `msp430-objdump -S` çıktısında kaynak satırları ile assembly birleşik görüntülenebilir.

---

## 7. ELF Yapısı Analizi

### ELF Header
21 section header, 6 program header. Section header string tablosu index 18.

### Section Haritası

| Nr | İsim | Tür | Adres | Boyut | Bayraklar |
|---|---|---|---|---|---|
| 1 | .far.text | PROGBITS | 0x10000 | 19.064 | AX |
| 2 | .text | PROGBITS | 0x3100 | 38.766 | AX |
| 3 | .rodata | PROGBITS | 0xC870 | 13.821 | A |
| 4 | .data | PROGBITS | 0x1100 | 336 | WA |
| 5 | .bss | NOBITS | 0x1250 | 5.704 | WA |
| 6 | .noinit | NOBITS | 0x2898 | 2 | WA |
| 7 | .vectors | PROGBITS | 0xFFC0 | 64 | AX |
| 8 | .comment | PROGBITS | — | 48 | — |
| 9–16 | .debug_* | PROGBITS | — | ~25KB | — |
| 19 | .symtab | SYMTAB | — | 18.288 | — |

### Relocation
`There are no relocations in this file.` — Tam bağlanmış EXEC dosyası; tüm sembol adresleri çözülmüş, runtime relocation yok.

### DWARF Debug Toplam Boyutu
`.debug_*` bölümleri toplamı: ~25KB. Bu veri dosya boyutunu şişirir ancak hedef flash'a yazılmaz.

---

## 8. Interrupt ve Donanım Analizi

### Kesme Vektör Tablosu
`.vectors` bölümü `0xFFC0`–`0xFFFF` arası 64 byte, 32 adet 2-byte'lık vektör içerir.

Ham içerik:
```
ffc0: 76337633... (varsayılan ISR)
ffe0: f4367837 3e35c235 76337633 ...
fff0: 24368c37 d8377633 fe357633 7633 0031
```
Son iki byte `0031` = `0x3100` (little-endian) → **reset vektörü entry point adresini gösteriyor**.

### Aktif Donanım Kesmesi Dağılımı
- **Port 1 (ISR 18):** GPIO buton/sensör kesmesi
- **Port 2 (ISR 19):** CC2420 radyo pin kesmesi
- **Timer A0/A1 (ISR 24/25):** Sistem zamanlayıcıları
- **UART0 RX (ISR 23):** Seri haberleşme alımı
- **I2C TX/RX (ISR 16/17):** TMP102 sensör iletişimi
- **Timer B1 (ISR 28):** CC2420 radyo zamanlama
- **Watchdog (ISR 26):** Sistem sıfırlama güvenliği

### MSP430 Register Erişimleri (Linker Sembollerinden)
`__P1IN`, `__P1OUT`, `__P1DIR`, `__P1IFG` — GPIO Port 1 doğrudan register eşlemesi. `__ADC12MCTL0–15` — 12 kanallı ADC kontrolü. `__UCB0I2CIE` — I2C kesme etkinleştirme.

---

## 9. Networking Analizi

### IPv6 Stack Bileşenleri
`uip_ds6_if` (IPv6 arayüzü), `uip_ds6_timer_periodic`, `uip_ds6_timer_ra`, `uip_aligned_buf` (paket tamponu), `uip_ext_len`, `uip_last_proto`, `uip_ext_bitmap`.

### RPL Yönlendirme
Tam `rpl-lite` sürümü içeriyor: DIS/DIO/DAO/DAO-ACK mesajlaşması, MRHOF (Minimum Rank with Hysteresis Objective Function), komşu tablosu yönetimi, DAG oluşturma/onarım mekanizmaları.

### 6LoWPAN
`6lowpan` log sembolü, `uncomp_hdr_len`, `packetbuf_hdr_len` — IPv6 başlık sıkıştırma/açma aktif. IEEE 802.15.4 üzerinden IPv6 çalıştırmak için zorunlu katman.

### MAC Katmanı
`mac_pan_id`, `mac_dsn`, `mac_max_payload`, `csma_output_packet` — **CSMA/CA** MAC protokolü kullanılıyor (TSCH değil).

### Komşu Tablosu
`neighbor_memb`, `neighbor_list_list`, `_rpl_neighbors_mem`, `nbr_table_keys_list` — dinamik komşu yönetimi.

---

## 10. Wireless / CSMA Analizi

### MAC Protokolü
Sembol analizine göre firmware **CSMA/CA** kullanmaktadır. TSCH'ye özgü `asn`, `slot_operation`, `channel_hopping` sembolleri bulunmamaktadır.

### Radyo Kanal Yönetimi
`cc2420_set_channel` (0x408A), `channel` değişkeni (0x1272) — IEEE 802.15.4 kanal seçimi (11–26 arası).

### Zamanlama
`rtimer_arch_schedule`, `msp430_sync_dco` — gerçek zamanlı zamanlayıcı ve DCO osilatör senkronizasyonu. `schedule_transmission` (0x45B8) — iletim planlama fonksiyonu.

### RPL Zamanlayıcıları
`rpl_timers_schedule_periodic_dis`, `rpl_timers_schedule_unicast_dio`, `rpl_timers_schedule_dao`, `rpl_timers_schedule_leaving`, `rpl_schedule_probing` — RPL protokolünün tüm zamanlayıcı altyapısı mevcut.

---

## 11. Sensor ve Peripheral Analizi

### Bağlı Sensörler

| Sensör | Protokol | Sembol | Görev |
|---|---|---|---|
| ADXL345 | SPI | `accmeter_process` | İvmeölçer |
| TMP102 | I2C | `i2c_tx/rx_interrupt` | Sıcaklık |
| Button | GPIO | `button_hal_buttons` | Kullanıcı girişi |

### Çevresel Birimler
- **UART0:** `uart0_rx_interrupt`, `uart0_input_handler` — seri debug/iletişim
- **SPI:** `spi_arch_has_lock` (undefined → platform HAL)
- **I2C (UCB0):** `__UCB0I2CIE` — I2C kesme kayıtçısı
- **ADC12:** `__ADC12MCTL0`–`__ADC12MCTL15` — 12 kanallı analog-dijital dönüştürücü
- **GPIO:** `gpio_hal_arch_*` fonksiyon ailesi (undefined → platform sürücüsü)

---

## 12. Algoritma / Matematiksel Analiz

### Kayan Nokta (Floating-Point)
Kayan nokta sembollerine (`__addsf3`, `__mulsf3` vb.) rastlanmamıştır. Firmware **tamsayı aritmetiği** kullanmaktadır.

### Yazılım Tamsayı Kütüphanesi
MSP430 donanım desteği olmadığından derleyici şu yazılım rutinlerini eklemiştir:

| Sembol | Boyut | İşlev |
|---|---|---|
| `__mulsi3` | — | 32-bit çarpma |
| `__udivsi3` / `__divsi3` | — | 32-bit bölme |
| `__modsi3` / `__umodsi3` | — | 32-bit mod |
| `__udivdi3` / `__umoddi3` | — | 64-bit bölme |
| `__udivmoddi4` | — | 64-bit bölme+mod |

### Yönlendirme Algoritması
`nbr_path_cost` — RPL MRHOF için komşu yol maliyeti hesaplama. `rpl_neighbor_select_best` — en iyi ebeveyn seçimi.

---

## 13. Güç ve Performans Analizi

### Kesme Yönetimi
`dint` (disable interrupts) ve `eint` (enable interrupts) komutları kod boyunca 20'den fazla kritik bölgede kullanılmaktadır. Bu atomik işlemler için gereklidir.

### Saat Sistemi
`clock_init`, `clock_time`, `clock_delay`, `clock_wait`, `clock_seconds` — tam saat yönetimi altyapısı. `msp430_sync_dco` — dijital kontrollü osilatör kalibrasyonu.

### Radyo Güç Yönetimi
`cc2420_set_txpower`, `cc2420_get_txpower` — dinamik iletim gücü kontrolü. `output_power` tablosu (0xC948) — olası güç seviyeleri. `cc2420_last_rssi` — alınan sinyal gücü ölçümü.

### Düşük Güç Mod Tespiti
`dint` kullanımı işlemcinin uyku moduna geçişinden önce kesmeler devre dışı bırakıldığına işaret eder. Contiki-NG event döngüsü boşta beklerken MSP430 LPM modlarını aktive eder.

---

## 15. Reverse Engineering Analizi

### Firmware Davranış Özeti
Sembol ve string analizinden firmware'in yaptıkları:
1. Contiki-NG event-driven process scheduler başlatılır
2. CC2420 radyo sürücüsü ve CSMA MAC başlatılır
3. RPL DAG oluşturulur, ağa katılım sağlanır
4. ADXL345 (ivme) ve TMP102 (sıcaklık) sensörleri periyodik okunur
5. Veriler IPv6/6LoWPAN üzerinden RPL ağı ile iletilir
6. `hello_world_process` uygulama katmanı mesaj gönderir

### Tespit Edilen Protokoller
- **MAC:** IEEE 802.15.4 CSMA/CA
- **Ağ:** IPv6 + 6LoWPAN
- **Yönlendirme:** RPL Lite (DIS/DIO/DAO)
- **Transport:** UDP (uip_udp_conn)

### Ağ Rolü
`rpl_dag_root_start` ve `NETSTACK_ROUTING.root_start()` fonksiyonlarının varlığı, bu firmware'in RPL ağının **root (kök) düğümü** olarak çalışabildiğini göstermektedir.

---

## 16. Compiler ve Optimization Analizi

### MSP430X Optimizasyon Komutları
```
pushm.a  → Çoklu register push (boyut kazancı)
popm.a   → Çoklu register pop
calla    → 20-bit fonksiyon çağrısı
reta     → 20-bit dönüş
mova     → 20-bit veri taşıma
bra      → 20-bit dal
```
Bu komutlar -Os bayrağı ile MSP430X hedefi için etkinleşir.

### -O0 ile Fark
`-O0`'da her register ayrı `push` ile kaydedilirdi (8× daha fazla komut). `-Os` ile `pushm.a #8, r11` tek komut olarak üretilir.

### Dead Code Elimination
385 fonksiyon tanımlı, `W` (weak) semboller varsayılan ISR olarak kullanılmış; bağlanmamış semboller binary'e dahil edilmemiştir.

---

## 17. Linker ve Build Sistemi Analizi

### Linker Sembol Dağılımı
```
__data_start = 0x1100   → RAM başlangıcı
__bss_start  = 0x1250   → BSS başlangıcı
__bss_end    = 0x2898   → BSS sonu = noinit başlangıcı
_stack       = 0x289A   → Stack başlangıcı
__far_data_start = 0x0  → Far veri yok
__vectors_start  = 0xFFC0
_vectors_end     = 0x10000
```

### Donanım Register Eşlemesi
Linker script, MSP430 IO register adreslerini sembolik isimlerle eşler:
```
__P1IN = 0x20, __P1OUT = 0x21, __P1DIR = 0x22
__ADC12MCTL0-15 = 0x80-0x8F
```

### Startup Kodu
`crt0.S` giriş noktası `0x3100`; `__ctors_start` = `__ctors_end` = `0x3376` (C++ constructor yok, salt C kodlaması).

---

## 18. Binary Transformation Analizi

### Nesne Özellikleri
```
Architecture: msp430:430X
Flags: EXEC_P (executable), HAS_SYMS (debug), D_PAGED
Start address: 0x3100
```

### ELF → Binary Dönüşümü
OTA aktarımda kullanılan `new-firmware.z1` ham binary değil ELF dosyasıdır. Ham binary üretmek için:
```bash
msp430-objcopy -O binary new-firmware.z1 new-firmware.bin
```
Ham binary'de yalnızca yüklenebilir bölümler (LOAD flag'li) yer alır; debug bilgisi çıkarılır.

### ELF'in Ham Binary'ye Üstünlüğü
ELF dosyası; section bilgisi, sembol tablosu, debug verisi ve yükleyici talimatlarını içerir. Ham binary yalnızca belleğe yazılacak ham byte dizisidir. ELF bootloader'ın hangi adreslere ne yazacağını bilmesini sağlar; bu OTA güncellemeler için kritiktir.

---

## 20. Contiki-NG Özel Analizler

### Aktif Process'ler

| Process | Adres | Thread Fonksiyonu |
|---|---|---|
| `accmeter_process` | 0x1100 | `process_thread_accmeter_process` (0x389C) |
| `cc2420_process` | 0x110C | `process_thread_cc2420_process` (0x4034) |
| `ctimer_process` | 0x1130 | `process_thread_ctimer_process` (0x4F94) |
| `etimer_process` | 0x113C | `process_thread_etimer_process` (0x51A8) |
| `hello_world_process` | 0x114A | `process_thread_hello_world_process` (0x5BA6) |
| `sensors_process` | 0x11D6 | `process_thread_sensors_process` (0xAB06) |
| `stack_check_process` | 0x11E2 | `process_thread_stack_check_process` (0xC074) |
| `tcpip_process` | 0x11EE | `process_thread_tcpip_process` (0xC5F4) |

### Event Timer Altyapısı
`etimer_set`, `etimer_reset`, `etimer_expired`, `etimer_stop`, `etimer_pending` — tam etimer API mevcut. `ctimer_set`, `ctimer_reset`, `ctimer_stop`, `ctimer_expired` — callback timer API.

### Protothread Mekanizması
Her `PROCESS_THREAD` bir protothread olarak çalışır. `process_run` (0x6D98) scheduler'ı, `process_post` (0x6E32) event gönderimini, `process_post_synch` (0x6E8E) senkron event iletimini yönetir.

### Packetbuf Yaşam Döngüsü
`packetbuf_aligned`, `packetbuf_ptr`, `packetbuf_payload_len`, `packetbuf_hdr_len`, `packetbuf_attrs`, `packetbuf_addrs` — tam packetbuf altyapısı. Paketler bu ortak tampon üzerinden katmanlar arasında geçer.

---

## 21. Güvenlik ve Robustness Analizi

### Stack Taşma Koruması
```
stack_check_process    → 0x11E2  (process)
stack_check_init       → 0xBFF2
stack_check_get_usage  → 0xC020
stack_top.2424         → 0x2264  (stack üst sınır işaretçisi)
_stack                 → 0x289A
```
Firmware kendi stack kullanımını izleyen yerleşik bir mekanizma içermektedir. Bu, MSP430'un sınırlı RAM'inde stack taşmasını erken tespite imkan tanır.

### TLV Checksum
`__TLV_CHECKSUM = 0x10C0` — MSP430 TLV (Tag-Length-Value) kalibrasyonu doğrulama. Bu, donanım kalibrasyonunun bütünlüğünü kontrol eder.

### Hardcoded Veri Analizi
`msp430-strings` ile `password`, `secret`, `key`, `token`, `auth`, `admin`, `backdoor` araması sonuç vermemiştir. **Hardcoded kimlik bilgisi tespit edilmemiştir.**

### Güvenlik Açığı Potansiyeli
`uip_process` (0x992 bayt) ve `input` fonksiyonu (0x118A byte) oldukça büyük fonksiyonlardır. Ağ paket işleme kodundaki sınır kontrolleri kritik önem taşır. Debug mesajlarının üretim firmware'inde kaldırılması önerilir.

---

## 22. Karşılaştırmalı Firmware Analizi

### Boyut Karşılaştırması

| Firmware | text (B) | data (B) | bss (B) | Flash | RAM | T Sembolleri |
|---|---|---|---|---|---|---|
| `new-firmware.z1` | 71.715 | 336 | 5.706 | 72.051 | 6.042 | 385 |
| `udp-server.z1` | 42.585 | 336 | 5.866 | 42.921 | 6.202 | 379 |
| `udp-client.z1` | 42.871 | 336 | 5.922 | 43.207 | 6.258 | 379 |

### Yorum
- `new-firmware.z1`, diğerlerine göre ~29KB daha büyük bir `.text` bölümüne sahiptir. Bu; tam sensör sürücüleri (ADXL345, TMP102), RPL yığını ve uygulama kodunu içerdiğini gösterir.
- `udp-server.z1` ve `udp-client.z1` boyut açısından neredeyse eşittir; aynı temel Contiki-NG altyapısını paylaşırlar.
- BSS bölümü `udp-client.z1`'de en büyüktür (+216B), bu OTA transfer için eklenen tampon değişkenleri yansıtır.
- Her üç firmware da 92KB flash ve 10KB RAM sınırı içinde kalmaktadır.

---

## Sonuç

`new-firmware.z1` analizi, Contiki-NG v4.8 tabanlı, CC2420 radyolu, CSMA/CA MAC, RPL Lite yönlendirmeli, ADXL345 ve TMP102 sensörlü, tam IPv6/6LoWPAN yığınına sahip bir Zolertia Z1 firmware'i olduğunu ortaya koymaktadır. MSP430X genişletilmiş mimarisi sayesinde 64KB sınırını aşan kod alanı kullanılmış, boyut odaklı `-Os` optimizasyonu ile 72KB flash içine sığdırılmıştır. Stack koruma mekanizması ve hardcoded kimlik bilgisi bulunmaması güvenlik açısından olumlu bulgulardır.
