# MSI Mystic Light — Reverse Engineering Notları

**Hedef cihaz:** USB `0db0:0076` "MYSTIC LIGHT" (MSI MAG B850 TOMAHAWK MAX WIFI anakartının dahili fan/case ARGB başlıkları). OpenRGB'de bu VID:PID için bir algılama girişi var ama yanlış protokol varyantını hedefliyor ("packet length = 0" hatası). Üst akış hatası: GitLab CalcProgrammer1/OpenRGB#4645 (açık, henüz yama yok).

## Durum (2026-09-12 itibarıyla)

- **MSI Center kuruldu ama çalışmıyor.** Sürekli "internet bağlantısı yok" hatası veriyor — halbuki VM içinden DNS/ping/HTTPS hepsi çalışıyor, NCSI sorunu değil.
- **Kök neden bulundu:** MSI Center'ın "Case" servisi cihazı `bCaseConnected=False` sanıyor ve bu yüzden RGB kontrol ekranını tamamen gizliyor. Muhtemelen bir VM-tespit mekanizması.
- **Denenen VM-gizleme yöntemleri** (işe yaramadı, sorun çözülmedi ama denemeye değer diğer fikirler için referans):
  - SMBIOS spoofing (`virt-xml --sysinfo`) ile gerçek anakart bilgileri (Micro-Star International, MS-7E62) VM'e enjekte edildi.
  - Ağ kartı modeli `virtio`'dan `e1000e`'ye çevrildi (virtio "sanal" algılanıyor olabilir diye).
  - vCPU topolojisi `sockets=1,cores=8` yapıldı (çok soketli görünüm VM izlenimi verebilir diye, ayrıca Windows Pro'nun 2-soket lisans sınırına takılmasın diye).
  - `API_Case.dll` / `MsiHid.dll` içinde statik string araması yapıldı, net bir VM-tespit kontrolü bulunamadı.
- **Sıradaki adım (henüz yapılmadı):** `API_Case.dll`/`MsiHid.dll`'in gerçek disassembly/decompile edilmesi (string araması yeterli olmadı) — Ghidra kurulu (bkz. `zalman-oz-lcd-re` reposundaki notlar), aynı yöntem burada da kullanılabilir. Alternatif: MSI Center yerine doğrudan OpenRGB'yi VM içinde deneyip ürettiği trafiği yakalamak (kullanıcı önce host-native hedefine odaklanmayı seçmişti, bu seçenek beklemede).

## USB trafiği yakalama — çalışıyor

Host tarafında `usbmon` (kernel modülü, `/dev/usbmon1`, izin `setfacl -m u:drol:rw /dev/usbmon1` ile açıldı) + `tshark` (Homebrew ile kuruldu) kullanılarak Mystic Light'tan (USB adres 5) gerçek trafik yakalandı:

- Cihaz standart HID descriptor'ları veriyor.
- Her ~1.1 saniyede bir **64 byte'lık periyodik interrupt-IN paketi** gönderiyor.
- Bu, USB seviyesinde iletişimin canlı ve gerçek olduğunu doğruluyor — ama henüz bir **renk/efekt değiştirme komutu** yakalanmadı çünkü MSI Center `bCaseConnected` engeli yüzünden o noktaya hiç ulaşamadı.

Eski yakalama dosyası (`mystic_capture.pcapng`) session-scoped scratchpad'de kalmıştı, muhtemelen artık erişilemez — gerekirse yeniden yakalanmalı.

## Sıradaki adımlar

1. MSI Center'ın `bCaseConnected=False` sorununu çözmek için `API_Case.dll`/`MsiHid.dll`'i Ghidra ile decompile etmek.
2. VEYA doğrudan VM içinde OpenRGB kurup GUI üzerinden renk değiştirmeyi denemek, üretilen trafiği yakalamak (MSI Center'ı tamamen bypass eder).
3. Renk komutu yakalanınca: mevcut OpenRGB algılama koduna (yanlış protokol varyantı sorunu olan) doğru komut formatını yamalamak, veya GitLab#4645 issue'sine katkı sunmak.
