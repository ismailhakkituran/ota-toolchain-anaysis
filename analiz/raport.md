# Contiki-NG `rpl-udp` Firmware — ELF Analiz Raporu

> **Hedef:** `new-firmware.z1` — Contiki-NG `examples/rpl-udp` örneğinden derlenmiş, Zolertia **Z1** (TI **MSP430**) mote firmware'i.
> **Kaynak dizin:** `/home/karakas10/contiki-ng/examples/rpl-udp/new-firmware.z1`
> **Analiz veri klasörü:** `analiz/new-firmware/`
> **Analiz tarihi:** 2026-06-06

---

## 1. Yönetici Özeti

| Özellik | Değer |
|---|---|
| Format | ELF32, little-endian, EXEC |
| Mimari | TI MSP430 (MSP430X — 20-bit adres uzayı) |
| ABI / OS | Standalone (bare-metal), statically linked |
| Derleyici | GCC 4.7.2 — `mspgcc dev 20120911` |
| Debug bilgisi | Mevcut (DWARF, **strip edilmemiş**) |
| Giriş noktası | `0x3100` (vector tablosu reset vektörüne işaret eder) |
| Section sayısı | 21 |
| Program header (segment) | 6 (5× LOAD, sıralı bellek bölgeleri) |
| Sembol sayısı | 1 143 (502 FUNC, 229 OBJECT) |
| Toplam görüntü | **77 757 B (0x12FBD)** — text 71 715 + data 336 + bss 5 706 |

**Tespit edilen bileşenler:** Contiki-NG çekirdeği (`process`, `etimer`, `ctimer`, `stack_check`), 6LoWPAN/uIPv6 ağ yığını (`sicslowpan`, `uip6`, `uip-ds6-*`, `uip-icmp6`, `uip-nd6`, `uip-sr`), **RPL-Lite** yönlendirme (`rpl_dag_init_root`, `rpl_icmp6_*`, `rpl_neighbor_*`, `rpl_timers_*`), CSMA MAC, **CC2420** 2.4 GHz radyo sürücüsü, Z1 platform sürücüleri (LED, sensör, accmeter, button-hal, UART0). Bu, klasik `rpl-udp` örneğinin Z1 hedefli derlemesidir.

---

## 2. ELF Başlığı (`01-elf-header.txt`)

```
Class:    ELF32                Type:    EXEC
Data:     little endian        Machine: TI MSP430
OS/ABI:   Standalone App       Version: 0x1
Entry:    0x3100               Flags:   0x10000001
Section headers: 21 (offset 0x17178)
Program headers:  6 (offset 52, size 32)
```

Notlar:
- `Flags = 0x10000001` → MSP430**X** (büyük model / 20-bit adres). Bunun göstergesi `.far.text` segmentinin `0x10000` üzerinde yerleşmesidir.
- `OS/ABI = Standalone App` (ABI = 0xFF) → bare-metal yürütme; dinamik link yok (`05-dynamic.txt`: *no dynamic section*).
- `Entry = 0x3100` → `.text` başlangıcı, klasik MSP430 `_reset_vector__` (bkz. `13-addr2line.txt`).

---

## 3. Bellek Yerleşimi — Section ve Segment Haritası

### 3.1 Section özetleri (`02-section-headers.txt`)

| # | Section | VMA aralığı | Boyut | Bayrak | Açıklama |
|---|---|---|---:|---|---|
| 1 | `.far.text` | `0x10000 – 0x14A77` | 19 064 B | AX | "Far" kod (MSP430X uzak çağrılar, ör. `input`, `uip6` rutinleri) |
| 2 | `.text`     | `0x03100 – 0x0C86D` | 38 766 B | AX | Birincil program kodu, reset/ISR yollarını içerir |
| 3 | `.rodata`   | `0x0C870 – 0x0FE6C` | 13 821 B | A  | String literalleri, sabit tablolar |
| 4 | `.data`     | `0x01100 – 0x0124F` |    336 B | WA | Başlangıç değerli RAM verileri (LMA `0xFE6E` flash içinde) |
| 5 | `.bss`      | `0x01250 – 0x02897` |  5 704 B | WA | Sıfırlanmış RAM verileri |
| 6 | `.noinit`   | `0x02898 – 0x02899` |      2 B | WA | İnit edilmeyen alan |
| 7 | `.vectors`  | `0xFFC0 – 0xFFFF`   |     64 B | AX | MSP430 kesme vektör tablosu |

> **Debug bölümler (LOAD edilmez):** `.debug_info` (~6.8 KB), `.debug_line` (~4 KB), `.debug_abbrev`, `.debug_loc`, `.debug_aranges`, `.debug_frame`, `.debug_str`, `.debug_ranges` — toplam ~22 KB. Cihaza yazılmaz ama analizi kolaylaştırır.
> **Sembol tabloları:** `.symtab` 18 288 B, `.strtab` 16 048 B → strip ile ciddi küçülme mümkün.

### 3.2 Program Header / Segment Haritası (`03-program-headers.txt`)

```
LOAD  VAddr=0x0000300C  FileSiz=0x09862  R E  → .text
LOAD  VAddr=0x0000C870  FileSiz=0x035FD  R    → .rodata
LOAD  VAddr=0x00001100  FileSiz=0x00150  RW   → .data + .bss   (LMA 0x0FE6E)
LOAD  VAddr=0x00002898  FileSiz=0x00000  RW   → .noinit         (LMA 0x0FFBE)
LOAD  VAddr=0x0000FFC0  FileSiz=0x00040  R E  → .vectors
LOAD  VAddr=0x00010000  FileSiz=0x04A78  R E  → .far.text
```

**Yorum:** `.data` ve `.bss` için `LMA ≠ VMA` ayrımı net görülüyor — başlangıç değerleri Flash'ta (`0xFE6E`) duruyor, çalışma zamanında RAM'e (`0x1100`) crt0 tarafından kopyalanıyor. `.far.text` MSP430X uzun adresleme bölgesinde (`0x10000+`).

### 3.3 Görsel bellek haritası

```
 RAM (0x1100 – 0x2899)               ROM/Flash
 ┌──────────────────────┐            ┌──────────────────────┐
 │ .data    0x1100      │ ←─ copy ── │  init image @ 0xFE6E │
 │ .bss     0x1250      │            ├──────────────────────┤
 │ .noinit  0x2898      │            │ .text     0x3100     │ 38 766 B
 └──────────────────────┘            │ .rodata   0xC870     │ 13 821 B
                                     │ .vectors  0xFFC0     │     64 B
                                     │ .far.text 0x10000    │ 19 064 B
                                     └──────────────────────┘
```

---

## 4. Boyut Profili (`09-size.txt`)

```
   text     data    bss     dec      hex     filename
  71715      336   5706   77757   0x12fbd   ../new-firmware.z1
```

- **Flash kullanımı:** 71 715 + 336 = **72 051 B** (Z1 92 KB flash — ~%78 doluluk).
- **RAM kullanımı:** 336 + 5 706 = **6 042 B** (Z1 8 KB RAM — ~%74 doluluk).
- Z1 için bu, RPL + 6LoWPAN + IPv6 yığınının tipik ayak izidir; ek özellik eklemek için RAM oldukça dar.

---

## 5. Bileşen / Modül Tespiti (sembol & string analizi)

`04-symbol-table.txt`, `08-nm-sorted.txt`, `10-strings.txt` ve `13-addr2line.txt` üzerinden çapraz okumayla tespit edilen modüller:

### 5.1 Contiki-NG çekirdeği
| Sembol | Adres | Rol |
|---|---|---|
| `process_thread_etimer_process` | `0x51A8` | Event timer process |
| `process_thread_ctimer_process` | `0x4F94` | Callback timer process |
| `process_thread_tcpip_process`  | `0xC5F4` | TCP/IP girdi/çıktı process |
| `process_thread_stack_check_pr` | `0xC074` | Stack overflow denetimi |
| `process_thread_sensors_proc`   | `0xAB06` | Sensor demultiplexer |
| `process_thread_cc2420_proces`  | `0x4034` | CC2420 receive process |
| `process_thread_accmeter_proce` | `0x389C` | Z1 ivmeölçer process |
| `process_thread_hello_world`    | `0x5BA6` | Örnek uygulama process'i |

### 5.2 Ağ / Yönlendirme yığını
- **6LoWPAN:** `sicslowpan_init` (`0xAC40`), `output` (`0xB1CC`, 3 572 B — büyük fonksiyon), `add_fragment`, `uncompress_addr`, `compress_addr_64`, `store_fragment`.
- **uIPv6:** `uip_icmp6_input/init`, `uip_sr_init`, `echo_request_input`/`echo_reply_input`, `ns_input`, `chksum`, `upper_layer_chksum`, `ext_hdr_options_process`, `uip_check_mtu`, `uip_update_ttl`, `uipbuf_*`.
- **RPL-Lite:** `rpl_dag_init_root` (`0x7E62`), `rpl_icmp6_init/dao_output`, `rpl_neighbor_init/get_from_lla`, `rpl_timers_init`, `init_dag`, `process_dio_init_dag`.
- **DS6 / Komşu yönetimi:** `uip-ds6-nbr.c`, `uip-ds6-route.c`, `uip-ds6.c` (defaultroutermemb, ds6_neighbors_struct, _ds6_neighbors_mem).

### 5.3 Z1 / MSP430 platform katmanı
- **Radyo:** `cc2420_init` (`0x4436`, 234 B), `cc2420_read`, `cc2420_last_rssi`.
- **MAC:** CSMA çağrı izleri (sembollerde `csma`, `packetbuf_copyto`, `packet_sent`).
- **Sürücüler:** `leds_arch_init`, `spi_init`, `rtimer_arch_now`, UART0 (`uart0_writeb`, `uart0_input_handler`, `putchar`), button-hal / gpio-hal stubları (UND — başka modülden bağlanıyor).
- **Platform:** `init_platform` (`0x45AC`), `platform_init_stage_three`, `stack_check_init`.
- **Crt0 / libgcc:** `_reset_vector__`, `__umoddi3` (msp430gcc 4.7.2).

### 5.4 String işaretçileri (`10-strings.txt`)
Yığının çalıştığını doğrulayan log formatları:
```
TTL value from rpl-lite: %u
rpl_icmp6_dao_output: not in an instance, skip sending DAO
rpl_icmp6_dao_output: no preferred parent, skip sending DAO
rpl_icmp6_dao_output: node not ready to send a DAO (prefix %p, parent addr %p, mop %u)
output: uip_len is smaller than uncomp_hdr_len (%d < %d)
uncompression: UDP length: %u (ext: %u) ip_len: %d udp_len: %d
udp: bad checksum 0x%04x 0x%04x
udp: zero port.
udp: no matching connection found
In udp_found
In udp_send
Removing IPv6 extension headers (extlen: %d, uiplen: %d)
Hello, EK-D103
Hello world process
```

> `Hello, EK-D103` ve `Hello world process` stringleri, örneğin standart `rpl-udp` üzerine eklenmiş bir özelleştirme olduğunu gösteriyor.

---

## 6. Kesme Vektör Tablosu (`.vectors` @ `0xFFC0`)

64 baytlık tablo, MSP430F2617 (Z1 MCU) için tipik. Reset vektörü (`0xFFFE`) → `_reset_vector__` → `0x3100` (entry point).

---

## 7. Disassembly Özeti (`06-disassembly.txt`)

`.far.text` başında `input` fonksiyonu (`0x10000`, 4 490 B — sicslowpan'ın `input` rutini):
- MSP430X **`calla`** (call absolute, 20-bit) bolca kullanılıyor → Büyük kod modeli doğrulanıyor.
- Erken çağrılar: `0x6AC8` (packetbuf erişimi), `0x5ED8` (RDC/CSMA katmanı), `0x6966` ve `0x6956` (packetbuf attr).
- `printf` çağrıları (`calla #0x13e88`) hata yolundan yapılıyor (log altyapısı).

---

## 8. Debug Bilgisi (`13-addr2line.txt` örnek doğrulama)

| Adres | Sembol | Kaynak |
|---|---|---|
| reset | `_reset_vector__` | `gcc-4.7.2/libgcc/config/msp430/crt0.S:118` |
| libgcc | `__umoddi3` | `gcc-4.7.2/libgcc/config/msp430/libgcc.S:992` |
| Z1 radyo | `cc2420_read` | `cc2420.c` |
| Contiki çekirdeği | `process_thread_ctimer_process` | `ctimer.c` |
| RPL | `link_stats_init` | `??:0` (satır eşlemesi yok) |

DWARF tutarlı, `.debug_aranges` ile sembol↔kaynak çözünürlüğü mümkün.

---

## 9. Strip / Optimizasyon Önerileri

Cihazda asıl ihtiyacınız olmayan, sadece analiz/debug için duran bölümler ~22 KB sembol + ~22 KB DWARF yer tutuyor. Üretim build'inde:

```bash
msp430-objcopy --strip-debug new-firmware.z1 new-firmware-stripped.z1
# veya tamamen:
msp430-strip new-firmware.z1
```

Bu, `.z1`/IHEX boyutunu düşürmez (yalnız LOAD segmentleri programlanır), **ama** dağıttığınız ELF artefaktının boyutunu ve sızdırdığı bilgiyi azaltır. (Z1'e yüklenen yalnızca 5 LOAD segmentidir; Flash ayak izi bu işlemden etkilenmez.)

---

## 10. Güvenlik / Tersine Mühendislik Notları

1. **Strip edilmemiş ELF** — fonksiyon ve dosya isimleri (`sicslowpan.c`, `uip6.c`, `rpl-*`, `cc2420.c`, `contiki-main.c`, `node-id-z1.c`) tamamen açık. Tersine mühendislik son derece kolay.
2. **DWARF kaynak yolları** — derleyen kullanıcının ev dizinini açığa çıkarıyor: `/home/karakas10/contiki-ng/...` ve `/home/user/tmp/gcc-4.7.2-msp430/...`. Üretim öncesi `-fdebug-prefix-map` veya strip önerilir.
3. **Stack check process** mevcut (`process_thread_stack_check_pr`) — taşma erken yakalanır, iyi bir güvenlik tedbiri.
4. **CC2420 link-layer şifreleme** sembolü görünmüyor → muhtemelen **AES/CCM kapalı**. Üretim deployment için `LLSEC` ayarı gözden geçirilmeli.
5. RAM oldukça dolu (%74). Ek 6LoWPAN bağlam, RPL parent veya komşu sayısının artırılması taşmaya yol açabilir.

---

## 11. Analiz Dosyaları İçindekiler

| Dosya | İçerik |
|---|---|
| `01-elf-header.txt` | `readelf -h` — ELF başlığı |
| `02-section-headers.txt` | `readelf -S` — section tablosu |
| `03-program-headers.txt` | `readelf -l` — segment haritası |
| `04-symbol-table.txt` | `readelf -s` — tam sembol tablosu (1 143 girdi) |
| `05-dynamic.txt` | `readelf -d` — boş (statik link) |
| `06-disassembly.txt` | `objdump -d` — tam disassembly |
| `07-objdump-sections.txt` | `objdump -h` — section özetleri |
| `08-nm-sorted.txt` | `nm -n` — adrese göre sıralı semboller |
| `09-size.txt` | `size` — text/data/bss özeti |
| `10-strings.txt` | `strings` — okunabilir yazılar |
| `11-file-info.txt` | `file` — format teşhisi |
| `12-comment.txt` | `.comment` bölümü (GCC sürümü) |
| `13-addr2line.txt` | Örnek sembol → kaynak çözünürlüğü |
| `16-hex-export.hex` | Cihaza yazılacak Intel HEX export |

---

*Rapor, `analiz/new-firmware/` klasöründeki çıktıların çapraz okunmasıyla otomatik üretildi. Cooja simülasyonunda farklı bir Z1 düğümü kullanıyorsanız sembol adresleri değişir; bu yapı (RPL-Lite + 6LoWPAN + CC2420) sabittir.*
