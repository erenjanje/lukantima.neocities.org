# C++ Şablon Bok Çukuru

Şu aralar C++ ile bir Entity Component System sistemi yapmakla uğraşmaktayım. Entity Component System'i (kısaca ECS) açıklamam gerekirse, genellikle oyunlarda kullanılan bir sistemdir ve temel amacı, oyundaki her bir varlığı (_Entity_) sadece veri içeren birden fazla bileşenden (_Component_) oluşturmak ve bu bileşenler üzerinde dönen sistemler (_System_) ile oyunu ayağa kaldırmaktır. Ana iki faydası ise sistemlerin, ardışık bileşenler üzerinde dönmesinden kaynaklı olarak işlemcinin önbelleğini oldukça verimli bir şekilde kullanması ile oyundaki birçok kavramın birbirinden bağımsız hâle gelmesidir -ki bu da oyunun kodunun anlaşılabilirliği için oldukça faydalıdır-. Bu konu üzerine konuşabileceğim şeyler var ama bu gönderinin konusu bu değil. Bu sistemin işinin çoğunu (mesela bileşen listeleri gibi) yürütme değil derleme zamanında gerçekleştirmek istemem sonucu olarak düştüğüm -affedersiniz- bok çukurunu anlatmak istiyorum bugün.

C++ şablonlarından (`template`) bahsetmem gerekiyor ilk olarak. C++ kullanmış aşağı yukarı herkes ama az ama çok şablonları görmüş olsa gerekir. İlk başta dile eklenme amacı, farklı türler üzerinde çalışması gereken ve bu türlerin hepsine aynı davranan bilimum veri yapıları ve algoritmaları yazarken kolaylık sağlamaktır. Klasik örneği vermek gerekirse:

```cpp
std::vector<int> v = {};
v.push_back(5);
v.push_back(10);

std::vector<double> c = {};
c.push_back(3.14);
c.push_back(2.71);
```

Burada `std::vector` bir şablon tip (_template type_) olduğundan dolayı farklı tiplerde veriyi depolayabileceğimiz varyantlarını istediğimiz gibi oluşturabiliyoruz. Örnekte `int` ve `double` tiplerinden veri tutabiliyor vektörlerimiz. Şablonların temel niteliklerinden birisi farklı tip parametresi (_type parameter_) almış tiplerin birbiriyle varsayılan olarak uyumsuz olmasıdır -yani `v = c` gibi bir ifade yasaktır-. Bu da dilin tip güvenliği açısından önemlidir. Keza bu vektörler sadece belirtilen tiplerden veri depolayabilir, aynı vektörde farklı tiplerde veriler depolamak için daha farklı tip parametreleri kullanmak gerekir.

## Şablonlar

Buraya kadar hava hoş. Şablonlar sayesinde hem tip güvenliğini ihlal etmiyoruz (C'de benzer bir vektör türü için `void*` kullanmamız gerekli ve bu da vektörün hangi tip veri depoladığı bilgisini kaybetmemiz demek, bir bakıma tip silme yani _type erasure_ uyguluyoruz, bunu yöntemi kullanan diller mevcut, o ayrı mesele) hem de kodu tekrarlamamız gerekmiyor (C'de makro cehennemine girmeden vektör yapmaya kalkışırsak vereceğimiz her tip parametresi için ayrı vektör yazmamız gerekir, bu da takdir edersiniz ki _çok_ koddur). C++'ın bir dönemler çağdaşı dillere göre en büyük üstünlüklerinden birisi bu şablon kullanımı. Ki derlenen kod da elle yazılmış gibi hızlı, yani dizilerin/vektörün işlenme şeklinde temel değişiklikler yapmadan `std::vector`'den daha hızlı bir dizi sistemi yapamayız. C++'ın çok sayıda bedelsiz soyutlamasından (_zero-cost abstraction_) birisi bu şablonlar. Dediğim gibi buraya kadar her şey mükemmel.

Ancak şablonlar sadece bu amaçla kullanılabilen şeyler değiller. Öncelikle şunu söylemem gerekir: C++'ın şablon sistemi Turing-complete'tir (bunu çeviremedim). Elbette bu hâliyle yapabileceklerimiz sınırlı. Fakat buna sebep olan birtakım ekstra özellik mevcut C++'ın şablon sisteminde -özellikle de yeni sürümlerinde-.

## Turing-complete Şablonlar

### Değişken Sayıda Argüman (_Variadic Arguments_)

C++ şablonlarını tanımlamaya ufaktan bakalım:

```cpp
template<typename T>
class Tip {/* ... */};
```

Buradaki `Tip`, sadece tek bir şablon parametresi alıyor: `T`. `typename` ise bu `T`'nin bir tip olduğunu belirtiyor. Bu tipi kullanırken `Tip<int>` gibi kullanabiliriz anlamına geliyor bu. Fakat C++'ta şablonlar C++11 standardından beridir değişken sayıda argüman da alabilir.

```cpp
template<typename... Ts>
class Tip {/* ... */};
```

Burada `Ts` aslında -boş olma ihtimali olan- bir parametre listesine tekabül ediyor. Artık `Tip<int>` dışında `Tip<int, float>` veya `Tip<int, float, std::string>` de verilebilir oluyor. Ancak bu `Ts` şu anki hâliyle kullanılabilir değil. `Ts` üzerinde `for` döngüsü dönemiyoruz herhangi bir şekilde. İşte bu noktada şunu söylemek gerekir: C++ şablonları _fonksiyonel programlama_ mantığıyla çalışır. Turing-complete derken aslında Turing makinesi usûlü döngüler ve şartlarla değil, Lambda kalkülüsü usûlü özyineleme ve desen eşleme (_pattern matching_) ile düşünmek gerekir. Birden fazla şablon tanımlayacağımız anlamına geliyor bu.

```cpp
template<typename... Ts>
class Tip {};
template<typename T, typename... Ts>
class Tip<T, Ts...> {};
template<>
class Tip<> {};
```

İkinci ve üçüncü tanımlarda değişik durumlar var görebileceğiniz üzere.

#### Eksik ve Tam Özelleştirmeler (_Partial_ and _Full Specializations_)

Üstteki kodda gerçekleşen şey <em>özelleştirme</em>lerdir.

İlk tanım aslında ana şablon (_primary template_), yani o şablonun genel tanımı, diğer özelleştirmeler tutmadığı takdirde kullanılacak olan tanımı, yani temel durum (_base case_).

İkinci tanım ise eksik özelleştirme (_partial specialization_). Eksik olmasının sebebi, kendisinin de şablon argümanına sahip olması (üstünde `T` ve `Ts` argümanları görülebiliyor). En az bir parametre içeren bütün `Tip` kullanımları bu tanımı kullanacaktır, Yani `Tip<int>`, `Tip<float>`, `Tip<int, float>`, `Tip<int, float, std::string, std::queue<long>>` gibi örneklerin hepsi bu tanımı kullanacakken `Tip<>` bu tanımı kullanmayacaktır (şablon parametresi olmadığı, dolayısıyla `T`'ye denk gelen bir parametreye sahip olmadığı için).

Üçüncü tanım ise bir tam özelleştirme (_full specialization_). Tam olmasının sebebi herhangi bir şablon parametresi almıyor olması, görebileceğiniz üzere `template` ifadesinden sonraki liste boş. Bunun anlamı, bu özelleştirmeye denk sadece bir tane olası parametre listesinin bulunmasıdır -`Tip<>` dışındaki hiçbir kullanım üçüncü tanımı kullanmayacaktır-.

Burada görebileceğimiz üzere aslında yazdığımız iki özelleştirme, bütün olası `Tip` kullanımlarını karşılıyor, parametre verilirse ikinci, verilmezse üçüncü tanım kullanılıyor. Haskell gibi desen eşleştirme destekleyen bir dil kullanmışsanız bunun şundan çok farklı olmadığını fark etmişsinizdir:

```hs
tip (t:ts)
tip ()
```

Ancak Haskell'de yazmak pek mümkün değil çünkü şu anda bu argümanlarla hiçbir şey yapılmıyor, yani `Tip`'in içerisinde hiçbir şey mevcut değil. Gelin `head` işlevini tanımlayalım bu şekilde, yani listenin ilk parametresini dönsün. Tabii ki şablonlarda "dönmek" kavramı yok, biz de bunun yerine döndüreceğimiz tipi tanımlayacağız.

```cpp
template<typename...>
class Head {};
template<typename T, typename... Ts>
class Head<T, Ts...> {
public:
   using type = T;
};
template<>
class Head<> {};
```

Artık `Head<int, float>::type` dediğimizde `int` anlamına geliyor. Ancak fark etmişsinizdir ki `Head<>::type` diye bir tip yok. Boş bir listenin head'i olması saçma olurdu keza. Haskell'de benzeri bir tanım yapmamız gerekirse şöyle olabilir

```hs
head (x:xs) = x
```

Boş tuple (_demet_ diye çevrilmiş de bana bile tuhaf geldi) desenine (`head ()`) fonksiyon tanımlamadım çünkü öyle bir durum/desen geldiği takdirde hata verilmesi gerekmeli, keza Haskell'in kendi `head`'i de bu şekilde çalışıyor. Aradaki en önemli fark hata mesajları olsa gerek, "`type`, `Head<>`'in bir elemanı değil" gibi sorunun kendisini belirtmeyen hata mesajları ile karşılaşmak oldukça kolay. Bazı durumlarda `static_assert` ile daha düzgün hata mesajları verilebilir fakat boş listede herhangi bir argüman olmadığı için _assert ettiğimiz_ ifadeyi dolaylayamıyoruz, dolayısıyla da boş `Head` olmasa dahi `static_assert` çalışıp kodumuzun derlenmesini önlüyor.

`Head`'de bahsetmek istediğim ufak bir kısım ise `using`. Anladığınızı düşünüyorum ama yine de anlatmam gerekirse yaptığı şey, o isim alanı içerisinde (`namepsace` veya `class`/`struct`) bir tipe başka bir isim vermek. C'deki `typedef`'in daha da güçlü hâli, nitekim kendisi de şablon olabiliyor.

### Tamsayı Argümanlar

Desen eşleme yetmediği gibi tamsayı parametre verebiliyoruz şablonlara. Bunun için argüman listesinde yapmamız gereken tek değişiklik `typename` yerine `int` -veya herhangi bir tamsayı türü- yazmak.

#### Derlenme Zamanı Değişkenler

C++11 ile hayatımıza `constexpr` diye bir ifade girdi. Bir değişken `constexpr` olarak tanımlandığı takdirde derlenme zamanında erişilebilir olur ve şablonlara parametre olarak verilebilir hâle gelir. `using`'in tip değil değere dönüşen hâli olarak düşünebiliriz `constexpr` değişkenleri. O zaman derleme zamanında faktoriyel hesaplayabilir anlamına geliyor bu!

```cpp
template<int N>
struct Factorial {
   static constexpr int value = N * Factorial<N-1>::value;
};
template<>
struct Factorial<0> {
   static constexpr int value = 1;
};
```

Haskell'de aynı fonksiyonu yazmak istesek şuna benzer bir şey olurdu:

```hs
factorial n = n * factorial (n - 1)
factorial 0 = 1
```

Görebileceğiniz üzere çok daha kalabalık olması haricinde C++ şablonları ile Haskell büyük ölçüde eşleşiyorlar mantık olarak.

## Şablon Cehennemi

Benim bu aralar cebelleştiğim şey ise bir tip listesi yaratmak. Bu tip listesi basitçe

```cpp
template<typename... Ts>
struct List {};
```

olarak tanımlanmış durumda. Bunun elbette çoğu işlevi için iki olasılık var: boş liste ve elemanlı liste.

```cpp
template<typename T, typename... Ts>
struct List<T, Ts...> {};
template<>
struct List<> {};
```

Gelin bu listenin birkaç özelliğini nasıl yaptığımı göstereyim.

```cpp
template<typename T, typename... Ts>
struct List<T, Ts...> {
   using head = T;
};
template<>
struct List<> {};
```

Daha önce de bahsettiğim gibi, `head` işlevi boş listede tanımlı değil.

```cpp
template<typename T, typename... Ts>
struct List<T, Ts...> {
   using tail = List<Ts...>;
};
template<>
struct List<> {
   using tail = List<>;
};
```

Boş listenin `tail`'ı da boş liste. Bunu hata olarak da belirtebilirdik tabii.

`at` işlevini oluşturmak biraz daha sıkıntılı. Ayrı bir "işlev şablonu" oluşturmak gerekiyor `using` ile tanımlanmış şablonları özelleştiremediğimiz için.

```cpp
template<int index, typename T>
struct At {};
template<typename T, typename... Ts>
struct At<0, List<T, Ts...>> {
	using type = T;
};
template<int index, typename T, typename... Ts>
struct At<index, List<T, Ts...>> {
	using type = At<index - 1, List<Ts...>>::type;
};

template<typename T, typename... Ts>
struct List<T, Ts...> {
	template<int index>
	using at = At<index, List<T, Ts...>>::type;
};
template<>
struct List<> {};
```

Bu örnek çok kalabalık görünüyor olabilir -ki öyle- ama parça parça gidersek daha anlaşılır olacağını düşünüyorum.

İlk başta `At` diye bir şablon tanımladık, bir tamsayı ve bir tip alıyor ve hiçbir şey içermiyor. Sonrasında ise eleman içeren bir listede `index`'in 0 olduğu durumu özelleştirdik. Bu durumda baş eleman sonucumuz olduğu için `type` tipini başa (`T`'ye) eşitledik. Sonrasında ise diğer olasılığa baktık yani listenin kalanında aramaya (elbette `index`'i 1 azaltarak). `List`'in kendisinde ise basitçe bu şablonu uygun parametrelerle "çağırdık".

Haskell'deki karşılığı şu şekilde olur (artık ayrı bir `List` olduğu için tuple değil Haskell listeleri ile fonksiyonu tanımlayabiliriz, çok da fark eden bir şey değil tabii).

```hs
at 0 [t:ts] = t
at index [t:ts] = at (index - 1) ts
```

`find`, bir değer (tamsayı) döndüren bir işlev ama yapılma şekli çok da farklı değil diğerlerine kıyasla.

```cpp
template<typename Q, int index, typename T>
struct find {};
template<typename Q, int index>
struct find<Q, index, List<>> {
	static constexpr auto value = -1;
};
template<typename Q, int index, typename... Ts>
struct find<Q, index, List<Q, Ts...>> {
	static constexpr auto value = index;
};
template<typename Q, int index, typename T, typename... Ts>
struct find<Q, index, List<T, Ts...>> {
	static constexpr auto value = find<Q, index + 1, List<Ts...>>::value;
};

template<typename T, typename... Ts>
struct List<T, Ts...> {
	template<typename Q>
	static constexpr auto find = ::find<Q, 0, List<T, Ts...>>::value;
};
template<>
struct List<> {
   template<typename Q>
   static constexpr auto find = -1;
};
```

Ancak burada "kuyruk özyinelemesi" (_tail recursion_) kavramını kullandım diğerlerinden farklı olarak. Eğer diğer türlü olsa idi indisin -1 olma durumunda (yani verilen tipin bulunamaması durumunda) sonucun -1 olarak kalması için başka uğraşmak zorunda kalacaktım. Haskell'de yazarsak da şu şekilde oluyor:

```hs
find q index [] = -1
find q index [q:ts] = index
find q index [t:ts] = find q (index + 1) ts
```

Son olarak da fonksiyonel programlamanın en temel işlevlerinden olan `map`'i göstermek istiyorum.

```cpp
template<typename Appended, typename T>
struct Append {};
template<typename Appended, typename... Ts>
struct Append<Appended, List<Ts...>> {
	using type = List<Ts..., Appended>;
};

template<template<typename, typename...> typename F, typename Ret, typename T>
struct map {};
template<template<typename, typename...> typename F, typename Ret>
struct map<F, Ret, List<>> {
	using type = Ret;
};
template<template<typename, typename...> typename F, typename Ret, typename T, typename... Ts>
struct map<F, Ret, List<T, Ts...>> {
	using type = map<F, Append<F<T>, Ret>, List<Ts...>>::type;
};

template<typename T, typename... Ts>
struct List<T, Ts...> {
	template<template<typename, typename...> typename F>
	using map = typename ::map<F, List<>, List<T, Ts...>>::type;
};
template<>
struct List<> {
   template<template<typename, typename...> typename F>
   using map = List<>;
};
```

Aslında bu `map`'ten çok `wrap`'e denk geliyor. Bu işlev aslında benim bu şablon batağına düşmemin temel sebebiydi diyebilirim (bir tip listesinin her elemanı için ayrı birer `std::vector`). Buradaki en fantastik kısım ise muhtemelen `template<typename, typename...> typename F` kısmı. Bu, "`F` en az bir şablon parametresi alan bir şablondur" anlamına geliyor. Eğer şablonlar işlevler ise (ki bizim kullanımımızda öyle çalışıyorlar) bu kullanım sayesinde "yüksek dereceli çeşit"leri (_higher order kind_) göstermek mümkün hâle geliyor. Burada ufak bir `Append` tanımlamak zorunda kaldım çünkü aynı anda hem `Ret`'i hem `T`'yi (`List`'imiz) bölemiyoruz. Haskell'de bu `Append` önceden `++` işelci olarak tanımlı ama.

Haskell'deki karşılığı da şu oluyor:

```hs
map f ret [] = []
map f ret [t:ts] = map f (ret ++ [(f t)]) [ts]
```

## Kapanış

C++ şablonları bir bok deliğidir. Diğer dillerin çoğunda pek mümkün olmayan ya da derleme değil yürütme zamanında ancak mümkün olan şeyleri yapabilmemizi sağlaması ise oldukça güçlü bir nitelik kılar şablonları. Zig, Rust ve Nim gibi daha modern dillerde veya Lisp gibi başından beri üstprogramlama (_metaprogramming_) düşünülmüş diller dışındaki dillerde derleme zamanında işlem yapmak oldukça sıkıntılı iken C++'ta her ne kadar üstprogramlama sonradan eklenen bir özellik olsa da güçlü ve hünerli biçimde yer almakta. Bu bok deliğine düşmemek diğer insanlar için çok zor değil muhtemelen, insanlar her yere `virtual` koyup geçiyor çoğunlukla veya C++'ı "Sınıflı C" (_C with classes_) olarak kullanıyor. Fakat benim gibi birine bu şablon imkanlarını verirseniz (ki sayıca o kadar az değiliz) şablonları istismar etmekten kaçınmayacaktır.

Şablonların oluşturduğu hiç de zarif olmayan fonksiyonel üstprogramlama dili için düşünmenin en kolay yolunun Haskell gibi bu amaç için tasarlanmış saf bir fonksiyonel programlama dili olduğu kanaatindeyim. Nitekim fark etmişsinizdir ki Haskell kodu ile C++ şablonu neredeyse birebir eşleşmekte ama C++ şablonunu görmek, algılamak, çözümlemek, yazmak ve okumak çok daha zor. Bunun bir "algoritma"sını (tırnak içinde çünkü muhtemelen katı bir algoritmadan ziyade insana yönelik bir tarif olacak) oluşturmayı düşünmüyor değilim açıkçası.

Bir dahaki C++ gönderim muhtemelen CRTP (_Curiously Recurring Template Pattern_, İlginç Şekilde Tekrar Eden Şablon Deseni) üzerine olacaktır -ki kendisi apayrı bir bok çukurudur-. Çağdaş C++'ın birçok nimeti var ancak insanlar bunların pek farkında değiller ne yazık ki. Rust'tan pek farklı olmayan bir hafıza güvenliğine ve Lisp seviyesinde bir üstprogramlama kapasitesine sahip olmasına karşın insanlar C++'ı çoğunlukla bol sınıf işaretçili `new` ve `delete` çorbası olarak öğrendiğinden ötürü maalesef C++'ın kötü bir dil olduğunu düşünüyorlar.


---
author: Erencan Ceyhan
lang: tr
tags:
- C++
- Programlama
- Teknoloji
- Gönderi
- Üstprogramlama
- Haskell
date: 2024-04-23 13:25:04+03:00
---
