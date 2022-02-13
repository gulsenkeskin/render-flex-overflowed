
Bir Flutter geliştiricisi olarak, muhtemelen uygulamanızda bir görüntünün kesilmesi (veya görünmemesi) gibi sorunlarla karşılaşmış veya  “viewport was given unbounded height” (görünüm alanına sınırsız yükseklik verildi) hatası almışsınızdır.

Aslında, Flutter hatalarının en sık görülen iki türü, düzen ile ilgili sorunlardır: **widget overflow** (taşması) ve **"renderbox not laid out"**(renderbox düzenlenmemiş) sorunları.

Neyse ki Dart DevTool'un [Flutter Inspector](https://docs.flutter.dev/development/tools/devtools/inspector)'ı , bunların neden oluştuğunu ve nasıl çözülebileceğini anlamanıza yardımcı olur. Bu makalede, RenderFlex overflowed'ın ne olduğunu ve 3 yaygın düzen sorununun hatalarını ayıklayarak aracın nasıl kullanılacağını öğreneceksiniz.


## A RenderFlex overflowed (Bir RenderFlex taştı)

RenderFlex overflowed, en sık karşılaşılan Flutter framework hatalarından biridir ve muhtemelen bununla zaten karşılaşmışsınızdır.

**Hata neye benziyor?**

Bu hatayla karşılaştığınızda, debug console'undaki hata mesajına ek olarak uygulama kullanıcı arayüzünde taşma alanını gösteren sarı ve siyah çizgiler görürsünüz:

```
The following assertion was thrown during layout:
A RenderFlex overflowed by 1146 pixels on the right.

The relevant error-causing widget was
    Row 	    lib/errors/renderflex_overflow_column.dart:23
The overflowing RenderFlex has an orientation of Axis.horizontal.
The edge of the RenderFlex that is overflowing has been marked in the rendering 
with a yellow and black striped pattern. This is usually caused by the contents 
being too big for the RenderFlex.
(Additional lines of this message omitted)
```

**Bu hatayla nasıl karşılaşırsınız?**

Hata, genellikle bir Column veya Row, boyutu sınırlı olmayan bir alt widget aracına sahip olduğunda oluşur.
Örneğin, aşağıdaki kod parçası ortak bir senaryoyu gösterir:


```
Widget build(BuildContext context) {
  return Container(
    child: Row(
      children: [
        Icon(Icons.message),
        Column(
          mainAxisSize: MainAxisSize.min,
          crossAxisAlignment: CrossAxisAlignment.start,
          children: [
            Text("Title", style: Theme.of(context).textTheme.headline4),
            Text(
                "Lorem ipsum dolor sit amet, consectetur adipiscing elit, sed"
                " do eiusmod tempor incididunt ut labore et dolore magna "
                "aliqua. Ut enim ad minim veniam, quis nostrud "
                "exercitation ullamco laboris nisi ut aliquip ex ea "
                "commodo consequat."),
          ],
        ),
      ],
    ),
  );
}
```

Yukarıdaki örnekte, Column, Row'un (üst öğesi- parent) ayırabileceği alandan daha geniş olmaya çalışır ve bir overflow hatasına (taşma) neden olur.
Column'un neden bunu yapmaya çalıştığını anlamak için, Flutter framework'ünün düzeni nasıl gerçekleştirdiğini bilmeniz gerekir:

Düzeni gerçekleştirmek için Flutter, render tree'yi (oluşturma ağacını) derinlik öncelikli (depth-first) bir geçişte yürütür ve boyut kısıtlamalarını parent'dan child'a aktarır…
Çocuklar, parent tarafından belirlenen kısıtlamalar dahilinde ana nesnelerine boyut aktararak yanıt verirler. – [Flutter architectural overview](https://docs.flutter.dev/resources/architectural-overview#layout-and-rendering)

Bu durumda, Row widget, alt öğelerinin boyutunu sınırlamaz ve onun alt  öğesi olan Column widget da altındaki öğelerin boyutunu sınırlamaz. İkinci Text widget da, parent widget  kısıtlamalarından yoksun olarak, görüntülemesi gereken tüm karakterler kadar geniş olmaya çalışır. Bu durumda da RenderFlex overflowed sorunu meydana gelir.

**Nasıl düzeltilir?**

Bunun için column'un genişliğini sınırlamanız gerekir. Bunu yapmanın bir yolu, colum'u bir Expanded widget'ın içine sarmaktır:

```
child: Row(
  children: [
    Icon(Icons.message),
    Expanded(
      child: Column(

      ),
    ),
  ]
)
```


Başka bir yol da Column'u bir Flexible widget içine sarmak ve bir flex faktörü belirlemektir. Aslında, Expanded widget da, flex faktörü 1.0 olan Flexible widget ile eşdeğerdir.


## Flutter Inspector Nedir?


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o6b7rnovff8j4cflc3ca.png)


Flutter Inspector, widget ağaçlarını keşfetmeyi ve görselleştirmeyi sağlayan bir araçtır (Dart DevTools paketinin bir parçasıdır). Uygulama düzeninizin içini ve dışını keşfetmek için mükemmeldir. Aşağıdaki GIF'den aracı görebilirsiniz.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qbd2p1m0fpr97a2eq5vp.gif)

Inspector ile uygulamadaki widget'ları seçebilir hatta debug banner'ı (hata ayıklama başlığını) bile kaldırabilirsiniz.


**Bu makale aşağıdaki özelliklere odaklanmaktadır:**

**Details Tree** Her bir widget'ın özelliklerini incelemenizi sağlar. Bir widget'ın gerçek boyutunu inceleyebilir ve parent widget'tan kısıtlamaların nasıl aktarıldığını görebilirsiniz. 


**Layout Explorer** Flex widget'larını (Row, Column, Flex) ve bunların alt öğelerini görselleştirmenizi sağlar. flex, fit, ve axis alignment'ı ayarlamayı ve uygulamanızdaki değişiklikleri görmenizi sağlar.

Hata ayıklama macerası 🤠
Hataları ayıklayacağınız 3 düzen problemini içeren bir demo uygulamasıyla başlayalım. Her sorunda ayrı ayrı hata ayıklamak için Flutter Inspector'ı kullanacak ve aşağıdaki ekran görüntüsüne benzeyen basit bir menü uygulamasını tamamlamak için sonunda düzeltilen sorunları birleştireceksiniz.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/1auzykm1g9qxuw6h7ei7.png)

 
**Bir düzen sorununa yaklaşırken aşağıdaki adımları kullanın:** 

1. Hata türünü ve hataya neden olan widget'ı belirlemek için debug console'undaki hata mesajını kontrol edin.

2. Bir Flex widget ve alt öğelerini görselleştirmek için Layout Explorer'ı açın.

3. Hataya neden olan widget'ın ve üst öğesinin/alt öğelerinin boyutunu ve kısıtlamalarını Details Tree ile inceleyin.

4. N kodunuza geri dönün ve sorunu düzeltin.




1. `menu` adlı yeni bir Flutter projesi oluşturun

2. lib/main.dart dosyasının içeriğini aşağıdaki kodla değiştirin:


```
import 'package:flutter/material.dart';

void main() {
  runApp(Menu());
}

class MenuItem extends StatelessWidget {
  const MenuItem(this.icon, this.itemText);
  final String icon;
  final String itemText;
  @override
  Widget build(BuildContext context) {
    return ListTile(
      leading: Text(
        icon,
        style: TextStyle(
          fontSize: 30.0,
        ),
      ),
      title: Text(itemText),
    );
  }
}

class Menu extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return MaterialApp(
      title: 'Flutter Demo',
      home: Scaffold(
        appBar: AppBar(
          title: Text('Menu Demo'),
        ),
        body: Padding(
          padding: EdgeInsets.all(20.0),
          child: Column(
            children: [
              // Modify code here
              Example1(),
            ],
          ),
        ),
      ),
    );
  }
}

// Problem 1: Overflow error
class Example1 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Padding(
      padding: EdgeInsets.only(bottom: 30.0),
      child: Row(
        children: [
          Text(
            'Explore the restaurant\'s delicious menu items below!',
            style: TextStyle(
              fontSize: 18.0,
            ),
          ),
        ],
      ),
    );
  }
}

// Problem 2: Viewport was given unbounded height error
class Example2 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return ListView(
      children: [
        MenuItem('🍔', 'Burger'),
        MenuItem('🌭', 'Hot Dog'),
        MenuItem('🍟', 'Fries'),
        MenuItem('🥤', 'Soda'),
        MenuItem('🍦', 'Ice Cream'),
      ],
    );
  }
}

// Problem 3: Invisible VerticalDivider
class Example3 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return Row(
      mainAxisAlignment: MainAxisAlignment.spaceEvenly,
      children: [
        RaisedButton(
          onPressed: () {
            print('Pickup button pressed.');
          },
          child: Text(
            'Pickup',
          ),
        ),
        // This widget is not shown on screen initially.
        VerticalDivider(
          width: 20.0,
          thickness: 5.0,
        ),
        RaisedButton(
          onPressed: () {
            print('Delivery button pressed.');
          },
          child: Text(
            'Delivery',
          ),
        )
      ],
    );
  }
}
```

test klasörünün altında bulunan widget_test.dart dosyasının 16. satırını aşağıdaki kodla değiştirin.

`await tester.pumpWidget(Menu());`

3. Uygulamayı çalıştırın.
4. Dart DevTools'u açın.


## Düzen sorunu 1: Overflow error (Taşma hatası)

Uygulamayı çalıştırdığınızda, satırın sonunda uyarı bandına benzer şekilde sarı ve siyah çapraz çizgili bir kutu görürsünüz :

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xl18lhp66e3b1fj38duf.png)
 

Bu, en yaygın Flutter layout hatası olan, overflow hastasını gösterir. Şimdi, sorunu belirlemek ve doğru düzeltmeyi bulmak için hata ayıklama adımlarını izleyelim.

1. Konsoldaki hata mesajını kontrol edin.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/obl1rfedv92p0tojafkz.png)
 
İlk olarak, hangi widget'ın soruna neden olduğunu belirleyin Hata mesajı, main.dart'ın 54. satırındaki Row'un soruna neden olduğunu gösterir. Row bir Flex widget olduğu için (yani, Flex sınıfını extend ettiği anlamına gelir), Layout Explorer ile inceleyebilirsiniz.


2. Layout Explorer'ı açın

DevTools'a gidin ve Layout Explorer sekmesini açın.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/imjgy5tnxsurxtrzf7j9.png)
![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/vrkmp1yk5i6pe8k8ce2q.png)
  
Row'a tıklayın. (Resimdeki sayılar aşağıdaki adımlarla ilişkilidir.)

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rftbj4ycqzp5q1f542ia.png)

1. Altta bir şeylerin yanlış olduğunu gösteren kırmızı kutular görünür. Banner'lara daha yakından baktığınızda, Text'in ( width = 447) ana widget , Row'dan (width=335) daha geniş olduğunu ve taşma hatasına neden olduğunu görebilirsiniz.

2.Text'e, yalnızca Row kadar geniş olabileceğini ve daha fazla genişleyemeyeceğini söylemenin bir yoluna ihtiyacınız var. 
Text'in flex değerini 1'e ayarlamayı deneyin. (Bu, Text widget'ını bir Expanded widget ile sarmaya benzer.) Sonuç olarak, Text küçülür ve kırmızı kutucuklar kaybolur. Düzelmiş gibi görünüyor ama düzelmedi.
Araç sadece bazı düzen özelliklerini değiştirirseniz ne olacağını gösterir kod üzerinde bir değişiklik yapmaz. Araç kodunuza dokunamadığı için kodu kendiniz güncellemelisiniz.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/aaesklmefrxv53isn8br.gif)

Neden tüm Row ve Column öğelerinin varsayılan olarak expanded olmadığını merak ediyor olabilirsiniz. Eğer tüm alt öğeler varsayılan olarak expanded yani genişletilmiş olsaydı bazı alt öğelerin çok sıkılması veya gerilmesi gibi birtakım başka sorunlar ortaya çıkabilirdi.

3. Details Tree ile boyutu ve kısıtlamaları inceleyin.
Bu senaryoda, sorun tanımlandığı için bu adımı atlayabilirsiniz.

4. Koda geri dönün ve kodunuzu düzeltin.

Text widget'ını Expanded widget ile sarın. Varsayılan olarak flex değeri 1'dir, bu nedenle bu özelliği belirtmeniz gerekmez.

## Düzen sorunu 2: Unbounded height error (Sınırsız yükseklik hatası)

Column içindeki Example1()'i Example2() ile değiştirerek bir sonraki örneğimize geçelim.

```
Column(
  children: [
    // Modify code here
    Example2(),
  ],
)
```

Example2 sınıfında çeşitli menü öğelerine sahip bir ListView olmasına rağmen uygulamada hiçbir şey gözükmüyor:


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/xi4bdavma6h8yb9ukeu5.png)

**1. Konsoldaki hata mesajını kontrol edin.**

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/87vahsm4vf90xxua88yh.png)
 
main.dart'ın 74. satırındaki ListView, “Vertical viewport was given unbounded height” (Dikey görünüm alanına sınırsız yükseklik verildi) hatasına neden oluyor. İlk bakışta, vertical viewport (dikey görüntü alanı) ve unbounded (sınırsız) terimleri belirsizdir, bu nedenle bir sonraki adıma geçin.


**2. Layout Explorer'ı açın**

DevTools'a geri dönün ve Layout Explorer sekmesini açın.

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/o28g62lz49lrd3rb6ea8.png)

Üstteki yenileme simgesine tıklayarak ağacı yenileyin. Layout Explorer yalnızca Flex widget'larını ve bunların doğrudan alt öğelerini desteklediğinden ListView'e tıkladıktan sonra hiçbir şey görünmez. İlginç bir şekilde, Example2'ye ve Column'a tıklamak da işe yaramıyor -Layout Explorer hala boş. Bir sonraki adıma geçin.

**3. Details Tree ile size ve constraint'leri (kısıtlamaları) inceleyin.**

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/q9rr3sgvylsbuw0ygetp.png)

Turuncu metin, boyutun eksik olduğunu gösteriyor - uygulamada ListView'in eksik olmasına şaşmamalı.

Constraints özelliğine baktıktan sonra, yükseklik kısıtlamasının infinity (sonsuz) olarak listelendiğine dikkat edin. Hata mesajı şimdi daha mantıklı. ListView, kaydırma yönünde sınırsız - başka bir deyişle sonsuz - yükseklik verilen bir görünüm alanıdır (viewport).

**4. Kodunuza geri dönün ve ListView'i bir Expanded widget ile sarın**

```
class Example2, StatelessWidget'ı genişletir { 
  @override 
  Widget build(BuildContext context) { 
    return Expanded( 
      child: ListView( 
        ... 
      ), 
    ); 
  } 
}
```


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/f7qk6jbmeqdh6zwwlo99.gif)


Bir widget'ı Expanded widget ile sarmanın, ona parent'ın (üst öğenin) ana ekseni (main axis) (Row için genişlik, Column için yükseklik) boyunca sınırlı bir kısıtlama sağladığını unutmayın. Bu durumda, parent Column olur, ve böylece Expanded widget bir yükseklik kısıtlaması sağlar.

## Düzen sorunu 3:  Invisible VerticalDivider (Görünmez VerticalDivider)

Example2()'yi Example3() ile değiştirin.

```
Column(
  children: [
    // Modify code here
    Example3(),
  ],
)
```
Koddaki Example3 sınıfına yakından bakın. VerticalDivider'ın var olduğuna dikkat edin, ancak uygulamada yalnızca iki düğme görünüyor:

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/7zy3qnwxaln74y7bi4zf.png)

**VerticalDivider neden görünmüyor?**

**1. Konsoldaki hata mesajını kontrol edin.**
Bu sefer herhangi bir hata mesajı almadık. Sonraki adıma devam edin.

**2. Layout Explorer'ı açın.**
DevTools'a geri dönün ve Layout Explorer sekmesine tıklayın


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/n22rc1imai3nkafpub7w.png)

Ağacı yeniledikten sonra VerticalDivider'a tıklayın ve Layout Explorer'ın sağ tarafına kaydırın. VerticalDivider'ın genişliğinin ve yüksekliğinin unconstrained (sınırlandırılmamış) olduğuna dikkat edin.

1. VerticalDivider'ın yüksekliğinin 0 olduğuna dikkat edin, bu da uygulamada neden görüntülenmediğini açıklıyor.


2. Daha önce yaptığınız gibi flex özelliğini 1'e getirin. Yükseklik 0 olarak kalır. Bir VerticalDivider'ı Expanded ile sarmak bu durumda işe yaramaz çünkü Expanded, bir yükseklik kısıtlaması yerine bir genişlik kısıtlaması sağlar.


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/byopel74djzvmyrkpvs8.gif)

**3. Details Treeile size ve kısıtlamaları(constraints) inceleyin.**

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/9hsj1j9fl45xk12yyyfv.png)
 

1. VerticalDivider altındaki ilk renderObject'i açın. Constraints özelliği, widget'ın genişlik ve yükseklik kısıtlamasına sahip olmadığını ve Layout Explorer'ın gösterdiğiyle eşleştiğini belirtir. Ancak, additionalConstraints altında genişlik 20'dir (örnek kodda açıkça tanımlandığı gibi), ancak yüksekliğin hala hiçbir kısıtlaması yok. Buradaki sorun genişlik değil, o yüzden yüksekliğe odaklanalım.

2. Üst widget'a, Row'a gidin ve Row'un bir yükseklik kısıtlamasına sahip olmadığını görmek için renderObject'i açın.

**4. Kodunuza geri dönün Row widget'ını SizedBox ile sararak yükseklik değerini 50 olarak belirleyin**

```
class Example3 extends StatelessWidget {
  @override
  Widget build(BuildContext context) {
    return SizedBox(
      height: 50.0,
      child: Row(
        ...
      ),
    );
  }
}
```

VerticalDivider'ın yüksekliğe sahip olabilmesi için bir yükseklik kısıtlaması verilmelidir. Row'u SizedBox ile sarın ve ona 50.0 sabit bir yükseklik verin. Bunu yapmak, Row'u VerticalDivider'a bir yükseklik kısıtlaması geçirmeye zorlar.

Hot reload'a tıklayın. Ve artık VerticalDivider'ı görebilirsiniz. :)


![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/0k5cz1nvbvvdmfx614tp.png)

 
VerticalDivider, benzersiz tanımları nedeniyle ListView'den farklı davranır. Kendi yüksekliklerini seçmeleri söylendiğinde, ListView mümkün olduğunca uzun olmak isterken VerticalDivider mümkün olduğunca kısa olmak ister. Ancak, uygulamada düzgün bir şekilde görünmesi için her ikisinin de bir yükseklik kısıtlamasına ihtiyacı vardır.

Şimdi, 3 örneğin hepsinin kodunu Column içinde bir araya getirelim:

```
Column(
  children: [
    // Modify code here
    Example1(),
    Example2(),
    Example3(),
  ],
)
```

Hot reload. Tebrikler, menü uygulamasını tamamladınız :)

![Image description](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/4yjpalkljzr4e8eiesac.png)

 

## Özet

Bu eğitim sayesinde şunları öğrendiniz:
• Constraint'ler (kısıtlamalar), widget ağacından geçirilir.
• Expanded, bir Row veya Column'un alt öğesi için sınırlı kısıtlama sağlar.
• Flutter Inspector, yerleşim sorunlarıyla uğraşırken en iyi arkadaşınızdır.


References:
https://medium.com/flutter/how-to-debug-layout-issues-with-the-flutter-inspector-87460a7b9db#738b
https://docs.flutter.dev/testing/common-errors
https://docs.flutter.dev/development/tools/devtools/inspector
https://docs.flutter.dev/development/tools/devtools/overview#how-do-i-install-devtools
