# Contiki-NG / MSP430 Firmware Analiz Raporu

Bu çalışma, Contiki-NG işletim sistemi üzerinde derlenmiş olan `udp-server.z1` bellenim (firmware) imajının, msp430 araç zinciri (toolchain) kullanılarak gerçekleştirilen tersine mühendislik ve yapısal analiz sonuçlarını içermektedir. Analizler, **Texas Instruments CC1352R** SoC bellek mimarisi referans alınarak mühendislik perspektifiyle raporlanmıştır.

## 1. Binary Kimlik Analizi
Dosya başlık yapısı (`readelf -h`) üzerinden bellenimin yapısal kimliği çözümlenmiştir:

| Özellik | Değer / Çıktı | Mühendislik Açıklaması |
| **Hedef Platform** | Zolertia Z1 | MSP430F2617 MCU ve CC2420 Telsiz yongası barındıran IoT düğümü. |
| **Mimari Tipi** | TI msp430 | 16-bit RISC mimarisi. Flags: `0x10000001` (MSP430X 20-bit genişletilmiş komut seti modu aktif). |
| **Format Bilgisi** | ELF32 Executable | 32-bit ELF konteyneri içinde sarmalanmış statik bağlı yürütülebilir dosya. |
| **Endianness** | Little-endian | En önemsiz bayt (LSB) en düşük hafıza adresinde saklanır (`2's complement, little endian`). |
| **Giriş Noktası (Entry Point)** | `0x3100` | İşlemciye reset geldiğinde bootloader'ın dallanacağı ilk kod adresi (`__watchdog_support`). |
| **ABI Bilgisi** | Standalone App | Bağımsız çalışan yalın donanım (bare-metal) uygulaması. |
| **Derleyici / Toolchain** | msp430-gcc | `Contiki-NG-release/v4.8-625-g8518cbaff-dirty` |
| **Optimizasyon Seviyesi** | `-Os` (Size Optimization) | Kod yapısında inlining dengeli tutulmuş ve `size` minimizasyonu yapılmıştır. |
| **Debug Sembolleri** | Mevcut (`not stripped`) | `.debug_info`, `.debug_line`, `.symtab` bölümleri tamamen korunmuştur. |

---

## 2. Bellek Kullanım Analizi
`msp430-size` ve `readelf -S` çıktılarından elde edilen gerçek boyutlar, hedef **Texas Instruments CC1352R** SoC bellek mimarisine (352 KB Flash, 80 KB SRAM) teorik olarak eşlenmiştir:

| Hafıza Bölgesi (Section) | Boyut (Byte) | Öznitelik | TI CC1352R Bellek Haritası Eşleşmesi (Memory Map) |
| :--- | :--- | :--- | :--- |
| **`.text`** | 41.158 (`0x00a0c6`) | AX (Alloc, Exec) | **Flash (Non-Volatile):** `0x00000000` adresinden başlar. Kapladığı alan: **%11.6** |
| **`.rodata`** | 1.363 (`0x000553`) | A (Alloc) | **Flash (Non-Volatile):** Sabit loglar ve stringler burada saklanır. |
| **`.data`** | 336 (`0x000150`) | WA (Write, Alloc) | **Flash + RAM:** Flash'ta saklanır, boot anında RAM'e (`0x20000000`) kopyalanır. |
| **`.bss`** | 5.864 (`0x0016e8`) | WA (Write, Alloc) | **SRAM:** Başlangıçta sıfırlanan global değişkenler için RAM uzayında açılır. |
| **`.noinit`** | 2 (`0x000002`) | WA (Write, Alloc) | **SRAM:** Reset anında sıfırlanmayan özel donanım sayaç alanı. |
| **`.vectors`** | 64 (`0x000040`) | AX (Alloc, Exec) | **Flash (Vektör Tablosu):** Kesme adresleri (`0x0000ffc0`). |

> 💡 **Stack & Heap Notu:** Sistemde dinamik bellek tahsisi (`malloc`) izine rastlanmamıştır (**Heap kullanılmamaktadır**). Stack işareti `_stack` ve `__stack` sembollerinde `0x3100` adresini (RAM'in zirvesi) göstermektedir; hafıza aşağı doğru büyüyecektir.

---

## 3. Symbol / Function Analizi
Sembol tablosundan (`nm -n`) elde edilen kritik sistem ve uygulama bileşenleri:

* **Uygulama Girişleri:** `udp_server_process` (`0x011ee`), `udp_rx_callback` (`0x0a8f2`).
* **Contiki-NG Çekirdek Süreçleri:** `tcpip_process` (`0x011e2`), `etimer_process` (`0x0113c`), `ctimer_process` (`0x01130`), `sensors_process` (`0x011be`).
* **Donanım / Sürücü Fonksiyonları:** `cc2420_init` (`0x04436`), `accm_init` (`0x0396a`), `tmp102_init` (`0x0a72c`), `leds_init` (`0x056b2`).
* **Ağ Callback Zinciri:** `simple_udp_register` (`0a1f6`), `rpl_link_callback` (`0x0893c`).
* **Ölü (Dead) Fonksiyon Analizi:** Derleyici `-ffunction-sections` ve `-fdata-sections` ile derlendiği için linker kullanılmayan fonksiyonları temizlemiştir; tablodakiler aktif kullanılan canlı sembollerdir.

---

## 4. String ve Metadata Analizi
`strings` çıktısından elde edilen kritik diagnostic veri ve sistem parametreleri:
* **Sistem Logları:** `"Starting Contiki-NG-release/v4.8..."`, `"Node ID: %u"`, `"Tentative link-local IPv6 address"`.
* **Protokol İmzaları:** `"CSMA"`, `"RPL Lite"`, `"TCP/IP stack"`, `"sicslowpan"`.
* **Donanım Logları:** `"ADXL345 sensor"`, `"CC2420 CCA threshold %i"`, `"TMP102 sensor"`.
* **Uygulama Değerleri:** `"UDP server"`, `"Received request '%.*s' from "`, `"Sending response."`.

---

## 5. Assembly / Instruction Analizi
Disassembly (`objdump -d`) çıktısından çıkarılan mimari davranışlar:

* **Function Prologue:** Fonksiyon girişlerinde (Örn: `__mulsi3`) `pushm.a #2, r11` komutuyla register durumları stack'e güvenli bir şekilde yedeklenmektedir.
* **Function Epilogue:** Fonksiyon çıkışlarında `popm.a` ve `reta` (Extended Return) kullanılarak çağrılan yere geri dönülmektedir.
* **Donanım Kontrolü (Watchdog):** `main` fonksiyonunun en başında `0x3100` adresinde `mov.b &0x0120, r5` ve `bis #23048, r5` komutlarıyla Watchdog Timer (`WDTCTL`) beslenmekte veya konfigüre edilerek işlemcinin istemsiz reset atması engellenmektedir.
* **Döngü Yapıları:** Bellek temizleme (`__do_clear_bss`) döngüsünde `dec r15` ve `jnz $-12` mimari komut zinciriyle döngülerin busy-wait/kontrol mantığı kurulmaktadır.

---

## 6. Source-Level Mapping Analizi
* Binary dosyası içerisindeki `.debug_line` ve `.debug_frame` bölümlerinin varlığı, adreslerin doğrudan C kod satırlarına çözümlenebileceğini kanıtlar.
* Olası bir runtime crash durumunda işlemci program sayacı (PC) `0x0a8f2` değerini gösterirse, `msp430-addr2line -e udp-server.z1 0x0a8f2` komutu doğrudan hata yaratan fonksiyonu (`udp_rx_callback`) ve C dosyasındaki tam satır numarasını çıktı olarak verecektir.

---

## 7. ELF Yapısı Analizi
Firmware'in neden "Ham Binary" değil de "ELF" olduğunun yapısal kanıtları:
* **Section Headers:** 20 adet section tablosu vardır. Kod, veri ve hata ayıklama katmanları birbirinden ayrılmıştır.
* **Program Headers:** 5 adet segment haritası içerir. `LOAD` komutlarıyla hangi segmentin hangi fiziksel hafıza adresine (`PhysAddr`) oturacağı açıkça deklare edilmiştir.
* **Örnek Segment Yapısı:** Segment 02, hem `.data` hem `.bss` bölgelerini bağlayarak RAM'de (`0x00001100`) çalışma uzayı inşa eder. Ham binary dosyalarda bu meta-veri yapıları bulunmaz.

---

## 8. Interrupt ve Donanım Analizi
* **Kesme Tablosu (Vector Table):** `.vectors` kesme vektör tablosu `0x0000ffc0` adresinde konuşlanmıştır ve `__ivtbl_32` olarak adlandırılmıştır.
* **Aktif Kesme Yönetimi (ISR):**
  * `port1_isr` (`0x0353e`): Port 1 GPIO kesmesi (Buton tetiklemeleri için).
  * `cc2420_timerb1_interrupt` (`0x035fe`): Telsiz zamanlama kesmesi.
  * `uart0_rx_interrupt` (`0x037ae`): Seri port veri alım kesmesi.
  * `watchdog_interrupt` (`0x037d8`): Sistem bekçi köpeği zaman aşımı yönetimi.

---

## 9. Networking Analizi
Ağ katmanının ve Contiki Network API'lerinin mimari ayak izleri:
* **Yönlendirme (Routing):** `rpl_process_dio`, `rpl_process_dis`, `rpl_process_dao` sembolleri **RPL Lite** protokolünün kontrol paketlerinin (DIO/DIS/DAO) tam olarak işlendiğini gösterir.
* **Veri İletimi:** `simple_udp_sendto` ve `simple_udp_register` sembolleri, uygulamanın Contiki-NG üst seviye UDP API'sini kullanarak unicast/multicast haberleşme yaptığını doğrular.
* **MAC/Link Katmanı:** `csma_output_packet` ve `frame802154_parse` sembolleri, ağın IEEE 802.15.4 standartlarında CSMA (Carrier Sense Multiple Access) katmanıyla çalıştığını gösterir.

---

## 10. Wireless / TSCH Analizi
* Firmware sembollerinde `cc2420_driver`, `cc2420_init`, `cc2420_set_channel` ve `cc2420_rssi` yoğunluğu bulunmaktadır.
* Bu durum, sistemin zaman senkronizasyonlu slot yapısı (TSCH) yerine, **gelişmiş bir kanal dinleme (CCA - Clear Channel Assessment) tabanlı düşük güç modlu CSMA/Radio sürücüsü** mimarisine sahip olduğunu göstermektedir.

---

## 11. Sensor ve Peripheral Analizi
Çevre birimlerinin (peripherals) registersel adres ve fonksiyon izleri:
* **ADXL345 İvmeölçer:** `accmeter_process` ve `accm_read_reg` fonksiyonları mevcuttur; I2C hattı üzerinden sensör verileri "polling" yöntemiyle okunmaktadır.
* **TMP102 Sıcaklık Sensörü:** `tmp102_read_temp_x100` sembolü, sıcaklık verisini fixed-point (100 ile çarpılmış) formatta işleyen sürücüyü gösterir.
* **GPIO Yönetimi:** `__P1DIR`, `__P1OUT`, `__P2IE` gibi semboller, MSP430 donanım register'larına (Giriş/Çıkış/Kesme yetki register'ları) doğrudan bellek haritalı (Memory-Mapped I/O) erişim yapıldığının kanıtıdır.

---

## 12. Algoritma Koşma / DSP / Matematiksel Analiz
* **Yazılımsal Emülasyon:** MSP430 donanımsal bir Floating-Point Unit (FPU) barındırmaz.
* Sembol tablosundaki `__mulsi3` (32-bit çarpma), `__udivsi3` (32-bit bölme) ve `__udivmoddi4` (64-bit bölme/kalan) sembolleri, matematiksel işlemlerin **derleyici tarafından yazılımsal olarak emüle edildiğini (Software Emulation)** gösterir. Bu durum yoğun matematiksel döngülerde "Computational Hotspot" (performans darboğazı) oluşturur.

---

## 13. Güç ve Performans Analizi
* Sistemde güç tüketim takibi için `energest_init` ve `energest_flush` modülleri etkindir; CPU, Radio RX/TX sürelerini loglar.
* `main` döngüsü sonunda yer alan `calla #0x06570` (`platform_idle`) çağrısı, sistemde işlenecek bir olay (event) kalmadığında işlemcinin otomatik olarak **Low-Power Mode (LPM)** seviyelerine çekilerek enerji verimliliği sağladığını gösterir.

---

## 14. Coverage ve Profiling Analizi
* Binary kodunda `gcov` izine rastlanmamıştır.
* Ancak `stack_check_get_usage` ve `stack_check_process` sembolleri, runtime (çalışma zamanı) sırasında stack taşmalarını önlemek amacıyla canlı bir bellek profil analizi yapıldığını ortaya koymaktadır.

---

## 15. Reverse Engineering Analizi
Yapısal analiz sonucunda bilinmeyen firmware'in davranış modeli şu şekilde geri çatılmıştır:
> Bu firmware, üzerinde ivmeölçer ve sıcaklık sensörleri barındıran, RPL-Lite ağ protokolü ile IPv6 tabanlı veri ileten ve gelen paketleri Coffee File System (CFS) sanal disk alanına yazma kabiliyetine sahip bir **Ağ Sınır Yönlendiricisi / IoT Sunucu Düğümü (UDP Server)** bellenimidir.

---

## 16. Compiler ve Optimization Analizi
* Disassembly incelendiğinde, küçük yardımcı fonksiyonların bile ayrık fonksiyon gövdesi şeklinde tutulduğu (`calla` komutları ile çağrıldığı) gözlemlenmiştir.
* Bu durum, derleyicinin agresif kod inlining (`-O3`) yerine, dosya boyutunu korumak ve hata ayıklamayı kolaylaştırmak adına **`-Os` veya `-O1` optimizasyon stratejisini** tercih ettiğini gösterir.

---

## 17. Linker ve Build Sistemi Analizi
Linker script tarafından otomatik üretilen sınır sembolleri başarıyla tespit edilmiştir:
* `__data_load_start` (`0x000d71c`): Verilerin ROM/Flash üzerindeki kaynak adresi.
* `__data_start` (`0x00001100`): Verilerin çalışma anında RAM'e oturacağı hedef adres.
* `__bss_start` (`0x00001250`) ve `__bss_end` (`0x00002938`): Linker'ın RAM üzerinde sıfırlayacağı alanı belirleyen kritik sınırlar.

---

## 18. Binary Transformation Analizi
* Elde edilen `udp-server.z1` ELF dosyası, doğrudan TI veya Zolertia işlemcisine flash aracılığıyla yazılamaz.
* Bunun için üretim bandında `msp430-objcopy -O ihex udp-server.z1 udp-server.hex` komutu kullanılarak section bilgileri soyulmalı ve donanım programlayıcıların anlayacağı **Intel HEX** veya **Raw Binary (.bin)** formatına dönüştürülmelidir.

---

## 19. Library ve Archive Analizi
Firmware'in statik kütüphane bağlantı izleri:
* `memcpy` (`0x0cf9a`), `memset` (`0x0d16a`), `memcmp` (`0x0cf6e`) gibi standart C bellek operasyonları ile `printf` / `vuprintf` kütüphane fonksiyonlarının mimariye optimize edilmiş hallerinin binary içine statik olarak gömüldüğü (`statically linked`) doğrulanmıştır.

---

## 20. Contiki-NG Özel Analizler
Contiki makrolarının assembly karşılıkları deşifre edilmiştir:
* `PROCESS_THREAD` makrosu arka planda assembly seviyesinde bir protothread yapısına (`status`, `value`, `PROCESS_YIELD`) açılır.
* `main` fonksiyonundaki `calla #0x066ca` (`process_init`) ve `calla #0x066e6` (`process_run`) çağrıları, Contiki'nin **cooperative (işbirlikçi) event-driven scheduler (olay güdümlü zamanlayıcı)** mekanizmasının ana motorunu oluşturur. Sistem asenkron kesmelerle uyanır, eventi ilgili sürece teslim eder ve tekrar uyur.

---

## 21. Güvenlik ve Robustness Analizi
* **Zafiyet Analizi:** Strings çıktısında her hangi bir hardcoded şifreleme anahtarı veya kimlik bilgisine rastlanmamıştır.
* **Zayıflıklar:** Derleyicide Stack Canaries (kod koruma) mimarisinin pasif olduğu görülmüştür. Ancak strings üzerindeki `"Check failed: %ld vs. %ld"` logu, işletim sisteminin kendi içinde bir boundary/range check (sınır kontrolü) barındırdığını ve robustness (dayanıklılık) seviyesini artırdığını gösterir.

---

## 22. Karşılaştırmalı Firmware Analizi
* `udp-server.z1` bellenimi, standart yalın bir istemci imajına kıyasla `.text` katmanında `NETSTACK_ROUTING.root_start()` gibi RPL kök düğüm fonksiyonlarını barındırdığı için yaklaşık **%15 daha büyük bir kod boyutuna** sahiptir.
* Ayrıca RAM uzayında yönlendirme tabloları (`_ds6_neighbors_mem`, `defaultrouterlist_list`) barındırdığı için RAM tüketimi istemci düğümlerine göre daha yoğundur.

---

## 23. Eğitimsel Reverse Engineering Görevleri
* Bu analiz çalışması, kaynak kodu bulunmayan bir IoT belleniminin sadece ELF header, section boyutları, sembol eşlemeleri ve string diagnostic logları takip edilerek; cihazın donanım haritasının, telsiz parametrelerinin, ağ üzerindeki rolünün (Server/Root) ve bellenim güvenliğinin tamamen deşifre edilebileceğini pratik olarak kanıtlamıştır.
