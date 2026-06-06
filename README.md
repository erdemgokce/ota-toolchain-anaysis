# ELF Analizi Raporu

Bu rapor, Contiki-NG işletim sistemi üzerinde derlenmiş `new-firmware.z1` imajının tersine mühendislik ve yapısal analiz sonuçlarını içermektedir. Bellenim, standart düğümlere kıyasla 64 KB bellek sınırını aşarak `.far.text` uzayına taşmış özel bir derleme profiline sahiptir. Analizler, **Texas Instruments CC1352R / MSP430X** SoC mimarisi referans alınarak 24 maddede özetlenmiştir.

---

## 1. Binary Kimlik Analizi

| Özellik | Değer |
| :--- | :--- |
| **Platform** | Zolertia Z1 / MSP430X Serisi |
| **Mimari** | MSP430X — Flags: 0x10000001 (20-bit genişletilmiş komut seti) |
| **ELF Formatı** | ELF32 Executable (Statik bağlı konteyner) |
| **Endianness** | Little-endian (2's complement) |
| **Giriş Noktası** | 0x3100 (Sistem ilk açıldığında bootloader'ın zıplayacağı adres) |
| **ABI Bilgisi** | Standalone App (İşletim sistemi olmadan çıplak donanımda çalışır) |
| **Derleyici İzi** | GCC: (GNU) 4.7.2 20120920 (mspgcc dev 20120911) |
| **Toolchain** | Contiki-NG-release/v4.8-625-g8518cbaff-dirty |
| **Optimizasyon** | -Os tahmini (Boyut minimizasyonu yapılmış) |
| **Debug Sembolü**| Mevcut (not stripped) — .debug_info, .debug_line korunmuş |

**Araçlar:** msp430-readelf -h, msp430-objdump -x, msp430-strings

---

## 2. Bellek Kullanım Analizi

| Özellik | Değer |
| :--- | :--- |
| **Flash Kullanımı** | 71.715 Byte (.text + .far.text + .rodata toplamı) |
| **RAM Kullanımı** | 6.042 Byte (.data + .bss + .noinit toplamı) |
| **.text Boyutu** | 71.715 Byte (64 KB barajını aştığı için .far.text alanı devrededir) |
| **.data Boyutu** | 336 Byte (İlk değer atanmış global/static değişkenler) |
| **.bss Boyutu** | 5.706 Byte (Başlangıçta sıfırlanan RAM alanı) |
| **Stack Kullanımı**| 0x3100 adresinden (RAM tavanı) aşağı doğru dinamik büyür |
| **Heap Analizi** | Dinamik tahsisat (malloc) yok |
| **Section Dağılımı**| 6 Program segmenti (Örn: Segment 05, 0x010000 adresine taşan kodları tutar) |
| **Memory Map** | Kodun bir kısmı 0x3100, taşan kısmı 0x10000 adreslerine oturur |

**Araçlar:** msp430-size, msp430-readelf -S, msp430-nm

---

## 3. Symbol / Function Analizi

| Özellik | Değer |
| :--- | :--- |
| **Fonksiyonlar** | main (0x0313e), rpl_process_dio (0x00344) |
| **Global Değişken** | rpl_neighbors, sensors, process_list |
| **Static Değişken** | dis_handler, dio_handler, debouncetimer |
| **ISR Fonk.** | port1_isr (0x0353e), cc2420_timerb1_interrupt |
| **Contiki Process**| tcpip_process, etimer_process, **hello_world_process** (Ekstra Yük) |
| **Radio Driver** | cc2420_process, cc2420_on, cc2420_off |
| **Network Callback**| rpl_link_callback, netstack_process_ip_callback |
| **Sensor Handler** | accmeter_process (ADXL345), sensors_process |
| **Kütüphaneler** | Matematik emülasyonları (__mulsi3, __udivhi3) |

**Araçlar:** msp430-nm -n, msp430-readelf -s, msp430-objdump -t

---

## 4. String ve Metadata Analizi

| Özellik | Değer |
| :--- | :--- |
| **Debug Mesajları**| "incomplete IPv6 header received (%d bytes)" |
| **printf Logları** | "initiating global repair (%s), version %u, rank %u" |
| **Sensör İsimleri**| "ADXL345 sensor", "TMP102 sensor" |
| **Protokol Adları**| "RPL Lite", "6LoWPAN", "CSMA", "IPv6" |
| **Diagnostic Log** | "Inconsistent DIO version", "Failed to retrieve max radio driver..." |
| **Developer Notları**| "NS: no space left for root node!", "failed to create a new RPL DAG" |

**Araçlar:** msp430-strings

---

## 5. Assembly / Instruction Analizi

| Özellik | Değer |
| :--- | :--- |
| **Instruction Seq**| pushm.a ile 20-bit register yedekleme işlemleri görülmüştür |
| **Pro/Epilogue**| pushm.a #8, r11 ile girilip, reta (Return Aligned) ile çıkılır |
| **Register** | r11 - r15 arası genel amaçlı parametre aktarımı için aktiftir |
| **ISR Akışı** | Kesme rutinlerinde dint (Disable Int.) ve eint (Enable Int.) yoğunluktadır |
| **Compiler Davranışı**| Fonksiyon çağrıları için 20-bit adreslemeye uygun calla komutları kullanılmıştır |

**Araçlar:** msp430-objdump -d, msp430-as

---

## 6. Source-Level Mapping Analizi

| Özellik | Değer |
| :--- | :--- |
| **Source Eşleme** | .debug_aranges, .debug_line vb. 8 adet DWARF debug bölümü mevcuttur |
| **Adres Çözümleme**| Derleyici yolları (/home/user/tmp/gcc-4.7.2.../crt0.S:118) bozulmamıştır |
| **Optimization Etkisi**| -Os nedeniyle inline edilmiş kodlar adres-satır eşleşmelerinde atlamalar yapar |

**Araçlar:** msp430-addr2line, msp430-objdump -S

---

## 7. ELF Yapısı Analizi

| Özellik | Değer |
| :--- | :--- |
| **Section Header** | 21 adet bölüm. .text (Normal kod) ve .far.text (Genişletilmiş kod) bir arada |
| **Program Header** | 6 adet segment. Segment 05, 0x010000 fiziksel adresine yükleme (LOAD) yapar |
| **Vector Table** | 0xffc0 adresine sabitlenmiştir (16-bit alan içinde kalmak zorundadır) |
| **Initialization** | __init_stack ve __data_start rutinleri RAM'i hazırlar |

**Araçlar:** msp430-readelf -a, msp430-elfedit

---

## 8. Interrupt ve Donanım Analizi

| Özellik | Değer |
| :--- | :--- |
| **Interrupt Vektörü**| 0xffc0 - 0xffff aralığında 32 kesme slotu donanım için ayrılmıştır |
| **Timer Kullanımı** | timera0, timera1 ve cc2420_timerb1_interrupt (Radyo zamanlayıcısı) |
| **GPIO / Sensor** | port1_isr (Buton), i2c_rx_interrupt (İvmeölçer I2C hattı) |
| **Donanım Erişimi** | __ADC12MCTL0, __UC1IE, __DMACTL0 gibi donanım register'ları memory-mapped |

**Araçlar:** msp430-objdump, msp430-readelf

---

## 9. Networking Analizi

| Özellik | Değer |
| :--- | :--- |
| **IPv6 Stack** | tcpip_ipv6_output, uip_ds6_init (Ağ katmanı tam kapasite devrede) |
| **RPL Routing** | rpl_process_dio, rpl_icmp6_dao_output (Mesh ağ yönlendirme aktif) |
| **MAC Layer** | csma_output_packet (Çarpışma önleyici MAC katmanı) |
| **Neighbor Table** | neighbor_addr_mem_memb_mem, _rpl_neighbors_mem üzerinden yönetilir |
| **Network Hataları** | "IPv6 cache full", "neighbor table full" hata mekanizmaları (Robustness) |

**Araçlar:** msp430-nm, msp430-objdump, msp430-strings

---

## 10. Wireless / TSCH Analizi

| Özellik | Değer |
| :--- | :--- |
| **TSCH Operation** | Zaman senkronizasyonlu TSCH yerine CCA tabanlı CSMA aktiftir |
| **Channel Logic** | cc2420_set_channel fonksiyonu ile sabit frekans bandı seçilir |
| **Radio Timing** | schedule_transmission ve rtimer_arch_schedule ile paket asenkron zamanlanır |

**Araçlar:** msp430-objdump, msp430-nm

---

## 11. Sensor ve Peripheral Analizi

| Özellik | Değer |
| :--- | :--- |
| **Button Handler** | button_sensor, gpio_hal_arch_port_interrupt_enable |
| **I2C / SPI** | __UCB0I2CIE (Sensörler arası seri haberleşme protokolleri aktif) |
| **Sensör Polling** | accmeter_process I2C hattı üzerinden ivme verisini sürekli okur (polling) |
| **ADC Routines** | __ADC12MCTL0 ile 12-bit Analog-Dijital çevirici devrededir |

**Araçlar:** msp430-objdump, msp430-nm

---

## 12. Algoritma Koşma / DSP Analizi

| Özellik | Değer |
| :--- | :--- |
| **Floating-Point** | Donanımsal FPU (Yüzen nokta ünitesi) çipte mevcut değildir |
| **Software Emul.**| __mulsi3 (Çarpma) ve __udivsi3 (Bölme) komutları yazılımsal emüle edilir |
| **DSP/Hotspot** | Emüle edilen matematik işlemleri hesaplama darboğazı (hotspot) yaratır |

**Araçlar:** msp430-objdump, msp430-nm

---

## 13. Güç ve Performans Analizi

| Özellik | Değer |
| :--- | :--- |
| **ISR Yoğunluğu** | 20'den fazla donanım kesmesi sürekli asenkron tetiklenmektedir |
| **Low-Power Mode** | Olay gelmediğinde CPU uyutulur. Kritik işlerde dint (Disable Int.) çekilir |
| **Radio Duty Cycle** | cc2420_set_txpower ile radyo iletim gücü dinamik ayarlanarak pil korunur |
| **Flash/RAM Etkisi** | 71 KB'lık dev kod alanı, MCU'nun Flash verimliliğini %90 sınırına yaklaştırır |

**Araçlar:** msp430-gprof, msp430-objdump, msp430-size

---

## 14. Coverage ve Profiling Analizi

| Özellik | Değer |
| :--- | :--- |
| **Test Coverage** | gcov enjeksiyonu tespit edilmemiştir (Üretim imajı) |
| **Runtime Bottleneck**| stack_check_process arka planda sürekli stack taşmalarını denetler |

**Araçlar:** msp430-gcov, msp430-gprof

---

## 15. Reverse Engineering Analizi

| Özellik | Değer |
| :--- | :--- |
| **Sınıflandırma**| İvme ve sıcaklık okuyabilen, ağı yöneten (RPL DAG Root) merkez düğüm |
| **Rol Çıkarımı** | "created a new RPL DAG" logları düğümün sıradan bir uç birim olmadığını kanıtlar |
| **Feature Inference** | Kodun içine fazladan gömülü olan hello_world_process, boyut şişkinliğinin sebebidir |

**Araçlar:** msp430-objdump, msp430-nm, msp430-readelf, msp430-strings

---

## 16. Compiler ve Optimization Analizi

| Özellik | Değer |
| :--- | :--- |
| **Optimizasyon** | -Os kullanılmasına rağmen kod 71 KB'a fırlamıştır |
| **Büyük Veri Yönetimi**| Derleyici, 64KB sınırını aşan input vb. fonksiyonları .far.text bölümüne atmıştır |
| **Address Translation**| calla (Call Aligned 20-bit) ve pushm.a komutları uzun adresler için zorunludur |

**Araçlar:** msp430-gcc, msp430-objdump

---

## 17. Linker ve Build Sistemi Analizi

| Özellik | Değer |
| :--- | :--- |
| **Section Placement** | Linker, __far_data_start ve __far_bss_start sembolleriyle RAM haritasını uzatır |
| **Vector Placement** | Donanım mimarisi gereği .vectors kesmesi 0xffc0 (16-bit alan) içine kilitlenmiştir |

**Araçlar:** msp430-ld, msp430-readelf

---

## 18. Binary Transformation Analizi

| Özellik | Değer |
| :--- | :--- |
| **ELF Dönüşümü** | Çip üzerine yazılırken .far.text alanı Intel HEX genişletilmiş adres formatı gerektirir |
| **Debug Removal** | strip komutu ile .debug_* alanları atılarak imaj programlamaya hazır hale getirilir |

**Araçlar:** msp430-objcopy, msp430-strip

---

## 19. Library ve Archive Analizi

| Özellik | Değer |
| :--- | :--- |
| **Kütüphane İçeriği** | vuprintf, __mulsi3 gibi rutinler Contiki libc arşivinden statik olarak koda gömülüdür |
| **Dynamic Link** | IoT platformlarında dinamik yükleyici olmadığı için harici DLL/SO bulunmaz |

**Araçlar:** msp430-ar, msp430-ranlib

---

## 20. Contiki-NG Özel Analizler

| Özellik | Değer |
| :--- | :--- |
| **Scheduler Analizi** | process_current, process_list ile asenkron olay güdümlü işletim sistemi kanıtlanmıştır |
| **etimer/ctimer** | etimer_request_poll, ctimer_set sistemin non-blocking zamanlayıcı motorlarıdır |
| **Packetbuf** | packetbuf_ptr, packetbuf_hdr_len tamponları ile uçtan uca ağ paketi işlenir |

**Araçlar:** msp430-cpp, msp430-objdump, msp430-nm

---

## 21. Güvenlik ve Robustness Analizi

| Özellik | Değer |
| :--- | :--- |
| **Buffer Handling** | Ağdan gelen paketlerin sınırları uip_check_mtu ile kontrol edilir |
| **Stack Güvenliği** | Donanım koruması yerine yazılımsal stack_check_process ile overflow engellenir |
| **Veri Bütünlüğü** | __TLV_CHECKSUM sembolü donanım kalibrasyon verilerinin bozulup bozulmadığını tartar |

**Araçlar:** msp430-strings, msp430-objdump

---

## 22. Karşılaştırmalı Firmware Analizi

| Özellik | Değer |
| :--- | :--- |
| **Code Size Farkı** | Yeni firmware, standart istemciye göre ~29.000 Byte (%60) daha obezdir |
| **Şişkinlik Sebebi** | RPL Kök Düğüm yönetimi, ekstra sensör kütüphaneleri ve hello_world_process eklenmesidir |

**Araçlar:** msp430-size

---

## 23. Eğitimsel Reverse Engineering Görevleri

| Özellik | Değer |
| :--- | :--- |
| **Kazanımlar** | 64 KB sınırını aşan (MSP430X 20-bit) bellenimlerin hafıza haritalaması çözülmüştür |
| **Rol Çıkarımı** | Cihazın sıradan bir istemci değil, ağır yük taşıyan bir IPv6 yönlendiricisi olduğu kanıtlanmıştır |
| **Gelişim** | Sadece statik analiz araçlarıyla donanım, yazılım, ağ ve güç profili başarıyla deşifre edilmiştir |

**Araçlar:** Bütün msp430-binutils araç zinciri

---

## 24. CC1352R OTA Yükleyici (Loader) İmplementasyonu ve Bellek Görselleştirmesi

Projenin hedef donanımı olan **Texas Instruments CC1352R SoC**, 352 KB Flash ve 80 KB SRAM'e sahiptir. Havadan (OTA) gelen `new-firmware.z1` imajının güvenli bir şekilde diske yazılıp çalıştırılabilmesi için donanım seviyesinde **BIM (Boot Image Manager)** adında bir yükleyici (loader) implementasyonu kullanılmalıdır.

### OTA Bootloader (BIM) Mantığı ve Flash Hafıza Haritası

OTA güncellemeleri sırasında cihazın çökmemesi için Flash bellek sanal olarak **Slot A** (Aktif) ve **Slot B** (Güncelleme) olmak üzere ikiye bölünür.

```text
=============================================================
|           CC1352R FLASH MEMORY MAP (352 KB)               |
=============================================================
| 0x00000000 | BIM (Boot Image Manager - Loader)            |
|------------|----------------------------------------------|
| 0x00001000 | SLOT A (Aktif Çalışan Firmware)              |
|            | -> udp-server imajı burada çalışır           |
|------------|----------------------------------------------|
| 0x0002B000 | SLOT B (OTA ile İnen Yeni İmaj Deposu)       |
|            | -> Havadan inen 71 KB'lık imaj buraya yazılır|
|------------|----------------------------------------------|
| 0x00056000 | NVS (Non-Volatile Storage) / Config Verileri |
|------------|----------------------------------------------|
| 0x00057FFF | Flash Sonu                                   |
=============================================================
