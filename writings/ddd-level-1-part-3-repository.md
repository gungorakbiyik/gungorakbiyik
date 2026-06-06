> 📝 _Originally published on Medium: [DDD Level 1 — Bölüm 3: Repository, Domain Service, Application Service](https://medium.com/@gungor.akbiyik/ddd-level-1-b%C3%B6l%C3%BCm-3-repository-domain-service-application-service-9088b1de844e)_


# DDD Level 1 — Bölüm 3: Repository, Domain Service, Application Service


> Üç bölümlük serinin son parçası. [1. bölümde](https://medium.com/@gungor.akbiyik/ddd-level-1-b%C3%B6l%C3%BCm-1-ddd-giri%C5%9F-entity-ve-value-object-db0b91d40d36) Entity ve Value Object'i, [2. bölümde](https://medium.com/@gungor.akbiyik/ddd-level-1-b%C3%B6l%C3%BCm-2-aggregate-ve-aggregate-root-b827bca2bb9c) Aggregate'i konuştuk. Bu bölüm aggregate'leri kullanan üç katmanı işliyor: Repository (aggregate'i depodan al/koy), Domain Service (iki aggregate'i koordine eden domain mantığı), Application Service (use case akışı, transaction sınırı). Sonunda sık yapılan hatalar, uygulama sırası ve sözlük var.

İki bölüm boyunca nesneleri konuştuk.

`Money` neden immutable olmalı. `Account` neden sadece `id` üzerinden eşit sayılmalı. Bir `Transaction` neden kendi içindeki `LedgerEntry`'leri dışarı vermez. Aggregate sınırını çizerken hangi üç soruyu sormak gerekiyor.

Bunları bilmek gerekli ama yetmiyor. `Account` aggregate'ini kim veritabanından çekecek? "A hesabından B hesabına transfer" gibi iki aggregate'i ilgilendiren bir kural nereye yazılacak? REST isteği geldiğinde idempotency kontrolü, aggregate yükleme, kayıt, event yayınlama akışını kim yönetecek?

Bu üç sorunun üç ayrı cevabı var: Repository, Domain Service, Application Service.

---

## Repository

Account aggregate'ini bir yerden okuyup bir yere yazmamız gerekiyor. Spring + JPA dünyasında refleks olarak şuna kayıyoruz:

```java
public interface AccountJpaRepository extends JpaRepository<AccountEntity, UUID> { }
```

Çalışır. Bir satır kod, CRUD hazır. Ama burada fark etmeden bir şey oldu: Account artık JPA'yı bilmek zorunda. `AccountEntity` adında bir JPA sınıfı doğdu, `@Column`, `@Id`, `@Table` annotation'ları girdi, domain modelimiz Hibernate'in lifecycle'ına bağlandı. "İyi nesne tasarımı" diyerek kurduğumuz Account, ORM'in dilini konuşmak zorunda kaldı.

Alternatife bak:

```java
public interface AccountRepository {
    Optional<Account> findById(AccountId id);
    void save(Account account);
    List<Account> findByCustomerId(CustomerId customerId);
}
```

İmzalarda hiçbir altyapı sözcüğü yok — JPA yok, PostgreSQL yok, SQL yok. `Optional<Account>` dönüyor; domain nesnesi, DTO değil. Parametreler de domain tipleri: `AccountId`, `CustomerId`. Baştan beri konuştuğumuz Value Object'ler.

Bu interface domain paketinin içinde yaşıyor. Domain onu görüyor, çağırıyor. Altında ne olduğunu bilmiyor.

JPA tarafı ayrı bir paketteki adapter:

```java
@Repository
public class JpaAccountRepository implements AccountRepository {
    private final SpringDataAccountRepo springRepo;
    private final AccountMapper mapper;

    public Optional<Account> findById(AccountId id) {
        return springRepo.findById(id.value()).map(mapper::toDomain);
    }

    public void save(Account account) {
        springRepo.save(mapper.toEntity(account));
    }
    // ...
}
```

Domain interface'i bir **port**, JPA implementasyonu bir **adapter** — hexagonal mimarinin terminolojisi. Yarın PostgreSQL'den MongoDB'ye geçsek, sadece adapter değişir; domain'de tek satır kod değişmez.

Bu yapıdan iki sessiz kural çıkıyor.

Birincisi: repository sadece aggregate'ler için var. 
`AccountRepository` evet — Account bir aggregate root. 
`StatusRepository` hayır — `Status` bir enum, aggregate değil. 
"Her tabloya bir repository" Spring refleksi; DDD'de repository "veritabanı tablosunu yöneten" değil, "aggregate'i tutan" şey.

İkincisi: imzada `Optional<Account>` görüyorsun, `AccountDto` değil. DTO çevirisini sunum katmanı yapar. Repository domain'in içinde yaşar; DTO bilmez, bilmesi de gerekmez.

Bir not: bazı projelerde JPA entity ile aggregate aynı sınıfta birleştirilir — `Account` hem domain hem persistence modeli olur. Boilerplate azalır, mapper kaybolur. Bedeli de var: domain artık JPA'ya bağımlı, Hibernate'in lifecycle kısıtları (default constructor, lazy loading, immutability kaybı) domain tasarımına sızar. Artıları ve eksileri ayrı bir tartışma — bu yazıda port/adapter ayrımıyla devam ediyoruz.

---

## Domain Service

Account'a `freeze()` koyduk, kural Account'ın içinde. 
Money'ye `add()` koyduk, kural Money'nin içinde. 
Aggregate'ler kendi sınırlarını korumayı öğrendi.

Peki ya iki Account'ı birden ilgilendiren bir kural varsa?

A hesabından B hesabına para transferi. 
Düşün: hangi nesnenin metodu olmalı bu?

`source.transferTo(target, amount)`? 
Mantıklı görünüyor, ama transferi başlatma kuralları sadece source'u değil target'ı da ilgilendiriyor — currency eşleşmesi her ikisinin sorumluluğu, ikisi de aktif olmak zorunda. Source'un içine yazmak target'a haksızlık. Üstelik bu metot bir `Transaction` döndürmek zorunda; bir Account'ın kendi içinden Transaction yaratması da garip kaçıyor.

`target.acceptFrom(source, amount)`? 
Aynı simetrik problem, taraflar değişti sadece.

Bir Application Service'e atalım o zaman, "TransferService" diyelim? 
Olabilir ama o zaman domain kuralları (currency eşleşmesi, aktiflik kontrolü, balanced entry üretimi) tekrar service katmanına kaçıyor — Anemic Domain Model'in geri dönüşü.

Şuna benzer bir sınıfa ihtiyacımız var:

```java
public class MoneyTransfer {

    public Transaction transfer(
        Account source,
        Account target,
        Money amount,
        TransactionId transactionId
    ) {
        source.assertActive();
        target.assertActive();
        if (!source.currency().equals(target.currency())) {
            throw new CurrencyMismatchException();
        }

        Transaction tx = new Transaction(transactionId, TransactionType.TRANSFER);
        tx.addEntry(source.id(), Direction.DEBIT, amount);
        tx.addEntry(target.id(), Direction.CREDIT, amount);
        tx.commit();           // Σ DR = Σ CR doğrulaması Transaction içinde
        return tx;
    }
}
```

Sınıfa bak — birkaç şey dikkat çekiyor.

Field yok. Sınıf durumsuz; her `transfer()` çağrısı bağımsız, kendi parametreleriyle yaşıyor.

Parametreler domain nesnesi — `Account`, `Money`, `TransactionId`. DTO yok, primitive tip yok. Aggregate'leri zaten yüklenmiş halde alıyor.

Metodun içinde infrastructure yok. Hiçbir veritabanı çağrısı, Kafka publisher, HTTP client geçmiyor. Saf domain mantığı, iki Account'ın bilmediği bir kuralı koordine ediyor.

İşte bu **Domain Service**. Tek bir aggregate'in içine sığmayan, ama tamamen domain kurallarından oluşan davranışların evi.

Bir test sorusu: *"Bu metot bir aggregate'in içine koyulabilir mi?"*

Cevap her zaman net değil. Üç durum:

**Evet — aggregate'e koy.**
- Kural tek aggregate'in kendi field'larıyla ifade ediliyorsa. `account.freeze()` — sadece Account state'i değişiyor.
- Saf hesaplama, parametre olarak primitive ya da Value Object. `money.add(other)` — Money + Money → Money.
- Sonuç yine aynı aggregate'in iç değişikliği. `transaction.addEntry(...)`, `transaction.commit()`.

Refleks önce buraya gitmeli; Domain Service tembel bir çıkış kapısı değil.

**Hayır — Domain Service'e taşı.**
- İki ya da daha fazla aggregate koordine ediliyorsa. Transfer — source + target, ikisinin de durumu kuralın parçası.
- Sonuç farklı bir aggregate üretiyorsa. Transfer, `Transaction` üretir; Account'ın Transaction yaratması garip kaçar.
- Aggregate sınırını ihlal etmeden ifade edilemiyorsa. "Source ve target currency aynı mı" — Account'ın içine koymak için Account'a target referansı geçmek gerekir, sınır sızar.

**Gri bölge.**
- Karmaşık yaratım (factory): aggregate içinde static method da olur, ayrı `AccountFactory` da. Tercih takım konvansiyonu.
- Sadece okuma yapan koordinasyon ("tüm donmuş hesapları listele") çoğunlukla Application Service tarafına yakın, Domain Service değil.

Transfer örneğinde üç "Hayır" koşulu birden tetiklendi — iki aggregate, farklı aggregate üretimi, sınır sızıntısı riski. Bu yüzden `MoneyTransfer` ayrı bir Domain Service olarak doğdu.

Bir de dikkat: `MoneyTransfer` içinde `LedgerEntry`'leri `new` ile yaratmıyoruz. Transaction aggregate'i `addEntry()` üzerinden kendi içinde yaratıyor. Aggregate sınırı korunuyor — "LedgerEntry'ye gitmenin tek yolu Transaction'dır" kuralı kodda da geçerli kalıyor.

---

## Application Service

`MoneyTransfer` iki Account'u koordine ediyor, kuralı uyguluyor. İşi temiz. Ama dış dünyadan REST üzerinden "şu IBAN'dan şuna 100 TL gönder" isteği geldiğinde, `MoneyTransfer.transfer()` çağrılana kadar bir sürü şey olması lazım.

Aggregate'leri kim yükleyecek? Source ve target Account'larını veritabanından kim çekecek?

Transaction sınırı nerede başlayıp nerede bitecek? `@Transactional` nereye gelecek?

Idempotency-key kontrolünü kim yapacak? Aynı istek iki kez geldiğinde paranın iki kez gitmesini engelleyen kod kimin?

Transaction kaydedildikten sonra event'i kim yayınlayacak?

Cevabı bir DTO'ya kim çevirip controller'a kim verecek?

Bunların hiçbiri domain kuralı değil. "Şu use case'i çalıştırmak için hangi adımlar lazım?" sorusunun cevabı. Akış meselesi.

Şöyle bir sınıf çıkıyor ortaya:

```java
@Service
public class TransferService {

    private final AccountRepository accountRepo;
    private final TransactionRepository transactionRepo;
    private final MoneyTransfer moneyTransfer;
    private final DomainEventPublisher eventPublisher;

    @Transactional
    public TransferResponse transfer(TransferCommand cmd) {
        // 1. Tekrarlanan istek kontrolü
        if (transactionRepo.existsByIdempotencyKey(cmd.idempotencyKey())) {
            return TransferResponse.duplicate(cmd.idempotencyKey());
        }

        // 2. Aggregate yükleme
        Account source = accountRepo.findById(cmd.sourceId())
            .orElseThrow(AccountNotFoundException::new);
        Account target = accountRepo.findById(cmd.targetId())
            .orElseThrow(AccountNotFoundException::new);

        // 3. Domain mantığı — domain service'e devret
        Transaction transaction = moneyTransfer.transfer(
            source, target, cmd.amount(), TransactionId.generate()
        );

        // 4. Kaydet ve event yayınla
        transactionRepo.save(transaction);
        eventPublisher.publish(new TransferCompleted(transaction.id()));

        // 5. DTO döndür
        return TransferResponse.from(transaction);
    }
}
```

Kısa bir not: `MoneyTransfer` constructor injection ile geliyor — Spring tarafında singleton bean. İki yol var: domain class'ına `@Component` koymak, ya da domain POJO bırakıp `@Configuration` ile `@Bean` tanımı yapmak. Tercihim ikincisi — domain Spring'i bilmemeli — ama küçük projelerde birincisi de yaygın. Tek seçim önemli: `new MoneyTransfer()` yapma; dependency saklanır, test edemezsin.

Bu kodun içinde **tek bir domain kuralı yok**. 
Currency eşleşmesi? `MoneyTransfer` içinde. 
Negatif tutar yasağı? `Money` Value Object'inde. 
Hesap aktif mi? `Account.assertActive()` içinde. 
Σ DR = Σ CR dengesi? `Transaction.commit()` içinde. 
`TransferService` sadece koordinatör: yükle, çağır, kaydet, yayınla, dönüştür.

`@Transactional` da bilinçli olarak burada. Transaction sınırı bu metodun başında açılıyor, sonunda kapanıyor. Domain Service'e koymadık çünkü "bir DB transaction içinde olalım" kararı use case seviyesinde alınır — bir transfer = bir DB transaction. Domain mantığı transaction sınırını bilmek zorunda değil.

Bu katman **Application Service**. Domain Service ile karıştırmak çok kolay — ikisi de "service" diye bitiyor, ikisi de bir use case'i akıtıyormuş gibi görünüyor. Ama işleri farklı:

**Domain Service**
- **Mantık türü:** Domain kuralları
- **Girdi / Çıktı:** Domain nesneleri
- **Bağımlılık:** Yalnızca domain
- **Transaction:** Hayır
- **Test:** Saf birim testi

**Application Service**
- **Mantık türü:** Kullanım senaryosu akışı
- **Girdi / Çıktı:** DTO'lar
- **Bağımlılık:** Repository, infrastructure, event bus
- **Transaction:** Evet (`@Transactional`)
- **Test:** Entegrasyon veya mock testi

Bu ayrım kaybolunca ne olur? Domain mantığı Application Service'e yığılır. `TransferService.transfer` 200 satıra çıkar; içinde currency kontrolü, aktiflik kontrolü, ledger entry oluşturma... Aggregate'ler bu mantığı bilmediği için yine veri torbasına dönüşür. Anemic Domain Model'in başka kılıkta geri dönüşü.

Service ile dolu bir Spring projesini DDD'ye taşımanın en zor kısmı bu ayrımı oturtmak. "Hangi kod aggregate'in içine, hangisi domain service'e, hangisi application service'e gidiyor?" — bu üç soruyu doğru cevaplayabilen ekip yarı yola gelmiş demektir.

---

## Hangi Banking Nesnesi Hangi DDD Kavramı?

Şu ana kadar parça parça konuştuk. Toparlayalım:

**Entity** — `Account`, `LedgerEntry`, `Transaction`, `Customer`. Kimliği var, zaman içinde değişiyor.

**Value Object** — `Money`, `AccountId`, `Currency`, `IBAN`, `Direction`. Kimliği yok, değişmez, değerleriyle eşit.

**Aggregate** — `Account`, `Transaction`, `Customer` (root'larıyla). Birlikte tutarlı kalan nesne kümesi.

**Aggregate Root** — `Account`, `Transaction`, `Customer`. Aggregate'in dışa açık tek arayüzü.

**Repository** — `AccountRepository`, `TransactionRepository`. Aggregate persistence soyutlaması.

**Domain Service** — `MoneyTransfer`, `InterestAccrual`, `LimitCheck`. Birden çok aggregate koordine eden domain mantığı.

**Application Service** — `TransferService`, `DepositService`, `AccountOpeningService`. Kullanım senaryosu akışı, transaction sınırı.

---

## Sık Yapılan Hatalar

Yazıyı bir araya getirirken aklımda tutmak istediğim, kendim de bir kısmına düştüğüm tuzaklar.

**Anemic Domain Model.** `Account` sadece getter/setter'dan ibaret, davranış `AccountService`'in içinde. En klasik tuzak. `account.freeze()`, `account.assertActive()` gibi metotlar nesneye geri taşınmadığı sürece DDD diye yazdığın hiçbir şey DDD değildir.

**Primitive Obsession.** Her yerde `String iban`, `BigDecimal amount`, `String currency`. IBAN format kontrolü 12 ayrı yerde tekrar ediyor, bir gün biri unutuyor, geçersiz IBAN sisteme giriyor. `IBAN`, `Money`, `Currency` gibi Value Object'ler bunu engellemek için var — kural tek bir yerde yaşasın diye.

**Şişirilmiş Aggregate.** `Account` içine `List<Transaction>` veya `List<LedgerEntry>` koymak. Aggregate sonsuz büyür, lock contention başlar, performans çöker. Aggregate konusunda uzun uzun anlattığımız sebepten — Account ve Transaction ayrı aggregate'ler, bağ sadece `accountId` üzerinden.

**Aggregate'ler Arası Nesne Referansı.** `Account.customer = Customer` yazıp Customer nesnesini Account'ın içine koymak. Aggregate sınırı sızıyor, iki aggregate aynı bellekte birbirine zincirleniyor. Doğrusu `Account.customerId = CustomerId` — bağ kimlikle.

**Repository'ye Domain Dışı Metotlar.** `AccountRepository.exportToCsv()` veya `AccountRepository.sendWelcomeEmail()`. Repository'nin işi aggregate'i tutmak ve geri vermek. CSV export ayrı bir application service'in, email gönderme ayrı bir adapter'ın işi.

**Domain Mantığını Application Service'e Yığmak.** Anemic Domain Model'in başka kılığı. `TransferService` 300 satır, içinde currency kontrolü, balance check, ledger entry üretimi... Her metot için tek soru: "Bu domain kuralı mı, akış koordinasyonu mu?" Kuralsa aggregate veya Domain Service'e gider, Application Service'e değil.

**Her Tablo İçin Repository.** `AccountRepository`, `LedgerEntryRepository`, `CurrencyRepository`, `StatusRepository`... Repository her **aggregate** için var, her **tablo** için değil. LedgerEntry Transaction aggregate'inin içinde yaşıyor — ayrı repository'si olmaz. Currency enum, repository'ye ihtiyacı yok. Aggregate sınırı tablo sınırlarıyla bire bir örtüşmüyor.

---

## Hangi Sırayla Uygulamalı?

Sıfırdan bir aggregate yazıyorsan kavramları rastgele değil, belli bir sırayla inşa etmek işi kolaylaştırıyor.

İlk adım Value Object'ler. `Money`, `AccountId`, `Currency`, `Direction`. Saf, değişmez, kurallar constructor'da. Bunlar olmadan Entity'leri yazınca primitive tipler her yerden sızar; sonra geri dönüp `String iban`'ları `IBAN`'a çevirmek tatsız iş.

Sonra Entity ve Aggregate Root'lar. `Account`, `Transaction`. Field'lar artık Value Object'lerden oluşuyor. Davranış metotları (`freeze`, `close`, `addEntry`, `commit`) buraya geliyor — setter yok, kurallar metotların içinde.

Repository interface'leri bunun üstüne gelir. `AccountRepository`, `TransactionRepository` — sadece interface, implementasyon henüz yok. Domain'de yaşıyor, JPA bilmiyor.

Domain Service ihtiyacı genelde tam bu noktada belirir. `MoneyTransfer` gibi iki aggregate'i koordine eden ama tamamen domain mantığı olan operasyonlar kendi sınıflarına ayrılır.

Application Service en son domain katmanı. `TransferService`. Repository + Domain Service + `@Transactional` + event publisher. Use case akışı burada oturur.

En son hexagonal adapter'lar. REST controller (giriş), JPA implementasyonu (çıkış), Kafka publisher. Domain bunların hiçbirini bilmez — her biri ayrı paketlerde, ayrı modüllerde yaşar.

Bu sıra "domain merkezde" felsefesini koda yansıtıyor. Tersinden başlarsan — controller, JPA, sonra domain — domain her zaman dışarının dilini konuşmak zorunda kalıyor.

---

## Bu Serinin Dışında Kalanlar

Üç bölümde birkaç konuya değinip geçtim, bunları burada netleştirmek istedim.

**Domain Event.** `TransferService` kodunda `eventPublisher.publish(new TransferCompleted(...))` satırı gördün. Domain Event tam olarak budur — aggregate içinde gerçekleşen bir şeyin, aggregate sınırının ötesine taşınması. `@TransactionalEventListener`, Outbox pattern, event-driven mimari bu konunun uzantıları. Aggregate'ler arası eventual consistency'nin taşıyıcısı. Ayrı bir yazıyı hak ediyor.

**Strategic DDD.** Bounded Context, Ubiquitous Language, Context Map — kod seviyesinin üzerindeki katman. Tactical pattern'lerin oturması için strategic kararların önce alınması gerekiyor aslında. Ama pratikte tersine de gidilebilir: önce kod, sonra büyük resim. Level 2 bunların üstüne oturacak.

**Hexagonal Mimari.** Repository tartışmasında port/adapter terminolojisi geçti — domain interface bir port, JPA implementasyonu bir adapter. Bu kendi başına bir mimari kararlar zinciri: primary vs secondary adapter ayrımı, ArchUnit ile sınır koruma, paket yapısı kuralları. DDD ile hexagonal sık birlikte kullanılır ama bağımsız konulardır — biri tactical pattern seti, diğeri mimari bir desen. Ayrı bir yazı dizisi olarak ele alınacak.

---

## Kaynaklar

- **Eric Evans** (2003) — *Domain-Driven Design: Tackling Complexity in the Heart of Software*. "Blue Book". Temel kaynak.
- **Vaughn Vernon** (2013) — *Implementing Domain-Driven Design*. "Red Book". Tactical pattern'lerin pratik rehberi.
- **Vaughn Vernon** (2011) — *Effective Aggregate Design*. 3-part PDF, dddcommunity.org. Aggregate tasarımının altın standardı. [Part 1](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_1.pdf) · [Part 2](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_2.pdf) · [Part 3](https://www.dddcommunity.org/wp-content/uploads/files/pdf_articles/Vernon_2011_3.pdf)
- **Vladimir Khorikov** — *Entity vs Value Object: the ultimate list of differences*. [enterprisecraftsmanship.com](https://enterprisecraftsmanship.com/posts/entity-vs-value-object-the-ultimate-list-of-differences/)
- **Martin Fowler** — *AnemicDomainModel*. [martinfowler.com](https://martinfowler.com/bliki/AnemicDomainModel.html)
- **Lev Gorodinski** (2012) — *Services in Domain-Driven Design*. [gorodinski.com](http://gorodinski.com/blog/2012/04/14/services-in-domain-driven-design-ddd/)

---

## Sözlük

**Entity** — Kimliği olan, zaman içinde değişebilen nesne. Eşitlik kimlik üzerinden.

**Value Object** — Kimliği olmayan, değişmez, değerleriyle tanımlanan nesne.

**Aggregate** — Birlikte tutarlı kalan nesnelerin kümesi ve bunların tutarlılık sınırı.

**Aggregate Root** — Aggregate'in dışa açık tek giriş noktası.

**Tutarlılık Sınırı** — Aggregate etrafına çizilen, "aynı transaction'da tutarlı" sınır.

**Invariant** — Her zaman doğru kalması gereken domain koşulu.

**Repository** — Aggregate persistence soyutlaması.

**Domain Service** — Birden fazla aggregate koordine eden, domain mantığı içeren durumsuz servis.

**Application Service** — Kullanım senaryosu akışı, transaction sınırı — domain mantığı içermez.

**Domain Event** — Domain içinde gerçekleşen bir olayın yayınlanması; aggregate'ler arası eventual consistency'nin taşıyıcısı.

**Anemic Domain Model** — Davranışı olmayan, yalnızca veri taşıyan sınıflar. DDD karşıtı desen.

**Primitive Obsession** — Value Object yerine `String`/`int`/`BigDecimal` kullanmak.

**Ubiquitous Language** — Domain uzmanı ve geliştiricinin ortak dili (strategic DDD, Level 2).

**Bounded Context** — Belirli bir modelin geçerli olduğu sınır (strategic DDD, Level 2).

---

## Seriyi Toparlarsak

Üç bölüm boyunca tactical DDD'nin temel kavramlarını banking üzerinden konuştuk:

- **Bölüm 1** — Entity (kimlik var, değişebilir) ve Value Object (kimlik yok, immutable).
- **Bölüm 2** — Aggregate (tutarlılık kümesi) ve Aggregate Root (tek kapı).
- **Bölüm 3** — Repository (aggregate persistence), Domain Service (çoklu aggregate koordinasyonu), Application Service (use case akışı).

Bu kavramlar tek başlarına anlamlı, ama gerçek değerleri bir arada konuşulduklarında ortaya çıkıyor. Aggregate sınırını bilmeden Repository tasarlayamazsın; Value Object olmadan Entity primitive tiplerle delik deşik olur; Domain Service ile Application Service arasındaki sınırı görmeden yazılmış bir kod Anemic Domain Model'e geri döner.

Level 2'de strategic katmana geçeceğim — Bounded Context, Ubiquitous Language, Context Map. Tactical pattern'leri ekipler arasında nasıl konumlandıracağımızı, hangi modülün hangi kavramı sahipleneceğini orada konuşacağız.
