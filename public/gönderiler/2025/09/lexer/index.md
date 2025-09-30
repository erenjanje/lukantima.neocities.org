# Sıfırdan Derleyici 01: Lexer

Herkese merhabalar! YouTube'da tuhaf tuhaf oyun videoları çekmekten biraz sıkıldım ve karşınızda blog serisi: derleyici! Tam olarak ne derleyicisi olduğunu açıkçası ben de bilmiyorum, baştan düzgün bir derleyici yazmak olacak amacım daha çok. Yine de kafamda bazı nitelikleri mevcut bu dilin. Derleyicinin ilk sürümünü de Rust'ta yazmayı hedefliyorum. Bu sefer lafı dolandırmak istemiyorum pek, o nedenle başlayalım!

## Dilin Sözdizimi

Öncelikle sözdizimsel (_syntactic_) özelliklerinden bahsedelim bu dilin.

* Genel olarak C benzeri bir sözdizimi olmalı. Örneğin `int foo(int x)` C'deki gibi fonksiyon başlığı olmalı
* C'de bildiğiniz (veya bilmediğiniz) üzere gramer olarak birtakım belirsizlikler (_ambiguities_) mevcut. En basit örnek `(Foo)(x)` olur muhtemelen. Kodun öncesinde `Foo`'nun ne olarak tanımlandığına bağlı olarak bu ifade farklı şekillerde yorumlanmak zorunda. Eğer `Foo` bir tip ise bir değiştirme (_casting_) işlemi olarak, eğer `Foo` bir fonksiyon ise `x`'in argüman olarak verildiği bir fonksiyon çağrısı olarak yorumlanmalı. Bunu çözmek için [C3](https://github.com/c3lang/c3c) dilinin yaklaşımından yararlanıp tipleri her zaman büyük, diğer belirteçleri (_identifier_) her zaman küçük harfle başlamaya _zorlamayı_ düşündüm. Ayrıca kod tarzı açısından da faydalı olacağını düşünüyorum.
* Bunun yanında, C++ gibi `<>` işaretleri ile bir genel (_generic_) tip veya fonksiyon oluşturacaksak, burada da belirsizler oluşuyor. Bunun çözümlerinden birisi C++ gibi kod anlam kazanana kadar kodu incelemek fakat bu ayırıcımızı (_parser_) Arap saçına çevirmeye yol açar. Rust'ın bu konudaki çözümü mümkün olan yerlerde `<>` kullanırken belirsizlik olan genel fonksiyon çağrılarında `::<>` sözdizimini kullanmak. Java'nın çözümü ise `<>` ile fonksiyon çağrılarını fonksiyonun _öncesine_ yerleştirmek. Rust'ın yaklaşımı bence oldukça güzel, sadece ufak bir değişiklik ile bunu `.<>` yapmak bana daha güzel geldi. Bunun güzelliğini ilerleyen kısımlarda biraz daha iyi anlayabileceğiz.
* Tipler katı bir şekilde sonek (_postfix_) yöntemiyle oluşturulmalı. C'de bildiğiniz üzere "nasıl kullanıyorsan öyle tanımla" konsepti var, fakat bunun sonucu `void(*(*f[])())()` oluyor. Birçok dil bunu önek (_prefix_) ile çözüyor, mesela benzeri bir tip Go'da `var f []func() func() void` gibi bir şey oluyor. Rust gibi diller de çok farklı değil. Fakat ben bu sözdiziminin C tarzı sözdizimine uygun olmadığını düşünüyorum, nitekim C# ve Java gibi dillerde dizge (_array_) tanımlama gibi tip işlemleri sonek mantığıyla yapılıyor. Ben bunu biraz daha net hâle getirmek istiyorum. Bir tipin sonuna
    + `&` eklenirse değişmez (_const_) referans
    + `&mut` eklenirse değişir (_mutable_) referans
    + `[]` eklenirse kesit (_slice_)
    + `[n]` eklenirse `n` elemanlı bir dizge
    + `?` eklenirse seçmeli (_optional_)
    + `!T` eklenirse `T` ile sonuç (_result_) (burada `T` bir tip belirteci değilse paranteze alınmak zorunda)
    + `(...)` eklenirse daha önceki verilen tipi dönen ve `...` içerisindeki parametre türlerine sahip bir fonksiyon
olmasını istiyorum açıkçası. Kısacası yukarıdaki tip `void()()[] f` olmalı.
* Klasik operatörler olmalı, farklı olarak `xor`, `lsh`, `rsh` anahtar kelime şeklinde olmalı. XOR'u şapka ile göstermek oldum olası saçma gelmiştir, bir kaydırmaları da genel tip ve fonksiyonlarla çakışabileceğinden dolayı anahtar kelime oluyorlar.
* Dile `let` ve `var` anahtar kelimelerini sokmamak adına `=` ile atamayı değişmez, `<-` ile atamayı değişir atama olarak kullanmayı ve bu ikisinin karıştırılmasının yasak olmasını düşündüm.
* `arena` ve `thread_scope` gibi birkaç fazladan anahtar kelime güzel olur gibi.
* Önişlemci (_preprocessor_) yok, üstprogramlama (_metaprogramming_) şimdilik düşünmediğim bir kısım. Genel olarak tip sistemi ile sorunu çözmeyi planlıyorum. Önişlemci olmadığı için boşluk karakterleri (_whitespace_) tokenlerin sonunu belirtmek dışında hiçbir işe yaramamış oluyor.
* Sayılarda `u32`, `s64` ve `f32` gibi sonekler olabilmeli.
* Metinler (_strings_) C'deki gibi `"` arasında olmalı, ama `"`'ın kendisinin kaçış karakteri (_escape character_) başına `\` koymak değil `''` olmalı. Daha fazla sayıda tek tırnak yan yana koyulduğunda konulan tek tırnak sayısının bir eksiği kadar tek tırnak metne dahil edilmeli. Bunun dışındaki kaçış karakterleri aynı, yani yeni satır hâlâ `\n`.
* Lambda fonksiyonlar `->` ile oluşturulabilmeli.

Aslında özellikle gramer üzerine daha fazlası yazılır ama bu gönderi için bu kadarı fazla bile. Elimizde hangi tokenlerin olacağını gösteren bir liste olmuş oluyor kabataslak yani. Şimdi gelin önce bunları Rust'ta tanımlayalım:

## Tasarım

```rs
#[derive(Debug, Eq, PartialEq)]
pub enum Keyword {
    With,
    Struct,
    Enum,

    // C types
    Uchar,
    Ushort,
    Uint,
    Ulong,
    Ullong,
    Char,
    Short,
    Int,
    Long,
    Llong,

    // Constant-width integers
    Int8,
    Int16,
    Int32,
    Int64,
    Uint8,
    Uint16,
    Uint32,
    Uint64,

    Float,
    Double,
}

#[derive(Debug, Eq, PartialEq)]
pub enum Punctuation {
    LeftParen,    // (
    RightParen,   // )
    LeftBracket,  // [
    RightBracket, // ]
    LeftBrace,    // {
    RightBrace,   // }

    Comma,           // ,
    Dot,             // .
    QuestionMark,    // ?
    ExclamationMark, // !
    Ampersand,       // &
    Arrow,           // ->

    Equate, // =
    Assign, // <-

    // Arithmetic operators
    Plus,     // +
    Minus,    // -
    Asterisk, // *
    Slash,    // /
    Percent,  // %

    // Comparison operators
    Equals,            // ==
    NotEquals,         // !=
    LessThan,          // <
    GreaterThan,       // >
    LessThanEquals,    // <=
    GreaterThanEquals, // >=

    // Logical operators
    And, // &&
    Or,  // ||
    // Not, // !

    // Bitwise operatos
    // Band, // &
    Bor,  // |
    Bxor, // ^
    Bnot, // ~
}

#[derive(Debug, PartialEq)]
pub enum Lexeme<'a> {
    Eof,
    Integer(i64),
    UnsignedInteger(u64),
    Float(f64),
    String(String),

    Identifier(&'a str),
    TypeIdentifier(&'a str),
    Keyword(Keyword),
    Punctuation(Punctuation),
}
```

Burada çok fazla kod var gibi gözüküyor ama aslında hepsi bu bahsettiğim farklı token türlerinin tek tek tanımlanmasından ibaret. String sahip (_owned_) iken neden belirteçlerin sahipsiz olduğunu sorabilirsiniz, cevabı çok zor değil. Metinler veri olarak kaçışlandıktan sonraki hâlleriyle tutulmalı, dolayısıyla yeni bir metin oluşturmak gerekiyor. Fakat belirteçlerde bu yok, dolayısıyla onlar direkt kaynak koda âtıfta bulunabilir.

Bunların yanında bir tokenin kaynağın neresinde olduğunu göstermek için ufak bir yapı tanımlamak mantıklı olacaktır:

```rs
#[derive(Debug, Eq, PartialEq, Copy, Clone)]
pub struct SourcePosition {
    pub line: usize,
    pub column: usize,
    pub character: usize,
}

impl SourcePosition {
    pub fn new(line: usize, column: usize, character: usize) -> SourcePosition {
        SourcePosition {
            line,
            column,
            character,
        }
    }
}
```

`new` fonksiyonu çok önemli değil, sadece pozisyon bilgisi oluştururken -özellikle testlerde- kolaylık olsun diye koydum.

## Kod

Bir token oluşturucu yazmanın, kalıp eşleme (_pattern matching_) olan bir dilde ne kadar kolay olduğunu bu kodu yazarken öğrendim açık konuşmak gerekirse. Bu bahsettiğimiz özellikler genel olarak tek bir `match` ifadesinin (_expression_) farklı kollarına tekabûl ediyor. Fakat öncelikle gelin bu tanımlamaları birleştirelim:

```rs
#[derive(Debug, Eq, PartialEq)]
pub enum LexError {
    UnknownCharacter(char),
}

#[derive(Debug, PartialEq)]
pub struct Token<'a> {
    pub result: Result<Lexeme<'a>, LexError>,
    pub from: SourcePosition,
    pub to: SourcePosition,
}
```

Okuma işlemi sırasında karşılaşabileceğimiz sorunları `LexError` içerisine koyarsak bir `Token` bu şekilde tanımlanabilir. Ufak oluşurucular (_constructors_) da ekleyebiliriz bu `Token`e:

```rs
impl<'a> Token<'a> {
    pub fn token(result: Lexeme<'a>, position: SourcePosition, next: SourcePosition) -> Self {
        Self {
            result: Ok(result),
            from: position,
            to: next,
        }
    }

    pub fn error(result: LexError, position: SourcePosition, next: SourcePosition) -> Self {
        Self {
            result: Err(result),
            from: position,
            to: next,
        }
    }
}
```

Şimdi gelelim asıl olaya. Bunlar sadece elimizdeki gereksinimleri veri yapılarına dökmek oldu bir bakıma, şimdi sırada bu gereksinimleri gerçekleştirmek var.

Öncelikle formatımız şu şekilde:

* Genel olarak fonksiyonlar `source: &[char]` alacak. `&str` yerine `&[char]` almamızın sebebi, `&str` bir kesit olmadığından ötürü kendisinin kalıplarla kullanımının zor olması. Fakat çağıran fonksiyon bize kaynağı bu şekilde verebilir. Ki sonra bahsedeceğim gibi bunu da daha otomatik hâle getireceğiz.
* Kesitler hiçbir zaman değişmemeli/dönüşmemeli, pozisyon verisi üzerinden nerede olduğumuz yürümeli. Bunu kolaylaştırmak için şu şekilde bir fonksiyon da yazdım:

```rs
impl SourcePosition {
    fn apply<'a>(&self, s: &'a [char]) -> &'a [char] {
        &s[self.character..]
    }
}
```

O zaman öncelikle ilk fonksiyonumuzdan başlayalım: `skip_whitespace`.

```rs
impl<'a> Token<'a> {
    fn skip_whitespace(source: &[char], position: SourcePosition) -> SourcePosition {
        let mut position = position;
        for c in position.apply(source) {
            match c {
                &c if c.is_whitespace() => position = position.next(c),
                _ => break,
            }
        }
        position
    }
}
```

Aslında oldukça basit bir fonksiyon: boşluk karakteri olduğu sürece karakterler üzerinde dön ve her boşluk karakteri için pozisyonu sonraki pozisyona getir. Başka bir karakter görünce döngüden çıkıp pozisyonu dön. Bu ilk karakter de olabilir son karakter de, ama son karaktere kadar gitse bile ötesine geçmiyor iterasyondan kaynaklı olarak. Dolayısıyla sadece boşluktan oluşan bir metinde pozisyon doğrudan metnin bir sonraki karakteri olacaktır.

Burada asıl büyü bir bakıma `next`'te oluyor ama aslında o da çok basit bir fonksiyon:

```rs
impl SourcePosition {
    fn next(&self, c: char) -> SourcePosition {
        match c {
            '\n' => SourcePosition {
                line: self.line + 1,
                column: 1,
                character: self.character + 1,
            },
            _ => SourcePosition {
                line: self.line,
                column: self.column + 1,
                character: self.character + 1,
            },
        }
    }
}
```

Açıklaması yine basit: yeni satır olursa sonraki satıra geçerken diğer karakterlerde sütunu ilerletiyor. Karakter konumu ise sürekli olarak devam ediyor karakterin ne olduğundan bağımsız olarak.

Tam olarak şimdi token oluşturucumuzun kalbine geliyoruz:

```rs
impl<'a> Token<'a> {
    fn dispatch(source: &'_ [char], position: SourcePosition) -> Token<'_> {
        let (result, next) = match position.apply(source) {
            [] => (Ok(Lexeme::Eof), position),
            [c @ '(', ..] => (
                Ok(Lexeme::Punctuation(Punctuation::LeftParen)),
                position.next(*c),
            ),
            [c @ ')', ..] => (
                Ok(Lexeme::Punctuation(Punctuation::RightParen)),
                position.next(*c),
            ),
            [c, ..] => (Err(LexError::UnknownCharacter(*c)), position.next(*c)),
        };
        Token {
            result,
            from: position,
            to: next,
        }
    }
}
```

Açıkçası bu kod tam olarak tamamlanmış değil. Fakat güzel yanı, genişletmesi genel olarak oldukça kolay. Asıl sorunlu olan iki durum üstteki boş metin durumu ile en alttaki bilinmeyen karakter durumu. Geri kalanı bizim gerçek tokenlerimizi oluşturacak. Dediğim gibi, kalıp eşlemeli bir dilde bunu yapmak oldukça kolay oluyor. İlk karakter parantez açma ise, geri kalanı umursamayıp sol parantez tokeni döndürüyoruz. Elimizdeki tokenin konumunu bilmesi biraz değişik olabilir sadece. Bize en son gelen konum aslında başlangıç konumumuz oluyor, tokenin kendisini çözen `match` kolunun bildireceği yeni başlangıç konumu ise tokenimizin yarı açık sonu oluyor. Bu yeni başlangıç konumunu token vermeli çünkü tokenin uzunluğunu tokenin kendisini kıyaslarken ancak öğrenebiliriz. Parantezler tek karakter ama mesela belirteçler tek karakter değil, herhangi bir sayıda karakter olabilir. Kaç tane olduğunu da ancak tokeni öğrendikten sonra bilebiliriz. Dolayısıyla tokeni öğrendikten sonra yeni başlangıç konumunu hesaplamamız gerekiyor.

Bundan sonrası oldukça basit oluyor aslında. `parse` fonksiyonu şu oluyor:

```rs
impl<'a> Token<'a> {
    pub fn parse(source: &'_ [char], position: SourcePosition) -> Token<'_> {
        let position = Self::skip_whitespace(source, position);
        Self::dispatch(source, position)
    }
}
```

Son olarak da bunu bir iteratöre çevirelim. Bunun için bir yardımcı tip oluşturmamız gerekiyor ama:

```rs
pub struct Lexer<'a> {
    pub source: &'a [char],
    pub next: SourcePosition,
}

impl<'a> Lexer<'a> {
    pub fn new(source: &[char]) -> Lexer {
        Lexer {
            source,
            next: SourcePosition::default(),
        }
    }
}
```

Eof görene kadar devam etmesini istiyorsak (hataları da dönsün) şu şekilde bir iteratör oldukça yeterli olacaktır:

```rs
impl<'a> Iterator for Lexer<'a> {
    type Item = Token<'a>;

    fn next(&mut self) -> Option<Self::Item> {
        let result = Token::parse(self.source, self.next);
        self.next = result.to;
        if let Token {
            result: Ok(Lexeme::Eof),
            from: _,
            to: _,
        } = result
        {
            None
        } else {
            Some(result)
        }
    }
}
```

Kullanımı da oldukça basit oluyor nitekim:

```rs
for lex in Lexer::new("(".chars().collect::<Vec<_>>().as_slice()) {
    println!("{:?}", lex);
}
```

Kodun yarısı `&str`'yi `&[char]`'a çevirmek! Çıktımız da tam olarak beklenecek şekilde:

```
Token {
    result: Ok(Punctuation(LeftParen)),
    from: SourcePosition { line: 1, column: 1, character: 0 },
    to: SourcePosition { line: 1, column: 2, character: 1 }
}
```

## Sonuç

Bir token oluşturucu yazmak işte bu kadar kolay aslında! Fakat yine de ciddi eksiklikleri var bu kodun. Sayıları, metinleri ve belirteçleri henüz token hâline getirmiyor fakat eklemesi kolay aslında baya. Her birinin ayrı türden bir karakterle başlaması büyük nimet gerçekten.

Bir sonraki gönderide görüşmek üzere!

---
author: Erencan Ceyhan
lang: tr
tags:
- Rust
- Programlama
- Teknoloji
- Gönderi
- Derleyiciler
date: '[2025](/gönderiler/2025)-[09](/gönderiler/2025/09)-30 22:36:06'
---
