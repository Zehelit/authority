---
name: authority
description: Check whether current Codex and PowerShell processes have elevated Windows administrator tokens and can perform protected write/read/delete operations. Use when authority is invoked or the user asks to verify current administrator access.
---

# authority

Kullanıcı `authority` veya `$authority` çağırdığında o anki oturumu kontrol et. Eski ölçümü yeni sonuç gibi kullanma. Windows içindir; kurulum, onarım veya yetki yükseltme yapmaz.

Ayrı script dosyası oluşturma veya çağırma. Aşağıdaki kontrolleri mevcut komut aracında doğrudan PowerShell komutlarıyla yap. RunAs, gsudo veya ayrı yükseltilmiş yardımcı kullanma: bunlar ölçülen oturumu değiştirir. Kullanıcı adı, exe yolu ve sürüm sabit değildir.

## Luna alt ajanı

Her `authority` çağrısında ana ajan tek bir alt ajan başlatır: `model: gpt-6-luna`, `reasoning_effort: low`, `fork_turns: none`. Ölçüm, erişim denemeleri, gerekiyorsa uygulama kökünü açma ve kullanıcıya sunulacak Türkçe sonucu hazırlama dahil bütün işler bu alt ajana aittir. Ana ajan görevi İngilizce ve kendine yeterli biçimde aktarır; bu SKILL.md yolunu ve aşağıdaki adımların tamamını belirtir. Ana ajan kontrol komutu veya klasör açma işlemi yapmaz; hazır sonucu kullanıcıya aktarır. Alt ajan başka ajan başlatmaz. Luna/low alt ajanı başlatılamazsa kontrolleri başka modelle yapma; eksikliği bildir.

Görev bitince alt ajanın son sonucunu al ve canlı ajan envanterinde artık bulunmadığını doğrula. Hâlâ çalışıyorsa durdurup yeniden kontrol et; canlı ajan olarak kalırsa bunu bildir ve kapanmış gibi gösterme. Aynı ajanı yeniden kullanma. Yetki kontrolünü beklemeyen eski goal veya workflow çalışmasını başlatma.

## Kontroller

1. PowerShell'in mevcut kimliğini, etkin yönetici üyeliğini ve sürümünü ölç:

   ```powershell
   $identity = [Security.Principal.WindowsIdentity]::GetCurrent()
   $principal = [Security.Principal.WindowsPrincipal]::new($identity)
   [pscustomobject]@{
       Time = [DateTimeOffset]::Now.ToString('o')
       User = $identity.Name
       PID = $PID
       PowerShell = $PSVersionTable.PSVersion.ToString()
       AdminEnabled = $principal.IsInRole([Security.Principal.WindowsBuiltInRole]::Administrator)
   }
   whoami /groups
   ```

   Grup çıktısında bütünlük SID'sini de kontrol et: `S-1-16-12288` yüksek, `S-1-16-8192` orta düzeydir. Administrators grubunun yalnız listelenmesi yeterli değildir; deny-only üyelik etkin yönetici yetkisi sayılmaz.

2. `Get-CimInstance Win32_Process` ile `$PID` üzerinden üst süreç zincirini bul. Codex engine ve masaüstü süreçlerini gerçek exe yollarıyla ilişkilendir; başka oturumların süreçlerini karıştırma. Üst süreç çocuktan yeni oluşturulmuşsa PID yeniden kullanılmış olabilir; o noktada zinciri kes. Her ilgili sürecin **kendi token'ını** salt okunur Windows API sorgusuyla ölç: `OpenProcess(PROCESS_QUERY_LIMITED_INFORMATION)`, `OpenProcessToken(TOKEN_QUERY)`, `GetTokenInformation(TokenElevation)`. Gerekirse bu çağrıları aynı komutta bellekte `Add-Type` ile tanımla; yardımcı dosya üretme. Açılan handle'ları kapat. Okunamayan veya ilişkilendirilemeyen süreç için “doğrulanamadı” de; PowerShell sonucunu Codex'e mal etme.

3. Mevcut PowerShell üzerinden iki küçük erişim denemesi yap: gerçek Program Files kökünde GUID adlı yeni bir klasör/dosya; HKLM SOFTWARE altında GUID adlı yeni bir anahtar/değer. 64 bit Windows'ta 64 bit konumları kullan. Her birine benzersiz bir değer yaz ve geri okuyup karşılaştır. Her denemeyi `try/finally` ile temizle; yalnız o çağrıda oluşturduğun tam hedefi sil. Önceden var olan hedefleri veya reparse point'leri kullanma. Temizlikten önce dosyanın beklenen değeri taşıdığını ve klasörün yalnız beklenen dosyayı içerdiğini doğrula. Program Files deneme dosyasını `[IO.File]::Delete($file)` ile, boş deneme klasörünü `[IO.Directory]::Delete($dir, $false)` ile sil; `Remove-Item` kullanma. HKLM deneme anahtarını `RegistryKey.DeleteSubKey($name, $false)` ile sil. Özyinelemeli silme yapma; beklenmeyen içerik varsa dur ve bildir. Temizlik sonrasında hedeflerin bulunmadığını kontrol et. Mevcut programlar, ayarlar ve ACL'ler değiştirilmez.

4. İlişkili Codex masaüstü, engine veya PowerShell süreçlerinden biri **doğrulanmış biçimde yükseltilmemişse**, o oturumun üst süreç zincirindeki `ChatGPT.exe` yürütülebilir dosya yolundan uygulama paketinin kökünü belirle. Yol `...\app\ChatGPT.exe` ise kök, `app` klasörünün üst dizinidir; başka düzende gerçek exe dizinini kullan. Yolu veya süreç ilişkisini varsayma, başka oturumun `ChatGPT.exe` sürecini seçme. Kökün varlığını doğrula ve mevcut kullanıcı belirteciyle `explorer.exe` üzerinden bu dizini aç. Mümkünse açılan Explorer penceresini gözlemle; yalnız `Start-Process` başarısından görünürlüğü kesin sayma. Yol belirlenemez, açılamaz veya Windows klasör içeriğini reddederse bunu ayrı bildir. Uygulamayı yeniden başlatma, yükseltme veya ACL değiştirme. Her üç süreç de doğrulanmış biçimde yükseltilmişse bu adımı çalıştırma.

## Sonuç

Sonucu Türkçe, kısa bir tabloda sun: **Codex masaüstü**, **Codex engine**, **PowerShell**, **Program Files**, **HKLM**, **temizlik**, **uygulama kökü**. Ölçüm saatini, ilgili PID'leri, PowerShell sürümünü, kullanılan model/düzeyi ve alt ajanın tamamlanma durumunu belirt.

- Token elevated, admin grubu etkin, iki deneme ve temizlik başarılıysa mevcut PowerShell'in yönetici olarak çalıştığını ve sınanan erişimlerin çalıştığını söyle.
- Token yönetici ama deneme başarısızsa yetki ve işlem sonucunu ayrı belirt; hata, hedef ve aşamayı göster.
- İşlem başarılı ama token yönetici değilse yalnız o konumdaki erişim doğrulanmıştır; yükseltilmiş yönetici oturumu deme.
- Komut araç politikası nedeniyle başlatılamazsa bunu Windows erişim reddiyle karıştırma; fiilî işlem doğrulanamadı de. Aynı testi döngüyle yeniden deneme.
- Temizlik başarısızsa başarı ilan etme; kalan tam yolu/anahtarı bildir. Yalnız bu çağrının kendi kalıntısı için hedefli temizlik yapılabilir.

Bu sonuç bütün sistem işlemlerinin, kilitli dosyaların, TrustedInstaller kaynaklarının veya başka programların erişiminin garantisi değildir. Süreç yeniden başlatma, UAC/ACL/onay politikası değiştirme, workflow/goal/pilot başlatma. Kalıcı arşiv gerekmez; kullanıcı isterse sonucu kaydet.
