# C++ Şablon Bok Çukuru 2: ~~Elektrik Bugalu~~ CRTP

Uzun bir aradan (ödev gönderileri olarak 6 ay, normal gönderi olarak 1,5 yıl!) sonra tekrardan merhabalar. [Önceki gönderide](/gönderiler/2024/04/şablon-cehennemi) bahsettiğim gibi bu gönderide biraz CRTP, yani "İlginç Şekilde Mükerrer Şablon Kalıbı" (_Curiously Recurring Template Pattern_) konsepti üzerine yazmayı düşünüyorum. Her ne kadar "C++ Şablon Bok Çukuru"nu bir seri olarak yapmayı düşünmemiş olsam da benzer bir başlığı hak ettiğini düşünüyorum, çünkü gerçekten _tuhaf_ bir kalıp bu CRTP denen meret.

## Tekrar

Şablonları ufaktan tekrar hatırlatarak başlayayım öncelikle. Şablonlar, çıkış amacı gereği aslında aynı şablonun farklı türlere uygulanması üzerine kurulu bir konsept. Birçok dilde -Go'da bile- benzeri konseptler var fakat şablon ismi kullanılmıyor, sanırım çok C++ çağrıştırıyor ve diğer dillerdeki araçlar C++ şablonlarının çoğu özelliğine sahip olmuyor. Mesela derlenme zamanı istediğimiz bir ifadenin cevabını elde edebilmemizi sağlıyor şablonlar doğru kullanıldığında. Detaylar için önceki gönderiye bakabilirsiniz.

Şimdi elimizde bir şablon sınıf olsun (ben `public` yazmak zorunda kalmamak için genelde `struct` kullanıyorum ama sınıf/_class_ olarak kabul edebilirsiniz):

```cpp
template<typename T>
struct Snf {};
```

Bu tip şu anki hâliyle herhangi bir tipi şablon argümanı olarak alabilir, çünkü şablon parametresi `T` ile hiçbir şey yapmıyor.

```cpp
template<typename T>
struct Snf {
    T data;
};
```

Bu sefer şablonumuz aldığı türden bir veriyi kendi içinde depoluyor. Hâlâ her türlü tipi alabilir gibi gözüküyor olabilir, ama artık gerçekleştirilemeyen tipleri ne yazık ki alamıyor. Bu tarz tiplerin en bilindik örneği `void` olsa gerek. Sonuçta `void` türünden veri depolamak tamamen anlamsız.

```cpp
template<typename T>
struct Snf {
    T data;
    T plus(T const& other) {
        return data + other;
    }
};
```

Gittikçe daha spesifik tipleri almaya doğru gidiyoruz. Burada sadece `T` türünden başka bir değer ile toplanabilip sonuç olarak `T` türünden (veya `T`'ye çevrilebilen bir türden) değer dönen tipleri alabilir oluyor `Snf`. Buna `int` gibi aritmetik türlerin yanında `std::string` gibi türler de dâhil.

Bütün bunlar aslında aldığımız `T` türünün "arayüzünü" (_interface_) tanımlıyor. C++ şablonları -C++20'deki konseptlere kadar- ördekleme (_duck typing_) tabanlı olduğu için bu kısıtları tip sisteminde göstermiyoruz.

Bu arayüzler sayesinde derlenme zamanında çokbiçimlilik (_compile time polymorphism_) yapabiliyoruz. `plus` metodu verilen türe göre bir `add` yönergesine (_instruction_) derlenebildiği gibi (mesela `T` `int` olursa) herhangi bir aşırıyüklenmiş (_overloaded_) toplama operatörünü de kullanabilir, `T` `std::string` olursa olacağı gibi.

İşte CRTP de bunun bir formu denebilir.

## Çokbiçimlilik

C++'ta çokbiçimlilik genelde sanal (`virtual`) metotlar aracılığıyla gerçekleştirilir. Ders kitabımsı bir örnek vermek gerekirse:

```cpp
struct Animal {
    virtual void move(vec3 towards) = 0;
};

struct Dog : Animal {
    void move(vec3 towards) override {
        if(towards.y != 0) {
            throw std::runtime_error("Dogs cannot fly!");
        }
        // run the dog
    };
};

struct Bird : Animal {
    void move(vec3 towards) override {
        // fly the bird
    }
};
```

Burada `Animal` sınıfı içinde `move` sanal metodunu "saf sanal" (_pure virtual_) olarak tanımladık ki bütün altsınıflar (_subclasses_) bu metodun üzerine yazmak zorunda kalsın. Bir hayvanın nasıl hareket edeceğini hayvanın kendisi bilir sonuçta sadece. Bu durum aslında bize bir arayüz oluşturuyor. Bütün hayvanların, yani `Animal` sınıfının bütün altsınıflarının sahip olmak _zorunda_ olduğu bazı metotlar var olmuş oluyor. Dinamik çokbiçimlilik (_dynamic polymorphism_) veya çalışma zamanı çokbiçimliliği (_runtime polymorphism_) oldukça güzel, fakat bunun bir bedeli de oluyor. Bu sınıfların hepsi birer sanal metot tablosuna (_virtual method table_) sahip olmak zorunda kalıyor. Ayrıca her `move` çağrısı (derleyici optimizasyonları bazen elimine edebilse de genel olarak) bu tabloya bakıp dolaylı çağrı (_indirect call_) yapmak zorunda. Her ne kadar günümüzdeki bilgisayarlarda bunun maliyeti _çoğunlukla_ (ama her zaman değil) ihmal edilebilir olsa da yine de bir maliyet. Peki bu maliyetinin karşılığında bize ne sağlıyor?

- Bütün hayvanlar hareket etme metodunu tanımlamak zorunda kaldığı için bir arayüz oluşturuyor.
- Bu arayüz üzerinden taban sınıfta (_base class_) yani `Animal`'da ekstra özellikler ekleyebiliriz. Mesela "A'dan B'ye git" metodu Dijkstra algoritması çalıştırarak o hayvanın A'dan B'ye en kolay nasıl gideceğini hesaplayabilir ve hayvanı uygun şekilde hareket ettirebilir. Bu kavrama _mixin_ (Türkçeye çeviremedim) deniyor genelde. 
- Farklı türden hayvanlar aynı `Animal` işaretçi/referansları tarafından depolanabilir oluyor (doğrudan `Animal` olarak depolanırsa nesne kesimi \[_object slicing_\] olur).

Aslında bu faydaların sadece üçüncüsü çalışma zamanı gerektiriyor. İlkini C++20'deki konseptler (`concept`) sağlıyor, veya önceki sürümlerde SFINAE (şu ikisinden de beter ayrı bir bok çukuru) veya `static_assert` kullanarak oluşturabiliyoruz. Benim şu an odaklanmak istediğim kısım ise ikincisi. Eğer ihtiyacımız olan temel kabiliyet mixinler ise sanal metot bedelini ödememizin pek bir anlamı yok. Peki bunu nasıl yapabiliriz? Sanal metotlar olmadan bir sınıf kendi altsınıflarının metotlarını çağıramaz, değil mi?

## Derleme Zamanı Çokbiçimliliği

### Mixinler

Sanal metotların aksine deleme zamanı çokbiçimliliği tam olarak C++ dilinin bir parçası değil. Nitekim bu nedenle adı _İlginç Şekilde Mükerrer Şablon Kalıbı_. Dilin bir özelliği olmamasına rağmen sıkça karşılaşılan bir sorunu çözdüğü için birbiriyle ilişkisiz insanların farklı şartlarda kendi kendilerine icat edip kullandığı bir tür "tasarım kalıbı" (_design pattern_) olarak karşımıza çıkıyor. Peki ne bu CRTP? Bir kod bin resme değer:

```cpp
template<typename Child>
struct Parent {};

struct Child : Parent<Child> {};
```

Bi' dakka bi' dakka bi' dakka. Altsınıf üstsınıfa şablon argümanı olarak _kendisini_ veriyor? Görünüşe bakılırsa dilin tasarımcılarının pek de isteyerek yapmadığı şekilde bir sınıf kendi üstsınıflarını belirtirken aslında sınıfın kendisi de tanımlı. Bunu mümkün kılan şey ise şablon gerçekleşmesinin (_instantiation_) tembel (_lazy_) olması. Yukarıdaki örnekte zaten çok sıkıntı yok, sonuçta şu hâliyle `Parent` her türlü tipi (`void` gibi gerçekleştirilemeyenler de dâhil) şablon argümanı olarak alabiliyor. Nitekim `Parent`'ın içerisinde `Child` gerçekleştirilebilir bir tip değil. Fakat şablon metotlarının derlenmesi, bu metotlar çağrıldığında oluyor ve bu metotların çağrılması neredeyse her zaman başka dış bir kod tarafından çok daha sonra gerçekleştiriliyor, dolayısıyla `Parent`, `Child`'ın metotlarını kullanabiliyor!

```cpp
template<typename Child>
struct Parent {
    void foo() {
        static_cast<Child*>(this)->bar();
    }
};

struct Child : Parent<Child> {
    void bar() {
        std::cout << "zartzurt" << std::endl;
    }
};
```

`Parent`'ın `bar` diye bir metodu yok, fakat bizzat _kendisini_ kendi altsınıfı olarak yorumlayıp (`static_cast`'in yaptığı) altsınıfının metodunu çağırabiliyor. Bu elbette bir arayüz oluşturuyor, artık `Parent`'tan kalıtan (_inherit_) bütün sınıflar `bar` metodunu oluşturmak zorunda. Yani üstte sanal metotlar ile yaptığımız arayüz kavramına benzer bir şeyi burada da yapmış oluyoruz. Fakat sanal metotların aksine burada herhangi bir dolaylı çağrı yok. Kod derlenirken derleyici hangi `bar`'ın çağrılacağını biliyor: `Child::bar`. Dolayısıyla `foo` fonksiyonu sadece tek bir doğrudan çağırmadan ibaret bu durumda ki bu da fazlasıyla ucuz bir işlem dolaylı çağrıya kıyasla.

Buradaki asıl güç ise mixinlerde aslında. Çok saçma bir kod olacak ama şu şekilde bir sınıf grubu düşünebiliriz denemek için:

- Elimizde kütük yazan (_log_) bir ana sınıf olmalı. Bu ana sınıf aldığı mesajı altsınıfa iletmeli ve onun ürettiği yazının başına `"[LOG]: "` eklemeli.
- Bilgi (`Info`) ve Hata (`Error`) diye iki altsınıf mesajın başına sırayla `"[INFO]: "` ve `"[ERR]: "` yazılarını eklemeli.

Önce bunu sanal metotlar ile yapalım isterseniz:

```cpp
struct Logger {
    void log(std::string const& message) {
        std::cout << "[LOG]: " << this->getLogString(message) << std::endl;
    }

    virtual std::string getLogString(std::string const& message) = 0;
};

struct Info : Logger {
    std::string getLogString(std::string const& message) override {
        return "[INFO]: " + message;
    }
};

struct Error : Logger {
    std::string getLogString(std::string const& message) override {
        return "[ERR]: " + message;
    }
};
```

Artık `Info` ile de `Error` ile de `log` metodunu çağırdığımızda bize uygun şekilde başına LOG diye etiketlenmiş şekilde yazı yazacaktır. Fakat bu işlem sanal metotlar üzerinden döndüğü için çalışma zamanında bir maliyeti var. Dediğim gibi bu maliyeti optimizasyonlar elimine edebilir, ama optimizasyonlara güvenmek çok da akıl kârı bir iş değil. Nitekim her şekilde o sanal tablo bedelini ödemek zorundayız. Nesnelerimiz küçükse (ki burada başka elemanları olmadığı için fazlasıyla geçerli) bu nesne başına 4 (32-bit sistemlerde) veya 8 (64-bit sistemlerde) byte kadar bir fazlalık demek. İsterseniz şimdi de CRTP uygulayalım:

```cpp
template<typename Child>
struct Logger {
    void log(std::string const& message) {
        std::cout << "[LOG]: " << static_cast<Child*>(this)->getLogString(message) << std::endl;
    }
};

struct Info : Logger<Info> {
    std::string getLogString(std::string const& message) {
        return "[INFO]: " + message;
    }
};

struct Error : Logger<Error> {
    std::string getLogString(std::string const& message) {
        return "[ERR]: " + message;
    }
};
```

Bu kodda herhangi bir dolaylı çağrı olmadığı gibi, sanal tablo bedeli de mevcut değil ve yine aynı şeyi yapıyor aslında: ana sınıf altsınıflarının yazacaklarının başına LOG etiketini yazıyor. Elbette bununla yapamayacağımız şeyler var, mesela `Logger` artık tek başına kullanılabilir bir sınıf değil. Çözüm olarak genelde `Logger`'a da bir üstsınıf koyulur ve `log`'un kendisi sanal metot hâline getirilir. Elbette bu kazancımızı yok etmiş olur fakat bazı durumlarda (mesela `log`'un `getLogString`'i tekrar tekrar çağırması) işimize yarayabilir. Ama günün sonunda yaptığımız şey aslında mixin olmuş oluyor, artık `Info` da `Error` da `log` metotlarını doğru çalıştırıyor ve `Logger`'ın eklemeleri tamamen statik, derleme zamanında gerçekleşiyor.

### Akışkan Arayüz (_Fluent Interface_)

Akışkan arayüz, sınıf metotlarının genel olarak kendini dönmesi oluyor. Özellikle sınıfların atayıcıları (_setters_) bunu kullanmaya müsait olur. Mesela

```cpp
auto a = MessageBox();
a.set_title("zort");
a.set_message("zurt");
a.show();
```

kodunda görebileceğiniz üzere `a`'yı sürekli tekrarladık. Oysa aynı sınıf üzerinde bütün bu işlemleri yapıyoruz, ama bütün bu metotlar `void` dönüyor, yani dönüş değerleri boşta bekliyor! Akışkan arayüz olduğu takdirde şöyle bir kod mümkün oluyor:

```cpp
MessageBox()
    .set_title("zort")
    .set_message("zurt")
    .show();
```

Eğer daha sonra kullanmayacaksak değişken tanımlamaya bile gerek kalmıyor!

Gelin [IUP](https://www.tecgraf.puc-rio.br/iup) grafik arayüz kütüphanesinin `VBox` sınıfı için bir C++ kaplayıcısı (_wrapper_) yazalım. Normalde bu kütüphane C'de yazılmış ve C üzerinden iletişim kuruyor, ama C++ kaplayıcısı yazmak zor değil. Ayrıca elementlerin nitelikleri (başlık, yazı, boyut vb.) metin olarak yazıldığı için fonksiyon yelpazesi de oldukça ufak, sadece "IupSetAttribute" ile aşağı yukarı her türlü nitelik uygun şekilde değiştirilebiliyor. Bir de sadece kaplar (_containers_) için geçerli olan bir `IupAppend` fonksiyonu var, kabın içine yeni bir element yerleştiriyor. Basit bir şekilde düzenlemek gerekirse şu şekilde bir sınıf hiyerarşisi oluşuyor:

```cpp
struct Element {
    Element(Ihandle* handle) : handle(handle) {}

    void set_attribute(char const* name, char const* value) const {
        IupSetAttribute(handle, name, value);
    }
    Ihandle* handle;
};

struct Container : Element {
    using Element::Element;

    void append(Element const& elem) {
        IupAppend(handle, elem.handle);
    }
};

struct VBox : Container {
    VBox() : Container(IupVbox(nullptr)) {}
};
```

Burada `using Element::Element` ile `Element`'in oluşturucusunu (_constructor_) `Container`'a da eklemiş olduk. Artık şu şekilde kullanabiliriz VBox'ımızı:

```cpp
auto v = VBox();
v.set_attribute("GAP", "4");
v.append(VBox());
```

Şimdi bunu akışkan arayüze çevirmeyi deneyelim isterseniz:

```cpp
struct Element {
    Element(Ihandle* handle) : handle(handle) {}

    Element const& set_attribute(char const* name, char const* value) const {
        IupSetAttribute(handle, name, value);
        return *this;
    }
    Ihandle* handle;
};

struct Container : Element {
    using Element::Element;

    Container const& append(Element const& elem) {
        IupAppend(handle, elem.handle);
        return *this;
    }
};

struct VBox : Container {
    VBox() : Container(IupVbox(nullptr)) {}
};
```

Ama şu şekilde kullanamayız:

```cpp
VBox()
    .set_attribute("GAP", "4")
    .append(VBox());
```

çünkü `set_attribute` bir `Element` dönüyor ve `Element`'lere `append` edilemiyor! Eğer `set_attribute` bir `Container` dönerse bu sefer de `Container` olmayan elementleri tanımlarken `Element`'i kullanamayız. Ayrı bir `Element` oluşturmamız gerekir. İşte bunu derleyicinin bizim için yapmasını CRTP ile sağlayabiliriz!

```cpp
template<typename Child>
struct Element {
    Element(Ihandle* handle) : handle(handle) {}

    Child const& set_attribute(char const* name, char const* value) const {
        IupSetAttribute(this->handle, name, value);
        return *static_cast<Child const*>(this);
    }
    Ihandle* handle;
};

template<typename Child>
struct Container : Element<Child> {
    using Element<Child>::Element;

    template<typename OtherChild>
    Child const& append(Element<OtherChild> const& elem) const {
        IupAppend(this->handle, elem.handle);
        return *static_cast<Child const*>(this);
    }
};

struct VBox : Container<VBox> {
    VBox() : Container(IupVbox(nullptr)) {}
};
```

Bir iki `this` ekledim derleyici `handle`'ı doğrudan bulamadığı için, ayrıca `append` de artık ayrı bir şablon sınıf oldu (çünkü elementlerin ana sınıfı artık farklı olabilir). Bunun çözümü olarak yukarıda bahsettiğim gibi `Element`'in de üstsınıfı olabilir. Bu sefer sanal tabloya vs. de ihtiyacımız yok, en üst ana sınıf sadece `handle`'ı tutabilir. Diğer sınıflar da sadece metotları eklemiş olur.

```cpp
auto v = VBox();
v
    .set_attribute("GAP", "5");
    .append(VBox());
```

Evet artık bu yukarıdaki kod çalışıyor!

## Kapanış

Bu sefer, önceki kadar derin bir bok çukuru olmasa da yine de tuhaf bir şeylerden bahsettim. Bir seneden daha fazla zaman geçmesine rağmen dediğimi yaptım sanırım. Bu arada yaşadıklarım da artık başka gönderinin konusu olsa gerek.

---
author: Erencan Ceyhan
lang: tr
tags:
- C++
- Programlama
- Teknoloji
- Gönderi
- Üstprogramlama
date: '[2025](/gönderiler/2025)-[07](/gönderiler/2025/07)-19 18:27:52+03:00'
---
