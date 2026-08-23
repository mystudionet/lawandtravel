# Law and Travel — Proje Devir Notları

**Site:** https://lawandtravel.com
**Geçici adres:** https://lawandtravel.turkishtv.workers.dev
**Durum:** Yayında ve çalışıyor
**Son güncelleme:** 23 Ağustos 2026

---

## 1. Proje Nedir

Amerikalı emniyet mensuplarına yönelik 11 günlük Türkiye turu için tek sayfalık tanıtım sitesi.
Voyarelle sitesindeki gibi **scroll ile oynayan tek parça video** mantığı üzerine kurulu.

**Rota:** Istanbul → Cappadocia → Antalya
**Tarih seçenekleri:** May 15–25 / June 4–14 / July 10–20 (2027)
**Grup limiti:** 24 kişi

---

## 2. Dosya Yapısı

```
lawandtravel/
├── index.html          # Ana sayfa (412 KB)
└── itinerary.html      # 11 günlük detaylı program (229 KB)
```

**Video sitede DEĞİL.** Cloudflare R2'de duruyor (aşağıda açıklandı).

Görseller `index.html` içine base64 olarak gömülü, ayrı dosya yok.

---

## 3. Teknik Yapı

### Scroll-Scrub Video Sistemi

Sayfanın kalbi bu. Video sabit (`position:fixed`) arka planda duruyor, üstünde 6 adet
şeffaf bölüm var. Scroll pozisyonu videonun `currentTime` değerine bağlanmış.

```js
var progress = -rect.top / (rect.height - vh);   // 0 ile 1 arası
video.currentTime = progress * video.duration;   // videoyu o ana sar
```

Aşağı scroll → video ileri sarar. Yukarı scroll → geri sarar.

### Bölüm Yükseklikleri (KRİTİK)

Her bölümün yüksekliği, videodaki o sahnenin gerçek süresiyle orantılı olmak zorunda.
Yoksa menü etiketi ile ekrandaki görüntü birbirini tutmaz.

```css
#beat-packing    { height: 150vh }   /* 0.0  - 6.5 sn  */
#beat-airport    { height:  83vh }   /* 6.5  - 10.5 sn */
#beat-flight     { height:  93vh }   /* 10.5 - 13.5 sn */
#beat-istanbul   { height: 108vh }   /* 13.5 - 17.5 sn */
#beat-cappadocia { height:  93vh }   /* 17.5 - 21.0 sn */
#beat-antalya    { height: 505vh }   /* 21.0 - 41.2 sn */
```

Videoyu değiştirirsen bu değerleri yeniden hesaplaman gerekir.

### Tasarım

| Öğe | Değer |
|---|---|
| Mavi (butonlar) | `#1A6BFF` |
| Koyu lacivert | `#0A1628` |
| Altın (vurgu) | `#C9A84C` |
| Metin | Siyah `#0B0D10` + beyaz hale |
| Başlık fontu | Unbounded 900 |
| Gövde fontu | Onest |
| Etiket fontu | JetBrains Mono |

Yazılar açık renkli video üzerinde okunabilsin diye siyah, arkalarında beyaz radial
gradient + `text-shadow` hale var.

Menü sağ üstte yatay, aktif bölüm altı çizili olarak takip ediliyor.

---

## 4. Video — ÖNEMLİ

### Neden R2'de?

Video önce Cloudflare Workers'a site dosyalarıyla birlikte yüklendi ama **çalışmadı**.
Sebep: Workers static assets **Range isteklerini desteklemiyor** (Status 200 dönüyor,
206 değil). Range olmadan tarayıcı videonun ortasına atlayamıyor, scroll-scrub çalışmıyor.

Çözüm: video Cloudflare R2'ye taşındı. R2 Range destekliyor.

### R2 Bilgileri

```
Bucket:      lawandtravel-video
Public URL:  https://pub-b2ba344316494c6abc68b274f0dc705f.r2.dev
Video yolu:  /video/journey-cf.mp4
Yedek:       /video/journey-1080.webm
```

**Maliyet:** Ücretsiz. R2 free tier aylık 10 GB depolama + sınırsız bant genişliği
(egress ücreti yok) veriyor. Video 20.6 MB, yani limitin %0.2'si.

### CORS Ayarı (olmazsa çalışmaz)

R2 → bucket → Settings → CORS Policy:

```json
[
  {
    "AllowedOrigins": [
      "https://lawandtravel.com",
      "https://www.lawandtravel.com",
      "https://lawandtravel.turkishtv.workers.dev"
    ],
    "AllowedMethods": ["GET", "HEAD"],
    "AllowedHeaders": ["*"],
    "ExposeHeaders": ["Content-Length", "Content-Range", "Accept-Ranges"],
    "MaxAgeSeconds": 3600
  }
]
```

Yeni bir domain eklersen `AllowedOrigins` listesine onu da ekle.

### Video Encode Ayarları

Orijinal 41 saniyelik master dosyadan üretildi:

```bash
ffmpeg -i master.mov \
  -c:v libx264 -crf 23 -preset slow -pix_fmt yuv420p \
  -g 12 -keyint_min 12 -sc_threshold 0 \
  -an -movflags +faststart journey-cf.mp4
```

**`-g 12` kısmı kritik:** her 0.5 saniyede bir keyframe koyuyor. Normal videoda keyframe
seyrek olur, scroll ederken aradaki kareler bulanık görünür. Yoğun keyframe = her scroll
pozisyonunda net görüntü. Bunu düşürürsen görüntü çamurlaşır.

**Kalite:** 1080p, 4.2 Mbps, 20.6 MB. Master'a göre PSNR 46.5 dB (45 üstü gözle ayırt edilemez).

---

## 5. Form Sistemi

Form gönderilince **iki şey aynı anda** olur:

### 1. Google Forms'a kaydedilir

Gizli iframe üzerinden POST edilir. `fetch` kullanılamaz çünkü Google CORS engelliyor.

```
Form ID: 1FAIpQLSc8_Q_Y35poBD8_dNLTj9m1qZJOWZeo8IO4X_972XAmJg2iuw
Endpoint: https://docs.google.com/forms/d/e/{FORM_ID}/formResponse
```

| Alan | Entry ID |
|---|---|
| Full Name | `entry.1864130971` |
| Email | `entry.484795080` |
| Phone | `entry.1132346057` |
| Group Size | `entry.1626982029` |
| Tour Dates | `entry.1693573793` |

**Değerler birebir eşleşmeli**, yoksa Google reddeder:
- Tarihler: `May 15 to 25`, `June 4 to 14`, `July 10 to 20`
- Grup: `1`, `2`, `3`, `4`, `5+`

### 2. WhatsApp açılır

Numara: `16314332021`

Mesaj şablonu:
```
Hi, I'm interested in your vacation package to Türkiye.
Please call me on the number below.

Dates: {tarih}
Name: {isim}
Email: {email}
Phone: {telefon}
Group size: {kişi}
```

### YAPILACAK

Google Form → **Yanıtlar** sekmesi → sağ üst üç nokta →
**"Yeni yanıtlar için e-posta bildirimleri al"** açılmalı.
Açık değilse yanıtlar sadece tabloda birikir, e-posta gelmez.

---

## 6. Cloudflare Kurulumu

```
Hesap:    Turkishtv@gmail.com
Proje:    lawandtravel  (Workers & Pages)
Domain:   lawandtravel.com + www.lawandtravel.com (ikisi de bağlı)
Registrar: Namecheap (nameserver'lar Cloudflare'e yönlendirildi)
```

**Not:** Proje "Pages" değil, static assets'li **Worker** olarak deploy edildi.
Fark yok, aynı şekilde çalışıyor.

---

## 7. GitHub'a Geçiş (sıradaki adım)

Şu an her değişiklikte dosyayı elle yükleme gerekiyor. GitHub bağlanınca `git push`
yeterli olacak.

### Kurulum

```bash
cd lawandtravel

git init
git add .
git commit -m "initial commit"
git branch -M main
git remote add origin https://github.com/KULLANICI_ADIN/lawandtravel.git
git push -u origin main
```

Sonra: Cloudflare → `lawandtravel` projesi → **Settings** → **Connect to Git** → repo seç.

Build ayarları (statik site olduğu için):
- Build command: **boş bırak**
- Build output directory: **/** (kök)

### Sonraki her güncelleme

```bash
git add .
git commit -m "ne değişti"
git push
```

Cloudflare otomatik deploy eder.

### .gitignore önerisi

```
.DS_Store
*.mov
*.mp4
*.webm
```

Video R2'de olduğu için repo'ya girmesine gerek yok, girerse repo şişer.

---

## 8. Sorun Giderme

### Video oynamıyor, tek karede donuk

1. F12 → Network → `journey-cf.mp4` satırındaki **Status**'a bak
   - **206** = doğru, sorun başka yerde
   - **200** = Range çalışmıyor, video yanlış yerden servis ediliyor
   - **404** = yol yanlış
2. CORS ayarını kontrol et
3. R2 adresini doğrudan tarayıcıda aç, açılıyor mu bak

### Mobilde video oynamıyor

Mobil tarayıcılar veri tasarrufu için videoyu kullanıcı dokunana kadar yüklemiyor.
Çözüm: ilk dokunuşta videoyu sessizce `play()` + `pause()` yapıp "kilidini açmak"
gerekiyor. **Henüz eklenmedi.**

### Menü etiketi ile görüntü uyuşmuyor

Bölüm yükseklikleri (bkz. bölüm 3) videodaki sahne süreleriyle orantılı değil demektir.

### Form Google'a düşmüyor

Değerlerin birebir eşleştiğini kontrol et (`5+` mi `5 or more` mu, tarih yazımı vs).

---

## 9. Yapılacaklar

- [ ] Google Form e-posta bildirimini aç
- [ ] GitHub repo kur, Cloudflare'e bağla
- [ ] Mobil video kilidi açma kodunu ekle
- [ ] Mobilde 20 MB video ağır gelirse 720p versiyon ekle
- [ ] favicon ekle (şu an 404 veriyor, zararsız ama konsolda hata görünüyor)

---

## 10. Notlar

- Görseller HTML içine gömülü olduğu için `index.html` 412 KB. Normal.
- `itinerary.html` ayrı sayfa, ana sayfadan "View the full day-by-day" ile açılıyor.
- Sitede login/üyelik yok, sadece Request formu var. Bilinçli tercih.
- Videonun master dosyası (41 sn, 54 MB) yerel makinede. Yedeğini sakla,
  yeniden encode gerekirse lazım olur.
