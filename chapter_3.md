# Statické stránky

V této kapitole se začneme věnovat vývoji aplikace na profesionální úrovni, která poslouží jako příklad i pro zbytek výuky. Ačkoliv bude mít aplikace nakonec uživatele, mikroposty a plnohodnotné přihlašování a autentifikaci, začneme zdánlivě jednoduchým témetam - tvorbou statických stránek. Navzdory zjevné jednoduchosti je tvorba statických stránek praktické cvičení, bohaté na implikace a provázanost.

Ačkoliv je Rails zaměřený především na tvorbu dynamických webových stránek opřených o databáze, zvládá i šikovně vytváření statických stránek za použití čistých HTML souborů. Použití Rails i pro statické stránky navíc skýtá jednoznačnou výhodu: můžeme jednoduše přidat i malé množství dynamického obsahu. V této kapitole se naučíme jak. Zároveň se nachomýtneme k automatickému testování, které je praktické k ujištění, že je náš kód napsaný správně.

## Příprava aplikace

Podobně jako v předchozí kapitole začneme vytvořením nového projektu Rails, tentokráte pojmenovaného "sample_app":

```
$ cd ~/environment
$ rails _5.1.6_ new sample_app
$ cd sample_app/
```

Dalším krokem je opět úprava souboru "Gemfile" tak, aby obsahoval gemy které naše aplikace bude potřebovat. I když vypadá podobně, jako minule, rozdíl je v sekci "test", která obsahuje gemy pro podporu pokročilého testování.

```
source 'https://rubygems.org'

gem 'rails',        '5.1.6'
gem 'puma',         '3.9.1'
gem 'sass-rails',   '5.0.6'
gem 'uglifier',     '3.2.0'
gem 'coffee-rails', '4.2.2'
gem 'jquery-rails', '4.3.1'
gem 'turbolinks',   '5.0.1'
gem 'jbuilder',     '2.7.0'

group :development, :test do
  gem 'sqlite3', '1.3.13'
  gem 'byebug',  '9.0.6', platform: :mri
end

group :development do
  gem 'web-console',           '3.5.1'
  gem 'listen',                '3.1.5'
  gem 'spring',                '2.0.2'
  gem 'spring-watcher-listen', '2.0.1'
end

group :test do
  gem 'rails-controller-testing', '1.0.2'
  gem 'minitest',                 '5.10.3'
  gem 'minitest-reporters',       '1.1.14'
  gem 'guard',                    '2.13.0'
  gem 'guard-minitest',           '2.4.4'
end

group :production do
  gem 'pg', '0.18.4'
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```



Následně spustíme příkaz "bundle install" pro instalaci a zahrnutí gemů. Mimojiné jsme si nastavili i přeskočení podpory databáze PostgreSQL (gem "pg"), jelikož SQLite, kterou budeme používat, se přecejen daleko snáze lokálně instaluje a konfiguruje.

```
$ bundle install --without production
```



# GITOVINY + DEPLOY, OPET PORESIT BEHEM REVIZE A REALIZACE



## Statické stránky

Když máme veškerou potřebnou přípravu za sebou, můžeme začít vyvíjet naší aplikaci. Prvním krokem bude vytvoření dynamických stránek pomocí sady Railsových "akcí" a "pohledů", které budou obsahovat pouze statické HTML. Railsové akce jsou pohromadě uvnitř ovladačů, které obsahují sady akcí se společným smyslem. V minulé kapitole jsme do nich nahlédli a v budoucnu se k nim vrátíme i dopodrobna, jakmile prozkoumáme celou REST architekturu. Pohybovat se budeme především v adresářích "app/controllers" a "app/views".

Jak jsme si řekli dříve, pokud používáme Git, je vhodné používat jinou větev než hlavní ("master"). Pro vytvoření větve týkající se statických stránek tedy provedeme následující příkaz:

```
$ git checkout -b static-pages
```



### Generované statické stránky

Pro začátek si vygenerujem ovladač použitím stejného skriptu "generate", jako jsme použili v minulé kapitole pro tvorbu scaffoldingu. Jelikož budeme vytvářet ovladač k osbluze statických stránek, nazveme ho "Static Pages controller", podle principu pojmenovávání [CamelCase](https://en.wikipedia.org/wiki/CamelCase) tedy "StaticPages". V plánu máme také akce pro Home page, Help page a About page, které si pojmenujeme (tentokrát podle pravidel pojmenovávání akcí malými písmeny) "home", "help" a "about". Skript "generate" umožňuje použití seznamu akcí, takže je rovnou zadáme jako parametry příkazu. Přesněji řečeno jen akce pro stránky Home a Help, přičemž akci pro stránku About si záměrně necháme na později, abychom viděli, jak ji lze přidat. Výsledný příkaz vypadá takto:

```
$ rails generate controller StaticPages home help
      create  app/controllers/static_pages_controller.rb
       route  get 'static_pages/help'
       route  get 'static_pages/home'
      invoke  erb
      create    app/views/static_pages
      create    app/views/static_pages/home.html.erb
      create    app/views/static_pages/help.html.erb
      invoke  test_unit
      create    test/controllers/static_pages_controller_test.rb
      invoke  helper
      create    app/helpers/static_pages_helper.rb
      invoke    test_unit
      invoke  assets
      invoke    coffee
      create      app/assets/javascripts/static_pages.coffee
      invoke    scss
      create      app/assets/stylesheets/static_pages.scss
```

Pro úplnost je dobré zmínit, že pro příkaz "rails generate" existuje zkratka v podobě příkazu "rails g". Takových zkratek existuje několik, a i když zde pro lepší orientaci používáme vždy plnou verzi příkazu, v praxi pak většina vývojářů v Rails používá zkratky uvedené v tabulce.

| **Plný příkaz**    | **Zkratka** |
| ------------------ | ----------- |
| `$ rails server`   | `$ rails s` |
| `$ rails console`  | `$ rails c` |
| `$ rails generate` | `$ rails g` |
| `$ rails test`     | `$ rails t` |
| `$ bundle install` | `$ bundle`  |

Než pokročíme dál, nahrajeme nově přidané soubory do vzdáleného repozitáře Gitu:

```
$ git add -A
$ git commit -m "Pridan ovladac statickych stranek"
$ git push -u origin static-pages
```

Poslední vypsaný příkaz pošle větev "static-pages" do repozitáře na Bitbucketu. Další odesílací ("push") akce dodatečné parametry nepotřebují a postačí jednoduché:

```
$ git push
```

Všimněme si také, že pro pojmenovávání tříd se používa CamelCase, ale vytvořené soubory naopak podléhají vzoru tzv. [snake case](https://en.wikipedia.org/wiki/Snake_case). V našem případě bude výsledný název souboru tímpádem "static_pages_controller.rb". I když se nejedná o povinnost, jakousi "štábní kulturu" pojmenovávání je záhodno dodržovat už od začátku - jednoduše jde o zvyklost.

#### Návrat zpět

Kdybychom náhodou někdy udělali při generování kódu chybu, je dobré znát postup pro vrácení věcí zpět do původního stavu. Naštěstí má Rails k dispozici několik praktických postupů.

Jedna z běžných situací například bývá, když si rozmyslíme pojmenování ovladače a cheme se zbavit vygenerovaných souborů. Jelikož Rails generuje spolu s ovladačem poměrně značné množství dodatečných souborů, není řešení tak jednoduché, že by stačilo odstranit samotný soubor s ovladačem. V předchozí lekci jsme si mohli všimnout, že takové generování může editovat i soubor "routes.rb", který pak musíme upravit ručně, aby nám vyhovoval. Naštěstí máme k dispozici příkaz "rails destroy", za kterým následuje jméno vygenerovaného prvku. Obecně vzato, tyto dva příkazy zadány postupně se navzájem vyruší:

```
  $ rails generate controller StaticPages home help
  $ rails destroy  controller StaticPages home help
```

V budoucnu si i vygenerujeme model v podobném duchu:

```
  $ rails generate model User name:string email:string
```

A můžeme ho zrušit příkazem:

```
  $ rails destroy model User
```

Další postup související s modely zahrnuje rušení migrací, kterým jsme se okrajově věnovali minule a probrání do detailu nás teprve čeká. Migrace mění stav databáze za použití příkazu:

```
  $ rails db:migrate
```

A ten samý krok můžeme vrátit zpět použitím:

```
  $ rails db:rollback
```

Pokud bychom se chtěli vrátit úplně na začátek verzí databáze, stačí použít:

```
  $ rails db:migrate VERSION=0
```

Číslo u parametru verze pak určuje, kam se má stav databáze vrátit, přičemž číslování samotné stoupá s každým použitím migrace.



Ale vraťme se ke statickým stránkám. Vytvoření jejich ovladače automaticky upravilo směrovací soubor ("config/routes.rb"), který je zodpovědný za implementaci směrovače, který definuje souvislost URL a webových stránek. V adresáři "config" Rails obecně ukládá soubory, které jsou potřeba ke konfiguraci aplikace, viz obr. 3.1.

// obr. 3.1 Obsah konfigurační složky "config" naší aplikace.

Jelikož jsme si přidali akce "home" a "help", směrovací soubor (config/routes.rb) už pro ně obsahuje pravidla:

```
Rails.application.routes.draw do
  get  'static_pages/home'
  get  'static_pages/help'
  root 'application#hello'
end
```

Pravidlo

```
get 'static_pages/home'
```

mapuje požadavky pro URL ""/static_pages/home" k akci "home" v ovladači Static Pages. Použitím "get" nastavíme cestu tak, ať reaguje na "GET" požadavek, což je jedno ze základních "HTTP sloves" podporovaných hypertextovým protokolem. V našem případě to znamená, že když vygenerujeme "home" akci v ovladači Static Pages, automaticky se nám zpřístupní stránka na adrese "/static_pages/home". Výsledek si můžeme prohlédnout spuštěním Rails serveru:

```
$ rails server
```

A zadáním adresy "/static_pages/home", viz obr. 3.2.

// obr. 3.2 Pohled (view) složky home (/static_pages/home).



#### GET a ostatní

Hypertextový přenosový protokol ([HTTP](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol#Request_methods)) definuje základní operace "GET", "POST", "PATCH" a "DELETE". Tyto odpovídají jednotlivým operacím mezi klientem (obvykle webovým prohlížečem jako je Chrome, Firefox nebo Safari) a serverem (obvykle ve formě webserveru jako je Apache nebo Nginx). Je důležité vědět, že i když je při vývoji aplikací v Rails na místním počítači klient i server na stejném fyzickém stroji, jde o dvě rozdílné věci. Důraz na HTTP slovesa je běžný pro webové [frameworky](https://en.wikipedia.org/wiki/Software_framework) (včetně Rails), které jsou ovlivněny architekturou REST.

"GET" je nejběžnější HTTP operací. Používá se pro čtení dat na webu; v podstatě jde o "get a page", tedy "získej stránku", a kdykoliv navštívíme stránky jako <http://www.google.com/> nebo <http://www.wikipedia.org/>, náš prohlížeč odesílá "GET" požadavek. "POST" je druhá nejběžnější operace; jde o požadavek odeslaný prohlížečem když např. vyplňíme a odešleme formulář. V aplikacích Rails se požadavky "POST" obvykle používají pro tvorbu věcí (i když HTTP také umožňuje používat "POST" jako způsob updatování). Například, když někde vyplníme a odešleme registraci, na vzdálené stránce se vytvoří uživatel právě skrze "POST". Zbylá dvě slovesa, "PATCH" a "DELETE", jsou vytvořena pro aktualizování a likvidaci věcí na vzdáleném serveru. Tyto požadavky jsou poměrně neobvyklé, jelikož je prohlížeče neumí za normálních okolností posílat, ale některé webové frameworky (včetně Ruby on Rails) mají k dispozici chytré způsoby, díky kterým vypadá, že prohlížeče takové požadavky odesílají. Rails tímpádem podporuje všechny čtyři typy požadavků.



Abychom správně pochopili původ stránky, podívejme se na ovladač Static Pages (app/controllers/static_pages_controller.rb) v textovém editoru. Měl by vypadat nějak takto:

```
class StaticPagesController < ApplicationController
  def home
  end

  def help
  end
end
```

Narozdíl od předešlých ovladačů uživatelů a mikropostů ovladač Static Pages nepoužívá standardní REST akce. To je ovšem pro statické stránky normální: architektura REST není nejlepším řešením pro úplně každý problém.

Ze slova "class" ve výpisu vidíme, že "static_pages_controller.rb" definuje třídu (angl. "class"), v tomto případě nazvanou "StaticPagesController". Třídy jsou v podstatě jen praktický způsob, jak organizovat funkce (též občas nazývané "metody"), jako jsou akce "home" a "help", které jsou definovány skrze slovo "def". Jak jsme si už vysvětlovali dříve, zobáček vlevo ("<") indikuje, že "StaticPagesController" dědí po třídě "ApplicationController". Jak záhy uvidíme, naše stránky díky tomu mají k dispozici spoustu funkcionality specifické pro Rails.

V ovladači Static Pages jsou obě metody nejprve prázdné:

```
def home
end

def help
end
```

V obyčejném Ruby by tyto metody neudělaly nic. V Rails je situace jiná - "StaticPagesController" je třída Ruby, ale protože dědí po "ApplicationController", je chování jejich metod specifické pro Rails: když navštívíme URL "/static_pages/home", Rails najde ovladač Static Pages, spustí kód v akci "home" a vykreslí pohled (view) související s akcí. V současné chvíli je akce "home" prázdná, takže návštěva "/static_pages/home" způsobí pouze vykreslení pohledu. Jak tedy pohled vypadá a jak ho najdeme?

Akce "home" má i související pohled nazvaný "home.html.erb". Ohledně ".erb" části si povíme záhy; část ".html" asi nepřekvapí tím, že v podstatě vypadá jako HTML.

Soubor "app/views/static_pages/home.html.erb" vypadá takto:

```
<h1>StaticPages#home</h1>
<p>Find me in app/views/static_pages/home.html.erb</p>
```

Soubor "app/views/static_pages/help.html.erb", tedy obdoba pro akci "help", vypadá téměř totožně:

```
<h1>StaticPages#help</h1>
<p>Find me in app/views/static_pages/help.html.erb</p>
```

V obou případech jde pouze o tzv. placeholders (což je forma výplňového textu, který může, ale i nemusí mit určitou informační hodnotu - doslova "drží místo").



### Vlastní statické stránky

I když si přidáme (malé množství) dynamického obsahu hned v další části, z předchozích souborů jsme zjistili důležitou věc: pohledy Railsu můžou jednoduše obsahovat statické HTML. Můžeme si tedy upravit naše stránky Home a Help i bez dodatečných znalostí Rails.

Soubor "app/views/static_pages/home.html.erb":

```
<h1>Aplikace</h1>
<p>
  Toto je domovská stránka pro naši aplikaci.
</p>
```

Soubor "app/views/static_pages/help.html.erb":

```
<h1>Pomoc</h1>
<p>
  Zatím snad nebude potřeba, ale je dobré být připraven!
</p>
```

Zobrazení obou stránek pak bude vypadat následovně (obr. 3.3 a 3.4).

// obr. 3.3 Upravená stránka Home.

// obr. 3.4 Upravená stránka Help.



## Začínáme testovat

Když jsme si přidali a upravili stránky Home a Help, vytvoříme si i stránku About. Při takových úpravách je rozumné připravit automatizovaný test, který ověří, že jsme prvek přidali správně. Pokud budeme do testů v průběhu vývoje přidávat dodatečnou funkcionalitu, může fungovat jako záchranná síť i jako jistá dokumentace pro zdrojový kód. Při správném provedení může být testování ve skutečnosti pro vývoj urychlujícím prvkem, protože navzdory kódu navíc pak ušetříme spoustu času při hledání případných chyb. Vyžaduje to ovšem jistý cvik a správné pochopení problematiky, takže je rozumné s přípravou testování začít co nejdříve.

I když obecně panuje shoda, že testování je dobrá věc, rozdíly bývají v postupech. Nejživější debaty se týkají tzv. "test-driven development" (TDD, česky "vývoj na bázi testů"), což je technika, během které programátor první napíše testy, při kterých aplikace selže (a tedy si tím vymezí, co nesmí, nebo naopak musí nastat), a následně píše kód aplikace tak, aby testy úspěšně procházel. Náš přístup bude odlehčený a pragmatický, takže TDD použijeme tam, kde se nám to bude hodit.



Když se rozhodujeme kdy a jak testovat, je dobré vědět, proč to vlastně děláme. Automatické testování ma tři hlavní výhody:

1. Testy chrání proti regresi, tedy stavu, kdy prvek přestane z nějakého důvodu fungovat, jak má. 
2. Testy umožňují refaktorování kódu (tedy změnu jeho formy aniž bychom měnili jeho funkci) s daleko větší jistotou.
3. Testy fungují jako "klient" pro aplikační kód, a tímpádem pomáhají určovat jeho návrh a rozhraní s ostatními částmi systému.

Ačkoliv žádná z uvedených výhod vysvloveně nevyžaduje, aby byly testy připraveny dopředu, je mnoho situací, kdy je "test-driven development" (TDD) cenným nástrojem. Když se rozhodujeme kdy a jak testovat, často rozhoduje i to, jak snesitelná pro nás příprava testů je; spousta vývojářů obvykle shledává, že čím lepší jsou v psaní testů, tím více jsou nakloněni jejich přípravě. Také záleží na tom, jak náročný je test ve srovnání s aplikačním kódem, jak přesně jsou nám známy funkce aplikace, kterou budeme připravovat, a jak velká je šance, že se v nich něco pokazí.

Když vezmeme výše zmiňované v potaz, je praktické mít určitou sadu orientačních pravidel, podle kterých se rozhodneme, jestli se vydáme cestou testování předem (nebo vůbec testování obecně). Pár praktických doporučení:

- Pokud je test obzvláště krátký nebo jednoduchý v porovnání s aplikací kterou testuje, je dobré se přiklonit k testování předem.
- Pokud není požadované chování a funkcionalita aplikace dopředu jasně daná, je lepší první napsat aplikační kód a testy až následně.
- Jelikož je bezpečnost vždy nejvyšší prioritou, testy pro model bezpečnosti píšeme přednostně.
- Kdykoliv se vyskytne bug (chyba), je dobré první napsat test pro jeho reprodukci a ochranu proti regresím, a pak až aplikační kód na jeho opravu.
- Pokud je velká šance, že se kód bude v budoucnu měnit (jako detailní HTML struktura), testy pro něj obvykle nepřipravujeme. 
- Testy píšeme před refaktorováním kódu, se zaměřením na testování částí, které mají velkou šanci, že se pokazí.

V praxi výše zmíněné poučky obvykle znamenají, že první napíšme testy pro ovladače a modely a integrační testy (které ověřují funkcionalitu skrze modely, pohledy i ovladače) až posléze. A pokud píšeme aplikační kód, který není obzvlášť křehký a náchylný k chybám, nebo je velká šance, že se změní (což je častý případ právě pohledů), často vynecháváme testování úplně.

Naše hlavní testovací nástroje budou testy ovladačů (se kterými začneme hned), a později i testy modelů a integrační testy. Integrační testy jsou obzvláště užitečné, jelikož nám umožňují simulovat akce uživatele který interaguje s naší aplikací skrze webový prohlížeč. Integrační testy budou nakonec naší primární testovací technikou, ale testy ovladačů jsou pro začátek jednodušší volba.

### Náš první test

Přidáme si tedy do naší aplikace stránku About. Jak uvidíme, test je krátký a jednoduchý, takže podle pouček zmíněných dříve si napíšeme test dopředu. Následně tomu pak přizpůsobíme psaní aplikačního kódu.

Učení se testování býva hlavně ze začátku výzva, která vyžaduje rozsáhlé znalosti jak Rails, tak Ruby. Nenechme se ale zastrašit - Rails už udělal tu nejtěžší práci za nás, jelikož "rails generate controller" automaticky vygeneroval i testovací soubor, díky kterému se máme od čeho odrazit:

```
$ ls test/controllers/
static_pages_controller_test.rb
```

Podívejme se tedy na jeho obsah ("test/controllers/static_pages_controller_test.rb"):

```
require 'test_helper'

class StaticPagesControllerTest < ActionDispatch::IntegrationTest

  test "should get home" do
    get static_pages_home_url
    assert_response :success
  end

  test "should get help" do
    get static_pages_help_url
    assert_response :success
  end
end
```

V současné chvíli není důležité chápat syntaxi do detailu, ale vidíme, že existují dva testy, každý pro jednu akci ovladače které jsme si prve vygenerovali. Každý z testů jednoduše vezme URL a ověří (skrze [aserci](https://en.wikipedia.org/wiki/Assertion_(software_development) )) že výsledek byl úspěšný. Použití "get" zase naznačuje předpoklad, že stránky Home a Help jsou obyčejné webové stránky, zpřístupněné právě použitím "GET" požadavku. Odpověď ":success" je abstraktní reprezentací HTTP [status code](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes) (v tomto případě kódu [200 OK](http://en.wikipedia.org/wiki/List_of_HTTP_status_codes#2xx_Success)). Jinak řečeno, test jako

```
test "should get home" do
  get static_pages_home_url
  assert_response :success
end
```

říká "Otestujeme stránku Home odesláním požadavku "GET" na URL "home" od Static Pages a pak si ověříme, že se nám vrátil stavový kód značící úspěch."

Pro zahájení našeho testovacího cyklu potřebujeme spustit testovou sadu pro ověření, že testy momentálně dopadají úspěšně. To lze provést příkazem "rails":

```
$ rails test
2 tests, 2 assertions, 0 failures, 0 errors, 0 skips
```

Výsledek je GREEN, tedy zelená - úspěch - a i když neuvidíme nutně i korespondující barvu, takováto terminologie je zcela běžná.

### Červená (RED)

Jak jsme si zmínili dříve, technika testování dopředu (TDD) zahrnuje nejprve napsání testu pro selhání, a pak až aplikační kód tak, aby prošel. V případě potřeby pak následuje i refaktorování kódu. Jelikož mnoho testovacích nástrojů vyjadřuje selhání testu červenou barvou a úspěch zelenou, takové sekvenci se pak občas říká "Red, Green, Refactor" cyklus. V této části dokončíme první krok tohoto cyklu - napsáním testu pro selhání se dostaneme k červené. Potom postupně přejdeme k zelené a i následnému refaktorování.

Prvním krokem bude napsat selhávací test pro stránku About. Soudě z výpisu testovacího souboru (test/controllers/static_pages_controller_test.rb) z předchozí části se dá uhádnout, jak by měl takový test vypadat:

```
require 'test_helper'

class StaticPagesControllerTest < ActionDispatch::IntegrationTest

  test "should get home" do
    get static_pages_home_url
    assert_response :success
  end

  test "should get help" do
    get static_pages_help_url
    assert_response :success
  end

  test "should get about" do
    get static_pages_about_url
    assert_response :success
  end
end
```

Jednoduše jsme si přidali i variantu ve stejném formátu i pro stránku About. A jak jsme i ostatně chtěli, test nyní selže.

```
$ rails test
3 tests, 2 assertions, 0 failures, 1 errors, 0 skips
```

### Zelená (GREEN)

Když teď máme k dispozici test selhání (red), využijeme jeho chybových zpráv jako vodítek pro úspěšné projití testu (green), a tedy implementováním funkční stránky About. Začneme prozkoumáním chybové hlášky, kterou nám test dodal:

```
$ rails test
NameError: undefined local variable or method `static_pages_about_url'
```

Chybová hláška říká, že kód Rails pro URL stránky About je nedefinovaný, což je pro nás náznak, abychom přidali potřebný řádek do směrovacího souboru (config/routes.rb). Použijeme stejný princip, jako u testovacího souboru ovladačů:

```
Rails.application.routes.draw do
  get  'static_pages/home'
  get  'static_pages/help'
  get  'static_pages/about'
  root 'application#hello'
end
```

Jednoduše přidáme další řádek, který řekne Railsu at směruje "GET" požadavek pro URL "/static_pages/about" k akci "about" v ovladači Static Pages, což i automaticky vytvoří pomocníka nazvaného "static_pages_about_url".

Když si spustíme test znovu tak zjistíme, že je sice stále červený, ale chybová hláška je jiná:

```
$ rails test
AbstractController::ActionNotFound:
The action 'about' could not be found for StaticPagesController
```

Problém je nyní v chybějící akci "about" v ovladači Static Pages (app/controllers/static_pages_controller.rb), kterou přidáme opět po vzoru "home" a "help" akcí.

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

Výsledek testu je opět červený, ale zároveň se i zase změnila chybová hláška:

```
$ rails test
ActionController::UnknownFormat: StaticPagesController#about is missing
a template for this request format and variant.
```

Tentokrát Railsu chybí šablona, což je v jeho kontextu v podstatě to samé, jako pohled. Jak jsme si uvedli dřívě, akce "home" je asociována s pohledem "home.html.erb" v adresáři "app/views/static_pages". Vytvoříme si tam tedy i nový soubor nazvaný "about.html.erb".

Způsobů, jakým vytvořit soubor je vícero. Grafické rozhraní obvykle umožňuje jednoduché řešení skrze pravé tlačítko myši, ale můžeme se naučit i další praktický trik unixových systémů: příkaz "touch".

```
$ touch app/views/static_pages/about.html.erb
```

I když je jeho původní určení aktualizace souboru bez změny jeho obsahu, jako vedlejší efekt i soubor vytvoří, pokud takový neexistuje. Soubor si tedy zase naplníme v editoru:

```
<h1>O stránce</h1>
<p>
  Demonstrativní text, ve kterém si můžeme časem uvést, k čemu aplikace slouží.
</p>
```



Teď už nám konečně test ohlásí zelenou:

```
$ rails test
3 tests, 3 assertions, 0 failures, 0 errors, 0 skips
```

Samozřejmě je dobré se podívat na výsledek i skrze prohlížeč, abychom se ujistili, že nám testy nevyvádí šílenosti (obr. 3.5).

// obr 3.5  Nová stránka About (/static_pages/about). 

### Refaktorování

Když jsme se dostali k zelené, můžeme kód v poklidu refaktorovat. Při vývoji aplikace se často stává, že kód začne "smrdět", tedy že začne být nevzhledny, nafouklý, nebo plný opakování. Počítači je jedno, jak kód vypadá, ale lidem tak úplně ne, takže je důležité udržovat kód čistý právě častým refaktorováním. I když je naše aplikace pořád příliš malá, než abychom ji teď refaktorovali, dostaneme se k tomu velmi brzy, protože pachy v kódu číhají v každém zákoutí.



## Lehce dynamické stránky

Když už máme vytvořeny akce a pohledy pro pár statických stránek, změníme je na trochu dynamické přidáním obsahu, který se mění stránku od stránky: název každé stránky bude odrážet její obsah. Jestli jde o dynamičnost v pravém slova smyslu je sice sporné, ale tak jako tak je to potřebný základ pro čistě dynamický obsah, ke kterému se dostaneme později.

Chceme tedy upravit stránky Home, Help a About tak, ať se jejich název mění s každou stránkou. To bude zahrnovat použití tagu "<title>" (název, titul) v pohledech našich stránek. Většina prohlížečů zobrazuje obsahy tagu title v jejich horní části, a nemalou důležitost mají i z hlediska optimalizace vyhledávání. Použijeme úplný "Red, Green, Refactor" cyklus: první přidáme jednoduché testy na title stránek (red), následně přidáme title jako takové (green) a nakonec použijeme soubor pro rozložení, abychom eliminovali duplicitu (refactor). Ve finále pak budeme mít u všech tří stránek title v podobě "<název stránky> | Ruby on Rails Tutorial", kde první část bude proměnlivá, jak je zobrazeno v následující tabulce:

| **Stránka** | **URL**             | **Základní název**         | **Proměnná** |
| ----------- | ------------------- | -------------------------- | ------------ |
| Home        | /static_pages/home  | `"Ruby on Rails Tutorial"` | `"Home"`     |
| Help        | /static_pages/help  | `"Ruby on Rails Tutorial"` | `"Help"`     |
| About       | /static_pages/about | `"Ruby on Rails Tutorial"` | `"About"`    |

Příkaz "rails new" vytvoří soubor pro rozložení automaticky, ale z hlediska výuky je dobré ho zprvu ignorovat, čehož docílíme změnou názvu:

```
$ mv app/views/layouts/application.html.erb layout_file   # docasna zmena
```

V reálné aplikaci se nic takového nedělá, ale pro pochopení významu souboru nám pomůže jeho dočasné odstavení.

### Testujeme názvy (červená)

Pro přidání názvů (titles) stránek se potřebujeme alespoň trochu seznámit se strukturou běžné webové stránky, která vypadá zhruba následovně:

```
<!DOCTYPE html>
<html>
  <head>
    <title>Pozdrav</title>
  </head>
  <body>
    <p>Ahoj světe!</p>
  </body>
</html>
```

Struktura výše obnáši tzv. "document type", nebo zkráceně jen "doctype", což je deklarace, díky které rozeznají prohlížeče jakou verzi HTML stránka používá (v tom to případě [HTML5](http://en.wikipedia.org/wiki/HTML5)); sekci "head", s textem "Pozdrav" uvnitř tagu "<title>" a sekce "body", ve které je text "Ahoj světe!" uvnitř tagu "<p>", který značí odstavec (paragraf). Odsazení je volitelné, jelikož HTML není citlivé na mezery, ale pro lepší orientaci v kódu je prakticky nutné ho používat.

Pro každý z názvů (titles) si vytvoříme jednoduchý test zkombinováním testů z předchozí části spolu s metodou "assert_select", která nám umožňuje ověřovat přítomnost specifických HTML tagů.

```
assert_select "title", "Home | Ruby on Rails Tutorial"
```

Uvedený kód ověřuje přítomnost tagu "<title>", který obsahuje řetězec "Home | Ruby on Rails Tutorial". Obdobu vytvoříme i pro zbylé stránky a soubor "test/controllers/static_pages_controller_test.rb" si tedy upravíme takto:

```
require 'test_helper'

class StaticPagesControllerTest < ActionDispatch::IntegrationTest

  test "should get home" do
    get static_pages_home_url
    assert_response :success
    assert_select "title", "Home | Ruby on Rails Tutorial"
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

Teď už jen zbývá test spustit, abychom si ověřili, že výsledek je červený.

```
$ rails test
3 tests, 6 assertions, 3 failures, 0 errors, 0 skips
```



### Přidání názvů stránek (zelená)

Aby test prošel úspěšně, přidáme každé stránce název (title). Použitím principů základní HTML struktury na naší Home page pak dostaneme tento výsledek:

```
<!DOCTYPE html>
<html>
  <head>
    <title>Home | Ruby on Rails</title>
  </head>
  <body>
    <h1>Aplikace</h1>
    <p>
      Toto je domovská stránka pro naši aplikaci.
    </p>
  </body>
</html>
```

Stránka bude v prohlížeči vypadat takto:

// obr. 3.6 Stránka Home (domovská) s názvem.

Aplikujeme tedy obdobný postup i na stránky Help a About:

```
<!DOCTYPE html>
<html>
  <head>
    <title>Help | Ruby on Rails Tutorial</title>
  </head>
  <body>
    <h1>Pomoc</h1>
    <p>
      Zatím snad nebude potřeba, ale je dobré být připraven!
    </p>
  </body>
</html>
```

```
<!DOCTYPE html>
<html>
  <head>
    <title>About | Ruby on Rails Tutorial</title>
  </head>
  <body>
    <h1>O stránce</h1>
    <p>
      Demonstrativní text, ve kterém si můžeme časem uvést, k čemu aplikace slouží.
    </p>
  </body>
</html>
```

Díky předchozím úpravám by testy měly proběhnout na zelenou:

```
$ rails test
3 tests, 6 assertions, 0 failures, 0 errors, 0 skips
```



### Rozložení a zapouzdřené Ruby

V této sekci jsme toho zvládli hodně, vytvořili jsme tři validní stránky za použití ovladačů a akcí Rails, ale pořád jde o statické HTML a tímpádem nevyužívá potenciálu Rails. Co je horší, trpí hroznou duplikací:

- Názvy stránek jsou téměř (i když ne úplně) stejné.
- "Ruby on Rails Tutorial" je shodný pro všechny tři názvy.
- Celá kostra struktury HTML se opakuje na všech stránkách.

Tento duplicitní kód je pak přímé porušení zásady "DRY" ("Don't Repeat Yourself", česky "neopakuj se"). Kód si tedy patřičně prosekáme odstraněním repeticí. Na konci si pak znovu spustíme testy, abychom si ověřili, že jsou názvy stránek stále v pořádku.

Paradoxně, první krok k eliminování duplikace bude přidání další: upravíme názvy stránek, které jsou si momentálně jen podobné, tak, aby byly stejné. Zdá se to zvláštní, ale zlikvidování duplikace nám to zjednoduší.

Postup bude zahrnovat použití zapouzdřeného Ruby ("embedded Ruby") v našich pohledech. Jelikož názvy stránek Home, Help a About mají proměnný prvek, použijeme speciální funkci Rails, zvanou "provide", k nastavení různých názvů na každou stránky. Jak to funguje můžeme vidět když nahradíme doslovný název "Home" v pohledu "home.html.erb" (soubor "app/views/static_pages/home.html.erb") následujícím kódem:

```
<% provide(:title, "Home") %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
  </head>
  <body>
    <h1>Aplikace</h1>
    <p>
      Toto je domovská stránka pro naši aplikaci.
    </p>
  </body>
</html>
```

Je to vlastně takový náš první příklad použití zapouzdřeného Ruby, také zkracovaného na ERb. (Tím jsme se i dozvěděli, proč mají HTML pohledy příponu ".html.erb") ERb je primární šablonovací systém pro zahrnování dynamického obsahu na webových stránkách.

```
<% provide(:title, "Home") %>
```

Tento kód, tedy konkrétně <% ... %> říká Rails ať zavolá funkci "provide" a asociuje řetězec "Home" se štítkem ":title". Poté, v samotném názvu, použijeme související značení <%= ... %> pro vložení názvu do šablony použitím Ruby funkce "yield":

```
<title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
```

Rozdíl mezi značením je ten, že <% ... %> jen provede kód uvnitř, kdežto <%= ... %> ho provede a výsledek vloží do šablony. Výsledná stránka je stejná, jako byla dříve, ale část s proměnnou je nyní dynamicky generovaná díky ERb.

Což si i můžeme ověřit spuštěním testu, který bude i nadále zelený:

```
$ rails test
3 tests, 6 assertions, 0 failures, 0 errors, 0 skips
```

Obdobné změny tedy provedeme i na stránkách Help a About.

```
<% provide(:title, "Help") %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
  </head>
  <body>
    <h1>Pomoc</h1>
    <p>
      Zatím snad nebude potřeba, ale je dobré být připraven!
    </p>
  </body>
</html>
```

```
<% provide(:title, "About") %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
  </head>
  <body>
    <h1>O stránce</h1>
    <p>
      Demonstrativní text, ve kterém si můžeme časem uvést, k čemu aplikace slouží.
    </p>
  </body>
</html>
```

Když už teď máme nahrazené proměnné části názvů stránek metodou skrze ERb, všechny naše stránky vypadají v podstatě takto:

```
<% provide(:title, "The Title") %>
<!DOCTYPE html>
<html>
  <head>
    <title><%= yield(:title) %> | Ruby on Rails Tutorial</title>
  </head>
  <body>
    Obsah
  </body>
</html>
```

Jinak řečeno, všechny stránky mají identickou strukturu, včetně obsahu tagu "<title>", s jedinou výjimkou v podobě obsahu tagu "<body>".

Abychom této podobné struktury využili, použijeme právě speciální soubor pro rozvržení (layout) zvaný "application.html.erb", který jsme si přejmenovali dříve, a nyní si ho obnovíme:

```
$ mv layout_file app/views/layouts/application.html.erb
```

Aby layout fungoval, nahradíme původní název (title) zapouzdřeným Ruby z příkladů výše:

```
<title><%= yield(:title) %> | Ruby on Rails Tutorial</title> 
```

Výsledný soubor pro layout (app/views/layouts/application.html.erb) pak bude vypadat takto:

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

Všimněme si řádku:

```
<%= yield %>
```

Je zodpovědný za vkládání obsahu každé stránky do šablony. Není momentálně důležité vědět, jak přesně to funguje; jde především o to, že použití tohoto rozvržení zajišťujě, že kupříkladu návštěva stránky /static_pages/home zkonvertuje obsah "home.html.erb" do HTML a pak ho vloží namísto "<%= yield %>".

Také je dobré vědět, že základní rozložení Rails obsahuje některé dodatečné řádky, jako:

```
<%= csrf_meta_tags %>
<%= stylesheet_link_tag ... %>
<%= javascript_include_tag "application", ... %>
```

Tento kód obstarává zahrnutí stylů a JavaScriptu aplikace, což jsou prvky, které spadají pod "assets", tedy zdroje, společně s metodou "csrf_meta_tags", která brání [CSRF](http://en.wikipedia.org/wiki/Cross-site_request_forgery), což je druh útoku na stránku.

V původních souborech pohledů máme samozřejmě použito příliš mnoho kódu, který nám nyní vloží šablona automaticky, takže všechny tři soubory trochu prostříháme, aby nám zůstal jen obsah:

```
<% provide(:title, "Home") %>
<h1>Aplikace</h1>
<p>
  Toto je domovská stránka pro naši aplikaci.
</p>
```

```
<% provide(:title, "Help") %>
<h1>Pomoc</h1>
<p>
  Zatím snad nebude potřeba, ale je dobré být připraven!
</p>
```

```
<% provide(:title, "About") %>
<h1>O stránce</h1>
<p>
  Demonstrativní text, ve kterém si můžeme časem uvést, k čemu aplikace slouží.
</p>
```

Jak je vidno, stránky Home, Help a About jsou stejné, jako byly, ale obsahují mnohém méně nadbytečného duplicitního kódu.

V praxi se bohužel často stává, že i jednoduché refaktorování je náchylné ke vzniku chyb. Proto je i jedním z důvodů, proč je důležité mít k dispozici dobrou sadu testů. Je přecejen lepší nechat proběhnout test, než podrobně procházet jednu stránku po druhé a hledat chyby. Ostatně, ze začátku vývoje aplikace je to ještě zvladatelné, ale v pozdějších fázích pak neskutečně frustrující a především neefektivní.

```
$ rails test
3 tests, 6 assertions, 0 failures, 0 errors, 0 skips
```

I když výsledek testu není důkaz v pravém slova smyslu, že je náš kód správně, tak výrazně zvyšuje pravděpodobnost, že tomu tak je, a v případě výskytů bugů v budoucnu se pak je o co opřít.

### Nastavení kořenové (root) cesty

Když jsme si upravili dle vlastní chuti stránky naší aplikace a zároveň položili základy testování, upravíme si před dalšími kroky kořenovou cestu. Jak jsme se už naučili dříve, obnáší to úpravu souboru "config/routes.rb" a změnou / na stránku dle vlastního výběru, což v našem případě bude stránka Home. (Stejně tak si odstraníme akci "hello", kterou už potřebovat nebudeme.)

Změníme si tedy cestu "root" z:

```
root 'application#hello'
```

na

```
root 'static_pages#home'
```

A požadavky na kořenovou adresu budou přesmerovány na akci "home" v ovladači Static Pages. Soubor "routes.rb" tedy bude vypadat takto:

```
Rails.application.routes.draw do
  root 'static_pages#home'
  get  'static_pages/home'
  get  'static_pages/help'
  get  'static_pages/about'
end
```

// obr. 3.7: Domovská (home) stránka po přesměrování na kořenovou cestu.



## Shrnutí

Při pohledu zvenčí by se zdálo, že jsme v této kapitole sotva něčeho dosáhli: začali jsme u statických stránek a skončili s... téměř statickými stránkami. Ale zdání klame: vývojem v duchu Railsových ovladačů, akcí, modelů a pohledů jsme se posunuli tak, že můžeme přidávat do našich stránek libovolný dynamický obsah. Jak přesně, to se dozvíme postupně.

Zároveň je teď dobrá chvíle pro odeslání změn do naší vedlejší větve a její spojení s hlavní větví. V prvé řadě tedy spustíme commit:

```
$ git add -A
$ git commit -m "Dokonceni statickych stranek"
```

Pak stačí spojit větve příkazem, který jsme se naučili dříve:

```
$ git checkout master
$ git merge static-pages
```

Je i vhodná chvíle na to, abychom odeslali současný stav projektu do vzdáleného repozitáře:

```
$ rails test
$ git push
```

Obecně také platí, že před odesláním dat je záhodno provést testování, pokud je k dispozici. Je to praktický zvyk, který se časem rozhodně vyplatí.

### Co jsme se naučili

- Už potřetí jsme si prošli celou procedurou vytváření aplikace od píky, včetně instalace potřebných gemů a odeslání ven. 
- Skript "rails" generuje nový ovladač přes parametry "rails generate controller ControllerName <volitelne nazvy akci>".
- Nová směrování jsou definována v souboru "config/routes.rb".
- Pohledy Railsu mohou obsahovat statické HTML nebo zapouzdrřené Ruby (ERb).
- Automatizované testování nám umožňuje používat sady testů, na kterých se dá stavět vývoj nových prvku, a umožňují spolehlivé refaktorování a odchytávání chyb.
- Test-driven development (TTD, "testování dopředu/předem") používá cyklus "Red, Green, Refactor" (červená, zelená, refaktorování). 
- Možnosti rozvržení v Rails umožňují použití šablon pro stránky naší aplikace, a tím eliminují duplicitu.