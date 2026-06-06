> 📝 _Originally published on Medium: [Account vs. Ledger — Core Banking’in Temelleri](https://medium.com/@gungor.akbiyik/account-vs-ledger-core-bankingin-temelleri-7958fa4b87b8)_


# Account vs Ledger — Core Banking'in Temelleri

Bir müşteri "bakiyem ne?" diye sorduğunda, banka bu cevabı nereden alıyor?

Birkaç ay önce buna "accounts tablosundaki balance kolonundan" derdim. Yanlış değil aslında — ama o cevabın altında çok daha ilginç bir mühendislik kararı yatıyor.

Bu yazı, bir Java developer'ın banking domain'ine girişinin notları. 19 yılda farklı sektörler gördüm. Şimdi core banking öğreniyorum ve her gün bir şeyin neden bu kadar karmaşık olmak zorunda olduğunu anlıyorum.

---

## "Bakiye" Bir Veri Değil, Bir Hesaplama

Klasik bir tabloda bakiye şöyle durur:

```sql
accounts
  id          UUID
  customer_id UUID
  type        VARCHAR  -- CHECKING, SAVINGS, CREDIT, LOAN
  currency    VARCHAR  -- ISO 4217
  status      VARCHAR  -- ACTIVE, FROZEN, CLOSED
  balance     DECIMAL  -- ← bu
```

Ve sezgi şu: para girince `balance + 100`, para çıkınca `balance - 100`. Basit, anlaşılır.

Banking'e girince ilk öğrendiğim şey bu sezginin bir tuzak olduğu.

`balance` kolonu bir **projection** — büyük defterdeki (ledger) tüm hareketlerin özeti, önbelleklenmiş bir türetilmiş değer. Gerçek kayıt ledger'da. `balance` orada sadece hızlı erişim için duruyor.

Muhasebe dünyasında bunun kökü çok daha eskiye gider. Luca Pacioli 1494'te çift kayıt muhasebesini kodladığında temel fikir şuydu: kasadaki nakit yanlışsa, kasayı değiştirmezsin. Deftere düzeltici kayıt eklersin. Kasa sonunda deftere uyar.

Yazılımda da aynı mantık. **Ledger gerçektir. Balance bir önbellektir.**

---

## Drift: İki Kaynak Ayrışırsa Ne Olur?

Bunu somutlaştırayım.

Önceki bir projemde şöyle bir yapı vardı: her işlemde hem `accounts.balance` güncelleniyordu hem de `account_transactions` tablosuna kayıt ekleniyordu. İki ayrı yazma, aynı transaction içinde.

Bir süre sonra gerçek bir sorun çıktı: aynı istek iki kez geldi (network retry, client-side bug — tam sebebi bilinmiyor). Idempotency yoktu. Para iki kez düştü.

Sonradan `idempotency_key` kolonu eklendi, işlem öncesi kontrol kondu. Yamandı. Çalıştı.

Banking gözlüğüyle bu hikayeye bakınca 4 kırılma noktası görüyorum:

1. `balance` kolonu ile işlem kayıtları **drift edebilir** — birisi güncellenip diğeri güncellenmeyebilir. Hangisi doğru?
2. Idempotency'yi sonradan eklemek sızıntıyı yamalamak gibi — asıl mimarinin tasarımında olması gerekir.
3. Çift kayıt (double-entry) yoksa, para nereden gelip nereye gittiği takip edilemiyor.
4. İç hesaplar (kasa, gelir, suspense) yoksa, bankacılık anlamında tam bir tablo çıkmıyor.

**Drift** — iki kaynak arasındaki bu sapma — banking'in en büyük düşmanı. Ledger-as-truth yaklaşımı bunu yapısal olarak kapatır: balance kolonu ya hiç olmaz, ya da açıkça yazma korumalıdır.

---

## Account ile Ledger: Farklı Sorumluluklar

İkisini aynı kavram gibi düşünmek kolay. Ama sorumlulukları tamamen farklı.

**Account** müşteriyle ilgili her şeyi bilir. Kim açtı, ne türü, hangi para birimi, şu an aktif mi. Bir state machine — OPENED'dan ACTIVE'e, oradan FROZEN veya CLOSED'a geçer. Balance'ı bilmez, bilmesi gerekmez.

**Ledger** ise her para hareketinin değişmez kaydıdır. Append-only çalışır — hiçbir kayıt silinmez, değiştirilmez. Hata varsa düzeltici kayıt eklenir. Ve her transaction'da tek bir demir kural vardır: toplam debit = toplam credit.

Sorumluluk matrisi:

| Soru | Cevap |
|------|-------|
| Hesap aktif mi? | Account |
| Bakiye ne? | Ledger (projection) |
| Transfer dengeli mi? | Ledger (yazma anında) |
| Hesap kapatılabilir mi? | Account (Ledger'a sıfır bakiye sorar) |
| Audit geçmişi? | Ledger |

Bunların karışmaması DDD'nin temel ilkelerinden biri. Account, debit/credit mekaniğini bilmez. Ledger, müşteri lifecycle'ını bilmez. Temiz ayrım.

---

## "Balance Kolonu Yazma Korumalı" — Bu Pratikte Ne Demek?

İki tasarım seçeneği var:

**Seçenek A:** Balance kolonu hiç olmaz. Bakiye lazım olunca `SUM(ledger_entries)` sorgulanır. Pedagojik olarak temiz — "balance is a projection" sloganı boşta kalmaz.

**Seçenek B:** Balance kolonu var ama yazma korumalı. CQRS okuma yolu — sadece projection servis yazabilir.

İkinci seçenekte yazma korumasının 5 kademesi var, gevşekten katıya:

1. **Convention + code review** — "bu kolona dokunma" kuralı. Kırılabilir.
2. **ArchUnit testi** — Java'da mimari kurallarını test olarak yazmanı sağlayan açık kaynak bir kütüphane. Normal JUnit testi gibi yazılır, CI'da çalışır. "Domain paketinin dışındaki hiçbir sınıf `balance` alanına yazmasın" gibi bir kural koyabilirsin. Kural ihlal edilirse build kırılır. Yazılı olmayan kuraldan farkı: makine kontrol ediyor, insan değil.
3. **JPA `@Column(updatable = false)`** — ORM katmanından koru.
4. **DB GRANT/REVOKE** — iki ayrı veritabanı kullanıcısı: `app_user` (balance'a UPDATE yetkisi yok), `projection_user` (UPDATE yetkili). Spring tarafında iki ayrı `DataSource`.
5. **Materialized view** — en katı. balance kolonu ayrı bir view, async refresh.

Bunu öğrendiğimde en etkileyici bulduğum 4. kademe oldu. Veritabanı seviyesinde yetki kısıtlaması — uygulama katmanındaki hata ne olursa olsun, `app_user` o kolona yazamaz. Sızdırmaz.

Bu konuyu ileride ayrı bir yazıyla derinlemesine anlatmak istiyorum (PostgreSQL column-level GRANT, Spring DataSource router, üretim kullanım rehberi). Şimdilik şunu söyleyeyim: bugüne kadar üzerinde çalıştığım sistemlerin çoğunda bu kademelerden hiçbiri yoktu — ekibin yazılı olmayan kuralına güveniliyordu ("balance kolonuna kimse el atmasın"). Bu kural genellikle yeni gelen birinin ya da acil bir bug fix'inin altında çatlıyor. Banking'de buna yetinilmiyor; tek bir hatanın bedeli yüksek.

---

## Direction vs Transaction Type: İki Ayrı Kavram

Bir işlemin "türü" ile muhasebedeki "yönü" farklı şeyler.

**Transaction Type** iş anlamı taşır: TRANSFER, DEPOSIT, WITHDRAWAL, FEE, INTEREST, REVERSAL, REFUND...

**Direction** ise muhasebe yönüdür: sadece DEBIT veya CREDIT. Başka değer yoktur.

Bu ayrımı kaçırırsanız tablo yapısı şöyle görünebilir:

```sql
-- Yaygın ama eksik tasarım
account_transactions
  id               UUID
  account_id       UUID
  transaction_type VARCHAR  -- TRANSFER, FEE, vb.
  amount           DECIMAL  -- pozitif veya negatif?
```

Doğrusu şöyle:

```sql
-- İki ayrı tablo, iki ayrı sorumluluk
transactions
  id              UUID
  type            VARCHAR   -- TRANSFER, DEPOSIT, FEE...
  idempotency_key VARCHAR
  status          VARCHAR
  created_at      TIMESTAMP

ledger_entries
  id             UUID
  transaction_id UUID       -- hangi transaction'a ait
  account_id     UUID
  direction      VARCHAR    -- sadece DEBIT veya CREDIT
  amount         DECIMAL    -- her zaman pozitif
  currency       VARCHAR
  occurred_at    TIMESTAMP
```

Neden bu ayrım önemli? Bir REVERSAL işleminin tipi yine REVERSAL, ama direction tam tersi. Bir komisyon işlemi (banking dilinde FEE — fee = ücret/komisyon) hem kesilir (müşteriden çıkar) hem iade edilebilir (müşteriye geri gider) — aynı tip, ters yönler. Type ve direction birbirine karıştırılınca bu senaryolar kodlanamıyor.

---

## Double-Entry Invariant: "İki Kayıt" Değil "Dengeli Kayıt Kümesi"

Çift kayıt muhasebesinin çok bilinen bir yanlış anlaşılması var: "her işlemde tam 2 kayıt olur."

Hayır. Kural şu: **her transaction'da toplam debit = toplam credit**. Entry sayısı 2, 3, 5 olabilir. Sayı değil, denge önemli.

Üç örnek:

**Basit transfer (2 entry):**
```
transaction: TXN-001, müşteri A → müşteri B, 100 TL

DEBIT  customer_A   100 TL    ← A'nın liability'si azaldı (banka ona daha az borçlu)
CREDIT customer_B   100 TL    ← B'nin liability'si arttı (banka ona daha fazla borçlu)

Σ DR = 100, Σ CR = 100 ✓
```

**Komisyonlu transfer (3 entry):**
```
transaction: TXN-002, müşteri A → müşteri B, 100 TL + 2 TL komisyon

DEBIT  customer_A   102 TL    ← A toplam 102 TL verdi
CREDIT customer_B   100 TL    ← B 100 TL aldı
CREDIT fee_income     2 TL    ← banka 2 TL komisyon kazandı

Σ DR = 102, Σ CR = 102 ✓
```

**FX transfer (4 entry):**
```
transaction: TXN-003, müşteri A TL → müşteri B USD

DEBIT  customer_A_TL   4500 TL
CREDIT nostro_USD_TL   4500 TL   ← TL tarafı kapandı

DEBIT  nostro_USD_USD    100 USD
CREDIT customer_B_USD    100 USD   ← USD tarafı kapandı

TL invariant: Σ DR = Σ CR = 4500 ✓
USD invariant: Σ DR = Σ CR = 100  ✓
(varsayılan kur: 1 USD = 45 TL)
```

Son örnekteki kritik nokta: **per-currency invariant**. 4500 TL ile 100 USD aynı kefeye konmaz. Her para birimi için ayrı denge kuralı geçerli.

DDD önden gönderme: `Money(amount, currency)` — currency olmadan amount anlamsız, bu iki alan bir bütün. Value Object tasarımının neden bu şekilde olduğunu bu örnekten anlayabilirsiniz.

---

## Banka İç Hesapları: Müşterilerin Hiç Görmediği Taraf

Ledger'daki hesapların hepsi müşterilere ait değil. Bankanın kendi işleyişini takip ettiği iç hesaplar da tam aynı mekanizmayı kullanıyor.

| Hesap                | Türü            | Ne İşe Yarar                                                   |
| -------------------- | --------------- | -------------------------------------------------------------- |
| **Cash Vault**       | Asset           | Banka kasasındaki fiziksel nakit                               |
| **Suspense**         | Liability       | Hedefi belirsiz, sınıflandırılamamış para. Hedef: sıfır bakiye |
| **Settlement**       | Asset/Liability | Bankalar arası clearing sürerken bekleyen para                 |
| **Fee Income**       | Income          | Komisyon gelirleri                                             |
| **Interest Expense** | Expense         | Mevduat sahiplerine ödenen faiz                                |
| **Nostro**           | Asset           | Bizim başka bankada tuttuğumuz para                            |
| **Vostro**           | Liability       | Başka bankanın bizde tuttuğu para                              |

**Suspense** özellikle ilginç. Bankalar arası wire transfer geldi, ama hedef hesap bulunamadı. Para nereye gidecek? Silindi mi? Havada mı kaldı? Hayır — suspense'e gider. Double-entry'yi bozmuyor, sadece çözüme kadar bekliyor. Sonra hedef netleşince oradan gerçek hesaba taşınır.

**Settlement ve Clearing** farkını da burada açıklayayım:
- **Clearing:** Bankalar birbirlerine gün içinde birçok transfer gönderdi. Bunları teker teker ödemek yerine netleştiriyorlar. A bankası B'ye 1000 TL gönderdi, B de A'ya 800 TL gönderdi — net 200 TL kalıyor.
- **Settlement:** O 200 TL net pozisyonun merkez bankası rezervleri üzerinden fiilen el değiştirmesi.

Clearing hesaplama, settlement gerçekleşme.

Bu iç hesaplar kodda nasıl modellenecek? Muhtemelen `Account` aggregate'inin bir `kind` enum field'ıyla: `CUSTOMER`, `CASH_VAULT`, `SUSPENSE`, `SETTLEMENT`, `FEE_INCOME`, `INTEREST_EXPENSE`. Ayrı aggregate değil — aynı mekanizma, farklı classification. Bu karar Hafta 1 koduna girerken ADR olarak yazılacak.

---

## DEBIT/CREDIT Perspektif Tuzağı

Muhasebeye yeni giren çoğu yazılımcının kafasını en çok karıştıran kısım burası. Çünkü aynı kelime — DEBIT veya CREDIT — iki farklı bağlamda iki farklı şey ifade ediyor gibi görünüyor.

İki bakış açısı var ve ikisini de anlamak gerekiyor.

### Müşteri perspektifi

Müşterinin gözüyle çok basit: "Hesabıma para girdi mi yoksa hesabımdan çıktı mı?" Banka ekstresinde gördüğümüz şey bu açıdan yazılıyor:

- Hesabıma giriş → **CREDIT** (alacaklanma)
- Hesabımdan çıkış → **DEBIT** (borçlanma)

Müşteri için hesap tek bir şey, bir kova. Para giriyor veya çıkıyor.

### Banka perspektifi

Ledger'da olay daha katmanlı. Çünkü banka için müşterinin vadesiz hesabı **bir yükümlülük** (liability) — müşteri istediğinde geri vermek zorunda olduğu para. Banka açısından hesabın "büyümesi" yükümlülüğün artması demek.

Burada DEBIT/CREDIT artık "para girdi/çıktı" değil; **hesap türünün muhasebedeki yönü**:

- Liability arttı → CREDIT
- Liability azaldı → DEBIT
- Asset arttı → DEBIT
- Asset azaldı → CREDIT

### Aynı işlem, iki ayrı anlatım — bir örnek

Müşteri Ali bankaya 500 TL yatırıyor (ATM'den nakit).

**Müşterinin gördüğü (ekstre):**
```
ALİ'NİN HESABI
İşlem      | Tür    | Tutar
NAKİT YATIRMA | CREDIT | +500 TL
```

Tek satır, basit.

**Bankanın ledger'ı:**
```
DEBIT  cash_vault       500 TL   ← banka kasası (asset) arttı
CREDIT customer_ali     500 TL   ← müşteri yükümlülüğü (liability) arttı
```

İki satır. Çünkü banka, parayı sadece müşteri hesabında değil, kendi kasasında da takip etmek zorunda. Müşteri ekstrede "kasa"yı hiç görmez.

Dikkat: Müşterinin ekstresinde **CREDIT**, bankanın ledger'ında müşteri tarafı yine **CREDIT**. İki bakış aslında aynı yere çıkıyor — sözcük "müşteri lehine artma"yı anlatıyor.

### Karışıklığın çıktığı an — ATM'den para çekme

Aynı Ali, başka bir gün ATM'den 200 TL çekiyor.

**Müşterinin gördüğü:**
```
ALİ'NİN HESABI
İşlem        | Tür   | Tutar
ATM ÇEKME    | DEBIT | -200 TL
```

**Bankanın ledger'ı:**
```
DEBIT  customer_ali     200 TL   ← müşteri yükümlülüğü (liability) azaldı
CREDIT cash_vault       200 TL   ← banka kasası (asset) azaldı
```

İşte burada "her şey CREDIT olmalı değil mi, kasada da para azaldı" gibi sezgi devreye girer. Cevap hayır:

- Müşteri yükümlülüğü (liability) — azaldı → DEBIT
- Banka kasası (asset) — azaldı → CREDIT

Aynı yön (azalma), iki farklı işaret. Çünkü hesap türleri farklı.

Müşteri ekstrede tek bir satır "DEBIT -200" görüyor; bankanın ledger'ında iki entry var. İki perspektif birbirinden bağımsız çalışıyor, ama matematik tutuyor.

### Kural: Ledger banka perspektifini kullanır, UI müşteri perspektifine çevirir

Kodda asla iki perspektifi karıştırmamak lazım. Ledger her zaman banka perspektifini kullanır. Müşteriye ekstre çıkarken bir adapter/DTO katmanı bu perspektifi müşteri açısına çevirir.

Yani: hesap müşteriye ait bir liability ise, ekstrede CREDIT = hesapta artış, DEBIT = hesapta azalış. Bu çeviri **sunum katmanında** yapılır, domain logic değildir.

### Hangi yönü nasıl bulurum? — Üç adımlı yöntem

Bir ledger entry yazarken kafam karıştığında bu üç soruyu sırayla soruyorum:

1. **Bu hesap ne türü?** (Asset / Liability / Income / Expense / Equity)
2. **Bakiye arttı mı azaldı mı?**
3. **Tablodan DR/CR'yi oku:**

| Tür | DR etkisi | CR etkisi |
|-----|-----------|-----------|
| Asset | Artar | Azalır |
| Liability | Azalır | Artar |
| Income | Azalır | Artar |
| Expense | Artar | Azalır |
| Equity | Azalır | Artar |

Yukarıdaki 200 TL ATM çekme örneği için:
- `customer_ali` → Liability, bakiye azaldı → **DEBIT**
- `cash_vault` → Asset, bakiye azaldı → **CREDIT**

Adım adım sezgi tutuyor.

### Klasik iki tuzak

**1. "Credit Card" tuzağı.** Kredi kartının adında "credit" geçiyor ama bu muhasebenin CR'si değil — ürünün ismi. Kart harcaması yaptığında müşteri açısından **borç** oluşuyor; ledger'da müşterinin kredi kartı hesabı (asset bankaya göre — banka müşteriden alacaklı) DEBIT'leniyor.

**2. Türkçe "borç" tuzağı.** Türkçe muhasebede Debit'in karşılığı **"Borç"**, Credit'in karşılığı **"Alacak"**. Ama bu, günlük dildeki "borçluyum" anlamı değil — T hesabının sol tarafına verilen isim. "Hesabın borç tarafı" demek "yükümlülük altındasın" demek değil.

İki dil aynı kelimeyi farklı bağlamda kullanıyor; muhasebeci ile yazılımcı aynı masaya oturduğunda buralarda kıvılcım çıkıyor.

---

## Öğrenirken Ne Fark Ettim

Banking domain'i öğrenmek ilk haftada "muhasebe öğreniyorum" gibi hissettiriyor. Ama aslında dağıtık sistemler, event sourcing, CQRS ve idempotency problemlerinin bir muhasebe sözlüğüyle yeniden adlandırılmış hali.

Ledger append-only → event log.
Balance projection → read model.
Reversing entry → compensating transaction.
Double-entry invariant → two-phase consistency guarantee.

Aslında hiçbir kavram yabancı değil. Sadece isimler farklı.

---

## Bundan Sonra Ne Yapacağım

Bu yazı banking domain'inin temellerini koydu. Sıradaki yazı Domain-Driven Design'ın taktiksel kalıplarına giriyor — Entity, Value Object, Aggregate, Repository, Domain Service, Application Service. DDD'yi havada anlatmak yerine **bu yazıdaki banking kavramlarını DDD'nin diliyle yeniden modelleyeceğim**: `Account` aggregate, `Money` value object, `MoneyTransfer` domain service, `LedgerEntry` ve aralarındaki sınırlar.

İkisini bilinçli olarak birlikte yürütüyorum. Önce domain (burada), sonra DDD (sonraki yazı), ardından kod — Java + Spring + JPA ile OpenCore Bank açık kaynak projesi olarak. Domain anlaşılmadan DDD ezberi olur; DDD'siz domain ise sadece kavram listesi kalır.

Eğer DDD yazısına geldiyseniz ve banking terminolojisi (asset, liability, ledger, double-entry, nostro) tanıdık değilse, bu yazıyı bir altlık olarak okumanızı öneririm — DDD makalesinde sık sık buraya gönderme yapılacak. Banking'i zaten biliyorsanız, doğrudan DDD yazısına geçebilirsiniz.

Kodlama aşaması da birkaç hafta sonra geliyor. Bu iki yazının kavramları somut Java sınıflarına, repository'lere, aggregate'lere dönüşecek; her sınıfı "bu hangi banking kavramı + hangi DDD kalıbı" şeklinde işaretleyeceğim. Domain → DDD → kod akışı bu projenin omurgası olacak.

---

## Terimler Sözlüğü

Bu yazıda geçen terimlerin kısa tanımları. Tam sözlük projenin glossary dosyasında büyümeye devam ediyor.

**Debit (DR) — Borç:** Hesabın sol tarafı. Etkisi hesap türüne göre değişir — asset ve expense'i artırır; liability, equity, income'u azaltır.

**Credit (CR) — Alacak:** Hesabın sağ tarafı. Liability, equity, income'u artırır; asset ve expense'i azaltır.

**Asset — Varlık:** Bankaya ait ekonomik değer. Kasa, verilen krediler, nostro hesapları.

**Liability — Yükümlülük:** Bankanın geri ödeme borcu. Müşteri vadesiz hesabı, mevduat — banka bunları geri ödemek zorunda.

**Double-Entry Bookkeeping — Çift Kayıt Muhasebesi:** Her ekonomik olay en az iki hesaba yansır; toplam DR = toplam CR. 1494'ten beri değişmeden kullanılıyor.

**Ledger — Büyük Defter:** Para hareketlerinin değişmez kaydı. Append-only, hiçbir kayıt silinmez veya değiştirilmez.

**Projection — Türetilmiş Görünüm:** Ham veriden hesaplanan özet. Balance, ledger entry'lerden türetilen bir projection'dır.

**Drift — Sapma:** İki kaynak arasındaki tutarsızlık. Balance kolonu ile ledger toplamının farklılaşması.

**Reversing Entry — Düzeltici Kayıt:** Hatayı silmek yerine karşı yönlü yeni bir kayıt eklemek. Append-only prensibinin uygulaması.

**Cash Vault — Kasa:** Bankanın fiziksel nakit envanteri. Tür: Asset.

**Suspense Account — Askıdaki Hesap:** Hedefi belirsiz işlemlerin geçici tutulduğu hesap. Hedef sıfır bakiye.

**Clearing — Netleştirme:** Bankalar arası transferlerin anlık değil, toplu netleştirme yöntemiyle gerçekleştirilmesi.

**Settlement — Gerçekleşme:** Netleşen tutarın merkez bankası rezervleri üzerinden fiilen el değiştirmesi.

**Nostro:** Banka A'nın Banka B'de tuttuğu hesap. A'nın bilançosunda Asset.

**Vostro:** Aynı hesap, B'nin perspektifinden. B'nin bilançosunda Liability.

**Idempotency:** Aynı isteğin birden çok kez gönderilmesinde yan etkinin bir kez gerçekleşmesi. Banking'de `idempotency_key` ile uygulanır.

**Aggregate:** DDD kavramı. Birlikte tutarlı kalması gereken nesnelerin tek kapıdan (root) erişilen kümesi.

**Value Object (VO):** Kimlik taşımayan, değerlerinden tanımlanan, immutable nesne. Örnek: `Money(amount, currency)`.

---

## Kaynaklar

- [Griffin Bank — The Immutable Bank](https://griffin.com/blog/the-immutable-bank)
- [Modern Treasury — Enforcing Immutability in Your Double-Entry Ledger](https://www.moderntreasury.com/journal/enforcing-immutability-in-your-double-entry-ledger)
- Pat Helland — "Immutability Changes Everything" (CIDR 2015): https://www.cidrdb.org/cidr2015/Papers/CIDR15_Paper16.pdf — "Accountants don't use erasers" cümlesi bu makaleden
- [Wikipedia — Double-entry bookkeeping](https://en.wikipedia.org/wiki/Double-entry_bookkeeping)
- [Wikipedia — Nostro and Vostro accounts](https://en.wikipedia.org/wiki/Nostro_and_vostro_accounts)

---

*#java #spring #ddd #banking #corebanking #softwareengineering*
