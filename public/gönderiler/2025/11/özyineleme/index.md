# C Bok Çukuru: `malloc`suz Dinamik Hafıza

Herkese selamlar! Ayda bir gönderi yazmaya çalışıyorum ama bazen yoğunluktan veya üşengeçlikten veyahut konu bulamamaktan kaynaklı olarak yazamayabiliyorum. Normalde ekimin sonuna yetiştirmeyi düşünüyordum ama ne yazık ki yetiştiremedim, dolayısıyla kasımın ilk gönderisi olmuş olacak bu arkadaş. Bu gönderideki konuyu ancak üç beş gün önce keşfettim ama bence hakkında gönderi yazmak için biçilmiş kaftan. İlk icat eden kişi ben değilim, HackerNews'te bunu yapan birilerini daha gördüm fakat çok yaygın bir teknik değil gibi gözüküyor. O zaman kollarımızı sıvayalım ve yeni bir ucubenin nasıl sunulduğunu izleyelim.

## Temel Bilgiler

Bir bok çukuru gönderisine dalmadan önce boku tanımakta fayda olduğuna inanmışımdır hep. C++ bok çukurlarında hep ufak hatırlatmalarla başladım bu sebeple. C bok çukurunda da aynı geleneği sürdürmemek için bir sebep göremiyorum açık konuşmak gerekirse.

Bu sefer aslında çok da detaylı veya ileri düzey bilgiye ihtiyacımız yok. C++ şablonları, özellikle de gönderilerde bahsettiğim biçimiyle, oldukça karmaşık konular fakat C doğası gereği o kadar da karmaşık olmayan bir dil. Nitekim bahsdeceğim kavram da çok temel kavramların etkileşimi ile oluşuyor.

### Dinamik Dosya Okuma

Öncelikle bunun ne _olmadığını_ söylesem iyi olabilir. `malloc`suz dinamik hafıza dendiğinde bazılarımızın aklına C99 ile hayatımıza giren ve C11 ile opsiyonel hâle gelen değişken boyutlu diziler (variable length array) gelebilir. Veya benzer bir işleve sahip olan ama bu sefer de platforma bağımlı olan `alloca` fonksiyonu da gelebilir. Fakat bu gönderide anlatacağım yöntem ikisinden de bağımsız. %100 saf C89'da, hatta hiçbir standart kütüphane fonksiyonuna ihtiyaç duymadan uygulanabilecek bir yöntem. En büyük güçlerinden biri de bu belki de. Ufak farklılıkları olacak ama elbette. Genel olarak kodlarda hata ile ilgilenme kısımlarını yazmıyorum, amacım işlevsel bir koddan ziyade yöntemi göstermek.

Bir dosyayı hafızaya okumayı şu şekilde yapmamız mümkün C99 varsa elimizde:

```c
void read_file(char const* filename) {
    FILE* file;
    long file_size;

    file = fopen(filename, "rb");

    fseek(file, 0, SEEK_END);
    file_size = ftell(file);
    fseek(file, 0, SEEK_SET);

    char buffer[file_size + 1];
    fread(&buffer, sizeof(char), file_size, file);
}
```

Bu fonksiyonun sonunda dosyanın içeriği `buffer` değişkeninde bulunuyor olacak. Hem de boyutunu `sizeof` ile alabiliyoruz! Bu fonksiyonun temel sorunu, akışın ileri geri gidilebildiğini var sayması diyebiliriz. Normal dosyalarda bu geçerli olsa da en basit akışlarda (_stream_) bu geçerli değil, mesela `stdin`. Kullanıcıdan interaktif girdi okuyamaz bu hâliyle yani. Böyle bir sorunun temelde en basit çözümü `malloc` kullanmaktır. `realloc` ile her bir boyut aşımında `buffer`'ın boyutunu arttırabiliriz, hatta geometrik bir dizi şeklinde arttırırsak yer ayırma (_allocation_) sayımız okuduğumuz boyuta göre logaritmik olacağından dolayı oldukça da verimli diyebiliriz.

### Özyineleme

Özyineleme, bilgisayar veya yazılım ile ilgili herhangi bir bölüm okumuşlarımız için genelde okulun ilk senesinde öğrendiğimiz bir yöntem. Müfredata bağlı olarak algoritma ve veri yapıları derslerinin dışında neredeyse hiç kullanmayız ama kolay kolay. Belki algoritmik derslerde işimize yarar, o da öyle derslerimiz varsa. Yazılımı daha karadüzen öğrenenlerimiz ise öğrenmemiş olabilir.

Özyineleme ile ilgili en meşhur espri, tanımının kendisinin de özyineleme olmasıdır herhalde. Özyinelemeyi özyineleme kelimesiyle tanımlarız yani. Daha ciddi bir tanımı, kendi üzerinden tanımlanan işlemler olsa gerek. Genellikle özyinelemeli bir algoritmadaki amaç problemi daha küçük bir parçaya bölüp, küçük parçanın aynı algoritma ile çözüldüğünü varsayıp onun üzerine çözümü tamamlamaktır. Buna _böl ve fethet_ (_divide and conquer_) yaklaşımı denir. En basit örneği Fibonacci sayılarıdır muhtemelen:

```c
int fib(int n) {
    if(n == 1 || n == 2) { return 1; }
    return fib(n - 1) + fib(n - 2);
}
```

Baştaki `if`'e _taban durum_ (_base case_) deniyor. Daha küçük iki sayının Fibonacci sayılarının hesaplandığını varsayıp bu ikisi üzerinden bir sonraki Fibonacci sayısını hesaplamış oluyoruz yani. Bu oldukça basit bir örnek ama bize gerekli her türlü konsepti tanıtıyor aslında.

### Bağlı Listeler (_Linked List_)

Bu da yine bilgisayar veya yazılım müfredatının ilk senesinin konularından birisi. Temel mantık, bir veri parçasının yanına bir sonraki veri parçasının yerini söyleyen bir veri daha koymak aslında.

```c
struct Node {
    struct Node* next;
    int data;
};
```

Bu düğüm (_node_) `int` türünden bir veri içeriyor, sonraki verinin adresini de `next` olarak tutuyor. En basit bağlı liste tanımı bu olsa gerek. Özyineleme ile çok güzel bir ikili oluyorlar aslında, çünkü bir düğüm üzerinde işlem yaptıktan sonra işin kalanını aynı işi, farklı parametrelerle belki, tekrar ederek bitirebiliyoruz genelde. Mesela `data`'ların her birini belli bir sayı ile çarpmak için şöyle bir işlev yazabiliriz:

```c
void multiply(struct Node* current, int number) {
    if(current) {
        current->data *= number;
        multiply(current->next, number);
    }
}
```

Bu sebeple fonksiyonel programlama dillerinin birçoğundaki en temel veri yapılarından birisi bağlı listeler olsa gerek. Bu tarz özyineleme ağırlıklı kod yazmak için birebir oluyorlar yani.

### Açılmış Bağlı Listeler (_Unrolled Linked List_)

İşte bu görece ileri denebilecek bir konu. Bağlı listede veriyi tutmak yerine, bağlı listede _birden fazla_ veri tuttuğumuzda buna açılmış bağlı liste diyoruz. Şuna benziyor bir bakıma yani:

```c
struct UnrolledNode {
    struct UnrolledNode* next;
    int data[32];
};
```

Burada otuz ikişer `int` tutuyoruz her bir düğümde. Bunun en büyük avantajı önbellek verimliliğini ciddi arttırması olsa gerek. Normal bir bağlı listede her adımda bir işaretçi (_pointer_) takip ederiz, sonuç olarak da belleğimizin orasına burasına gider dururuz, bu da modern önbellek sistemleri için fazlasıyla kötü. Açılmış bağlı listelerde ise verinin sayısını arttırabilir, böylece bir seferde bir önbellek bölgesin dolduracak kadar veriyi işleyebiliriz. İşlemlerimizin karmaşıklığını arttırıyor elbette, ama bence oldukça güzel ve şık bir optimizasyon.

### Devam Yollama Tarzı (_Continuation Passing Style_)

Bu konsept aslında C'de pek az kullanılan bir yöntem, ama C'de ne kadar enderse Lisp'te de o kadar yaygın kullanılan bir yöntem aynı zamanda. Buradaki temel olay aslında saf (_pure_) işlemlere bir "sıra" eklemek oluyor. Normalde fonksiyonel programlamada tam olarak "şunu yap, sonra şunu yap, ondan sonra da bunu yap" diye belirtmek mümkün değil. Günün sonunda ana giriş fonksiyonumuz tek bir değer döndürecek. Fakat bir sonucu başkasına doğrudan bağlamak mümkün devam yollama tarzı ile. Yapacağımız şey basit: bir işlem gerekli parametrelerinin yanında _ne ile devam etmesi gerektiğini_ de alacak ve işleminin sonucunu argüman olarak bu devam fonksiyonuna verecek. Lisp'e çok hâkim olmadığım için C ile göstereceğim:

```c
void print_integer(int value) {
    printf("%d\n", value);
}

void multiply_cps(int x, int y, void(*cont)(int)) {
    int result = x * y;
    cont(result);
}

int main() {
    multiply_cps(3, 5, &print_integer); // prints 15
}
```

Tabii ki bu tek aşamalı bir uygulama. İki aşama olursa aradaki aşamaya devamını nasıl belirteceğimizin derdi oluşuyor. Değişken sayıda argüman alan fonksiyonlar (_variadic function_) ve biraz ekstra kod ile bu sorunu çözebiliyoruz aslında.

## Yöntem

Bu kadar boş yaptıktan sonra yönteme gelelim. Ya şimdiye kadar tahmin etmişsinizdir ya da görünce "Bunun için mi zamanımı bu kadar harcadın!" diye bana kızacaksınız.

Temel mantık, bu açılmış bağlı listeyi _doldurma_ işini de özyinelemeli bir şekilde yapmak. Normalde bir fonksiyon listeyi doldurur, listeyi döner ve en başta bunu çağıran fonksiyon işine devam eder. Fakat bunu yapmak için hafızayı ayırabileceğimiz bir yere ihtiyacımız var. İlk seçeneğimiz `malloc` kullanmak olur elbette, `free` kullanmayı da bizi çağıran fonksiyonun sorumluluğu olarak belirlemiş oluruz. Bunu otomatikleştirmenin en temel yolu akıllı işaretçi (_smart pointer_) kullanmak olur, veya bunun genellenmiş hâli olan RAII. C++ ve Rust'ta yazılan çoğu program bu yöntemi kullanıyor nitekim. O `std::vector`/`Vec` havadan gelmiyor sonuçta. Çöp toplayıcı (_garbage collector_) sahibi dillerde bu sorumluluk çağıran fonksiyona değil çöp toplayıcının kendisine verilir, en azından izlemeli çöp toplayıcı (_tracing garbage collector_) dillerde. Atıf sayaçlı (_reference counting_) çöp toplama sistemleri akıllı işaretçilerden pek farklı değil mantık olarak. İzlemeli olanlar biraz daha değişik. Bir diğer yöntem ise belli bir arenaya toplamak oluyor. Bu durumda sorumluluk çağıran fonksiyon veya çöp toplayıcı değil, içinde yer ayrımış olduğumuz arenada oluyor. Bu arena ileride bir noktada silmekle yükümlü, veya hafıza kaçağında (_memory leak_) sakınca yoksa silmeyebilir de tabii.

İşte bu noktada işleri biraz tersdüz etmek aklıma geldi açıkçası. Bu mantıkta kodun çalışması düzden gerçekleşirken hafıza işleri tersten çalışıyor bir bakıma. Tabii ki nedenselliği bozmuyoruz, ama sorumluluk olarak bu şekilde olmuş oluyor. Çağıran fonksiyon veriyi üreten fonksiyonu da veriyi tüketen fonksiyonu da çağırıyor, bu noktada sorumluluk gayet düz. Ancak üreten fonksiyon veriyi ürettikten sonra sorumluluğunu kendisini çağıran fonksiyona veriyor, yani verinin sorumluluğunu çeviriyoruz. İşte tam bu noktada dedim ki, tersini yaparsak ne olur? Yani işlev çağırma sorumluluğunu devam yollama tarzı ile tersine çevirsek ve üretilen verinin sorumluluğunu üreten fonksiyondan çıkarmasak ne olur?

Bu şekilde anlatınca havada kalmıştır muhtemelen, o nedenle hadi gelin bir akıştaki verileri dinamik olarak bir açılmış bağlı listeye okuyup bunun üzerinden işlem yapan bir fonksiyon yazalım.

```c
struct BufferNode {
    struct BufferNode* next;
    char data[BUFSIZ - sizeof(struct BufferNode*)];
};
```

Bu basit bir açılmış bağlı liste düğüm tanımı. Sadece toplam boyutu `BUFSIZ` değerini (C standardına göre stdio başlık dosyasında tanımlanan ve stdio fonksiyonlarının kullandığı buffer boyutunu belirten sabite) bulacak şekilde kendini boyutlandırıyor. Burada herhangi başka bir boyut da verilebilir, o kısım çok önemli değil açıkçası.

```c
int read_stream_to_node(FILE* stream, struct BufferNode* node) {
    size_t read_count;

    read_count = fread(&node->data, sizeof(char), sizeof(node->data), stream);
    if(read_count < sizeof(node->data)) {
        assert(!ferror(stream));
        return 0;
    }

    return 1;
}
```

Bu da bizim temel yardımcı fonksiyonumuz. Bir düğüme veriyi okuyor, eğer okuma bitmişse 0 bitmemişse 1 dönüyor.

```c
void read_stream_helper(FILE* stream, struct BufferNode* root, struct BufferNode* prev, void (*cont)(struct BufferNode* root)) {
    struct BufferNode buffer;
    buffer.next = NULL;
    prev->next = &buffer;

    if(read_stream_to_node(stream, &buffer)) {
        read_stream_helper(stream, root, &buffer, cont);
    } else {
        cont(root);
    }
}

void read_stream(FILE* stream, void (*cont)(struct BufferNode* root)) {
    struct BufferNode buffer;
    buffer.next = NULL;

    if(read_stream_to_node(stream, &buffer)) {
        read_stream_helper(stream, &buffer, &buffer, cont);
    } else {
        cont(&buffer);
    }
}
```

Ha? Neredeyse aynı fonksiyondan iki tane olması hoş değil elbette, belki `root` ve `prev` parametreleri sona taşınıp `NULL` verilirse `read_stream` gibi çalışması sağlanabilir `read_stream_helper`'ın. Ama mantık basit: buffera veriyi okuduktan sonra eğer veri okuma bitmişse devam fonksiyonunu çağırmak. Gelin `tee`'nin tek girdiden tek çıktıya aktaran basit versiyonunu yazalım, yani verilen girdiyi değiştirmeden çıktı olarak yazdırsın:

```c
void print(struct BufferNode* root) {
    struct BufferNode* current;
    current = root;

    while(current) {
        size_t i;
        for(i = 0; i < sizeof(current->data); i++) {
            printf("%c", current->data[i]);
        }
        current = current->next;
    }
}

int main() {
    read_stream(stdin, &print);
    return EXIT_SUCCESS;
}
```

Devam fonksiyonuna argüman vermek dediğim gibi görece zor C'de, dolayısıyla çıktı akışını `stdout` olarak vermek yerine direkt `printf` kullandım ama mantık bu şekilde. Bu sayede yığın (_heap_) kullanmadan tamamen yığıt (_stack_, neden bu ikisinin çevirileri bu kadar benzer kelimeler?) üzerinden yerimizi ayırabilir ve işlemimizi gerçekleştirebiliriz.

## Sonuç

Sonuç olarak bu yöntem ile `malloc` kullanmadan dinamik hafıza ile işlem yapmak mümkün hâle geliyor. Fakat elbette birtakım sorunları yok değil. Çağrı yığıtı (_call stack_) yığının aksine çok daha kısıtlı boyuta sahip, dolayısıyla bu şekilde çok fazla veri oluşturursak yığıt taşımı (_stack overflow_) yaşamamı mümkün. Ayrıca nasıl ki normal yöntemlerde, veri yeri ayırmanın sorumluluğunun çağıran fonksiyona aktarıldığı, verinin yerini ayırma işi dinamik bir şekilde oluyorsa burada da terslediğimiz diğer etken yani fonksiyonun kendisi dinamik oluyor. Fakat bunun bir çözümü var, devam fonksiyonunu veri oluşturma fonksiyonunun içinde belirtmek. Bunu her durum için elle yazmak C'de çok akıl kârı değil ama C++'ta şablon kullanarak bunu derleyiciye yaptırabiliriz. Tabii ki sonrasındaki her işlem için fonksiyonlarımız değişecek, bu da kod miktarının hızlı bir şekilde çoğalmasına yol açacak ama performans kaybımızı engelleyecek en azından.

Bir diğer eksiğimiz ise bunun bağlı liste kullanması. Ha, açılmış bağlı liste olduğu için performans o kadar da kötü değil ama bir dizge hâline getirilse daha iyi olabilir. C89'da bu pek mümkün değil ama C99'da gelen değişken boyutlu diziler ile bu da mümkün fakat bu şekilde çalışan kodu yazmayı okuyucuya bırakıyorum. Mantık aynı, sadece oluşturulan veri yapısı bir bağlı liste değil de taştıkça boyutunu arttıran veya katlayan bir dizi olmalı.

Kısacası eğer yığıtınız sınırlıysa çok işinize yaramaz ama değilse arena sistemini tamamen yığıt üzerinde gerçekleştirebilmenizi sağlar bu yöntem.

---
author: Erencan Ceyhan
lang: tr
tags:
- C
- Programlama
- Teknoloji
- Gönderi
date: 2025-11-02 03:33:19+03:00
---
