> 📝 _Originally published on Medium: [DDD Level 1 — Bölüm 1: Entity ve Value Object](https://medium.com/@gungor.akbiyik/ddd-level-1-b%C3%B6l%C3%BCm-1-ddd-giri%C5%9F-entity-ve-value-object-db0b91d40d36)_

---
type: medium-article
status: draft
project: opencore-bank
series: ddd-level-1
part: 1
created: 2026-05-19
tags: [ddd, tactical-patterns, entity, value-object, banking, java, spring]
---

# DDD Level 1 — Bölüm 1: DDD Giriş + Entity ve Value Object

> Bu yazı üç bölümlük bir serinin ilk parçası. Banking domain'i (Account, Transaction, LedgerEntry) üzerinden tactical DDD pattern'lerini yerleştirmeye çalışıyorum. Bu bölümde Entity ve Value Object var — kimliği olan ve olmayan nesneler. 2. bölüm Aggregate'i, 3. bölüm Repository / Domain Service / Application Service'i işliyor.

Account, Transaction, LedgerEntry. OpenCore Bank koduna baktığımda bu sınıfların ne *olduğunu* biliyordum — banking domain'ini [bir önceki yazıda](https://medium.com/@gungor.akbiyik/account-vs-ledger-core-bankingin-temelleri-7958fa4b87b8) konuşmuştuk.

Ama neyi *temsil ettiklerini* anlamak için farklı bir gözlüğe ihtiyacım vardı.

O gözlük DDD.

---

DDD'yi kavramsal olarak biliyordum — "nesnelerin davranışı olsun" tarzı. "İyi tasarım" fikrini yıllar içinde defalarca öğrendim, unuttum, yeniden keşfettim. Ama banking domain'inde pratikte görmek başka bir şey.

Bu yazı o sürecin notları. Tactical DDD pattern'lerini — Entity, Value Object, Aggregate, Repository, Domain Service, Application Service — banking örnekleriyle yerleştirmeye çalışıyorum. Teknik teori değil; "bu sınıf nereye ait?" sorusunun cevabı.

---

## DDD Nedir? — Vehicle, Car ve Davranışın Kaybolduğu Yer

OOP'u Vehicle/Car ile öğrendik. `Car.accelerate()`, `Car.brake()` — hız, vites, frenleme mantığı sınıfın içindeydi. Davranış nesneye aitti. "Veri + davranış birlikte" diye anlattılar, mantıklı geldi.

Sonra üretim koduna girdik, ortaya farklı bir şey çıktı.

Spring Boot projesi açtık. `OrderEntity`, `Order`, `Customer` — hepsi sadece field + getter/setter. Davranış nereye gitti? Servislere. `OrderService.cancel(order)`, `CustomerService.freeze(customer)`. Sınıflar veri torbasına döndü; mantık servis metotlarına dağıldı. Üç ay sonra "iptal kuralı nerede yazıyor?" sorusunun cevabı dört dosyada birden.

Martin Fowler buna 2003'te bir isim taktı: **Anemic Domain Model.** Adı üzerinde — davranışsız, kansız nesne. OOP'un karşılığı değil; prosedürel programlamanın nesne kostümü giymiş hali.

Aynı yıl Eric Evans bir kitap çıkardı: *Domain-Driven Design*. Tek cümleyle: **davranışı geri nesneye taşı, sınırını dikkatle çiz.**

Neden `AccountService.freeze(account)` değil de `account.freeze()` olmalı? Çünkü "donduran kim" değil, "dondurma kuralı kimin sorumluluğu" sorusu önemli. Kapalı hesap dondurulamaz, zaten dondurulmuş hesap tekrar dondurulamaz — bu kurallar Account'a ait. Servis katmanı her seferinde bunları kontrol etmek zorunda kalmaz, etmeyi de unutmaz.

Bu yazıda Car ve Order yerine Account, Money, Ledger üzerinden ilerleyeceğim — fikir aynı.

---

## DDD'nin İki Katmanı: Bu Yazı Hangisi?

DDD'yi konuşurken iki farklı şeyden bahsediliyor genelde.

Biri büyük resim — sistem nasıl parçalanır, ekipler nasıl ortak dil konuşur, hangi modül hangi kavramı sahiplenir. Bounded Context, Ubiquitous Language, Context Map gibi konular bu katmanda. Buna "strategic" deniyor; mimari ve organizasyon seviyesi.

Diğeri kod seviyesi — bir `Account` sınıfını nasıl yazarız, davranış nereye gider, sınırı nereye çizeriz. "Tactical" denen kısım bu.

Bu yazı tactical olanı işliyor. Strategic'i Level 2'ye bırakıyorum — taktiksel kalıplar tek başına da değerli, "iyi nesne tasarımı" diye baksak bile karşılığını alıyoruz.

---

## Entity

İlk durağımız `Account` sınıfı:

```java
public class Account {
    private final AccountId id;     // kimlik — değişmez
    private Status status;          // zaman içinde değişir
    private final Currency currency;
    private final AccountKind kind;

    public boolean equals(Object o) {
        if (!(o instanceof Account other)) return false;
        return this.id.equals(other.id);   // sadece kimlik karşılaştırması
    }

    public int hashCode() {
        return id.hashCode();
    }
}
```

Bir saniye dur ve `equals` metoduna bak. Sadece `id`'ye bakıyor. `status` farklı olsa bile, `currency` farklı olsa bile — `id` aynıysa aynı Account.

`id` field'ı `final`; bir kez set edildi mi değişmez. `status` ise zamanla değişen bir şey: `OPENED → ACTIVE → FROZEN → CLOSED`. Hesap kapatıldığında nesne yok olmuyor, `status` `CLOSED` oluyor. Account aynı kalıyor — çünkü `id` aynı.

Tersi de var. İki Account aynı müşteriye, aynı currency'ye, aynı tipe sahip olabilir; ama `id`'leri farklıysa **farklı** hesaplardır. Müşteri "iki vadesiz hesap istiyorum" der, sistemde iki ayrı Account oluşur. Her şey aynı, kimlikleri farklı.

Bu kalıba **Entity** diyoruz. Kimliği olan, zaman içinde değişebilen nesne. Eşitlik kimlikle ölçülür, değerle değil.

Vladimir Khorikov bir test öneriyor: *"Bu nesneyi tıpatıp aynı bir kopyasıyla değiştirsen fark eder mi?"* Fark ediyorsa Entity — çünkü kimlik önemli, kopya orijinalin yerine geçemez. Fark etmiyorsa farklı bir kavram — birazdan göreceğimiz Value Object.

`LedgerEntry` de Entity. Audit için her kaydın kendi kimliği şart — "şu kayıt değil bu kayıt" diyebilmek gerekiyor.

---

## Value Object

Bir 100 TL düşün. Cebimde 100 TL var, senin cebinde de 100 TL var. Bunların farkı var mı? Yok. İkisi de 100 TL.

`Money` sınıfı:

```java
public record Money(BigDecimal amount, Currency currency) {
    public Money add(Money other) {
        if (!this.currency.equals(other.currency)) {
            throw new IllegalArgumentException("Farklı currency'ler toplanamaz");
        }
        return new Money(this.amount.add(other.amount), this.currency);
    }
}
```

Üç şey dikkat çekiyor.

Birincisi `record` ile yazılmış olması. Java'da `record` otomatik immutable — bir kez yaratıldı mı içindeki `amount` ve `currency` değişmez. "100 TL'yi 200 TL yap" diye bir şey olmaz; yeni bir `Money` yaratılır.

İkincisi `add` metodu. `this.amount`'a eklemiyor, yeni bir `Money` döndürüyor. Mevcut nesne dokunulmaz kalıyor — bu da immutability'nin bir sonucu.

Üçüncüsü: farklı currency toplamaya izin vermiyor. 100 TL ile 30 USD'yi toplayamazsın, exception. Bu kural Money'nin içinde yaşıyor; her servis katmanı bunu tekrar tekrar kontrol etmek zorunda değil.

Bir de `id` yok. Aynı `amount` ve `currency`'ye sahip iki `Money` nesnesi eşittir. Khorikov'un testi: bir 100 TL'yi başka bir 100 TL ile değiştirsen fark eder mi? Etmez. Kimlik yok, sadece değer var.

Bu kalıba **Value Object** diyoruz. Kimliği olmayan, immutable, değerleriyle eşit olan nesne. Tek başına yaşamaz — bir Entity'nin (Account, LedgerEntry) parçasıdır.

### Niye immutable olmak zorunda?

Bir saniye dur — niye mutable yapmayalım Money'yi? Bir `setAmount` koysak ne olur?

```java
class Money {                                     // YAPMA — mutable Money
    BigDecimal amount;
    Currency currency;
    public void setAmount(BigDecimal v) { this.amount = v; }
}
```

Sonra şu kodu hayal et:

```java
Money price = product.getPrice();    // 100 TL
auditLog.record(price);              // log'a aynı referansı verdik
price.setAmount(BigDecimal.ZERO);    // başka bir yerde "temizle"
product.getPrice();                  // hala 100 mü?
```

Hayır. `product.getPrice()` artık `0` döner. `auditLog`'a verdiğin nesne de aynı referans, o da `0` görür. Kimse istemediği halde, üç ayrı yer birden değişti.

Çözüm "defensive copy" gibi görünür — `getPrice` her seferinde yeni bir Money döndürsün. Olur ama her metoda kopyalama yükü bindirir, hâlâ kafa karıştırıcı: "şu Money'yi alanlardan kim değiştirebilir?" sorusu kod tabanına yayılır.

Asıl mesele daha derinde. **100 TL'yi 0 TL yapmak mantıksız.** 100 TL bir değer — sabit. Sayılar gibi düşün: `int x = 5; x = 6;` derken 5'i 6 yapmadın, sadece `x` değişkenine başka bir sayı atadın. 5 hâlâ 5'tir. Para da öyle. Hesabın bakiyesi değişiyorsa, hesabın referans tuttuğu **yeni bir Money** vardır; eski Money ortadan kalkmaz, değişmez.

Pratik üç kazanç bunun yan ürünü:

- **Thread-safety.** Birden çok thread aynı Money'yi paylaşabilir, hiçbir senkronizasyon gerekmez. Banking gibi yüksek eşzamanlılıklı bir sistemde bu büyük avantaj.
- **Hash-based collection güvenliği.** Money'yi Map key veya Set element olarak kullanırsın. Mutable olsa `hashCode` sonradan değişir, collection nesneyi kaybeder. Immutable bunu yapısal olarak engeller.
- **Okuma kolaylığı.** "Acaba bu metot bana verdiğim Money'yi değiştirdi mi?" sorusu hiç çıkmaz. Cevap her zaman "hayır" — çünkü değiştiremez.

Banking'de başka Value Object örnekleri:

```java
record AccountId(UUID value) { }
record Currency(String iso4217Code) { }
record IBAN(String value) {
    public IBAN {
        if (!IbanValidator.isValid(value)) {
            throw new IllegalArgumentException("Geçersiz IBAN: " + value);
        }
    }
}
enum Direction { DEBIT, CREDIT }
```

`AccountId` UUID'yi sarıyor — tip güvenliği için. Bir metoda `AccountId` parametresi koyduğunda, yanlışlıkla `CustomerId` veremezsin; derleme zamanında yakalanır. `IBAN` constructor'ında format kontrolü yapıyor — geçersiz IBAN sisteme hiç girmez. `Currency` bir string'i değil ISO 4217 kodunu temsil ediyor.

Hepsinin ortak özelliği: **kuralın evidir**. Geçersiz IBAN, negatif tutar, farklı currency toplamak — hepsi Value Object'in içinde, constructor'da veya metotta yakalanır. Domain'in geri kalanı "Money geldi" gördüğünde geçerli olduğundan emin olabilir. Savunma kodunu her yere dağıtmak gerekmez.

Veritabanında Value Object'ler özel bir yer tutmaz. `Account.balanceCap` bir Money VO ise, `accounts` tablosunda `balance_cap_amount` ve `balance_cap_currency` aynı satırda iki kolon olarak yaşar. Ayrı tablo, join, foreign key yok — çünkü VO'nun bağımsız ömrü yok, parent'ıyla birlikte yaşar.

---

## Buraya kadar ne aldık?

İki kalıp ayırt etmeyi öğrendik:

- **Entity** — kimliği var, zaman içinde değişebilir, eşitlik kimlikten gelir. Account, LedgerEntry, Transaction.
- **Value Object** — kimliği yok, değişmez, eşitlik değerden gelir. Money, IBAN, AccountId, Currency.

Her nesneye bakarken artık sorabiliyoruz: "Bunu birebir kopyasıyla değiştirsem fark eder mi?" Cevap kalıbı söylüyor.

Bu bölümde Entity'nin **kimliğini** konuştuk ama davranışını değil. `account.freeze()`, `account.debit()` nereye, hangi kurallarla? Bunlar bir sonraki bölümün konusu — çünkü tutarlılık sınırını çizmeden "davranış nereye" sorusu eksik kalıyor. **Aggregate** işte tam o sınır. Tek başına yaşamayan nesneleri kim sahipleniyor, "Account'un içine `List<Transaction>` koyalım mı" sorusunun cevabı neden hayır — hepsi 2. bölümde.
