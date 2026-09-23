IL2CPP Tersine Mühendislik ve Canlı Proses Bellek Yaması — Profesyonel Rehber

Sürüm: 1.0
Hedef Platform: Android 16 (API 36), ARM64 (aarch64)
Kapsam: Unity IL2CPP ile derlenmiş oyunlarda dump.cs analizi, offset kavramları ve root erişimli canlı proses bellek yaması
Dış Bağımlılık: Sadece Termux binutils paketi (Python, radare2, Frida, Magisk modülü yok)

---

İçindekiler

1. IL2CPP Mimarisi ve Derleme Süreci
2. dump.cs ve Offset Kavramları
3. Offset Türleri ve Semantiği
4. Statik vs Dinamik (Runtime) Yama
5. Android 16 Güvenlik Katmanları
6. Bellek Erişim Mekanizmaları
7. ARM64 ABI ve Çağrı Sözleşmesi
8. Uçtan Uca Simülasyon
9. Doğrulama ve İleri Teknikler

---

1. IL2CPP Mimarisi ve Derleme Süreci

1.1 IL2CPP Nedir?

Unity, C# kodunu doğrudan çalıştırmak yerine IL2CPP (Intermediate Language to C++) adlı bir AOT (Ahead-of-Time) derleyicisi kullanır. Süreç:

```
C# kaynak kodu
    ↓ (Roslyn)
CIL / IL bytecode  (assembly-csharp.dll)
    ↓ (Unity IL2CPP derleyicisi)
C++ kaynak kodu  (il2cpp-output/*.cpp)
    ↓ (NDK clang)
ARM64 makine kodu  (libil2cpp.so)
```

1.2 Çıktı Bileşenleri

Dosya Rol
libil2cpp.so Derlenmiş ARM64 makine kodu — tüm C# metodlarının native karşılıkları
global-metadata.dat Tip tanımları, alan isimleri, metod imzaları, string literaller, offset bilgileri
dump.cs IL2CPP Dumper tarafından metadata'dan yeniden inşa edilen okunabilir sözde C#

global-metadata.dat şifrelenmiş veya gizlenmiş olabilir (Unity 2021+ bazı sürümlerde). Bu durumda dump almak için Il2CppDumper'a doğru metadata header'ı beslemek gerekir.

1.3 Native Kod-İçi Yapılar

Her IL2CPP derlemesinde bulunan zorunlu yapılar:

· Il2CppClass — her C# sınıfı için runtime tip bilgisi
· MethodInfo — her metodun pointer, isim, parametre bilgisi
· Il2CppObject — heap'teki her nesnenin başlığı (klass pointer'ı + monitor)
· VTable — sanal metodların dispatch tablosu

Bir C# nesnesinin heap yerleşimi tipik olarak:

```
+0x00  Il2CppClass*  klass
+0x08  void*         monitor (senkronizasyon)
+0x10  ... kullanıcı alanları ...
```

Bu yüzden dump.cs'teki ilk kullanıcı alanı offseti genelde 0x10'dan başlar.

---

2. dump.cs ve Offset Kavramları

2.1 dump.cs Yapısı

Tipik bir dump satırı:

```csharp
public class PlayerWallet : MonoBehaviour
{
    public int money;              // 0x18
    public int totalEarned;        // 0x1C

    public void AddMoney(int amount);   // RVA: 0x1A2B3C Offset: 0x1A2B3C VA: 0x1A2B3C
    public int  GetMoney();             // RVA: 0x1A2D40 Offset: 0x1A2D40 VA: 0x1A2D40
}
```

2.2 // 0x18 Ne Demek?

Bu bir alan offsetidir. PlayerWallet tipinde bir nesne heap'te allocate edildiğinde, money alanı nesne başlangıcından 24 byte sonra yer alır.

```
PlayerWallet instance:
offset 0x00 → Il2CppClass* klass
offset 0x08 → void* monitor
offset 0x10 → (muhtemelen base sınıf alanı veya padding)
offset 0x18 → int money         ← burada
offset 0x1C → int totalEarned   ← burada
```

Erişim: *(int32_t*)((uint8_t*)obj + 0x18)

2.3 // RVA: 0x1A2B3C Ne Demek?

RVA = Relative Virtual Address. Metodun libil2cpp.so belleğe yüklendiğindeki taban adresine göre göreceli konumu.

Çalışma zamanındaki gerçek adres:

```
absolute_address = module_base + RVA
```

module_base, /proc/<pid>/maps çıktısından okunur. ASLR nedeniyle her proses başlangıcında farklıdır.

2.4 // Offset: ... ve // VA: ...

· Offset (dump.cs bağlamında): Genellikle RVA ile aynıdır. Bazı dumper sürümleri burada dosya offseti verir.
· VA (Virtual Address): Mutlak sanal adres. Statik analiz sırasında readelf'in verdiği VirtAddr + RVA toplamıdır. Çalışma zamanında ASLR ile kayar.

---

3. Offset Türleri ve Semantiği

3.1 Sınıflandırma

Offset Türü Taban İşaret Ettiği Şey
Alan (field) offseti Nesne başlangıcı Veri (int, float, pointer, obje ref)
Metod RVA Modül tabanı Kod (makine talimatları)
Statik alan RVA Modül tabanı Statik veri bölgesi
String literal offset Metadata tabanı UTF-8 string bloğu

3.2 Kritik Ayrım

Offset ne bir değer tutar ne de bir fonksiyonun kendisidir. Offset konum bilgisidir.

· Alan offseti verilirse: değerin nerede olduğunu söyler.
· Metod RVA'sı verilirse: kodun nerede olduğunu söyler.

Bu ayrımı karıştırmamak için iki soruyu sor:

1. Hangi yapının offseti? (nesne mi, modül mü?)
2. Ne işaret ediyor? (veri mi, kod mu?)

3.3 İleri: Statik Alan ve Thread Static

```csharp
public static int globalScore;   // RVA: 0x2F3A10
```

Bu bir global değişkendir, nesne başına değil. Adresi:

```
absolute = module_base + 0x2F3A10
```

Değeri okumak/yazmak için PTRACE_POKEDATA doğrudan bu adrese uygulanır.

---

4. Statik vs Dinamik (Runtime) Yama

4.1 Karşılaştırma

Kriter Statik Yama Dinamik (Runtime) Yama
Hedef Diskteki libil2cpp.so RAM'deki kopya
Kalıcılık Kalıcı (APK yeniden imzalanmalı) Geçici (proses kapanınca biter)
Bütünlük kontrolü Tetikler Tetiklemez
İmza doğrulaması Bozulur Etkilenmez
Root gerekli mi? Hayır (repack için) Evet (ptrace için)
APK değişir mi? Evet Hayır
Öğrenme/analiz Uygun Uygun

4.2 Statik Yamanın Riskleri

1. İmza doğrulaması — APK'yı yeniden imzalayınca orijinal imza bozulur. Bazı oyunlar PackageManager.getPackageInfo().signatures kontrol eder.
2. Sunucu taraflı doğrulama — Oyun sunucusu, client'tan gelen verinin bütünlüğünü doğrularsa patch tespit edilir.
3. AVB / dm-verity — Sistem bölümüne yazılırsa cihaz boot etmez.
4. Oyun içi checksum — libil2cpp.so'nun hash'i karşılaştırılırsa patch tespit edilir.

4.3 Dinamik Yamanın Avantajları

· APK dosyası dokunulmaz.
· İmza doğrulaması geçer.
· Diskteki lib bozulmaz.
· Oyun her yeniden başlatıldığında temiz state.
· Deneme yanılma kolay (yanlış patch → oyunu yeniden başlat).

4.4 Dinamik Yamanın Sınırları

· Proses kapanınca patch kaybolur.
· Root + CAP_SYS_PTRACE gerekir.
· SELinux enforcing ise engellenebilir.
· Bazı hardened kernel'lar PTRACE_POKEDATA'yı kod sayfalarında kısıtlar.
· Anti-tamper: bazı oyunlar kendi kod segmentini periyodik olarak kontrol eder.

---

5. Android 16 Güvenlik Katmanları

5.1 SELinux

Type Enforcement modeli. Her proses bir domain'e aittir (ör. untrusted_app, platform_app). Her dosya bir type'a aittir (ör. apk_data_file, system_file). Policy, hangi domain'in hangi type'a ne yapabileceğini tanımlar.

Permissive mod: Kural ihlalleri log'a yazılır ama engellenmez.

```bash
su -c "getenforce"       # Enforcing mi Permissive mi?
su -c "setenforce 0"     # Geçici permissive (yeniden başlatmada sıfırlanır)
```

Kalıcı permissive için kernel cmdline'a androidboot.selinux=permissive eklenir (bootloader unlock gerekir).

5.2 AVB (Android Verified Boot)

Boot zinciri boyunca imza doğrulaması. Bootloader → boot.img → system.img. Değiştirilmiş bir bölüm, hash uyuşmazlığı nedeniyle boot'u durdurur.

5.3 dm-verity

Sistem bölümünün blok bazlı bütünlük kontrolü. Salt-okunur bölümde bir blok değişirse, hash tree uyuşmazlığı tespit edilir.

5.4 W^X (Write XOR Execute)

Bir bellek sayfası aynı anda hem yazılabilir hem çalıştırılabilir olamaz. ARM64'te PXN (Privileged Execute Never) ve UXN (User Execute Never) bitleri ile donanım seviyesinde uygulanır.

Ancak: ptrace(PTRACE_POKEDATA) bu kısıtı baypas eder. Çünkü ptrace kernel seviyesinde çalışır ve sayfa izinlerini kontrol etmez — sadece hedef prosesin adres uzayına yazar.

5.5 PAC (Pointer Authentication Code)

ARMv8.3+ ile gelen imzalama mekanizması. Fonksiyon pointer'ları ve dönüş adresleri PAC ile imzalanır. Yanlış bir pointer kullanımı → crash.

Kod yaması için PAC kritik değildir (biz kod byte'larını değiştiririz, pointer değil), ama hook yazarken dikkat gerekir.

5.6 BTI (Branch Target Identification)

ARMv8.5+ ile gelen mekanizma. Dolaylı branch'lerin hedefi BTI talimatıyla başlamalıdır. Aksi halde SIGILL.

Fonksiyon başına trambolin yazarken BTI c (call) veya BTI j (jump) talimatı eklemek gerekebilir.

5.7 ASLR (Address Space Layout Randomization)

Her proses başlangıcında libil2cpp.so'nun base adresi değişir. Bu yüzden:

· RVA sabittir.
· Base adres dinamiktir.
· Absolute adres = base + RVA, her açılışta yeniden hesaplanmalı.

---

6. Bellek Erişim Mekanizmaları

6.1 /proc/<pid>/maps

Her prosesin sanal adres haritası. Örnek satır:

```
7a00000000-7a012345000 r-xp 00000000 fd:00 4321 /data/app/.../lib/arm64/libil2cpp.so
```

Kolonlar: start-end perms offset dev inode path

· r-xp → read + execute, private
· 7a00000000 → segment başlangıç sanal adresi (base buradan)
· 00000000 → dosya içi offset

libil2cpp.so birden fazla segment içerebilir (kod, rodata, data). İlk r-xp segmenti tipik olarak kodu içerir; base adres budur.

6.2 /proc/<pid>/mem

Prosesin sanal bellek imajını dosya gibi sunan özel dosya. read() ve write() çağrıları, ptrace(PTRACE_PEEKDATA/PTRACE_POKEDATA) ile eşdeğerdir.

Kritik özellik: write() çağrısı, hedef sayfa salt-okunur (r-x) olsa bile başarılı olur. Çünkü kernel, ptrace yolunu kullanır ve W^X kontrolünü atlar.

Ön koşullar:

· Çağıran proses CAP_SYS_PTRACE yetkisine sahip olmalı (root).
· Hedef proses dumpable olmalı (/proc/<pid>/status içinde CoreDumping veya TracerPid kontrolü).
· Yamaç ptrace_scope ayarı izin vermeli (/proc/sys/kernel/yama/ptrace_scope).

6.3 dd ile Erişim

```bash
dd of=/proc/<pid>/mem bs=1 seek=<addr> conv=notrunc
```

· bs=1 → tek byte biriminde yaz (lseek için gerekli).
· seek=<addr> → sanal adrese lseek. dd içeride lseek() çağırır; kernel bunu /proc/<pid>/mem için özel olarak handle eder ve ptrace POKEDATA yoluna yönlendirir.
· conv=notrunc → dosya sonu kırpma; bellek dosyası için semantik olarak önemsiz ama güvenli.

Okuma için:

```bash
dd if=/proc/<pid>/mem bs=1 skip=<addr> count=<n>
```

6.4 ptrace_scope ve Yama

Bazı dağıtımlarda ptrace_scope=1 ile sadece parent proses ptrace yapabilir. Root, ptrace_scope=0 veya CAP_SYS_PTRACE ile baypas eder. Android'de varsayılan olarak root erişimi yeterlidir.

6.5 Alternatif Mekanizmalar

Yöntem Avantaj Dezavantaj
/proc/<pid>/mem + dd Basit, dosya gerekmez Root + ptrace
Frida Memory.patch Yüksek seviye, dinamik Frida server gerekir
process_vm_writev syscall Kernel-native, ptrace'siz C kodu gerekir
LKM (Loadable Kernel Module) En güçlü Kernel derleme, imza
LD_PRELOAD Kütüphane seviyesi hook Sadece dinamik link edilenler

---

7. ARM64 ABI ve Çağrı Sözleşmesi

7.1 Kayıtçılar

Kayıtçı Kullanım
x0-x7 Fonksiyon argümanları ve dönüş değeri
x8 Dolaylı sonuç kayıtçısı
x9-x15 Geçici (caller-saved)
x16-x17 Intra-procedure-call (IP0, IP1)
x18 Platform register (Android'de TLS)
x19-x28 Kalıcı (callee-saved)
x29 Frame Pointer (FP)
x30 Link Register (LR) — dönüş adresi
sp Stack Pointer
xzr / wzr Sıfır kayıtçısı

7.2 IL2CPP Metod Çağrı Kuralı

C# tarafında:

```csharp
public void AddMoney(int amount);
```

Instance metodu olduğu için:

· x0 = this pointer'ı (PlayerWallet*)
· w1 = amount (int32, 32-bit görünüm)
· Dönüş değeri yok

Eğer static olsaydı:

```csharp
public static void AddMoney(int amount);
```

· w0 = amount

7.3 Tip Genişliği

· int → w register (32-bit)
· long → x register (64-bit)
· float → s register (32-bit)
· double → d register (64-bit)
· bool → w register (0 veya 1)
· Referans → x register (64-bit pointer)

7.4 Örnek Disassembly

```asm
AddMoney:
    ldr  w2, [x0, #0x18]    ; w2 = this->money
    add  w2, w2, w1          ; w2 = w2 + amount
    str  w2, [x0, #0x18]    ; this->money = w2
    ldr  w3, [x0, #0x1c]    ; w3 = this->totalEarned
    add  w3, w3, w1          ; w3 += amount
    str  w3, [x0, #0x1c]    ; this->totalEarned = w3
    ret                      ; x30 → pc
```

ldr (Load Register), str (Store Register), add, ret — hepsi standart ARM64.

---

8. Uçtan Uca Simülasyon

8.1 Ortam

```
~/coinclicker/
├── libil2cpp.so        (analiz için; diskte değiştirilmeyecek)
├── global-metadata.dat
└── dump.cs
```

Bağımlılıklar:

```bash
pkg update && pkg upgrade -y
pkg install binutils -y
```

Sadece binutils — içinde readelf, objdump, as, objcopy, xxd, dd, cmp.

Ön koşullar:

· Root erişimi (su)
· SELinux permissive veya uygun policy
· Hedef oyun kurulu ve çalışır durumda
· uname -m → aarch64

8.2 Hedef Metodun Bulunması

```bash
grep -n -i "AddMoney\|Earn\|Wallet\|money" dump.cs
```

Çıktı:

```
1209: public void AddMoney(int amount);  // RVA: 0x1A2B3C Offset: 0x1A2B3C
```

RVA = 0x1A2B3C

8.3 Orijinal Kodun İncelenmesi

objdump ile diskteki lib üzerinden:

```bash
objdump -d --start-address=0x1A2B3C --stop-address=0x1A2B60 libil2cpp.so
```

Çıktı:

```
1a2b3c:  ldr  w2, [x0, #24]
1a2b40:  add  w2, w2, w1
1a2b44:  str  w2, [x0, #24]
1a2b48:  ldr  w3, [x0, #28]
1a2b4c:  add  w3, w3, w1
1a2b50:  str  w3, [x0, #28]
1a2b54:  ret
```

Anlam: this->money += amount; this->totalEarned += amount;

Not: #24 = 0x18, #28 = 0x1C. objdump bunları decimal gösterir.

8.4 Yeni Fonksiyonun Tasarlanması

İstenen: her çağrıda money = totalEarned = 999999.

999999 = 0xF423F — 20 bit, tek movz yetmez. movz + movk gerekir.

Yeni assembly:

```asm
movz w2, #0x423f
movk w2, #0xf, lsl 16
str  w2, [x0, #0x18]
str  w2, [x0, #0x1c]
ret
```

8.5 Byte Kodlamasının Hesaplanması

Elle hesaplanmış ARM64 kodlamaları (little-endian byte'lar):

Talimat 32-bit kod Byte sırası (LE)
movz w2, #0x423f 0x528847E2 E2 47 88 52
movk w2, #0xf, lsl 16 0x72A001E2 E2 01 A0 72
str w2, [x0, #0x18] 0xB9001802 02 18 00 B9
str w2, [x0, #0x1c] 0xB9001C02 02 1C 00 B9
ret 0xD65F03C0 C0 03 5F D6

Toplam 20 byte. Orijinal fonksiyon da ~28 byte. Sığıyor.

8.6 Canlı Proses Bilgilerinin Toplanması

```bash
am start -n com.example.coinclicker/.MainActivity
sleep 3
```

PID ve base adresi:

```bash
su -c '
PID=$(pidof com.example.coinclicker)
BASE=0x$(grep libil2cpp.so /proc/$PID/maps | head -1 | cut -d"-" -f1)
ADDR=$((BASE + 0x1A2B3C))
printf "PID=%d  BASE=0x%x  TARGET=0x%x\n" "$PID" "$BASE" "$ADDR"
'
```

Örnek çıktı:

```
PID=12345  BASE=0x7a00000000  TARGET=0x7a001a2b3c
```

BASE'in ilk r-xp segmentinden okunduğuna dikkat: /proc/<pid>/maps birden çok satır döndürebilir, head -1 kod segmentini alır.

8.7 Yamanın Uygulanması

Dosya oluşturmadan, printf ile byte'ları pipe'a bastırıp dd ile doğrudan /proc/<pid>/mem'e yaz:

```bash
su -c '
PID=$(pidof com.example.coinclicker)
BASE=0x$(grep libil2cpp.so /proc/$PID/maps | head -1 | cut -d"-" -f1)
ADDR=$((BASE + 0x1A2B3C))

printf "\xe2\x47\x88\x52\xe2\x01\xa0\x72\x02\x18\x00\xb9\x02\x1c\x00\xb9\xc0\x03\x5f\xd6" \
  | dd of=/proc/$PID/mem bs=1 seek=$ADDR conv=notrunc 2>/dev/null

echo "Yama yazıldı: $(printf 0x%x $ADDR)"
'
```

Neden bu çalışır?

· dd of=/proc/<pid>/mem gördüğünde open(O_WRONLY) çağırır. Kernel, bu yola yazmayı ptrace mekanizmasına yönlendirir.
· lseek() çağrısı hedef sanal adresi belirler.
· write() çağrısı her byte için PTRACE_POKEDATA'ya dönüşür (kernel içinde).
· Hedef sayfa r-x (salt-okunur + çalıştırılır) olsa bile yazma başarılı olur, çünkü ptrace kernel seviyesinde çalışır ve sayfa izin kontrolünü atlar.

8.8 Doğrulama

Canlı bellekten oku:

```bash
su -c '
PID=$(pidof com.example.coinclicker)
BASE=0x$(grep libil2cpp.so /proc/$PID/maps | head -1 | cut -d"-" -f1)
ADDR=$((BASE + 0x1A2B3C))

dd if=/proc/$PID/mem bs=1 skip=$ADDR count=20 2>/dev/null | xxd
'
```

Beklenen:

```
00000000: e247 8852 e201 a072 0218 00b9 021c 00b9
00000010: c003 5fd6
```

Bu, yazdığımız byte'ların aynısı. Yama başarılı.

8.9 Oyun İçi Test

Oyunu aç, para kazanma butonuna bas. money ve totalEarned her tıklamada 999999 olur.

Harcama yapıldığında SpendMoney gerçek değeri düşürür (o metodun içinde money -= amount vardır), ama bir sonraki AddMoney çağrısı tekrar 999999'a yazar.

8.10 Temizlik

Oyunu kapat:

```bash
am force-stop com.example.coinclicker
```

Bellek sıfırlanır, yama kaybolur. Diskteki libil2cpp.so bozulmamıştır.

---

9. Doğrulama ve İleri Teknikler

9.1 Yamanın Kalıcı Hale Getirilmesi (Opsiyonel)

Her oyun açılışında otomatik uygulamak için:

· Bir init script (/data/adb/service.d/ altına) oyun başlangıcını algılayıp aynı dd komutunu çalıştırabilir.
· inotifywait ile /proc altında yeni proses oluşumunu izleyip tetikleyebilir.
· Magisk modülü yazılabilir (ama önceki konuşmada dış bağımlılık istenmedi).

9.2 Karmaşık Yama: Code Cave

Yeni fonksiyon 20 byte'tan uzun olsaydı ve orijinal alana sığmasaydı:

1. Yeni kodu dosyanın sonundaki bir boş alana yaz (dd ile sona ekle).
2. Orijinal fonksiyonun başına b <yeni_adres> (branch) talimatı koy.

Branch talimatı 4 byte'tır, 26-bit offset taşır (±128 MB). Hedef adres pc'ye göre hesaplanır:

```
offset = (target - pc) / 4
b_encoding = 0x14000000 | (offset & 0x03FFFFFF)
```

9.3 Hook (Kanca) vs Patch

Yöntem Davranış
Patch Orijinal kod değiştirilir; fonksiyon tamamen farklı çalışır
Hook Fonksiyonun başına atlama konur; özel kod çalışır, orijinal çağrılabilir

Hook için x30 (LR) ve caller-saved kayıtçıları saklamak gerekir. Ayrıca PAC ve BTI aktifse BTI c / PACIASP talimatları eklenmelidir.

9.4 Anti-Tamper Tespiti

Bazı oyunlar şunları yapar:

· Kendi kod segmentini periyodik olarak hashler → yama tespit edilir.
· /proc/self/maps okur → bilinmeyen kütüphane arar.
· ptrace(PTRACE_TRACEME) çağırır → debugger varlığını anlar.
· TracerPid alanını kontrol eder.

Bu tespitleri baypas etmek için ek hook'lar gerekir (ör. fopen("/proc/self/maps") üzerine hook).

9.5 ARM64 Talimat Kodlaması Elle Hesaplama

movz:

```
31     23  22 21        5 4    0
1 0 1 0 0 1 0 1 hw imm16 Rd
```

· hw = 16-bit shift (0, 16, 32, 48 için 0, 1, 2, 3)
· imm16 = 16-bit immediate
· Rd = hedef kayıtçı

movz w2, #0x423f:

```
sf=0 (32-bit)  opc=10 (movz)  hw=00  imm16=0x423F  Rd=00010 (w2)
= 0x528847E2
```

9.6 IL2CPP Metod Çağrı Sözleşmesi Detayı

IL2CPP'de instance metodu native imzası:

```c
ReturnType MethodName(TypeName* __this, Arg1 a1, Arg2 a2, ..., const MethodInfo* method);
```

Son parametre her zaman const MethodInfo*'dır ve genellikle x8 üzerinden geçirilir (IL2CPP runtime özel kuralı). Ancak basit metodlarda x8 kullanılmaz ve ihmal edilebilir.

Static metod:

```c
ReturnType MethodName(Arg1 a1, ..., const MethodInfo* method);
```

Bu yüzden AddMoney(int amount) native imzası:

```c
void AddMoney(PlayerWallet* __this, int32_t amount, const MethodInfo* method);
```

x0 = this, w1 = amount, x8 = method (görmezden gelinir).

9.7 Bellek Haritası Örneği

Tipik bir IL2CPP prosesinin libil2cpp.so segmentleri:

```
7a00000000-7a00xxxxxx r-xp 00000000  ... libil2cpp.so    (kod, .text)
7a00xxxxxx-7a00yyyyyy r--p 000xxxxx  ... libil2cpp.so    (.rodata)
7a00yyyyyy-7a00zzzzzz rw-p 000yyyyy  ... libil2cpp.so    (.data, .bss)
```

RVA'lar .text segmentine düşer. Eğer bir RVA .rodata veya .data aralığındaysa, o bir veri offsetidir, kod değil.

Doğru segmenti bulmak için:

```bash
readelf -S libil2cpp.so | grep -E "text|rodata|data"
```

9.8 RVA → Dosya Offseti Dönüşümü

ELF dosyasında bir RVA'nın hangi dosya offsetine karşılık geldiğini bulmak için Program Headers kullanılır:

```bash
readelf -l libil2cpp.so
```

Her LOAD segmenti için:

```
if (VirtAddr <= RVA < VirtAddr + FileSiz):
    file_offset = Offset + (RVA - VirtAddr)
```

Çoğu IL2CPP derlemesinde ilk LOAD segmenti Offset == VirtAddr şeklindedir (yani RVA = dosya offseti). Ama garanti değildir.

9.9 Metodun Doğru Sınırını Bulmak

objdump ile metod başlangıcından itibaren 32-64 byte disassemble edip ret talimatına kadar bak:

```bash
objdump -d --start-address=0x1A2B3C --stop-address=0x1A2B80 libil2cpp.so
```

ret (0xD65F03C0) genellikle son talimattır. Bundan sonraki byte'lar bir sonraki fonksiyona aittir.

Bazı metodlarda birden fazla ret olabilir (koşullu dönüşler). Tüm ret'ler aynı fonksiyona aittir.

---

Ek: Terimler Sözlüğü

Terim Açıklama
IL2CPP Unity'nin C# → C++ → native derleme zinciri
RVA Relative Virtual Address — modül tabanına göre offset
VA Virtual Address — mutlak sanal adres
ASLR Address Space Layout Randomization
PAC Pointer Authentication Code
BTI Branch Target Identification
W^X Write XOR Execute
SELinux Security-Enhanced Linux
AVB Android Verified Boot
dm-verity Device-mapper bütünlük kontrolü
ptrace Linux proses izleme syscall'ı
PTRACE_POKEDATA ptrace ile hedef belleğe yazma operasyonu
Code cave Yeni kod için kullanılan boş alan
Hook Fonksiyona müdahale (çağrıyı ele geçirme)
Patch Fonksiyonun kendi kodunu değiştirme

---

Ek: Sık Karşılaşılan Hatalar

Hata Neden Çözüm
dd: /proc/<pid>/mem: Permission denied Root yok veya SELinux engelliyor su -c ile çalıştır, SELinux permissive yap
Yanlış adrese yazma → oyun crash RVA yanlış veya base yanlış segment /proc/<pid>/maps'i dikkatle oku, ilk r-xp satırını al
Patch uygulandı ama etkisi yok Yanlış fonksiyon veya this pointer'ı yanlış dump.cs'te metodun sınıfını doğrula, x0'ın this olduğundan emin ol
Oyun açılışta kapanıyor Patch anti-tamper'ı tetikledi Farklı bir fonksiyonu patch'le veya hook kullan
ASLR nedeniyle adres sürekli değişiyor Normal davranış Her seferinde /proc/<pid>/maps'ten base'i yeniden oku

---

Son not: Bu rehber yalnızca eğitim ve güvenlik araştırması amaçlıdır. Üçüncü taraf oyunlarda izinsiz değişiklik yapmak, oyunun kullanım şartlarını ihlal eder ve yasal sonuçlar doğurabilir.