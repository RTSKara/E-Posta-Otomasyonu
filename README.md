# E‑Posta – n8n Otomatik E‑posta Etiketleme ve Sınıflandırma İş Akışı

Bu repo; **Gmail’e düşen e‑postaları** otomatik olarak dinleyen, **OpenAI** ile içeriği analiz edip **kategoriye ayıran** ve Gmail’de ilgili **etiketleri** uygulayan bir **n8n iş akışını (workflow)** içerir.

> Kısa özet: `Gmail Trigger` → `Gönderici Adı Çıkar` → `Selamlama` → `Metin Sınıflandırma` → **(Links | Incoming codes | Payments | Others)** → **Gmail’e etiket ekle**

---

## 🚀 Özellikler

- **Gmail Trigger**: Gelen e‑postaları her dakikada bir tarar.
- **Bilgi Çıkarımı (LangChain Information Extractor)**: E‑posta başlığından **gönderici adını** çeker.
- **Akıllı Selamlama**: İsim varsa “Merhaba *İsim*”, yoksa genel bir “Merhaba” mesajı hazırlar.
- **Metin Sınıflandırma (LangChain Text Classifier)**: E‑posta gövdesini şu kategorilere ayırır:
  - **Links** – bağlantı içeren e‑postalar
  - **Incoming codes** – doğrulama/şifre sıfırlama kodları
  - **Payments** – ödeme/ücret soruları veya makbuzlar
  - **Others** – yukarıdakilere girmeyenler
- **Gmail Etiketleme**: Kategoriye göre uygun **Gmail label**’ı ekler.

---

## 🧰 Gereksinimler

- Çalışır durumda bir **n8n** örneği (Docker, Desktop, Sunucu vb.)
- **Gmail** erişimi için n8n’de ayarlanmış Google kimlik bilgisi (OAuth)
- **OpenAI API Key** (n8n Credentials → *OpenAI API*)
- n8n içinde **LangChain düğümleri**: `@n8n/n8n-nodes-langchain` (n8n Cloud/masaüstü sürümlerde hazır gelir; gerekirse Güncellemeler/NPM paketleri ile etkinleştirin)
- Gmail tarafında kullanacağınız **etiketlerin oluşturulmuş olması** (ID’lerini n8n düğümlerine gireceksiniz)

---

## 📦 Kurulum

1. Bu repodaki iş akışını JSON olarak indirin: **`e-posta-workflow.json`**  
   → [İndir](sandbox:/mnt/data/e-posta-workflow.json)
2. n8n arayüzünde **Workflows → Import from file** deyip bu JSON’u içe aktarın.
3. **Credentials** atayın:
   - `Gmail Trigger` ve `Gmail` düğümleri için **Google (Gmail)** hesabınızı yetkilendirin.
   - `OpenAI Chat Model` düğümleri için **OpenAI API** kimliğinizi seçin.
4. **Etiket ID’lerini** güncelleyin:  
   Aşağıdaki düğümlerde kendi Gmail etiket **ID**’lerinizi girin:
   - `Lİnkler` → `labelIds: ["Label_…"]`
   - `Kodlar` → `labelIds: ["Label_…"]`
   - `Ödemeler` → `labelIds: ["Label_…"]`
   - `Diğerleri` → `labelIds: ["Label_…"]`

> **İpucu:** Gmail etiket ID’lerini görmek için n8n’de `Gmail` node’unu **List → Labels** modunda çalıştırabilir veya Gmail API referansını kullanabilirsiniz. İsterseniz etiket adlarıyla eşleştirecek bir adım ekleyebilirsiniz.

---

## ⚙️ Yapı ve Akış

```
Gmail Trigger (every minute)
        │
        ▼
Information Extractor (senderName)
        │
        ▼
If (senderName not empty?) ─────► İsim Varsa (introduction = senderName)
        │                          │
        └──────────────► İsim Yoksa (introduction = "Merhaba")
        │
        ▼
Notlar (emailbody, messageId, threadId, introduction)
        │
        ▼
Mail Düzenleyici (Text Classifier: Links | Incoming codes | Payments | Others)
        │
        ├──► Lİnkler    (Gmail: add label to messageId)
        ├──► Kodlar     (Gmail: add label to messageId)
        ├──► Ödemeler   (Gmail: add label to messageId)
        └──► Diğerleri  (Gmail: add label to messageId)
```

### Önemli Düğümler
- **Gmail Trigger**: `pollTimes = everyMinute`
- **Information Extractor**: `text = {{$json.from.value[0].name}}` → `senderName`
- **If**: `senderName notEmpty`
- **İsim Varsa/İsim Yoksa (Set)**: `introduction` değeri atanır  
  (Not: “Mehaba” yazımı **“Merhaba”** olarak düzeltmenizi öneririz.)
- **Mail Düzenleyici (Text Classifier)**: Kategorileri JSON olarak döndürür.
- **Gmail (Add Labels)**: `messageId` üzerine ilgili **labelId** eklenir.

---

## 🧪 Test & Doğrulama

1. Gmail’inize dört tipe uygun örnek e‑posta gönderin:
   - Bir e‑posta **link** içersin.
   - Bir e‑posta **doğrulama kodu** içersin.
   - Bir e‑posta **ödeme/ücret** içeriği taşısın.
   - Bir e‑posta **diğer** genel içerik olsun.
2. n8n’de workflow’u **Activate** edin; **Executions** ekranından akışı izleyin.
3. Gmail’de her e‑postanın **doğru etiket** ile işaretlendiğini kontrol edin.

---

## 🧩 Özelleştirme Önerileri

- **Kategori setini genişletin** (ör. *Faturalar*, *İş Başvuruları*, *Destek Talepleri*).  
- **Yanıt taslağı üretin**: Kategoriye göre otomatik **reply draft** oluşturup `Gmail → Send` node’u ekleyin.
- **Spam/önemsiz** filtre adımı ekleyin (ör. *Bayes + anahtar kelime listesi*).
- **Çok dilli sınıflandırma**: `systemPromptTemplate` içinde TR/EN yönergeler.
- **Rate limit** ve **hata yönetimi**: `Error Trigger` + `Retry` politikaları.

---

## 🔐 Güvenlik

- OpenAI ve Gmail kimlik bilgilerini **Credentials** ekranında güvenli saklayın.
- **Kişisel veri** içeren e‑postaları 3. taraflara gönderirken gizlilik kurallarına uyun.
- Gerekirse sınıflandırma için **yerel model** (LLM) seçeneklerini değerlendirin.

---

## 🛠 Sorun Giderme

- **Etiket uygulanmıyor**: Doğru **labelId** girildi mi? `messageId` doğru mu set ediliyor?
- **Trigger çalışmıyor**: n8n örneği aktif mi, credential izinleri doğru mu, `pollTimes` açık mı?
- **Sınıflandırma hatalı**: System prompt’u netleştirin, örnekler (few‑shot) ekleyin, model sürümünü güncelleyin.
- **Türkçe karakter/encoding**: Node’lar arası veri geçişinde `UTF‑8` olarak koruyun.

---

## 📄 Lisans

Bu iş akışı örneği **MIT** lisansı ile paylaşılabilir. Kurumsal kullanımda gizlilik ve güvenlik politikalarınızı dikkate alın.
