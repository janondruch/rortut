# Ruby s nádechem Rails

Na základech předchozí části nás čeká prozkoumání některých prvku programovacího jazyku Ruby, které jsou pro Rails důležité. I když je Ruby jazyk rozsáhlý, naštěstí nám postačí jeho relativně malá znalost pro úspěšný vývoj v Rails. Zároveň to bude tak trochu jiný vstup do problematiky Ruby, než bývá běžné, ale nabereme si díky tomu solidní základ pro práci v Ruby s nádechem Rails, nehledě na předchozí zkušenosti. Tato část bude zároveň informačně poměrně obsáhlá, ale budeme se k ní vracet ve formě odkazů i v budoucnu.

## Motivace

Jak jsme si ukázali minule, je možné vyvíjet i na základě kostry aplikace v Rails, dokonce ji i testovat, bez prakticky jakýchkoliv znalostí Ruby, na kterém vše stojí. Jelikož ale nemůžeme spoléhat na takto dodané věci navěky, budeme se muset vydat na hlubší průzkum.

Opět pro vývoj použijeme oddělenou větev, kterou nakonec spojíme s větví hlavní:

```
$ git checkout -b rails-flavored-ruby
```

### Zabudovaní pomocníci (helpers)

Naposledy jsme si upravili naše vcelku statické stránky tak, ať používají možností Rails k eliminaci duplikace. Pro připomenutí tedy následující pohled ("app/views/layouts/application.html.erb"):

```
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all',
                                              'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>

  <body>
    <%= yield %>
  </body>
</html>
```

Zaměřme se na jeden konkrétní řádek:

```
<%= stylesheet_link_tag 'application', media: 'all',
                                       'data-turbolinks-track': 'reload' %>
```

Jde o využití zabudované Rails funkce "stylesheet_link_tag", která zahrne soubor "application.css" pro všechny [typy médií](http://www.w3.org/TR/CSS2/media.html) (včetně i např. tiskáren). Pro zkušeného vývojáře v Rails vypadá řádek jednoduše, ale obsahuje alespon čtyři potenciálně matoucí prvky Ruby: zabudované Rails metody, vyvolání metody s chybějícími závorkami, symboly a hašemi. Naštěstí si všechno ve zbytku této části postupně projdeme.



### Vlastní pomocníci (helpers)

I když Rails obsahuje mnoho zabudovaných funkcí určených k použití v pohledech, nebylo by to ono, kdyby neumožňoval i tvorbu vlastních. Tyto funkce se nazývají "pomocníci" (angl. "helpers"), a abychom si ukázali, jak si takového pomocníka vytvořit, blíže se podívame na řádek "title" z předchozího výpisu:

```
<%= yield(:title) %> | Ruby on Rails Tutorial
```
Tento řádek spoléhá na definování názvu stránky (skrze "provide") v každém pohledu, například:

```
<% provide(:title, "Home") %>
<h1>Aplikace</h1>
<p>
  Toto je domovská stránka pro naši aplikaci.
</p>
```

Ale co když název nedodáme? Je dobrým zvykem mít "základní název" (base title) na každé stránce, s možností volitelného názvu pokud chceme být více specifičtí. Současným rozvržením jsme toho téměř dosáhli, ale problém nastane, když smažeme volání "provide" v jednom z pohledů. Bez specifického titulu bude název totiž vypadat takto:

```
 | Ruby on Rails Tutorial
```

Jinak řečeno, jako základ by to stačilo, ale symbol | na začátku to trochu kazí.

Tento problém vyřešíme právě skrz vlastního pomocníka, kterého si nazveme "full_title", a který nám vrátí název "Ruby on Rails Tutorial" v případě, že název stránky není přímo definován, a v případě, že je, dodá nám původní variantu se svislou čarou. Upravíme si tedy soubor "app/helpers/application_helper.rb":

```
module ApplicationHelper

  # Vraci dynamicky plny nazev stranky.
  def full_title(page_title = '')
    base_title = "Ruby on Rails Tutorial"
    if page_title.empty?
      base_title
    else
      page_title + " | " + base_title
    end
  end
end
```

Když máme pomocníka vytvořeného, zjednodušíme jím rozvržení nahrazením řádku

```
<title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
```

řádkem

```
<title><%= full_title(yield(:title)) %></title>
```

Celý výpis souboru "app/views/layouts/application.html.erb" pak vypadá následovně:

```
<!DOCTYPE html>
<html>
  <head>
    <title><%= full_title(yield(:title)) %></title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all',
                                              'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
  </head>
  <body>
    <%= yield %>
  </body>
</html>
```

Aby nám pomocník skutečně pomáhál, můžeme smazat nepotřebné "Home" z domovské stránky, čímž dosáhneme právě varianty základního názvu. Nejprve si tedy upravíme test ("test/controllers/static_pages_controller_test.rb") následujícím způsobem, což aktualizuje předešlý test názvu a přidá jeden pro variantu chybějícího řetězce "Home" v titulu.

```
require 'test_helper'

class StaticPagesControllerTest < ActionDispatch::IntegrationTest
  test "should get home" do
    get static_pages_home_url
    assert_response :success
    assert_select "title", "Ruby on Rails Tutorial"
  end

  test "should get help" do
    get static_pages_help_url
    assert_response :success
    assert_select "title", "Help | Ruby on Rails Tutorial"
  end

  test "should get about" do
    get static_pages_about_url
    assert_response :success
    assert_select "title", "About | Ruby on Rails Tutorial"
  end
end
```

Následně si test spustíme, abychom si ověřili, že podle předpokladu selže:

```
$ rails test
3 tests, 6 assertions, 1 failures, 0 errors, 0 skips
```

Aby test proběhl úspěšně, odstraníme řádek "provide" z pohledu domovské stránky ("app/views/static_pages/home.html.erb"):

```
<h1>Aplikace</h1>
<p>
  Toto je domovská stránka pro naši aplikaci.
</p>
```

Teď už by měl test projít:

```
$ rails test
```

I když se obsah našeho pomocníka může zdát zkušenějším Rails vývojářum jako jednoduchý, obsahuje spoustu prvků z Ruby, ale žádný strach, postupně si je projdeme.



## Řetězce a metody

Jedním z hlavních nástrojů pro výuku Ruby pro nás bude představovat konzole Rails, což je program v prostředí příkazového řádku určený k interakci s aplikacemi Rails. Konzole jako taková je postavena na tzv. "interactive Ruby" (irb, interaktivní Ruby) a má tedy přístup ke všem možnostem jazyka.

Konzoli spustíme jednoduše v příkazovém řádku:

```
$ rails console
Loading development environment
>>
```

Konzole se nativně spouští v tzv. vývojovém prostředí (development environment), což je jedno ze tří samostatných prostředí, ktere Rails definuje (zbylé dvě jsou "testovací" a "produkční"). V této chvíli pro nás toto rozdělení nehraje roli, ale ještě se k němu vrátíme později.

Konzole je skvělý výukový nástroj, takže se není třeba bát prozkoumávat a experimentovat, rozbít se nám něco může povést jen sotva. Důležité jsou klávesové zkratky Ctrl-C, v případě zaseknutí, a Ctrl-D pro vypnutí konzole. Jako i v jiných podobných prostředích funguje šipka nahoru jako historie předchozích příkazů.

I když bude následující výuka poměrně detailní, pro případné bližší informace lze zaskočit i do oficiální dokumentace [Ruby API](http://ruby-doc.org/).



### Komentáře

Komentáře v Ruby začínají symbolem # (mřížka, křížek, hash, nebo nesprávně v dnešní době často i "hashtag") a vliv mají až do konce řádku. Ruby následně tzv. "zakomentované/odkomentované" řádky ignoruje, ale jsou užitečné pro čtení člověkem (často i samotným autorem). V kódu

```
# Vraci dynamicky plny nazev stranky.
def full_title(page_title = '')
  .
  .
  .
end
```

pak první řádek obsahuje komentář, který popisuje funkci, která po něm následuje.

V rámci konzole se obvykle komentáře nezahrnujou, ale pro názornou ukázku si ho můžeme zahrnout, jako například:

```
$ rails console
>> 17 + 42   # Celociselne scitani
=> 59
```

Jak vidno, komentář konzole úplně odignorovala a dodala žádaný výsledek.



### Řetězce

Řetězce (angl. strings) jsou pravděpodobně nejdůležitější datová struktura pro webové aplikace, jelikož stránky se nakonec skládají z řetězců textu, které server posílá prohlížeči. V konzoli si je tedy trochu prozkoumáme:

```
$ rails console
>> ""         # Prazdny retezec
=> ""
>> "foo"      # Neprazdny retezec
=> "foo"
```

Takovým řetězcům se říká "string literals", a jsou tvořeny skrze použití dvojitých uvozovek ("). Výpis konzole pak odpovídá vyhodnocení každého řádku, což je v případě string literals jednoduše řetězec samotný.

Aby terminologie nakonec nebyla příliš jednoduchá, můžeme na řetězce použít i operaci zřetězení, a to operátorem "+":

```
>> "foo" + "bar"    # Zretezeni retezcu
=> "foobar"
```

Kde je výsledkem "foo" plus "bar" řetězec "foobar".

Další způsob tvorby řetězců je skrze interpolaci (vkládání), použitím speciálni syntaxe "#{}":

```
>> first_name = "John"    # Prirazeni promenne
=> "John"
>> "#{first_name} Doe"     # Vlozeni retezce
=> "John Doe"
```

Nejprve jsme si přiřadili hodnotu "John" proměnné "first_name" a pak jí vložili do řetězce "#{first_name} Doe". Stejně tak bychom mohli vložit oba řetězce skrze proměnnou:

```
>> first_name = "John"
=> "John"
>> last_name = "Doe"
=> "Doe"
>> first_name + " " + last_name    # Zretezeni s vlozenou mezerou uprostred
=> "John Doe"
>> "#{first_name} #{last_name}"    # Obdobne reseni s pouzitim interpolace
=> "John Doe"
```

Výsledek je totožný, i když metoda s vkládáním mezery je poněkud krkolomná.



### Vypsání, zobrazení (print)

Pro vypsání (print) řetězce na obrazovku se obvykle používá Ruby funkce "puts" (zkráceně "put string" - vlož řetězec):

```
>> puts "foo"     # put string
foo
=> nil
```

Metoda "puts" má ale vedlejší efekt: příkaz puts "foo" vypíše řetězec na obrazovku a pak vrátí  [doslova nic](http://www.answers.com/nil) ("literally nothing"): "nil" je speciální Ruby hodnota pro "vůbec nic" (nothing at all).

Metoda "puts" rovněž automaticky vloží nový řádek poté, co vypíše řetězec, čímž se lisí od souvisejícího příkazu "print", který je jinak prakticky totožný.

```
>> print "foo"    # vypsani retezce bez vlozeni noveho radku
foo=> nil
```

Technické označení nového řádku je "newline" (nový řádek), obvyklé značený obráceným lomítkem a písmenem "n", tedy "\n". Je tedy možné napodobit chování "puts" i skrze "print", a to právě explicitním přidáním znaku pro nový řádek:

```
>> print "foo\n"  # stejne jako puts "foo"
foo
=> nil
```

### Řetězce v jednoduchých uvozovkách

Všechny dosavadní příklady používaly dvojité uvozovky, ale Ruby také podporuje uvozovky jednoduché. Pro většinu použití jsou tyto dva druhy řetězců identické:

```
>> 'foo'          # retezec v jednoduchych uvozovkach
=> "foo"
>> 'foo' + 'bar'
=> "foobar"
```

Rozdíl je hlavně v tom, že Ruby neumožňuje interpolaci do jednoduchých uvozovek:

```
>> '#{foo} bar'     # retezce v jednoduchych uvozovkach neumoznuji interpolaci
=> "\#{foo} bar"
```

Všimněme si, že konzole vrací hodnoty ve dvojitých uvozovkách a je pak nutné použít tzv únikový znak (escape character, metoda obecně se slangově nazývá escapování) pro speciální kombinace znaků, jako je "#{".

Když tedy metoda dvojitých uvozovek umožňuje všechno, co jednoduché uvozovky, a navíc zvládá interpolaci, proč používat jednoduché uvozovky? Často jsou totiž praktické díky své doslovnosti, a tímpádem obsahují doslova to, co do nich vepíšeme. Kupříkladu, znak zpětného lomítka je považován ve většině systémů za speciální, jako třeba ve formě nového řádku (\n). Pokud ale chceme, ať proměnná obsahuje zpětné lomítko, je jednodušší použít jednoduché uvozovky:

```
>> '\n'       # doslovna kombinace 'zpetne lomitko n'
=> "\\n"
```

Podobně, jako u kombinace '#{' v předchozím příkladu, Ruby potřebuje escapovat zpětné lomítko dalším zpětným lomítkem. Ve dvojitých uvozovkách je tedy doslovné lomítko reprezentováno skrze dvě zpětné lomítka.

```
>> 'Nove radky (\n) a tabulatory (\t) pouzivaji znak pro zpetne lomitko \.'
=> "Nove radky (\\n) a tabulatory (\\t) pouzivaji znak pro zpetne lomitko \\."
```

V praxi se pak jednoduché a dvojité uvozovky používají jednoduše podle potřeby, a v drtivé většině případů jsou vcelku beztrestně zaměnitelné.

### Objekty a předávání zpráv

Všechno v Ruby, včetně řetězců a dokonce i "nil" je tzv. objekt. Techničtější význam si vysvětlíme později, ale obzvlášť v tomto případě se nejlépe učí příkladem (tedy, spoustou příkladů, ideálně).

Je jednodušší vysvětlit, co objekty dělají, a to že reagují na tzv. zprávy (messages). Objekt jako je řetězec, například, zvládá reagovat na zprávu "length" (délka), která vrátí počet znaků v řetězci.

```
>> "foobar".length        # Predani zpravy "length" retezci
=> 6
```

Obvykle jsou zprávy, které jsou předány objektům tzv. metody, což jsou funkce, ke kterým mají tyto objekty přístup. Řetězce také reagují na metodu "empty?" (prázdný?):

```
>> "foobar".empty?
=> false
>> "".empty?
=> true
```

Otazník na konci značí, že hodnota, kterou nám funkce vrátí, bude tzv. boolean, tedy pravda (true) nebo nepravda (false).

```
>> s = "foobar"
>> if s.empty?
>>   "Retezec je prazdny"
>> else
>>   "Retezec je neprazdny"
>> end
=> "Retezec je neprazdny"
```

Pro přidání více podmínek můžeme použít tzv. "elsif" (else + if, česky "nebo pokud"):

```
>> if s.nil?
>>   "Promenna je nil"
>> elsif s.empty?
>>   "Retezec je prazdny"
>> elsif s.include?("foo")
>>   "Retezec obsahuje 'foo'"
>> end
=> "Retezec obsahuje 'foo'"
```

Booleany mohou být kombinovány také použitím operátorů && ("logické A"), || ("logické NEBO") a ! ("inverzní operátor", resp. negace, také "není", podle kontextu):

```
>> x = "foo"
=> "foo"
>> y = ""
=> ""
>> puts "Oba retezce jsou prazdne" if x.empty? && y.empty?
=> nil
>> puts "Jeden z retezcu je prazdny" if x.empty? || y.empty?
"Jeden z retezcu je prazdny"
=> nil
>> puts "x neni prazdne" if !x.empty?
"x neni prazdne"
=> nil
```

Jelikož je vše v Ruby objekt, může i "nil" reagovat na metody. Kupříkladu metoda "to_s", která převede prakticky jakýkoliv objekt na řetězec:

```
>> nil.to_s
=> ""
```

Což, není velkým překvapením, vrátí prázdný řetězec. Zprávy (metody) jdou za sebou nicméně také řetězit:

```
>> nil.empty?
NoMethodError: undefined method `empty?' for nil:NilClass
>> nil.to_s.empty?      # retezeni zprav
=> true
```

Objekt "nil" jako takový na metodu "empty?" nereaguje, ale "nil.to_s", jelikož uz byl převeden na řetezec, ano. Na testování, zda jde o "nil" existuje také přímo metoda:

```
>> "foo".nil?
=> false
>> "".nil?
=> false
>> nil.nil?
=> true
```

Kód

```
puts "x neni prazdne" if !x.empty?
```

je i dobrým příkladem alternativního použití slova "if": Ruby umožňuje psaní tvrzení, která jsou vyhodnocena jen tehdy, pokud je následující tvrzení pravdivé. Současně existuje i slovo "unless", které funguje obdobně:

```
>> string = "foobar"
>> puts "Retezec '#{string}' je neprazdny." unless string.empty?
Retezec 'foobar' je neprazdny.
=> nil
```



### Definování metod

Konzole nám umožňuje definovat metody stejně, jako jsme si definovali akci "home" nebo pomocníka "full_title" minule. (V praxi je samozřejmě tvorba metod v konzoli nepraktická a obvykle se prostě použije soubor.) Definujme si tedy funkci "string_message", která vezme jeden parametr a vrátí zprávu na základě toho, zda je parametr prázdný, nebo ne:

```
>> def string_message(str = '')
>>   if str.empty?
>>     "Je to prazdny retezec!"
>>   else
>>     "Retezec je neprazdny."
>>   end
>> end
=> :string_message
>> puts string_message("foobar")
Retezec je neprazdny.
>> puts string_message("")
Je to prazdny retezec!
>> puts string_message
Je to prazdny retezec!
```

Na posledním příkladu je vidět, že je možné parametr vynechat úplně (a spolu s ním i závorky). Díky tomu, že kód

```
def string_message(str = '')
```

obsahuje nativní (default, v češtině slangově často "defaultní") parametr, což je v tomto případě prázdný řetězec. Tím je argument "str" volitelný, a pokud ho nezadáme, automaticky je mu přiřazena nativní hodnota.

Funkce v ruby mají "implicitní návrat", což znamená, že vrací poslední vyhodnocené tvrzení - v tomto případě jednu ze dvou zpráv, podle toho, zda je parametr "str" prázdný, nebo ne. Ruby obsahuje i možnost explicitního návratu; následující funkce je ekvivalentní s předchozí:

```
>> def string_message(str = '')
>>   return "Je to prazdny retezec!" if str.empty?
>>   return "Retezec je neprazdny."
>> end
```

Je důležité vědět i to, že jméno parametru funkce je irelevantní z pohledu volání. Jinak řečeno, "str" bychom mohli nahradit jakýmkoliv platným jménem proměnné, jako třeba "the_function_argument", a výsledek se nezmění:

```
>> def string_message(the_function_argument = '')
>>   if the_function_argument.empty?
>>     "Je to prazdny retezec!"
>>   else
>>     "Retezec je neprazdny."
>>   end
>> end
=> nil
>> puts string_message("")
Je to prazdny retezec!
>> puts string_message("foobar")
Retezec je neprazdny.
```

### Zpět k "title" pomocníkovi

Díky předchozí exkurzi už máme podstatně lepší pochopeni "full_title" pomocníka (app/helpers/application_helper.rb), kterého si můžeme i okomentovat:

```
module ApplicationHelper

  # Vraci dynamicky plny nazev stranky.               # Dokumentacni komentar
  def full_title(page_title = '')                     # Definice metody, volitelny parametr
    base_title = "Ruby on Rails Tutorial"             # Prirazeni promenne
    if page_title.empty?                              # Boolean test
      base_title                                      # Implicitni vraceni hodnoty
    else
      page_title + " | " + base_title                 # Zretezeni retezcu
    end
  end
end
```

Tyto prvky - definice funkce, přiřazení proměnné, boolean testy, řídící struktura a zřetězení řetězců znaků společně vytvoří úsporného pomocníka pro rozvržení naší stránky. Poslední prvek je "module ApplicationHelper": moduly nám umožňují spolu související metody úhledně ukládat do balíčků, které pak lze zakomponovat do Ruby použitím "include". V případě modulu pomocníků za nás nicméně inkluzi provede Rails. Díky tomu je metoda "full_title" automaticky dostupná ve všech našich pohledech.



## Jiné datové struktury

I když webové aplikace sestávají primárně z řetězců, pořád na jejich tvorbu často potřebujeme i jiné datové typy. Přiblížíme si tedy některé datové struktury Ruby, které jsou důležité pro psaní Railsových aplikací.



### Pole (arrays) a rozsahy (intervaly - ranges)

Pole ("array", tedy pole hodnot, ne to plné řepky) je vlastně jen seznam prvků ve specifickém pořadí. Jejich pochopení bude praktické i pro pozdější pochopení haší, a pro koncepci modelování dat v Rails (jako asociace "has_many", a podobně).

Zatím jsme věnovali hodně času pochopení řetězců, a existuje jednoduchý způsob, jak přenést řetězce do polí metodou "split":

```
>>  "foo bar     baz".split     # Rozdeleni retezce do trojkprvkoveho pole.
=> ["foo", "bar", "baz"]
```

Výsledkem operace je pole tří řetězců. Pokud se nezadá metodě "split" specifický parametr, pak dělí řetězec na základe mezer, ale obecně lze dělit řetězec prakticky kterýmkoliv znakem:

```
>> "fooxbarxbaz".split('x')
=> ["foo", "bar", "baz"]
```

Jak je v mnoha počítačových jazycích zvykem, pole v Ruby začínají na hodnotě 0 (zero-offset), takže první prvek v poli má index 0, druhý 1, a tak dále:

```
>> a = [42, 8, 17]
=> [42, 8, 17]
>> a[0]               # Pro pristup k polim pouziva ruby hranate zavorky
=> 42
>> a[1]
=> 8
>> a[2]
=> 17
>> a[-1]              # Indexy mohou byt i zaporne!
=> 17
```

Jak jsme si ukázali, Ruby používá hranaté závorky pro přístup k prvkům pole. Jako praktický doplňek Ruby umožňuje i synonyma k některým prvkům, ke kterým se často přistupuje:

```
>> a                  # Jen pripominka, z ceho se sklada 'a'
=> [42, 8, 17]
>> a.first
=> 42
>> a.second
=> 8
>> a.last
=> 17
>> a.last == a[-1]    # Porovnani pouzitim ==
=> true
```

Poslední řádek obsahuje operátor pro porovnání rovnosti == ("je rovno"), který Ruby sdílí s mnoha dalšími jazyky, spolu se souvisejícím != ("není rovno"), atd.:

```
>> x = a.length       # Podobne jako retezce i pole reaguji na metodu 'length'
=> 3
>> x == 3
=> true
>> x == 1
=> false
>> x != 1
=> true
>> x >= 1
=> true
>> x < 1
=> false
```

Kromě "length" pole reagují i na spoustu dalších metod:

```
>> a
=> [42, 8, 17]
>> a.empty?
=> false
>> a.include?(42)
=> true
>> a.sort
=> [8, 17, 42]
>> a.reverse
=> [17, 8, 42]
>> a.shuffle
=> [17, 42, 8]
>> a
=> [42, 8, 17]
```

Jak vidíme, žádná z vypsaných metod "a" jako takové nemění. Pro změnu (mutaci) pole je třeba použít příslušné "bang" (slangové označení vykřičníku v podobném kontextu) metody:

```
>> a
=> [42, 8, 17]
>> a.sort!
=> [8, 17, 42]
>> a
=> [8, 17, 42]
```

Do polí lze přidávat hodnoty metodou "push", nebo operátorem << (zvaný "shovel operator", tedy "lopatový"):

```
>> a.push(6)                  # Vlozeni 6 do pole
=> [42, 8, 17, 6]
>> a << 7                     # Vlozeni 7 do pole
=> [42, 8, 17, 6, 7]
>> a << "foo" << "bar"        # Retezeni vkladani do pole
=> [42, 8, 17, 6, 7, "foo", "bar"]
```

Poslední příklad také demonstruje, že lze vkládání navazovat. Také, a tak to v mnoha jazycích nebývá, je možno v Ruby mít pole které obsahuje různé datové typy (v našem případě tedy řetězce a celá čísla).

Už víme, že metodou "split" lze převést řetězec na pole. Funguje to ale i obráceně, pomocí metody "join":

```
>> a
=> [42, 8, 17, 6, 7, "foo", "bar"]
>> a.join                       # Spojeni bez parametru
=> "4281767foobar"
>> a.join(', ')                 # Spojeni s carkou a mezerou
=> "42, 8, 17, 6, 7, foo, bar"
```

Blízko k polím mají tzv. rozsahy, někdy též intervaly (ranges), které jsou nejsnáze pochopitelné jejich převedením na pole metodou "to_a":

```
>> 0..9
=> 0..9
>> 0..9.to_a              # Hopla, volame to_a na 9
NoMethodError: undefined method `to_a' for 9:Fixnum
>> (0..9).to_a            # Budeme muset pouzit zavorky
=> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
```

I když je 0..9 platný rozsah, je třeba ho schovat do závorek, aby na něj šlo použít metodu.

Rozsahy jsou praktické pro vytahování prvků pole:

```
>> a = %w[foo bar baz quux]         # Pouzijeme %w pro vytvoreni pole retezcu
=> ["foo", "bar", "baz", "quux"]
>> a[0..2]
=> ["foo", "bar", "baz"]
```

Obzvlášť praktický trik je použití indexu -1 na konci rozsahu, což vybere veškeré prvky od udaného prvního bodu až na konec pole, aniž bychom museli jakkoliv zjišťovat délku pole:

```
>> a = (0..9).to_a
=> [0, 1, 2, 3, 4, 5, 6, 7, 8, 9]
>> a[2..(a.length-1)]               # Explicitni pouziti delky pole
=> [2, 3, 4, 5, 6, 7, 8, 9]
>> a[2..-1]                         # Pouziti triku s -1
=> [2, 3, 4, 5, 6, 7, 8, 9]
```

Rozsahy také fungují se znaky:

```
>> ('a'..'e').to_a
=> ["a", "b", "c", "d", "e"]
```



### Bloky

Jak pole, tak rozsahy reagují na metody, které přijímají bloky, což je jedna z nejsilnějších a zároveň nejvíce matoucích možností Ruby:

```
>> (1..5).each { |i| puts 2 * i }
2
4
6
8
10
=> 1..5
```

Tento kód zavola metodu "each" na rozsah (1..5) a předá mu blok { |i| puts 2 * i }. Svislé čáry okolo proměnné v |i| je syntaxe Ruby pro blokovou proměnnou, a metoda sama pak musí vědět, jak se k ní chovat. V tomto případě metoda "each" použitá na rozsah zpracuje blok s jednou místní proměnnou, kterou jsme nazvali "i", a provede obsah bloku na každou hodnotu v rozsahu.

Složené závorky jsou jednou metodou, jak označit blok, ale existuje i druhá: 

```
>> (1..5).each do |i|
?>   puts 2 * i
>> end
2
4
6
8
10
=> 1..5
```

Bloky mohou být i víceřádkové, a často také bývají. Budeme používat obvyklý přístup, kdy nám složené závorky poslouží pro krátké, jednořádkové bloky a syntaxe "do..end" zase pro delší jednořádkové, nebo víceřádkové:

```
>> (1..5).each do |number|
?>   puts 2 * number
>>   puts '--'
>> end
2
--
4
--
6
--
8
--
10
--
=> 1..5
```

Proměnnou "number" jsme použili namísto "i" jen proto, aby bylo zřejmé, že její jméno může být jakékoliv.

Bohužel, pokud člověk nema výrazné zkušenosti s programováním, nejde bloky pochopit kdovíjak jednoduše; prostě je potřeba je vídat dostatečně často a časem se vstřebají do povědomí. Příklady hodně pomáhají, tak si ještě pár uvedeme (i s použitím metody "map"):

```
>> 3.times { puts "Betelgeuse!" }   # 3.times vezme blok bez promennych
"Betelgeuse!"
"Betelgeuse!"
"Betelgeuse!"
=> 3
>> (1..5).map { |i| i**2 }          # ** je operator pro umocneni
=> [1, 4, 9, 16, 25]
>> %w[a b c]                        # %w tvori pole retezcu
=> ["a", "b", "c"]
>> %w[a b c].map { |char| char.upcase }
=> ["A", "B", "C"]
>> %w[A B C].map { |char| char.downcase }
=> ["a", "b", "c"]
```

Metoda "ma" vrací výsledek aplikace bloku na každý prvek pole nebo rozsahu. V posledních dvou případech blok uvnitř "map" zahrnuje volání specifické metody na blokovou proměnnou, a často se zkracuje technikou zvanou "symbol-to-proc":

```
>> %w[A B C].map { |char| char.downcase }
=> ["a", "b", "c"]
>> %w[A B C].map(&:downcase)
=> ["a", "b", "c"]
```

K symbolu, který je zde použit se dostaneme záhy. Jako poslední příklad bloků se můžeme podívat na individuální test, kterému jsme se nedávno věnovali (test/controllers/static_pages_controller_test.rb):

```
test "should get home" do
  get static_pages_home_url
  assert_response :success
  assert_select "title", "Ruby on Rails Tutorial"
end
```

Můžeme předpokládat, že díky přítomnosti slova "do" je tělo testu vlastně blok. metoda "test" vezme řetězec jako parametr (popis) a blok, a provede obsah těla bloku v rámci testování jako takového.



### Haše a symboly

Haše jsou v podstatě pole, která nejsou omezena na indexování pomocí integerů (celočíselných hodnot). V jiných jazycích se jim proto říká také "asociativní pole". Namísto toho mohou být v haši indexy, nebo též "klíče" prakticky jakýkoliv objekt. Například můžeme jako klíče použít řetězce:

```
>> user = {}                          # {} je prazdna has.
=> {}
>> user["first_name"] = "John"        # Klic "first_name", hodnota "John"
=> "Michael"
>> user["last_name"] = "Doe"          # Klic "last_name", hodnota "Doe"
=> "Hartl"
>> user["first_name"]                 # Pristup k prvkum je jako v polich
=> "Michael"
>> user                               # Doslovna reprezentace hase
=> {"last_name"=>"Hartl", "first_name"=>"Michael"}
```

Haše se značí složenými závorkami, které obsahují páry klíč-hodnota (key-value); prázdné závorky {} bez takových párů značí prázdnou haši. I když je to zprvu matoucí, složené závorky haší nemají nic společného s jejich obdobou v blocích. I když jsou haše hodně podobné polím, jeden z hlavních rozdílů je ten, že haše obecně nezaručují posloupnost prvků v konkrétním pořadí. Pokud na pořadí záleží, je lepší použít pole.

Namísto definování haší po jedné za pomocí hranatých závorek, je praktičtější použití syntaxe "klíč" => "hodnota":

```
>> user = { "first_name" => "John", "last_name" => "Doe" }
=> {"last_name"=>"Doe", "first_name"=>"John"}
```

Je zvykem mezi operátor vepisovat i mezery pro lepší přehlednost, i když je výstup konzole nakonec ignoruje.

Doteď jsme jako klíče haší používali řetězce, ale je daleko běžnější použít tzv. symboly. Symboly trochu vypadají jako řetězce, ale začínají dvojtečkou namísto toho, aby byly uzavřeny v uvozovkách. Například :name je symbol. Jde v podstatě o takové "odlehčené" řetězce.

```
>> "name".split('')
=> ["n", "a", "m", "e"]
>> :name.split('')
NoMethodError: undefined method `split' for :name:Symbol
>> "foobar".reverse
=> "raboof"
>> :foobar.reverse
NoMethodError: undefined method `reverse' for :foobar:Symbol
```

Symboly jsou speciální datový typ Ruby, který s ním sdílí jen velmi málo jiných jazyků, a ze začátku se tedy jeví podivně, nicméně Ruby je používá poměrně často a je to tedy hlavně otázka zvyku. Narozdíl od řetězců nejsou všechny znaky platné:

```
>> :foo-bar
NameError: undefined local variable or method `bar' for main:Object
>> :2foo
SyntaxError
```

Dokud ale začínají symboly písmenem a používají jen bežné znaky, nebývá s nimi problém.

Ve smyslu symbolů jako klíčů haše můžeme definovat haš "user" následovně:

```
>> user = { :name => "John Doe", :email => "john@example.com" }
=> {:name=>"John Doe", :email=>"john@example.com"}
>> user[:name]              # Pristup k hodnote odpovidajici :name
=> "John Doe"
>> user[:password]          # Pristup k hodnote klice, ktery nebyl definovan
=> nil
```

Jak vidíme, hodnota haše pro nedefinovaný klíč je prostě "nil".

Jelikož je běžné používat symboly jako klíče haší, od verze 1.9 Ruby podporuje novou syntaxi právě pro tento případ:

```
>> h1 = { :name => "John Doe", :email => "john@example.com" }
=> {:name=>"John Doe", :email=>"john@example.com"}
>> h2 = { name: "John Doe", email: "john@example.com" }
=> {:name=>"John Doe", :email=>"john@example.com"}
>> h1 == h2
=> true
```

Druhá varianta syntaxe nahrazuje kombinaci => jménem klíče, které následuje dvojtečka a hodnota:

```
{ name: "John Doe", email: "john@example.com" }
```

Tato struktura daleko více připomíná styl zápisu v jiných jazycích (jako je JavaScript) a její obliba roste i v komunitě Rails. Jelikož se ale běžně používají obě varianty, je třeba je znát.

Hodnoty haše mohou být prakticky cokoliv, dokonce i jiné haše:

```
>> params = {}        # Definovani hase zvane 'params' (zkracene 'parameters').
=> {}
>> params[:user] = { name: "John Doe", email: "john@example.com" }
=> {:name=>"John Doe", :email=>"john@example.com"}
>> params
=> {:user=>{:name=>"John Doe", :email=>"john@example.com"}}
>>  params[:user][:email]
=> "john@example.com"
```

Tyto "haše v haši", nebo jednodušeji "vnořené haše" (nested hashes) se v Rails často používají.

Stejně jako pole a rozsahy, i haše reagují na metodu "each". Například mějme haši zvanou "flash" s klíči pro dvě podmínky, ":success" a ":danger":

```
>> flash = { success: "Fungovalo to!", danger: "Nefungovalo to." }
=> {:success=>"Fungovalo to!", :danger=>"Nefungovalo to."}
>> flash.each do |key, value|
?>   puts "Key #{key.inspect} has value #{value.inspect}"
>> end
Key :success has value "Fungovalo to!"
Key :danger has value "Nefungovalo to."
```

Všimněme si, že metoda "each" v rámci polí vezme blok jen s jednou proměnnou, ale u haší vezme dvě: klíč a hodnotu. Tímpádem "each" metoda prochází haší po krocích sestávajících z párů klíč-hodnota.

Poslední příklad používá praktickou metodu "inspect", která vrací řetězec s doslovnou reprezentací objektu, na který je volána:

```
>> puts (1..5).to_a            # Vypis pole jako retezec
1
2
3
4
5
>> puts (1..5).to_a.inspect    # Vypis pole doslovne
[1, 2, 3, 4, 5]
>> puts :name, :name.inspect
name
:name
>> puts "Fungovalo to!", "Fungovalo to!".inspect
Fungovalo to!
"Fungovalo to!"
```

Mimochodem, použití "inspect" pro vypsání objektu je tak běžné, že pro to existuje i zkratka "p":

```
>> p :name             # Stejny vystup jako 'puts :name.inspect'
:name
```

### Návrat k CSS

Vrátíme se k řádku v souboru rozvržení (app/views/layouts/application.html.erb), kterému už téměř zvládneme porozumět.

```
<%= stylesheet_link_tag 'application', media: 'all',
                                       'data-turbolinks-track': 'reload' %>
```

Jak jsme si zmínili dříve, Rails má speciální funkci pro zahrnutí stylů, a

```
stylesheet_link_tag 'application', media: 'all',
                                   'data-turbolinks-track': 'reload'
```

je právě volání této funkce. Je tu ale několik záhad. V prvé řadě, kde jsou závorky? V Ruby jsou nepovinné, tímpádem jsou tyto varianty ekvivalentní:

```
# zavorky jsou pri volani funkci nepovinne
stylesheet_link_tag('application', media: 'all',
                                   'data-turbolinks-track': 'reload')
stylesheet_link_tag 'application', media: 'all',
                                   'data-turbolinks-track': 'reload'
```

V druhé řadě, parametr "media" vypadá jako haš, ale kde má složené závorky? Pokud je haš posledním prvkem ve volání funkce, jsou složené závorky také nepovinné, takže jsou i obě tyto varianty platné:

```
# slozene zavorky jako posledni parametr hase jsou nepovinne
stylesheet_link_tag 'application', { media: 'all',
                                     'data-turbolinks-track': 'reload' }
stylesheet_link_tag 'application', media: 'all',
                                   'data-turbolinks-track': 'reload'
```

A nakonec, jaktože si Ruby správně vykládá řádky

```
stylesheet_link_tag 'application', media: 'all',
                                   'data-turbolinks-track': 'reload'
```

i když je mezi posledními prvky zalomení řádku? Jednoduše, Ruby nerozlišuje v takovém kontextu mezi novým řádkem a mezerou. Kód jsme si zalomili čistě z důvodu lepší čitelnosti.

Vidíme tedy, že řádek

```
stylesheet_link_tag 'application', media: 'all',
                                   'data-turbolinks-track': 'reload'
```

volá funkci "stylesheet_link_tag" se dvěma parametry: řetězcem, který obsahuje cestu ke stylům, a haší se dvěma prvky, indikuje typ médií a říká Rails ať použije funkci [turbolinks](https://github.com/rails/turbolinks), přidanou v Rails 4.0. Díky uzavření v <%= %> jsou výsledky vloženy do šablony skrze ERb, a pokud se podíváme na zdrojový kód stránky v prohlížeči, zjistíme, jaké HTML je potřeba k zahrnutí stylů:

```
<link data-turbolinks-track="true" href="/assets/application.css" media="all"
rel="stylesheet" />
```



## Třídy v Ruby

Dříve jsme si ujasnili, že vše v Ruby je objekt, a v této části si konečně nadefinujeme nějaké vlastní. Jako i spousta dalších objektově orientovaných jazyků používá Ruby pro organizaci metod třídy; tyto třídy jsou pak instancovány a vytvoří objekty. Pokud člověk s objektově orientovaným programováním (OOP) nemá zkušenosti zní předchozí tvrzení podivně, ale vše se ujasní, když si uvedeme pár příkladů.

### Konstruktory

Vlastně za sebou už máme hodně případů, kdy jsme použili třídy pro instancování objektů, ale ještě jsme to neudělali explicitně. Například jsme instancovali řetězec za pomocí dvojitých uvozovek, což je doslovný konstruktor pro řetězce:

```
>> s = "foobar"       # doslovny konstruktor pro retezce za pouziti dvojitych uvozovek
=> "foobar"
>> s.class
=> String
```

Řetězec reaguje na metodu "class" a vrátí třídu, do které spadá.

Namísto použití doslovného konstruktoru ale můžeme použít ekvivalentní metodu "jmenného" (pojmenovaného) konstruktoru, která zahrnuje zavolání metody "new" na jméno třídy:

```
>> s = String.new("foobar")   # jmenny konstruktor pro retezec
=> "foobar"
>> s.class
=> String
>> s == "foobar"
=> true
```

Oba postupy jsou ekvivalentní, ale v tomto případě jsme mnohem explicitnější v tom, co chceme udělat.

Pole fungují stejně, jako řetězce:

```
>> a = Array.new([1, 3, 2])
=> [1, 3, 2]
```

Haše, na druhou stranu, fungují trochu jinak. Konstruktor pro pole "Array.new" přijme prvotní hodnotu pro pole, ale "Hash.new" vezme nativní hodnotu pro haš, což je hodnota haše pro neexistující klíč:

```
>> h = Hash.new
=> {}
>> h[:foo]            # pokus o pristup k hodnote neexistujiciho klice :foo.
=> nil
>> h = Hash.new(0)    # takto nam neexistujici klice budou vracet 0 namisto nil
=> {}
>> h[:foo]
=> 0
```

Když je metoda zavolána na třídu samotnou, jako v případě "new", říká se jí třídní metoda. Výsledek zavolání "new" na třídu je objekt této třídy, také zvaný instance třídy. Metoda volaná na instanci, jako "length", se nazývá metoda instance (instanční metoda).

### Dědičnost tříd

V rámcí výuky o třídách je praktické moct zjistit, jaká je třídní hierarchie pomocí metody "superclass":

```
>> s = String.new("foobar")
=> "foobar"
>> s.class                        # najdi tridu s
=> String
>> s.class.superclass             # najdi supertridu retezce
=> Object
>> s.class.superclass.superclass  # zakladni trida Ruby je BasicObject
=> BasicObject
>> s.class.superclass.superclass.superclass
=> nil
```

Ukážeme si i diagram hierarchie dědičnosti (obr. 4.1). Je na něm vidět, že supertřída řetězce je objekt, a supertřída objektu je BasicObject, ale ten už žádnou supertřídu nemá. Tento vzorec se dá aplikovat na v podstatě všechny objekty Ruby: postupně projít třídní hierarchií až do chvíle, než se dostaneme k BasicObject, ze kterého tímpádem všechny třídy přeneseně dědí. Proto to tvrzení, že "vše v Ruby je objekt".

// obr. 4.1 Hierarchie dědičnosti pro třídu "String" (řetězec).

Pro lepší pochopení tříd není nic lepšího, než si vytvořit nějakou vlastní. Připravíme si tedy třídu "Word" s metodou "palindrome?", která vrátí true pokud je slovo stejné při čtení dopředu i pozpátku:

```
>> class Word
>>   def palindrome?(string)
>>     string == string.reverse
>>   end
>> end
=> :palindrome?
```

Třídu můžeme použít následovně:

```
>> w = Word.new              # vytvoreni noveho Word objektu
=> #<Word:0x22d0b20>
>> w.palindrome?("foobar")
=> false
>> w.palindrome?("level")
=> true
```

Tento příklad se ale netváří krapet nepřirozeně jen tak pro nic za nic. Je přecejen divné, abychom vytvářeli novou třídu jen kvůli metody, která vezme řetězec jako parametr. Jelikož slovo je řetězec, je daleko přirozenější, aby naše třída "Word" dědila ze "String": (Bude třeba konzoli vypnout a zapnout, aby se zbavila původní definice "Word".)

```
>> class Word < String             # Word dedi ze String.
>>   # vraci true pokud je retezec stejny i obracene
>>   def palindrome?
>>     self == self.reverse        # self je samotny retezec
>>   end
>> end
=> nil
```

Syntaxe "Word < String" se používá pro dědění, která zaručí, že krom nové metody "palindrome?" budou mít slova přístup i ke stejným metodám, jako řetězce:

```
>> s = Word.new("level")    # vytvor novy Word, inicializovany s hodnotou "level"
=> "level"
>> s.palindrome?            # Word ma pristup k metode palindrome?
=> true
>> s.length                 # Word zaroven dedi vsechny metody, ktere maji retezce
=> 5
```

Jelikož třída "Word" dědí ze "String", můžeme zase použít konzoli k explicitnímu výpisu hierarchie:

```
>> s.class
=> Word
>> s.class.superclass
=> String
>> s.class.superclass.superclass
=> Object
```

Hierarchie je k vidění zde (obr. 4.2):

// obr 4.2: Hierarchie dědičnosti pro vlastní třídu "Word"

V rámci definování třídy "Word" jsme mimochodem přistoupili ke slovu uvnitř třídy "Word". Ruby nám takový přístup umožňuje skrze použití klíčového slova "self": uvnitř třídy "Word" je "self" objekt samotný, a tedy můžeme použít

```
self == self.reverse
```

pro ověření, zda je slovo palindrom. V rámci třídy "String" je dokonce použití "self." volitelné, pokud jde o metody nebo atributy (pokud nepřiřazujeme), takže

```
self == reverse
```

by fungovalo také.



### Upravování zabudovaných tříd

I když je dědičnost silný koncept, v případě palindromů by bylo asi ještě přirozenější přidat metodu "palindrome?" přímo do třídy "String", abychom mohli volat metodu "palindrome?" i na doslovnou definici řetězce, což momentálně nemůžeme:

```
>> "level".palindrome?
NoMethodError: undefined method `palindrome?' for "level":String
```

Ruby nám ale něco takového umožňuje; třídy v ruby mohou být otevřeny a modifikovány, což umožňuje obyčejným smrtelníkum do nich přidávat metody:

```
>> class String
>>   # vraci true pokud je retezec stejny i obracene
>>   def palindrome?
>>     self == self.reverse
>>   end
>> end
=> nil
>> "deified".palindrome?
=> true
```

Upravování zabudovaných tříd je mocná technika, ale s velkou mocí přichází velka zodpovědnost, a obecně se neholduje přístupu přidávání metod do zabudovaných tříd, pokud pro to nemáme opravdu dobrý důvod. Rails často dobré důvody má; například se u webových aplikací často stává, že chceme zamezit tomu, aby byly proměnné prázdné - tedy že například uživatelské jméno nesmí sestávat jen z mezer, nebo dokonce vůbec ničeho. Rails proto do Ruby přidá metodu "blank?". Jelikož konzole automaticky obsahuje rozšíření Rails, můžeme si to vyzkoušet i na příkladu:

```
>> "".blank?
=> true
>> "      ".empty?
=> false
>> "      ".blank?
=> true
>> nil.blank?
=> true
```

Jak vidíme, řetězec mezer není "prázdný" v doslovném smyslu, ale je "nevyplněný" (tedy "blank"). Další příklady toho, co Rails přidává do tříd Ruby si ještě projdeme později.

### Třída ovladače

Všechno to povídání o třídách a dědičnosti nám mohlo připomenout, že už jsme něco takového viděli, konkrétně v ovladači statických stránek:

```
class StaticPagesController < ApplicationController

  def home
  end

  def help
  end

  def about
  end
end
```

Můžeme se teď už dovtípit, alespon zhruba, co onen kód znamená: "StaticPagesController" je třída, která dědí z "ApplicationController" a má k dispozici metody "home", "help" a "about". Jelikož si konzole Rails nahraje místní prostředí, můžeme si dokonce vytvořit ovladač explicitně a prozkoumat jeho třídní hierarchii:

```
>> controller = StaticPagesController.new
=> #<StaticPagesController:0x22855d0>
>> controller.class
=> StaticPagesController
>> controller.class.superclass
=> ApplicationController
>> controller.class.superclass.superclass
=> ActionController::Base
>> controller.class.superclass.superclass.superclass
=> ActionController::Metal
>> controller.class.superclass.superclass.superclass.superclass
=> AbstractController::Base
>> controller.class.superclass.superclass.superclass.superclass.superclass
=> Object
```

Diagram hierarchie vypadá následovně (obr. 4.3):

// obr. 4.3 Hierarchie dědičnosti pro statické stránky.

Dokonce můžeme zavolat akce ovladače z konzole, což jsou ale pouze metody:

```
>> controller.home
=> nil
```

Hodnota "nil" se nám vrátila proto, že je akce "home" prázdná.

Ale moment - akce nemají hodnoty, které by vracely, alespoň tedy ne takové, na kterých by záleželo. Smysl akce "home", jak jsme viděli dříve, je vykreslit webovou stránku, ne vrátit hodnotu. A určitě jsme si nikde nevolali "StaticPagesController.new". Co se to tedy děje?

Jde totiž o to, že Rails je napsaný v Ruby, ale to neznamená, že by Rails byl Ruby. Některé třídy Rails se používají jako běžné objekty Ruby, ale některé jsou jen "vodou pro kouzelný mlýn" Ruby. Rails je "sui generis" (lat. "svého druhu"), a je třeba ho studovat a pochopit od Ruby zvlášť.

### Třída uživatele

Zakončíme naši exkurzi Ruby vlastní třídou "User", kterou využijeme v modelu uživatelů.

Prozatím jsme si definovali třídy skrze konzoli, což ale začne být brzy krapet únavné; raději si vytvoříme soubor "example_user.rb" v kořenovém adresáři aplikace a naplníme ho následovně:

```
class User
  attr_accessor :name, :email

  def initialize(attributes = {})
    @name  = attributes[:name]
    @email = attributes[:email]
  end

  def formatted_email
    "#{@name} <#{@email}>"
  end
end
```

Děje se zde poměrně dost věcí, takže si je postupně probereme. První řádek,

```
  attr_accessor :name, :email
```

vytváří "atributové přistupovače" (attribute accessors), které odpovídají uživatelově jménu a emailové adrese. Tento postup vytvoří tzv. "getter" a "setter" metody, které nám umožní získat/zjistit (get) a nastavit (set) instanční proměnné "@name" a "@email", které jsme krátce zmiňovali už dříve. Základní smysl instančních proměnných v Rails je ten, že jsou automaticky přístupné ve všech pohledech, ale obecně se používají především pro proměnné, které musí být k dispozici skrze celou třídu Ruby. Instanční proměnné vždy začínají znakem "@", a pokud jsou nedefinované, mají hodnotu "nil".

První metoda, "initialize", je v Ruby speciální: tato metoda se volá když provádíme "User.new". Tato konkrétni "initialize" bere jeden parametr, "attributes":

```
  def initialize(attributes = {})
    @name  = attributes[:name]
    @email = attributes[:email]
  end
```

Proměnná "attributes" zde má základní hodnotu rovnou prázdné haši, abychom mohli zadefinovat uživatele beze jména nebo emailové adresy.

Naše třída nakonec definuje metodu nazvanou "formatted_email", která použije hodnoty přiřazené proměnným "@name" a "@email" pro vytvoření pěkně formátované verze uživatelova emailu za použití interpolace řetězců:

```
  def formatted_email
    "#{@name} <#{@email}>"
  end
```

Jelikož jsou "@name" a "@email" instanční proměnné, jsou automaticky přístupné i v metodě "formatted_email".

Zbývá jen nahodit konzoli, použít "require" pro zapojení kódu v souboru a trochu si naši novou třídu otestovat:

```
>> require './example_user'     # takto se nahrava kod z example_user
=> true
>> example = User.new
=> #<User:0x224ceec @email=nil, @name=nil>
>> example.name                 # nil, jelikoz attributes[:name] je nil
=> nil
>> example.name = "Example User"           # prirazeni jmena, ktere neni nil
=> "Example User"
>> example.email = "user@example.com"      # a obdobne i emailove adresy, ktera neni nil
=> "user@example.com"
>> example.formatted_email
=> "Example User <user@example.com>"
```

Použili jsme unixové "." pro "adresář, ve kterém se nacházíme" a "./example_user" pro prohledání relativního umístění, kde se soubor nalézá. Následující kód vytvoří testovacího uživatele a na naplní jeho jméno a emailovou adresu přiřazením do souvisejících atributů (přiřazení jsou umožněna řádkem "attr_accessor" v našem souboru). Když napíšeme

```
example.name = "Example User"
```

Ruby nastaví hodnotu proměnné "@name" na "Example User" (obdobně i v případě emailu), kterou pak použijeme v metodě "formatted_email". Pro konečné parametry haše můžeme opět vynechat složené závorky a také vytvořit nového uživatele s předdefinovaným obsahem předáním haše metodě "initialize":

```
>> user = User.new(name: "John Doe", email: "john@example.com")
=> #<User:0x225167c @email="john@example.com", @name="John Doe">
>> user.formatted_email
=> "John Doe <john@example.com>"
```

V budoucnu se setkáme i s běžnou praxí v podobě techniky zvané masové přiřazování (mass assignment), kde se inicializují objekty za použití haše jako parametru ve velkém.



## Shrnutí

Tím končí náš přehled jazyka Ruby. V další části nově nabraných informací patřičně zužitkujeme ve formě dalšího vývoje naší aplikace.

Soubor "example_user.rb" už potřebovat nebudeme, takže ho můžeme klidně smazat:

```
$ rm example_user.rb
```

Poté odešleme ostatní změny do repozitáře hlavního zdrojového kódu a spojíme do hlavní větve.

Then commit the other changes to the main source code repository and merge into the `master` branch, push up to Bitbucket, and deploy to Heroku:

```
$ git commit -am "pridan pomocnik full_title"
$ git checkout master
$ git merge rails-flavored-ruby
```

Nikdy není od věci provést test, než data kamkoliv odešleme:

```
$ rails test
```

Šup s nimi na Bitbucket:

```
$ git push
```



### Co jsme se naučili

- Ruby má k dispozici spoustu metod pro operace s řetězci znaků.
- Vše v Ruby je objekt.
- Ruby podporuje definici metody skrze slovo "def"
- Ruby podporuje definici třídy skrze slovo "class"
- Railsové pohledy mohou obsahovat statické HTML nebo zapouzdřené/vnořené Ruby (ERb).
- Zabudované datové struktury Ruby zahrnují pole, rozsahy (intervaly) a haše (asociativní pole).
- Bloky v Ruby jsou pružný konstrukt který (krom jiného) umožňuje přirozené iterace počítatelných datových struktur.
- Symboly jsou označení, jako řetězce bez jakékoliv dodatečné struktury.
- Ruby podporuje dědičnost v rámci objektů.
- Je možné otevřít a upravit zabudované třídy Ruby.
- Slovo "deified" je palindrom.