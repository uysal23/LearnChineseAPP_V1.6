# Konuşma alıştırması: cihaz üzerinde ses tanıma (sürüm 1.5)

## Karar
Önceki sürümlerde (1.3–1.4) bir "telaffuz değerlendirmesi" (0–100 puan, alt ölçümler, 3 aşamalı
sesli sınav) vardı ve puan çevrimiçi bir Azure sunucusundan ya da takılabilir bir yerel motordan
geliyordu. Araştırmaya rağmen (bkz. eski `docs/YEREL_MOTOR.md`, bu sürümde kaldırıldı) telefonda
çalışan, doğrulanmış bir Mandarin **telaffuz puanlama** motoru bulunamadı.

Bu sürümde talimatla **karar 3** uygulandı:
- Sınav geçme şartlarından ses eşleştirme/telaffuz puanı tamamen **kaldırıldı**. Sınav yeniden
  **2 aşamalıdır**: Aşama 1 kelime (%90), Aşama 2 gramer/boşluk doldurma (%85). Önceki sürümdeki
  3. aşama, `ST.pr`, `PronunciationTask/VoiceAttempt/PronunciationAssessment` veri modeli, Azure
  sunucusu (`server/server.js`) ve ilgili tüm ekranlar **silindi**.
- Ses tanıma yalnızca **alıştırmalar için** var: her sahnede "🗣 Konuşma alıştırması" (10 kelime +
  10 cümle, karışık). Cihazın kendi (Android `SpeechRecognizer`, çevrimdışı tercih edilir) konuşma
  tanımasını kullanır, söylediğini **"duyulan metin"** olarak gösterir ve hedef metinle karakter
  karakter eşleştirir (✔/✖). **Hiçbir puan, yüzde ya da 0–100 skoru üretilmez veya gösterilmez.**
  Bu alıştırma ilerlemeye/sınava girmez, tamamen isteğe bağlı kendi kendine çalışmadır.

## Neden ses eşleştirmesi bir "telaffuz puanı" değil
Ses tanımanın döndürdüğü metin, hedef metne ne kadar yakınsa o kadar "doğru söylenmiş" demek
**değildir** — tanıyıcı hata yapmış olabilir, tonu hiç değerlendirmez, gürültüde yanlış duyabilir.
Bu yüzden arayüzde bunu her zaman "puanı değildir" notuyla birlikte, yalnızca bilgilendirici bir
"ne duyuldu" göstergesi olarak sunuyoruz; ✔/✖ işaretleri de bir "doğruluk yüzdesi" değil, yalnızca
o karakterin duyulan metinde bulunup bulunmadığını gösterir.

## Teknik
- **Motor:** `android.speech.SpeechRecognizer` + `RecognizerIntent` (`EXTRA_PREFER_OFFLINE=true`,
  dil `zh-CN`). Cihazda Çince çevrimdışı konuşma tanıma paketi kuruluysa tamamen çevrimdışı çalışır;
  yoksa cihazın kendi ayarları devreye girer (uygulama ayrıca bir yere bağlanmaz).
  Köprü: `AndroidASR` (`isAvailable/hasPermission/requestPermission/start/stop/cancel`) →
  JS `window.__asr(evt, payload)` (`partial`/`final`/`error`).
- **Yayın (claude.ai) sürümü / tarayıcı:** `AndroidASR` yok, bu yüzden konuşma alıştırması
  ekranında net bir "yalnızca APK'da, cihaz üzerinde çalışır" açıklaması gösterilir; uygulamanın
  geri kalanı normal çalışır.
- **Eşleştirme:** basit LCS (en uzun ortak alt dizi) ile hedef metindeki her karakterin duyulan
  metinde bulunup bulunmadığı işaretlenir (`matchChars`). Bu bir dizgi karşılaştırmasıdır, ses
  analizi değildir ve öyle sunulmaz.
- **Gizlilik:** Ses kaydı **hiçbir zaman diske yazılmaz veya ağa gönderilmez** — sistemin konuşma
  tanıma servisi ses akışını işler, uygulamaya yalnızca metin döner. Mikrofon yalnızca kullanıcı
  "Söyle" düğmesine basınca açılır; arka plana geçince ya da ekran değişince `cancel()` ile kapanır.
  İlk kullanımda Türkçe gerekçeli bir izin açıklaması gösterilir.
- **Hata/izin durumları:** ASR yok / cihaz desteklemiyor / izin reddedildi / Çince paketi eksik /
  konuşma algılanmadı — hepsi ayrı, anlaşılır Türkçe mesajlarla gösterilir; hiçbiri diğer
  bölümleri (kelime/gramer alıştırmaları, sınav, kart, senaryo oynatma) etkilemez.
- **İzinler/manifest:** yalnızca `RECORD_AUDIO`. **İnternet izni kaldırıldı** — uygulama artık uçtan
  uca çevrimdışıdır (TTS de zaten cihazın kurulu Çince sesini kullanıyordu).

## Test
`tests/asr.test.js` — Chromium (Playwright), sahte `AndroidASR` ile: 2 aşamalı sınavın 3. aşama
şartı olmadan tamamlanması, ASR yokken/izin reddinde diğer bölümlerin çalışması, "duyulan metin"
ekranında puan/yüzde **gösterilmediğinin** doğrulanması, hata kodlarının Türkçe açıklanması,
mikrofonun yalnızca kullanıcı eylemiyle açılıp arka planda kapanması, 115 sahnenin hepsinde
10+10 görev üretimi. **20/20 geçti.** Gerçek Android cihazda gerçek `SpeechRecognizer` ile
**denenmedi**; ilk kurulumda telefon Çince çevrimdışı paketi indirmemiş olabilir (uygulama bu
durumu algılayıp yönlendirme mesajı gösterir, ama gerçek cihazda doğrulanmadı).
