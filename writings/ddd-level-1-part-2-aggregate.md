> 📝 _Originally published on Medium: [DDD Level 1 — Bölüm 2: Aggregate ve Aggregate Root](https://medium.com/@gungor.akbiyik/ddd-level-1-b%C3%B6l%C3%BCm-2-aggregate-ve-aggregate-root-b827bca2bb9c)_


# DDD Level 1 — Bölüm 2: Aggregate ve Aggregate Root

> Üç bölümlük serinin ikinci parçası. [1. bölümde](#) Entity ve Value Object'i konuşmuştuk — kimliği olan ve olmayan nesneleri. Bu bölüm bu nesneleri bir araya getiren kavrama bakıyor: Aggregate. Tutarlılık sınırı, root, "Account'un içine `List<Transaction>` koyalım mı" sorusunun cevabı. 3. bölüm Repository, Domain Service ve Application Service ile devam edecek.

Önce sade bir tanım: **Aggregate**, birlikte değişen ve birlikte tutarlı kalması gereken nesnelerin oluşturduğu kümedir. **Aggregate Root** ise bu kümenin dışa açılan tek kapısıdır — kümeye yapılan tüm değişiklikler bu kapıdan geçer.

Bu cümle ilk okunduğunda soyut gelebilir. Anlaşılır olması için önce alıştığımız yapıyı hatırlayalım, sonra DDD'nin neyi değiştirdiğini görelim.

## Eskiden ne yapıyorduk?

Klasik Spring projesinde `Account` şuna benzer:

```java
@Entity
@Getter @Setter
public class Account {
    @Id private UUID id;
    private UUID customerId;
    private String currency;
    private Status status;
    // getter/setter Lombok'tan
}
```

Boş bir veri torbası. JPA entity, sadece field var, davranış yok. Yanına `AccountService` yazarız:

```java
@Service
public class AccountService {
    private final AccountRepository repo;

    public void freezeAccount(UUID accountId) {
        Account account = repo.findById(accountId).orElseThrow();

        if (account.getStatus() == Status.CLOSED) {
            throw new IllegalStateException("Kapalı hesap dondurulamaz");
        }
        if (account.getStatus() == Status.FROZEN) return;

        account.setStatus(Status.FROZEN);
        repo.save(account);
    }

    public void closeAccount(UUID accountId) { /* benzer */ }
    public void activateAccount(UUID accountId) { /* benzer */ }
}
```

Burada her şey alışıldık. Controller `freezeAccount`'u çağırır, service hesabı yükler, validasyonu yapar, status'u set eder, kaydeder.

Sorun şu: "Kapalı hesap dondurulamaz" kuralı `AccountService.freezeAccount`'ın içinde yaşıyor. Yarın bir başka yerden — bir `BatchProcessor`, bir `AdminController`, bir Kafka consumer — hesap dondurmak gerekirse, ya bu service'i çağırırsın ya da aynı kontrolü orada yeniden yazarsın. Birinci yol her zaman mümkün olmayabilir; ikincisi de kuralı koddan koda dağıtır. Üç ay sonra kuralın bir kopyası kontrolünü atlar.

Ayrıca `account.setStatus(Status.FROZEN)` cümlesi her yerden çağrılabilir. Bir junior developer "şu account'u manuel olarak frozen yapayım" diye doğrudan setter'a girebilir. Kural baypas edilir.

Mantık servise yığılmış, nesne savunmasız.

## DDD ile ne değişiyor?

Davranışı nesneye geri taşıyoruz:

```java
public class Account {
    private final AccountId id;
    private final CustomerId customerId;
    private final Currency currency;
    private final AccountKind kind;
    private final AccountClass classification;
    private Status status;

    // status için public setter YOK

    public void freeze() {
        if (status == Status.CLOSED) {
            throw new IllegalStateException("Kapalı hesap dondurulamaz");
        }
        if (status == Status.FROZEN) return;
        this.status = Status.FROZEN;
    }

    public void close() {
        if (status == Status.CLOSED) return;
        this.status = Status.CLOSED;
    }
}
```

İki kritik fark var.

Birincisi: `status` private, **setter yok**. Dışarıdan kimse `account.setStatus(CLOSED)` yazamaz. Status'u değiştirmek isteyen `freeze()` veya `close()` çağırmak zorunda.

İkincisi: "Kapalı hesap dondurulamaz" kuralı `freeze()`'in içine taşındı. Kural artık metodla aynı yerde yaşıyor — birinden bahseden mutlaka diğerini de görüyor.

Service ne hale geliyor?

```java
@Service
public class AccountService {
    private final AccountRepository repo;

    public void freezeAccount(AccountId accountId) {
        Account account = repo.findById(accountId).orElseThrow();
        account.freeze();
        repo.save(account);
    }
}
```

Üç satır. Service "yükle, çağır, kaydet" diyor; karar Account'ın içinde. Yarın `BatchProcessor` da `account.freeze()` çağırır, aynı kural devreye girer. Kural tek noktada yaşıyor — `Account.freeze()`.

## İşte Aggregate Root bu

Bu yeni `Account` bir **Aggregate Root**. Yani:

- İçindeki state'e (`status`) dışarıdan doğrudan erişilemez.
- Tüm değişiklikler `freeze()`, `close()` gibi root metotlarından geçer.
- Service ve diğer kod sadece Account'a referans tutar; Account içeriği yönetir.
- Kurallar root'un içinde yaşar, koddan koda dağılmaz.

Spring developer mantığıyla: aggregate root, eskiden service'in dışarıdan elle yönettiği "veri torbası" nesnenin yerine geçen, kuralları kendi içinde taşıyan, dışa tek arayüzlü bir sınıftır. Service ince bir koordinatör katmanına döner.

## Peki "aggregate küme demek" demiştik, küme nerede?

Şu ana kadar gösterdiğim Account aggregate'inin içinde tek nesne var: kendisi. Yani aggregate = root = aynı nesne. Bu da geçerli bir aggregate (tek nesneli aggregate yaygındır), ama "küme" hissini vermez.

Multi-nesneli bir örnek için Transaction'a bakalım. Ama önce kısa bir domain hatırlatması — Transaction ve LedgerEntry nedir, niye birlikte yaşıyorlar.

### Transaction ve LedgerEntry: kısa bir hatırlatma

Bir para transferi düşün: A hesabından B hesabına 100 TL.

Bu işlem veritabanında iki ayrı tabloda izini bırakır.

**`transactions` tablosu** — işlemin başlığı:

```
id        | type     | idempotency_key | status     | created_at
─────────────────────────────────────────────────────────────────────────
TXN-A1F3  | TRANSFER | idem-9c4e       | COMMITTED  | 2026-05-18 09:42:11
```

**`ledger_entries` tablosu** — işlemin muhasebe satırları:

```
id      | transaction_id | account_id  | direction | amount  | currency
─────────────────────────────────────────────────────────────────────────
LE-1001 | TXN-A1F3       | customer_A  | DEBIT     | 100.00  | TL
LE-1002 | TXN-A1F3       | customer_B  | CREDIT    | 100.00  | TL
```

`ledger_entries` tablosundaki her satır bir **LedgerEntry**: hangi hesap, hangi yön (DEBIT/CREDIT), ne kadar, hangi para birimi. `transaction_id` ile başlığa bağlı.

`transactions` tablosundaki o tek satır da bir **Transaction**: "A'dan B'ye transfer" işleminin kendisi. Transaction "ne yaptık"ı söylüyor, LedgerEntry'ler "muhasebede nasıl göründü"yü. Aynı zamanda idempotency, status, zaman damgası gibi işleme dair üst bilgiler de burada.

Bir transaction birden fazla entry üretebilir. Mesela aynı transferi 2 TL komisyonla yapsaydık:

```
transactions
id        | type     | idempotency_key | status     | created_at
─────────────────────────────────────────────────────────────────────────
TXN-B2C4  | TRANSFER | idem-3f10       | COMMITTED  | 2026-05-18 10:15:33

ledger_entries
id      | transaction_id | account_id  | direction | amount  | currency
─────────────────────────────────────────────────────────────────────────
LE-1003 | TXN-B2C4       | customer_A  | DEBIT     | 102.00  | TL
LE-1004 | TXN-B2C4       | customer_B  | CREDIT    | 100.00  | TL
LE-1005 | TXN-B2C4       | fee_income  | CREDIT    |   2.00  | TL
```

Üç entry, hepsi aynı `TXN-B2C4`'e bağlı. Σ DEBIT = Σ CREDIT = 102, dengeli.

Entry sayısı işin türüne göre değişir:

- Basit transfer: 2 entry
- Komisyonlu transfer: 3 entry
- FX transferi: 4 entry (TL ayağı + USD ayağı)

İlişki şu: bir Transaction → birden fazla LedgerEntry. Entry'ler tek başına yaşamaz; mutlaka bir transaction'a aittirler. Bir entry başka bir transaction'a "taşınamaz" — çünkü o işlemin bir parçasıdır.

Komisyonlu örnekte göründüğü gibi, dengeyi koruyan bir kural var: bir transaction'daki tüm entry'lerin **Σ DEBIT = Σ CREDIT** (her currency için ayrı) olması zorunlu. Yoksa para sıfırdan oluşmuş veya buhar olmuş gibi gözükür. Çift kayıt muhasebesinin temel kuralı bu.

(Bu konunun ayrıntısını [bir önceki yazıda](https://medium.com/@gungor.akbiyik/account-vs-ledger-core-bankingin-temelleri-7958fa4b87b8) konuştuk. Burada özet yetiyor.)

### Transaction'ı aggregate olarak yazalım

```java
public class Transaction {                    // Aggregate Root
    private final TransactionId id;
    private final TransactionType type;
    private Status status;
    private final List<LedgerEntry> entries = new ArrayList<>();

    public void addEntry(AccountId accountId, Direction direction, Money amount) {
        if (status != Status.PENDING) {
            throw new IllegalStateException("Sonlanmış transaction'a entry eklenemez");
        }
        entries.add(new LedgerEntry(LedgerEntryId.generate(), accountId, direction, amount));
    }

    public void commit() {
        validateBalanced();    // Σ DR = Σ CR (her currency için)
        this.status = Status.COMMITTED;
    }

    public List<LedgerEntry> entries() {
        return Collections.unmodifiableList(entries);
    }

    private void validateBalanced() {
        // her currency için DR toplamı = CR toplamı kontrolü
    }
}
```

Burada **aggregate** = Transaction + içindeki `entries` listesi. Birlikte tutarlı kalan küme. Yukarıda konuştuğumuz `Σ DR = Σ CR` invariant'ı bu sınır içinde geçerli ve Transaction tarafından korunuyor.

**Aggregate root** = Transaction. Dışarıdan tek erişim noktası.

Niye `LedgerEntry`'leri doğrudan dışarıya açmıyoruz? Çünkü eğer biri `transaction.entries().add(new LedgerEntry(...))` diyebilseydi, Transaction kontrol etmeden entry eklenmiş olur, denge bozulurdu. Şu an `entries()` `Collections.unmodifiableList()` dönüyor — dışarısı listeyi okuyabilir ama değiştiremez. Değişiklik için `addEntry()` çağırmak zorunda; orası da `status`'a bakıyor.

Yani aggregate root sadece "ana nesne" değil; aynı zamanda kümenin içindeki her şeyin **bekçisi**. Kurallar buradan uygulanır, başka yerden değil.

İkisini yan yana koyarsak:

| | İçinde ne var? | Root | Kuralı kim koruyor? |
|---|---|---|---|
| **Account aggregate** | Tek nesne — Account | Account | Account (`freeze`, `close`) |
| **Transaction aggregate** | Transaction + LedgerEntry listesi | Transaction | Transaction (`addEntry`, `commit`) |

Aggregate'in kaç nesne içerdiği değişebilir; **root'tan geçme zorunluluğu** değişmez. Bu DDD'de aggregate sınırının temel anlamı.

## LedgerEntry'nin durumu — bir nüans

Yukarıdaki tabloda LedgerEntry'i Transaction aggregate'inin içinde gösterdik. Ama LedgerEntry de bir Entity'dir — kimliği var, "şu kayıt değil bu kayıt" diyebilmek için ID'si lazım. Buna rağmen **bağımsız bir aggregate değil**; Transaction aggregate'inin içinde yaşar. Aggregate'ler iç içe geçmez: bir nesne ya tek başına aggregate'tir, ya da başka bir aggregate'in parçasıdır.

Pratik sonucu: LedgerEntry'ye gitmenin tek yolu Transaction'dır. Repository'de bile ayrı bir `LedgerEntryRepository` yoktur — `TransactionRepository` var, Transaction'ı yüklediğinde LedgerEntry'ler onunla birlikte gelir.

## Vaughn Vernon'un üç kuralı

Aggregate sınırlarını çizmek DDD'nin en kafa karıştırıcı kısmı. Vaughn Vernon "Effective Aggregate Design" yazı dizisinde (2011) üç kural önerir. Sırayla bakalım.

### Kural 1 — Gerçek invariant'ı modelle, daha fazlasını değil

Bir aggregate'in içine sadece aynı anda tutarlı kalmak **zorunda** olan şeyleri koy. "Zorunda" kelimesi kritik. Tek soru: bu iki şey aynı transaction içinde, aynı anda tutarlı olmak zorunda mı, yoksa biraz gecikme tolere edilir mi?

İki kavramı ayıralım. **Gerçek invariant** = bir an bile bozulursa sistem yanlış cevap verir, para kaybolur, audit kırmızı yanar. **Sahte invariant** = eninde sonunda doğru olması yeter, 200 ms gecikme sorun değil.

Gerçek invariant örneği: Transaction içindeki `Σ DR = Σ CR` kuralı. Commit anında dengeli olmak zorunda. "Önce DEBIT'i yazalım, CREDIT 100 ms sonra eklenir" diyemezsin — o aralıkta sistemde dengesiz bir transaction var demek, audit raporları bozuk çıkar. Bu yüzden Transaction ve entry'leri **aynı aggregate** içinde, tek `save()` çağrısında atomik yazılır.

Sahte invariant örneği: müşterinin adı değişince hesap dökümünde gösterilen ad da değişmeli. Bu invariant mı? Hayır — 14:00'te ad güncellendi, 14:00:01'de ekran yenilense yeter. Bu yüzden Customer ve Account **ayrı aggregate**. Account içinde Customer nesnesi yok, sadece `customerId` tutuluyor. Customer adı değiştiğinde Account'a dokunulmaz; hesap dökümünde ad gösterilecekse Customer ayrıca okunur.

**Pratik test (üç soru):**

- Bu kural 1 saniye bozulursa ne olur? "Para kaybolur / audit kırmızı yanar" → gerçek invariant, aynı aggregate.
- Eninde sonunda doğru olsa yeter mi? "Evet" → sahte invariant, farklı aggregate.
- Bu bir kural mı yoksa bir sorgu mu? "Toplam göster", "listeyi getir" türündeki şeyler invariant değil, query. Aggregate sınırını şekillendirmezler — projection ile çözülür.

### Kural 2 — Aggregate'leri küçük tut

Büyük aggregate üç ayrı sorun çıkarır: belleğe yüklemek pahalı, eşzamanlı işlemler birbirini bekletir, içerideki koleksiyon sınırsız büyüyebilir. Şüphede kaldığında küçük tut.

İyi örnek: **Transaction aggregate'i doğası gereği küçüktür.** İçindeki entry sayısı sınırlı — basit transfer 2 entry, komisyonlu transfer 3, FX transferi 4-5. Yapı gereği şişmez. Yükleme her zaman ucuz, eşzamanlı kilit kısa, hafıza yükü sabit. Vernon'un terminolojisinde buna "naturally small aggregate" denir.

Karşı örnek: **Customer aggregate'inin içine bütün Account'ları koymak.** Bireysel müşteride 1-5 hesap olur, sorun yok. Ama kurumsal müşteride 200 hesap olabilir; her hesabın da kendi current_balance'ı, status'ı, limit'i var. Customer'ı bir kere yüklemek = 200 nesneyi belleğe almak. Müşterinin sadece adresini güncellemek istediğinde bile 200 hesap yüklenir. Bu yüzden Customer ve Account ayrı aggregate, ilişki `account.customerId` üzerinden kurulur.

Account'ın içine LedgerEntry listesi koymak da bu kuralın tipik ihlalidir — detayı az aşağıda ele alacağız.

**Pratik test:**

- İçindeki bir koleksiyon teorik olarak sınırsız büyüyebilir mi? Evet → o koleksiyon ayrı aggregate'lerden oluşmalı.
- Aggregate'i yüklemenin maliyeti sub-millisecond mı, saniyeler mi? Yüksekse böl.
- Aynı aggregate'e eşzamanlı kaç işlem dokunuyor? Yüksek concurrency varsa küçük tut, kilit süresi kısalsın.

### Kural 3 — Aggregate'ler arası referans kimlikle, nesne referansıyla değil

Bir aggregate başka bir aggregate'i bilmek zorundaysa, onu **kimlik** ile tutar, nesne referansıyla değil. Account `Customer owner` yerine `CustomerId customerId` tutar.

Niye? Nesne referansı aggregate sınırını sızdırır. `account.getOwner().updateName(...)` yazabiliyorsan, Account'tan Customer'ın içine girip onu değiştiriyorsun demektir. Customer'ın kendi invariant'larına Account üzerinden müdahale ediliyor — bu, DDD'nin temel ayrımını bozar.

Domain'imizdeki iki örnek:

`LedgerEntry`, Transaction aggregate'inin parçası. Ama hangi Account'ı etkilediğini biliyor — kimlikle:

```java
public final class LedgerEntry {
    private final LedgerEntryId id;
    private final AccountId accountId;    // Account aggregate'e kimlikle referans
    private final Direction direction;     // DEBIT / CREDIT
    private final Money amount;
}
```

`Account` nesnesi LedgerEntry'nin içinde değil. Bu sayede Transaction commit edilirken Account'ları yüklemek şart değil — entry'ler kimliklerle yazılır, sonradan ayrı bir projeksiyon süreci o entry'leri okuyup `Account.balance`'ı günceller. Eventual consistency.

Aynı sezgi Account → Customer için de geçerli:

```java
public class Account {
    private final CustomerId customerId;   // YOK: private Customer owner
    private Money balance;
    ...
}
```

Müşteri adı güncellenecekse `customerRepository.findById(...)` ile Customer aggregate'i çekilir, değiştirilir, kaydedilir. Account hiç dokunulmaz.

Nesne referansının çıkardığı sorunlar:

- **JPA refleksi.** `@ManyToOne Customer owner` yazınca Account yüklenince Customer otomatik gelir; cascade fetch zinciri büyür, performans çöker.
- **Sahte invariant tuzağı.** Birisi `account.owner.totalAssets`'a bakan bir kural yazar. Customer'ın state'i Account üzerinden okunup yazılmaya başlar, sınır bulanır.
- **Yaşam döngüsü karışır.** Customer silinmek isteniyor, ama Account hâlâ ona nesne referansı tutuyor. Kimlik referansında bu sorun yok — Account'ın `customerId`'sine karşılık gelen Customer yoksa hesap "ex-customer" durumunda yaşamaya devam eder (audit/legal sebebiyle doğal bir durum).

**Pratik test:**

- Bu referans aynı aggregate içinde mi? (Transaction → LedgerEntry) → nesne referansı OK.
- Farklı aggregate içinde mi? (Account → Customer, LedgerEntry → Account) → kimlik referansı (id).
- Şüphede: kimlik. Sonradan ihtiyaç olursa repository ile yükle.

## Account'ın içinde niye Transaction veya LedgerEntry listesi yok?

JPA dünyasından gelirken refleks `@OneToMany`. Account'un içine ya bir `List<Transaction>` ya da bir `List<LedgerEntry>` koymak çok doğal görünür. İkisi de yaygın refleks; ben de başta karıştırmıştım. Ama ikisi de yanlış — farklı sebeplerden. Tek tek ele alalım.

### Önce Transaction'ı hatırlayalım

Bir Transaction **tek bir Account'a ait değildir, olamaz.** Çünkü her Transaction birden fazla LedgerEntry üretir, bu entry'lerin her biri farklı bir Account'a bağlı olabilir.

- Basit transfer: A için DEBIT + B için CREDIT → **2 farklı** Account
- Komisyonlu transfer: A + B + `fee_income` → **3 farklı** Account
- FX transferi: A_TL + nostro_TL + nostro_USD + B_USD → **4 farklı** Account

`transactions` tablomuzu hatırla — orada `account_id` field'ı yoktu. Bilinçli olarak yok. Transaction "olay"ın kendisidir; tek bir Account'a "aitlik"i tanımsız. Bağ hep `ledger_entries.account_id` üzerinden, entry seviyesinde kuruluyor.

Bu yüzden:

```java
class Account {
    private List<Transaction> transactions;   // YANLIŞ — yapısal olarak imkansız
}
```

Aynı Transaction iki farklı Account'ın listesinde birden mi yaşayacak? Hangi tarafa gidecek? Yapı baştan kırık. Bu sezgi DDD'den bağımsız zaten domaine ters.

### Peki ya LedgerEntry?

```java
class Account {
    private List<LedgerEntry> entries;   // YANLIŞ — bu sefer farklı sebepten
}
```

Bu yapı kendi içinde çelişmez — her LedgerEntry tek bir Account'a bağlı, "A'nın hareketleri" listesi mantıklı görünür. Yine de DDD bunu reddediyor, iki sebepten:

**(a) Aggregate sınırı çakışır.** LedgerEntry zaten Transaction aggregate'inin parçası. Eğer Account aggregate'i de aynı LedgerEntry'leri tutarsa, aynı nesne iki ayrı aggregate sınırının içinde birden bulunur. "Bu LedgerEntry'yi kim koruyor — Account mı, Transaction mı?" sorusunun cevabı bulanıklaşır. DDD'de bir nesne ya bir aggregate'in parçasıdır ya değil; iki yere birden ait olamaz.

**(b) Aggregate kontrolsüz büyür (Vernon 2. kuralı).** Aktif bir hesap on yıl sonra milyonlarca entry barındırabilir. Account'u her yüklediğinde:

- Eşlenen tüm entry'ler yüklenmeye çalışılır (JPA cascade fetch).
- Aggregate'i kaydetmek için tüm liste belleğe girmek zorunda.
- İki eşzamanlı transfer aynı Account üzerinde kilitlenir; biri diğerini bekler.

Saniyede binlerce işlem yapan bir bankacılık sisteminde bu darboğaz kabul edilemez.

### Peki doğrusu?

Account ile diğer aggregate'ler arasında **doğrudan nesne referansı yoktur**. Bağ ters yönde, sadece kimlik üzerindendir. Hesaba dair tüm bilgi LedgerEntry'nin içindeki `accountId` field'ında saklanır:

```java
record LedgerEntry(
    LedgerEntryId id,
    AccountId accountId,    // Account nesnesi değil, kimliği
    Direction direction,
    Money amount,
    ...
)
```

- Account, ne transaction'lardan ne entry'lerden haberdardır. Kendi sınırı içinde yaşar.
- LedgerEntry, ilgili Account'a `accountId` ile bağlı.
- "Bu hesap hangi işlemlerde geçti?" sorusu repository sorgusuyla cevaplanır:

```java
List<Transaction> txs = transactionRepo.findByAccountId(accountId);
```

Veritabanı seviyesinde bağ duruyor (`ledger_entries.account_id` foreign key), ama domain modelinde nesne grafiği olarak yaşamıyor. Yukarıda anlattığımız Vernon 3. kuralının doğrudan uygulaması.

JPA'nın `@OneToMany` refleksinden çıkmak DDD geçişinin en zor adımlarından biri. "İlişki = nesne grafiği" sezgisini bırakıp "ilişki = kimlik bağı" sezgisine geçmek alışkanlık değişimi gerektiriyor.

Sonuç olarak banking domain'imizde iki ayrı aggregate var: **Account** (tek nesneli) ve **Transaction** (root + LedgerEntry'ler). Aralarındaki bağ doğrudan değil, sadece kimlik. Aggregate sınırını koruyan budur.

---

## Buraya kadar ne aldık?

Aggregate ve Aggregate Root iki ayrı kavram:

- Aggregate = birlikte tutarlı kalan nesnelerin kümesi (1 nesne de olabilir, N nesne de).
- Aggregate Root = bu kümeye dışarıdan tek erişim noktası, kuralın bekçisi.

Aggregate sınırını çizerken üç soru rehber oluyor: invariant gerçek mi (Vernon 1), aggregate küçük mü (Vernon 2), aggregate'ler arası bağ kimlikle mi (Vernon 3).

JPA'nın `@OneToMany` refleksi DDD'de bilinçli bir karara dönüşmek zorunda — "ilişki = nesne grafiği" yerine "ilişki = kimlik bağı" sezgisine geçmek gerekiyor.

Sonraki bölümde aggregate'leri kullanan üç katmana geçiyoruz: **Repository** (aggregate'i depodan al/koy), **Domain Service** (iki aggregate'i koordine eden domain mantığı), **Application Service** (use case akışı, transaction sınırı). Hangi kod hangi katmana ait sorusunun cevabı 3. bölümde.
