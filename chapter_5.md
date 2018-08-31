# Naplňujeme layout

Během naši exkurze Ruby jsme se naučili, jak zapojit styly aplikační styly do aplikace samotné, ale zatím v nich nemáme žádné reálne CSS. V této části začneme naplňovat naše vlastní stylování pomocí zahrnutí CSS frameworku, a ještě navrch přidáme pár vlastních stylů. Také do layoutu (rozvržení) zahrneme odkazy na stránky (jako Home a About), které jsme si dříve vytvořili. Během toho se něco málo dozvíme o tzv. partials, cestách v Rails, zřetězeném zpracovávání zdrojů (asset pipelining) a čeká nás i úvod do Sass. Zakončíme pak naše snažení důležitým krokem směrem k umožnění uživatelům se na naši stránku přihlásit.

## Přidání struktury

I když jsme primárně tématicky zaměření na vývoj webu, ne jeho design, bylo by přecejen poněkud depresivní pracovat na aplikaci, která vypadá naprosto otřesně, takže si v této části přidáme do layoutu trochu struktury a i minimální stylování pomocí CSS. Krom několika vlastních řádků ale budeme hlavně spoléhat na [Bootstrap](http://getbootstrap.com/), což je běžně používaný framework pro webdesign od Twitteru.

Při vývoji webových aplikací je dobré si rozvrhnout uživatelské rozhraní co možná nejdříve. Pro představu tedy budeme používat i tzv. mockups, což jsou hrubé návrhy toho, jak bude nakonec aplikace vypadat. Zatím se budeme věnovat především vývoji statických stránek z předchozích částí včetně loga stránky, navigační hlavičky a patičky stránky. Mockup (náhled/návrh) nejdůležitější stránky, Homepage, je na následujícím obrázku (obr. 5.1).

// obr. 5.1 Mockup domovské stránky naší aplikace

Jako obvykle je teď dobrá chvíle na vytvoření nové větve v Gitu:

```
$ git checkout -b filling-in-layout
```

### Navigace stránky

Jako první krok k přidání odkazů a stylů do aplikace aktualizujeme soubor s layoutem "application.html.erb" dodatečnou HTML strukturou, která zahrnuje pár dodatečných oddílů, CSS tříd a začátek navigace naší stránky. Soubor si tedy upravíme následovně, a hned si i jednotlivé prvky vysvětlíme:

```
<!DOCTYPE html>
<html>
  <head>
    <title><%= full_title(yield(:title)) %></title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all',
                                              'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
    <!--[if lt IE 9]>
      <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/r29/html5.min.js">
      </script>
    <![endif]-->
  </head>
  <body>
    <header class="navbar navbar-fixed-top navbar-inverse">
      <div class="container">
        <%= link_to "sample app", '#', id: "logo" %>
        <nav>
          <ul class="nav navbar-nav navbar-right">
            <li><%= link_to "Home",   '#' %></li>
            <li><%= link_to "Help",   '#' %></li>
            <li><%= link_to "Log in", '#' %></li>
          </ul>
        </nav>
      </div>
    </header>
    <div class="container">
      <%= yield %>
    </div>
  </body>
</html>
```

Jak jsme krátce zmiňovali dříve, Rails používá nativně HTML5; jelikož HTML5 standard je relativně nový, některé prohlížeče (především starší verze Internet Exploreru) ho plně nepodporují, a proto přidáme kus JavaScriptového kódu, který tento problém obchází:

```
<!--[if lt IE 9]>
  <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/r29/html5.min.js">
  </script>
<![endif]-->
```

Poněkud podivný zápis

```
<!--[if lt IE 9]>
```

zahrnuje následný řádek pouze tehdy, když je verze IE nižší, než 9. Podivná část [if lt IE 9] není součástí Rails; ve skutečnosti jde o podmínečný komentár, které podporují prohlížeče IE právě pro tyto případy. Je to praktické především proto, že vsuvku vnímá jen IE s verzí nižší, než 9, a prohlížeče jako Firefox, Chrome a Safari si ho vůbec nevšímají.

Další sekce zahrnuje "header" (hlavičku) pro logo stránky, několik oddílů (tag <div>, z angl. "division" ) a seznam prvků s odkazy navigace:

```
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", '#', id: "logo" %>
    <nav>
      <ul class="nav navbar-nav navbar-right">
        <li><%= link_to "Home",   '#' %></li>
        <li><%= link_to "Help",   '#' %></li>
        <li><%= link_to "Log in", '#' %></li>
      </ul>
    </nav>
  </div>
</header>
```

Tag "header" označuje prvky, které by měly přijít na vrch stránky. Samotnému tagu jsme dali tři CSS třídy, zvané "navbar", "navbar-fixed-top" a "navbar-inverse", oddělené mezerami:

```
<header class="navbar navbar-fixed-top navbar-inverse">
```

Všechny HTML prvky mohou mít přiřazeny jak třídy, tak tzv. "id"; jsou to pouhá označení, praktická pro stylování skrze CSS. Základní rozdíl mezi třídami a id je ten, že třídy lze na stránce použít libovolněkrát, kdežto id musí být unikátní a tímpádem použité jen jednou. V našem případě mají všechny navbar třídy speciální význam pro Bootstrap, který si záhy nainstalujeme a zapojíme.

Uvnitř tagu "header" máme tag "div":

```
<div class="container">
```

Tag "div" je obecný oddíl (division); nedělá nic krom toho, že rozděluje dokument do jednoznačných částí. Ve starším HTML se "div" tagy používaly pro téměř všchny oddíly stránky, ale HTML5 přidalo tagy "header", "nav" a "section" pro oddíly, které má většina stránek tak jako tak. Opět, v našem případě má tag "div" svou třídu ("container"). Stejně jako u tagu "header" je určená pro Bootstrap.

Po tagu "div" narazíme na trochu vnořeného Ruby:

```
<%= link_to "sample app", '#', id: "logo" %>
<nav>
  <ul class="nav navbar-nav navbar-right">
    <li><%= link_to "Home",   '#' %></li>
    <li><%= link_to "Help",   '#' %></li>
    <li><%= link_to "Log in", '#' %></li>
  </ul>
</nav>
```

Pomocí pomocníka Rails "link_to" jsou vytvořeny odkazy (které jsme vytvořili přímo kotvou "a" dříve); první parametr pro "link_to" je text odkazu, přičemž druhý je URL samotná. Později naplníme URL jmennou cestou, ale zatím použijeme "pahýl" ve formě "#" (jde o běžný postup ve webdesignu, kdy mřížka slouží jako dočasný zabírač místa pro reálnou URL). Třetí parametr je haše s možnostmi, v tomto případě přidá id "logo" k odkazu na aplikaci. (Ostatní tři odkazy nemají haš s možnostmi, což nevadí, jelikož je nepovinná.) Pomocníci Rails často přijímají tímto způsobem dodatečné možnosti, čímž nám dávají možnost přidat dodatečné HTML aniž bychom opouštěli Rails.

Druhý prvek v divech je seznam navigačních odkazů, vytvořený za použití tagu pro neřazený seznam (unordered list) "ul", spolu s prvky seznamu (list item) "li":

```
<nav>
  <ul class="nav navbar-nav navbar-right">
    <li><%= link_to "Home",   '#' %></li>
    <li><%= link_to "Help",   '#' %></li>
    <li><%= link_to "Log in", '#' %></li>
  </ul>
</nav>
```

Tag "nav", ač v tomto případě formálně vzato nepovinný, jsme použili z důvodu lepšího vyjádření významu navigačních odkazů. Třídy "nav", "navbar-nav" a "navbar-right" na tagu "ul" mají opět zvláštní význam pro Bootstrap a jakmile ho nainstalujeme, budou nastylovány automaticky. Jak si pak můžeme ověřit v prohlížeči, Rails layout zpracoval a vyhodnotil zapouzdřené Ruby, takže výsledek vypadá takto:

```
<nav>
  <ul class="nav navbar-nav navbar-right">
    <li><a href="#">Home</a></li>
    <li><a href="#">Help</a></li>
    <li><a href="#">Log in</a></li>
  </ul>
</nav>
```

Poslední část layoutu je div s hlavním obsahem:

```
<div class="container">
  <%= yield %>
</div>
```

Třída "container" opět slouží Bootstrapu. Jak jsme se naučili dříve, metoda "yield" vkládá obsah každé stránky do layoutu.

Krom patičky, kterou přidáme posléze, je náš layout hotový a můžeme se podívat na výsledek navštívením domovské stránky. Přidáme si ještě pár dodatečných prvků do souboru "home.html.erb", abychom stylování trochu více využili:

```
<div class="center jumbotron">
  <h1>Vítejte v aplikaci</h1>

  <h2>
    Toto je domovská stránka naší tréningové aplikace.
  </h2>

  <%= link_to "Zaregistrovat se", '#', class: "btn btn-lg btn-primary" %>
</div>

<%= link_to image_tag("rails.png", alt: "Rails logo"),
            'http://rubyonrails.org/' %>
```

V rámci přípravy pozdějšího přidávání uživatelů vytvoří první "link_to" pahýlový odkaz k formuláři:

```
<a href="#" class="btn btn-lg btn-primary">Zaregistrovat se</a>
```

V tagu "div" je třída "jumbotron" opět určená Bootstrapu, stějne jako "btn", "btn-lg" a "btn-primary" třídy v tlačítku.

Druhý "link_to" představuje pomocníka "image_tag", který vezme parametry pro cestu k obrázku a volitelnou haš možností, v tomto případě nastavujíc atribut "alt" image tagu. Aby všechno fungovalo, musí existovat obrázek nazvaný "rails.png", který by měl jít stáhnout z [této adresy](http://railstutorial.org/rails.png) a stačí jej umístit do složky "app/assets/images/". Jelikož pracujeme v unixovém systému, můžeme využít možnosti příkazu "curl":

```
$ curl -o app/assets/images/rails.png -OL railstutorial.org/rails.png
```

Jelikož jsme použili pomocníka "image_tag", Rails automaticky projde "app/assets/images/" a vyhledá kýžený obrázek.

Konečně můžeme vidět plody naší práce. Bude možná zapotřebí restartovat server Rails, aby se změny projevily, ale výsledky by měly vypadat jako na obrázku (obr. 5.2):

// obr. 5.2 Domovská stránka bez aktivního vlastního CSS.

Abychom lépe pochopili efekt "image_tag", podívejme se na HTML které produkuje prozkoumáním obrázku v prohlížeči:

```
<img alt="Rails logo" src="/assets/rails-9308b8f92fea4c19a3a0d8385b494526.png" />
```

Řetězec "9308b8f92fea4c19a3a0d8385b494526", který bude na každém systému vypadat jinak, je přidán Railsem pro ujištění unikátního názvu souboru, což má za následek správné načtení obrázku prohlížečem, když dojde k jeho změně. Atribut "src" neobsahuje část cesty "images", namísto toho používá složku "assets", která je všem zdrojům sdílená (obrázky, JavaScript, CSS apod.). Na serveru totiž Rails asociuje obrázky ze složky "assets" se správnou složkou "app/assets/images", ale z pohledu prohlížeče jde stále o jednu a tutéž složku, díky čemuž je zvládá nahrát rychleji. Obsah atributu "alt" bude zobrazen tehdy, když je stránka navštívena programem, který neumí zobrazit obrázky.

I když vypadá výsledek zatím poněkud fádně, vzhledem ke vší té práci, položili jsme si ve skutečnosti dobrý základ v podobě HTML prvků s rozumnými třídami, díky čemuž jsme ve skvělé pozici pro přidání CSS stylů.

### Bootstrap a vlastní CSS

Díky předchozí části se tedy můžeme vrhnout na tvoření layoutu založeném na CSS. Jak jsme si už řekli, spousta tříd je specifických pro [Bootstrap](http://getbootstrap.com/), framework od Twitteru který umožňuje jednoduché formátovaní uživatelského prostředí a jeho prvků v HTML5 aplikaci. V této části nakombinujeme Bootstrap s vlastními CSS pravidly, aby naše aplikace trochu vypadala. Je také dobré vědět, že samotné použití Bootstrapu činí naši stránku [responzivní](http://en.wikipedia.org/wiki/Responsive_web_design), tedy že bude vypadat rozumně na široké škále přístrojů a displejů (PC, laptop, tablet, mobilní telefon apod).

V prvé řadě si tedy přidáme Bootstrap, k čemuž slouží v aplikacích Rails gem "bootstrap-sass", jak je vidět na výpisu níže. Bootstrap nativně používá jazyk [Less CSS](http://lesscss.org/) k vytváření dynamických stylů. Rails podporuje (velmi podobný) jazyk Sass, takže "bootstrap-sass" konvertuje Less na Sass a zpřístupňuje naši aplikaci všechny potřebné Bootstrapové soubory. Do gemfile si tedy přidáme:

```
source 'https://rubygems.org'

gem 'rails',          '5.1.6'
gem 'bootstrap-sass', '3.3.7'
.
.
.
```

Pro instalaci Bootstrapu spustíme jako obvykle "bundle install":

```
$ bundle install
```

Ačkoliv "rails generate" automaticky vytváří pro každý ovladač zvlášť CSS soubor, je překvapivě obtížné je všechny zahrnout ve správném pořadí. Umístíme si tedy v tomto případě všechno potřebné CSS do jednoho souboru. Prvním krokem k vlastnímu CSS bude vytvoření jeho souboru:

```
$ touch app/assets/stylesheets/custom.scss
```

Jak název složky, tak přípona souboru jsou důležité prvky. Složka

```
app/assets/stylesheets/
```

je součástí zřetězeného zpracování, a jakékoliv soubory stylů v této složce budou automaticky zahrnuty jako část souboru "application.css" v layoutu stránky. Přípona .scss označuje soubor jako "Sassy CSS" a dává tím procesu zpracování najevo, ať soubor zpracuje za použití Sass.

Uvnitř souboru pro vlastní CSS pak můžeme použít funkci "@import" pro zahrnutí Bootstrapu (společne se související utilitou Sprockets):

```
@import "bootstrap-sprockets";
@import "bootstrap";
```

Tyto dva řádky pak zahrnují celý Bootstrapový framework. Po restartování serveru pro aktualizaci změn (Ctrl+C v konzoli běžícího serveru a následné opětovné spuštění skrze "rails server") můžeme vidět výsledek podobný, jako na následujícím obrázku (obr. 5.2). Umístění textu není správně a logo nemá žádně styly, ale barvy a tlačítko pro registraci už vypadají slibně.

// obr. 5.5 Aplikace s Bootstrap CSS.

Dále si tedy přidáme trochu CSS, které se následně použije pro stylování layoutu všech jednotlivých stránek podle výpisu níže. Výsledek pak bude vypadat jako na obrázku (obr. 5.6). Řádků je vícero, a pro experimentování a zjištění, co který dělá, je dobré si zkusit zakomentovat některá pravidla (pomocí syntaxe /* ..... */) a pozorovat změny.

```
@import "bootstrap-sprockets";
@import "bootstrap";

/* universal */

body {
  padding-top: 60px;
}

section {
  overflow: auto;
}

textarea {
  resize: vertical;
}

.center {
  text-align: center;
}

.center h1 {
  margin-bottom: 10px;
}
```

// obr 5.6 Přidání odsazení a dalšího univerzálního stylování.

Všimněme si, že CSS má konzistentní podobu. Obecně vzato, pravidla CSS se vztahují na třídu, id, HTML tag nebo kombinaci předchozího a následuje seznam stylovacích příkazů. Například

```
body {
  padding-top: 60px;
}
```

odsadí vrch stránky o 60 pixelů. Díky třídě "navbar-fixed-top" v tagu "header" Bootstrap zafixuje navigační panel k vrchu stránky, takže odsazení slouží k oddělení hlavního textu od navigace. CSS v pravidle

```
.center {
  text-align: center;
}
```

nastaví třídě "center" vlastnost "text-align: center". Jinak řečeno, tečka v ".center" označuje třídu jako cíl pro stylování. Tedy, všechny prvky uvnitř jakéhokoliv tagu (třeba "div") se třídou "center" budou na stránce zarovnány na střed.

I když Bootstrap nabízí CSS pravidla pro poměrně peknou typografii, přidáme si i vlastní pravidla pro vzhled textu na naší stránce, viz výpis níže. (Ne všechny pravidla se vztahují na domovskou stránku, ale dříve nebo později je využijeme.) Výsledek je opět vidět na obrázku (obr. 5.7).

```
@import "bootstrap-sprockets";
@import "bootstrap";
.
.
.
/* typography */

h1, h2, h3, h4, h5, h6 {
  line-height: 1;
}

h1 {
  font-size: 3em;
  letter-spacing: -2px;
  margin-bottom: 30px;
  text-align: center;
}

h2 {
  font-size: 1.2em;
  letter-spacing: -1px;
  margin-bottom: 30px;
  text-align: center;
  font-weight: normal;
  color: #777;
}

p {
  font-size: 1.1em;
  line-height: 1.7em;
}
```

// obr. 5.7 Přidání několika typografických stylů.

Nakonec přidáme pár pravidel i pro logo stránky, které sestává z textu "sample app". CSS v následujícím výpisu převede text na verzálky a změní jeho velikost, barvu a umístění. (Použili jsme id namísto třídy proto, že očekáváme na stránce jen jeden výskyt loga v jednu chvíli, ale šla by samozřejmě použít i třída.)

```
@import "bootstrap-sprockets";
@import "bootstrap";
.
.
.
/* header */

#logo {
  float: left;
  margin-right: 10px;
  font-size: 1.7em;
  color: #fff;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
}

#logo:hover {
  color: #fff;
  text-decoration: none;
}
```

Prvek "color: #fff" mění barvu loga na bílé. HTML barvy mohou být zapsány třemí páry hexadecimálních čísel, kde každé odpovídá jedné ze základních barev, tedy Red (červená), Green (zelená) a Blue (modrá). Kód "#ffffff" nastaví všechny barvy na maximum, což má za výsledek bílou, a "#fff" je jeho zkrácená podoba, kterou lze také použít. CSS také definuje poměrně velké množství synonym pro barvy, například "white" právě pro "#fff". Výsledek je opět na obrázku (obr. 5.8).

// obr. 5.8 Aplikace s pěkně nastylovaným logem.

### Partials (dílčí prvky)

I když základní layout plní svůj účel, je poněkud zanesený. HTML vsuvka pro Internet Explorer používá podivnou syntaxi a zabír tři řádky, takže by bylo praktické ji uklidit někam, kde nebude zavazet. Navíc tvoří HTML hlavička logickou jednotku, takže by měla být celá umístěna na jednom místě. Tohle všechno vyřeší "funkce" Rails zvaná partials (dílčí prvky, které jsou určeny k použití na více místech). Nejprve se podíváme, jak bude layout vypadat poté, co si partials zadefinujeme (soubor "app/views/layouts/application.html.erb"):

```
<!DOCTYPE html>
<html>
  <head>
    <title><%= full_title(yield(:title)) %></title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all',
                                              'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
    <%= render 'layouts/shim' %>
  </head>
  <body>
    <%= render 'layouts/header' %>
    <div class="container">
      <%= yield %>
    </div>
  </body>
</html>
```

HTML vsuvku jsme nahradili jednořádkovým zavoláním pomocníka Rails "render":

```
<%= render 'layouts/shim' %>
```

Výsledkem tohoto řádku je vyhledání souboru nazvaného "app/views/layouts/_shim.html.erb", vyhodnocení jeho obsahu a vložení výsledku do pohledu. Za zmínku stojí podtržítko na začátku souboru, kterým se obecně označují partials, a mimojiné je díky tomu možno je rozeznat na první pohled.

Aby partial fungoval, bude třeba samozřejmě kýžený soubor vytvořit a naplnit. V případě našeho vsuvkového partialu jde jen o tři řádky pro IE, a obsah souboru "app/views/layouts/_shim.html.erb" pak bude vypadat takto:

```
<!--[if lt IE 9]>
  <script src="//cdnjs.cloudflare.com/ajax/libs/html5shiv/r29/html5.min.js">
  </script>
<![endif]-->
```

Obdobně můžeme přesunout obsah hlavičky do svého vlastního partialu a vložit ho do layoutu zavoláním funkce "render". (U partials je obvyklý postup jejich vytváření skrze textový editor.) Soubor "app/views/layouts/_header.html.erb" si tedy naplníme takto:

```
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", '#', id: "logo" %>
    <nav>
      <ul class="nav navbar-nav navbar-right">
        <li><%= link_to "Home",   '#' %></li>
        <li><%= link_to "Help",   '#' %></li>
        <li><%= link_to "Log in", '#' %></li>
      </ul>
    </nav>
  </div>
</header>
```

Když už teď umíme vytvářet partials, přidáme si k hlavičče i patičku stránky. Jak se dá uhodnout, pojmenujeme ji "\_footer.html.erb" a vložíme ji do složky s layouty ("app/views/layouts/\_footer.html.erb"):

```
<footer class="footer">
  <small>
    Ruby on Rails Tutorial
  </small>
  <nav>
    <ul>
      <li><%= link_to "About",   '#' %></li>
      <li><%= link_to "Contact", '#' %></li>
    </ul>
  </nav>
</footer>
```

Podobně jako u hlavičky jsme použili "link_to" pro interní odkazy ke stránkám About a Contacts (a zatím je ponechali s URL "pahýlem"). Do layoutu si patičkový partial vložíme na stejném principu, jako soubory stylů a hlavičku:

```
<!DOCTYPE html>
<html>
  <head>
    <title><%= full_title(yield(:title)) %></title>
    <%= csrf_meta_tags %>
    <%= stylesheet_link_tag    'application', media: 'all',
                                              'data-turbolinks-track': 'reload' %>
    <%= javascript_include_tag 'application', 'data-turbolinks-track': 'reload' %>
    <%= render 'layouts/shim' %>
  </head>
  <body>
    <%= render 'layouts/header' %>
    <div class="container">
      <%= yield %>
      <%= render 'layouts/footer' %>
    </div>
  </body>
</html>
```

Samozřejmě by naše patička byla bez trochy stylování ošklivá, tak si ho přidáme. Výsledek je na obrázku (obr. 5.9).

```
.
.
.
/* footer */

footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid #eaeaea;
  color: #777;
}

footer a {
  color: #555;
}

footer a:hover {
  color: #222;
}

footer small {
  float: left;
}

footer ul {
  float: right;
  list-style: none;
}

footer ul li {
  float: left;
  margin-left: 15px;
}
```

// obr. 5.9 Domovská stránka s přidanou patičkou.



## Sass a zřetězené zpracování (asset pipeline)

Jedna z nejužitečnějších vlastností Rails je asset pipeline (zřetězené zpracování zdrojů), která značně zjednodušuje práci se statickými zdroji jako je CSS, JavaScript a obrázky. Podíváme se jí tedy na zoubek, a pak se budeme věnovat i Sass, užitečnému nástroji pro psaní CSS.

### Asset pipeline

Z pohledu běžného vývojáře Rails existují tři základní prvky, které je třeba pochopit v rámci asset pipeline: složky zdrojů, seznamové soubory a předprocesorové zpracovávání.

####Složky zdrojů

Rails používá tři standardní složky pro statické zdroje, kde má každá svůj význam:

- "app/assets":  zdroje specifické pro současnou aplikaci
- "lib/assets": zdroje pro knihovny napsané vývojářským týmem
- "vendor/assets": zdroje třetích stran

Každá z těchto složek má podsložky pro každou ze tříd zdrojů: obrázky, JavaScript a Cascading Style Sheets (CSS):

```
$ ls app/assets/
images/  javascripts/  stylesheets/
```

Proto jsme si umístili naše vlastní CSS právě do složky "app/assets/stylesheets".

#### Seznamové soubory (manifest files)

Jakmile máme umístěny zdroje do jejich náležitých lokací, můžeme použít seznamové soubory (manifest files), skrze gem [Sprockets](https://github.com/rails/sprockets), jako informační zdroj pro Rails, který je následně nakombinuje do jednotlivých souborů. (Což se samozřejmě vztahuje jen na CSS a JavaScript, na obrázky ne.) Podívejme se například na základní seznamový soubor pro aplikační styly ("app/assets/stylesheets/application.css"):

```
/*
 * This is a manifest file that'll be compiled into application.css, which
 * will include all the files listed below.
 *
 * Any CSS and SCSS file within this directory, lib/assets/stylesheets,
 * vendor/assets/stylesheets, or vendor/assets/stylesheets of plugins, if any,
 * can be referenced here using a relative path.
 *
 * You're free to add application-wide styles to this file and they'll appear
 * at the bottom of the compiled file so the styles you add here take
 * precedence over styles defined in any styles defined in the other CSS/SCSS
 * files in this directory. It is generally better to create a new file per
 * style scope.
 *
 *= require_tree .
 *= require_self
 */
```

Klíčové řádky jsou ve skutečnosti CSS komentáře, které ale používá Sprockets pro začlenění příslušných souborů:

```
/*
 .
 .
 .
 *= require_tree .
 *= require_self
*/
```

Řádek

```
*= require_tree .
```

zajišťuje, že všechny CSS soubory ve složce "app/assets/stylesheets" (včetně podsložek) jsou zahrnuty do aplikačního CSS. Řádek

```
*= require_self
```

určuje, kde v rámci nahrávací sekvence CSS má být umístěno "application.css" samo o sobě.

Rails má k dispozici rozumné seznamové soubory, a v rámci naší výuky do nich nebudeme potřebovat zasahovat.

####Předprocesorové zpracování (preprocessor engines)

Jakmile máme shromážděné zdroje, Rails je připraví pro šablonu stránky tím, že je prožene několika předprocesory, a za pomocí seznamových souborů je nakombinuje pro finální zobrazení prohlížečem. Jaký předprocesor použít dáváme Railsu vědět skrze souborové přípony; tři nejběžnější jsou ".scss" pro Sass, ".coffee" pro CoffeeScript a ".erb" pro vnořené Ruby (embedded ruby, ERb). ERb jsme si už prošli, Sass nás čeká záhy a CoffeeScript v rámci výuky potřebovat nebudeme, ale jde o elegantní jazyk, který je nakonec převeden na JavaScript.

Předprocesory mohou být zřetězeny, takže

`foobar.js.coffee`

je prohnán skrze CoffeeScript, a

`foobar.js.erb.coffee`

je první zpracován CoffeeScriptem a následně ERb (posloupnost je zprava doleva!).

####Efektivita v produkční fázi

Jedna z nejlepších věcí ohledně asset pipeline je ta, že její výsledek je vždy optimalizován, aby byl efektivní i v produkčním (finálním) použití. Tradiční metody pro organizování CSS a JavaScriptu zahrnují rozdělování funkcionality do specifických souborů a spoustu formátování a odsazování. I když praktické pro programátora, nepraktické pro produkční nasazení. Zahrnutí několika neoptimalizovaných nabobtnalých souborů může výrazně zvýšit nahrávací časy stránek, což je jeden z nejdůležitějších faktorů, které ovlivňují kvalitu uživatelského dojmu. S asset pipeline si nemusíme vybírat jen jedno z toho. Ve vývojové fázi máme k dispozici hezky formátované soubory, které jsou pak jednoduše převedeny do formátů, které jsou efektivní v ostrém nasazení. Asset pipeline první nakombinuje veškeré CSS do jednoho souboru ("application.css"), veškerý JavaScript též ("application.js") a pak je minifikuje, díky čemuž odstraní přebytečné odsazení a mezery, které soubor nafukují. Výsledkem je pak to nejlepší z obou světů: přehlednost ve vývoji a efektivita při nasazení.



###Syntakticky skvělé styly

Sass je jazyk určený k psaní stylů, který vylepšuje v mnoha směrech klasické CSS. Projdeme si dvě jeho nejdůležitější dovednost, "nesting" (vnořování) a proměnné.

Jak jsme zmínili dříve, Sass podporuje formát zvaný SCSS (s příponou ".scss"), který je v podstatě nadřazený CSS samotnému; tedy, SCSS pouze přidává CSS další dovednosti, než aby definoval nový styl zápisu. Každý validní CSS soubor je tedy i validním SCSS souborem, což je praktické pro projekty s existující sadou stylů. V našem případě jsme použili SCSS už ze startu, abychom mohli využít výhod Bootstrapu. Jelikož Railsová asset pipeline automaticky používá Sass ke zpracování souborů s příponou ".scss", soubor "custom.scss" bude zpracován Sass předprocesorem předtím, než bude připraven k odeslání prohlížeči.

#### Nesting (vnořování)

Běžnou praxí v rámci stylů bývá existence pravidel, které se aplikují na vnořené prvky. Dříve jsme si zadefinovali pravidla jak pro ".center", tak pro ".center h1":

```
.center {
  text-align: center;
}

.center h1 {
  margin-bottom: 10px;
}
```

V Sass to můžeme nahradit praktičtější formou:

```
.center {
  text-align: center;
  h1 {
    margin-bottom: 10px;
  }
}
```

Vnořené pravidlo pro "h1" automaticky dědí kontext ".center".

Máme i druhého adepta na princip vnořování, který ale vyžaduje poněkud jinou syntaxi. Máme kód:

```
#logo {
  float: left;
  margin-right: 10px;
  font-size: 1.7em;
  color: #fff;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
}

#logo:hover {
  color: #fff;
  text-decoration: none;
}
```

Logo "#logo" se zde objevuje dvakrát, jednou samo o sobě a podruhé s parametrem "hover" (který definuje jeho chování v případě, že se nad takovým prvkem objeví kurzor myši). Abychom zanořili druhé pravidlo, budeme potřebovat odkaz na rodičovský element "#logo"; v SCSS je tohoto dosaženo skrze znak ampersand (&):

```
#logo {
  float: left;
  margin-right: 10px;
  font-size: 1.7em;
  color: #fff;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
  &:hover {
    color: #fff;
    text-decoration: none;
  }
}
```

Sass změní "&:hover" na "#logo:hover" v rámci převedené SCSS na CSS.

Obě vnořovací techniky lze použít i na CSS patičky, která bude vypadat následovně:

```
footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid #eaeaea;
  color: #777;
  a {
    color: #555;
    &:hover {
      color: #222;
    }
  }
  small {
    float: left;
  }
  ul {
    float: right;
    list-style: none;
    li {
      float: left;
      margin-left: 15px;
    }
  }
}
```

Ruční předělání je dobrým cvičením, ale je vždy třeba ověřit, že CSS stále funguje jak má i po konverzi.



#### Proměnné

Sass nám pro eliminaci duplicity a více výstižný kód umožňuje použít také proměnné. Kupříkladu můžeme vidět, že máme definováno vícero referencí na stejnou barvu:

```
h2 {
  .
  .
  .
  color: #777;
}
.
.
.
footer {
  .
  .
  .
  color: #777;
}
```

V tomto případě je #777 světle šedá, a můžeme jí pojmenovat skrze definování proměnné:

```
$light-gray: #777;
```

SCSS pak můžeme přepsat takto:

```
$light-gray: #777;
.
.
.
h2 {
  .
  .
  .
  color: $light-gray;
}
.
.
.
footer {
  .
  .
  .
  color: $light-gray;
}
```

Jelikož proměnné jako "$light-gray" jsou výstižnější než "#777", často se definují proměnné i pro hodnoty, které se neopakují. Dokonce i Bootstrap definuje velké množství [proměnných pro různé barvy](http://getbootstrap.com/customize/#less-variables). I když stránka definuje proměnné skrze Less, ne Sass, "bootstrap-sass" gem poskytuje Sass ekvivalenty. Souvislost je lehké uhodnout; kde používá Less znak @, Sass používá znak dolaru \$. Na stránce najdeme i proměnnou pro světle šedou:

```
 @gray-light: #777;
```

Skrze gem "bootstrap-sass" by tedy měla existovat související SCSS proměnná "$gray-light". Tou můžeme nahradit naši vlastní proměnnou "\$light-gray":

```
h2 {
  .
  .
  .
  color: $gray-light;
}
.
.
.
footer {
  .
  .
  .
  color: $gray-light;
}
```

Aplikace obou metod, jak vnořování, tak proměnných na celý SCSS soubor ("app/assets/stylesheets/custom.scss") pak vypadá následovně. Zejména stoji za povšimnutí znatelné vylepšení pravidel pro tag "footer".

```
@import "bootstrap-sprockets";
@import "bootstrap";

/* mixins, variables, etc. */

$gray-medium-light: #eaeaea;

/* universal */

body {
  padding-top: 60px;
}

section {
  overflow: auto;
}

textarea {
  resize: vertical;
}

.center {
  text-align: center;
  h1 {
    margin-bottom: 10px;
  }
}

/* typography */

h1, h2, h3, h4, h5, h6 {
  line-height: 1;
}

h1 {
  font-size: 3em;
  letter-spacing: -2px;
  margin-bottom: 30px;
  text-align: center;
}

h2 {
  font-size: 1.2em;
  letter-spacing: -1px;
  margin-bottom: 30px;
  text-align: center;
  font-weight: normal;
  color: $gray-light;
}

p {
  font-size: 1.1em;
  line-height: 1.7em;
}


/* header */

#logo {
  float: left;
  margin-right: 10px;
  font-size: 1.7em;
  color: white;
  text-transform: uppercase;
  letter-spacing: -1px;
  padding-top: 9px;
  font-weight: bold;
  &:hover {
    color: white;
    text-decoration: none;
  }
}

/* footer */

footer {
  margin-top: 45px;
  padding-top: 5px;
  border-top: 1px solid $gray-medium-light;
  color: $gray-light;
  a {
    color: $gray;
    &:hover {
      color: $gray-darker;
    }
  }
  small {
    float: left;
  }
  ul {
    float: right;
    list-style: none;
    li {
      float: left;
      margin-left: 15px;
    }
  }
}
```

Sass nám nabízí ještě dodatečné způsoby zjednodušení stylů, ale předešlý kód používá ty nejdůležitější, což už je dobrý začátek sám o sobě. Vícero možností lze najít na [stránce Sass](http://sass-lang.com/).



## Odkazy v layoutu

Když teď máme stránku jakž takž nastylovanou, začneme naplňovat odkazy, které jsme doteď ponechali ve formě pahýlu ("#"). Samozřejmě bychom mohli použít hard-code odkazy jako

```
<a href="/static_pages/about">O nás</a>
```

ale to by bylo v Rails téměř rouhavé. V prvě řadě by bylo hezké kdyby URL na stránku "about" byla /about, namísto /static_pages/about. Rails navíc používá jmenné cesty, které zahrnují kód jako

```
<%= link_to "About", about_path %>
```

Kód je takto mnohem transparentnější, a také více flexibilní, jelikož můžeme změnit definici "about_path" a URL se nám změní všude, kde jsme "about_path" použili.

Kompletní seznam plánovaných odkazů je v tabulce, spolu s jejich cestami a URL. Do konce této části si krom posledního doimplementujeme všechny.

| **Stránka** | **URL**  | **Jmenná cesta** |
| ----------- | -------- | ---------------- |
| Domů        | /        | `root_path`      |
| O nás       | /about   | `about_path`     |
| Pomoc       | /help    | `help_path`      |
| Kontakt     | /contact | `contact_path`   |
| Registrace  | /signup  | `signup_path`    |
| Přihlášení  | /login   | `login_path`     |

### Stránka kontaktů

Začneme tedy stránkou kontaktů (Contact page). Postupujeme opět TDD způsobem a upravíme si soubor "test/controllers/static_pages_controller_test.rb", tedy první krok bude červený test.

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
    assert_select "title", "Pomoc | Ruby on Rails Tutorial"
  end

  test "should get about" do
    get static_pages_about_url
    assert_response :success
    assert_select "title", "O nás | Ruby on Rails Tutorial"
  end

  test "should get contact" do
    get static_pages_contact_url
    assert_response :success
    assert_select "title", "Kontakty | Ruby on Rails Tutorial"
  end
end
```

Test by měl tedy proběhnout červeně.

```
$ rails test
```

Aplikační kód odráží přidání stránky About v předchozí části: první aktualizujeme cesty ("config/routes.rb"), přidáme akci "contact" do ovladače statických stránek ("app/controllers/static_pages_controller.rb") a nakonec vytvoříme pro kontakty i pohled ("app/views/static_pages/contact.html.erb").

```
Rails.application.routes.draw do
  root 'static_pages#home'
  get  'static_pages/home'
  get  'static_pages/help'
  get  'static_pages/about'
  get  'static_pages/contact'
end
```

```
class StaticPagesController < ApplicationController
  .
  .
  .
  def contact
  end
end
```

```
<% provide(:title, 'Contact') %>
<h1>Kontakty</h1>
<p>
  Stránka s kontakty na autora aplikace.
</p>
```

Všechny testy by měly proběhnout zeleně.

```
$ rails test
```

### Cesty v Rails

Pro přidání pojmenovaných cest pro statické stránky upravíme soubor s cestami "config/routes.rb", který Rails používá pro definování mapování URL. V prvé řadě zrevidujeme cestu na domovskou stránku, která je sama o sobě speciální případ, a pak zadefinujeme sadu cest pro zbylé stránky.

Zatím jsme viděli tři příklady, jak jde zadefinovat kořenová cesta, počínaje kódem

```
root 'application#hello'
```

v první aplikaci, poté

```
root 'users#index'
```

v druhé, a

```
root 'static_pages#home'
```

naposledy. V každém případě metoda "root" nastaví cestu / k ovladači a akci podle našeho výběru. Definování kořenové cesty tímto způsobem má i druhý důležitý efekt, a to vytvoření jmenné cesty, která nám umožní se odkazovat na cesty podle jména namísto čisté URL. V tomto případě jsou cesty "root_path" a "root_url", kde je jediny rozdíl ten, že druhá jmenovaná obsahuje celou URL:

```
root_path -> '/'
root_url  -> 'http://www.example.com/'
```

Budeme se držet běžné praxe, a to použití variantu "\_path", s výjimkou přesměrování, kde použijeme variantu "\_url". (Z toho důvodu, že HTTP standard technicky vzato vyžaduje po přesměrování plnou URL, i když ve většině prohlížečů fungují obě možnosti.)

Jelikož jsou základní cesty poněkud kypré, využijeme příležitosti a zadefinujeme si kratší jmenné cesty pro stránky Help, About a Contacts. V "config/routes.rb" si tedy upravíme pravidla "get" takovýmto způsobem:

```
get 'static_pages/help'
```

na

```
get  '/help', to: 'static_pages#help'
```

Tento nový vzorec zápisu směruje požadavek GET na URL /help na akci "help" v ovladači statických stránek. Jako u kořenové cesty, i zde pak dojde k vytvoření dvou jmenných cest, "help_path" a "help_url":

```
help_path -> '/help'
help_url  -> 'http://www.example.com/help'
```

Aplikování této změny na zbylé statické stránky pak bude vypadat takto:

```
Rails.application.routes.draw do
  root 'static_pages#home'
  get  '/help',    to: 'static_pages#help'
  get  '/about',   to: 'static_pages#about'
  get  '/contact', to: 'static_pages#contact'
end
```

Odstranili jsme cestu pro "static_pages/home", jelikož budeme vždy používat variantu "root_path" nebo "root_url".

Jelikož testy používaly staré cesty, výsledek je nyní červený. Aby prošel na zelenou, musíme si upravit i cesty následujícím způsobem, včetně nepovinné formy "*_path" u každé jmenné cesty:

```
require 'test_helper'

class StaticPagesControllerTest < ActionDispatch::IntegrationTest

  test "should get home" do
    get root_path
    assert_response :success
    assert_select "title", "Ruby on Rails Tutorial"
  end

  test "should get help" do
    get help_path
    assert_response :success
    assert_select "title", "Pomoc | Ruby on Rails Tutorial"
  end

  test "should get about" do
    get about_path
    assert_response :success
    assert_select "title", "O nás | Ruby on Rails Tutorial"
  end

  test "should get contact" do
    get contact_path
    assert_response :success
    assert_select "title", "Kontakty | Ruby on Rails Tutorial"
  end
end
```



### Použití jmenných cest

Když máme cesty zadefinované, můžeme použít výsledne jmenné cesty v layoutu. Obnáší to pouze naplnění druhých parametrů funkcí "link_to" náležitými cestami. Kupříkladu, převedeme

```
<%= link_to "About", '#' %>
```

na

```
<%= link_to "About", about_path %>
```

a podobně.

Začneme s partialem hlavičky, "app/views/layouts/_header.html.erb", který obsahuje odkazy na stránky Home a Help. Když už jsme u toho, budeme následovat bězný webový postup a nalinkujeme také logo na domovskou stránku.

```
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", root_path, id: "logo" %>
    <nav>
      <ul class="nav navbar-nav navbar-right">
        <li><%= link_to "Home",    root_path %></li>
        <li><%= link_to "Help",    help_path %></li>
        <li><%= link_to "Log in", '#' %></li>
      </ul>
    </nav>
  </div>
</header>
```

Pro přihlášení zatím jmennou cestu nemáme (přidáme si ji později), takže jsme ji zatím nechali jako pahýl. Další místo s odkazy je partial patičky ("app/views/layouts/_footer.html.erb"), který má odkazy na stránky About a Contact.

```
<footer class="footer">
  <small>
    Ruby on Rails Tutorial    
  </small>
  <nav>
    <ul>
      <li><%= link_to "About",   about_path %></li>
      <li><%= link_to "Contact", contact_path %></li>
    </ul>
  </nav>
</footer>
```

V této fázi má náš layout vytvořené odkazy ke všem statickým stránkám, takže například /about vede na stránku O nás.

// obr. 5.10 Stránka O nás na /about.

### Testování odkazů layoutu

Po naplnění odkazů layoutu bude dobrý nápad si je otestovat, ať máme jistotu, že fungují správně. Jde to samozřejmě i ručně pomocí prohlížeče, kde první navštívíme kořenovou cestu a pak jednotlivé odkazy jeden po druhém. Ale to by nebylo příliš praktické. Raději nasimulujeme stejnou sérii kroků za použití integračního testu, který nám umožňuje ozkoušet chování naší aplikace z pohledu koncového uživatele. Začneme vytvořením šablony testu, kterou si nazveme "site_layout":

```
$ rails generate integration_test site_layout
      invoke  test_unit
      create    test/integration/site_layout_test.rb
```

Rails automaticky přidá ke jménu souboru "_test".

Náš plán bude zahrnovat otestování odkazů layoutu ověřením HTML struktury stránky:

1. Zjisti kořenovou cestu (domovská stránka).
2. Ověř, že je vykreslena správná šablona.
3. Zjisti správné odkazy na stránky Home, Help, About a Contact.

Následující výpis ("test/integration/site_layout_test.rb") demonstruje, jak můžeme použít integrační test Rails pro preložení těchto kroků do kódu, počínaje "assert_template" metodou pro ověření, že domovská stránka je vykreslena za pomocí správného pohledu.

```
require 'test_helper'

class SiteLayoutTest < ActionDispatch::IntegrationTest

  test "layout links" do
    get root_path
    assert_template 'static_pages/home'
    assert_select "a[href=?]", root_path, count: 2
    assert_select "a[href=?]", help_path
    assert_select "a[href=?]", about_path
    assert_select "a[href=?]", contact_path
  end
end
```

Použity jsou i některé více pokročilé možnosti metody "assert_select". Konkrétně tedy syntaxe, která nám umožňuje ověřit přítomnost konkrétní kombinace odkaz-URL specifikováním jména tagu "a" a atributu "href", jako např. v:

```
assert_select "a[href=?]", about_path
```

Rails automaticky vloží hodnotu "about_path" namísto otazníku a tím ověří existenci HTML tagu ve tvaru:

```
<a href="/about">...</a>
```

Verze pro kořenovou cestu zahrnuje podmínku pro dva takové odkazy (je totiž jak v logu, tak v navigaci):

```
assert_select "a[href=?]", root_path, count: 2
```

Díky tomu si můžeme být jisti, že jsou oba odkazy přítomné.

Několik dalších možností použití "assert_select" jsou v následující tabulce. I když je "assert_select" flexibilní a užitečný nástroj, z praxe se obvykle člověk naučí, že je lepší volit lehčí přístup a ověřovat pouze HTML prvky, u kterých je nízká pravděpodobnost, že se časem změní.

| **Kód**                                       | **Odpovídající HTML**            |
| --------------------------------------------- | -------------------------------- |
| `assert_select "div"`                         | `<div>foobar</div>`              |
| `assert_select "div", "foobar"`               | `<div>foobar</div>`              |
| `assert_select "div.nav"`                     | `<div class="nav">foobar</div>`  |
| `assert_select "div#profile"`                 | `<div id="profile">foobar</div>` |
| `assert_select "div[name=yo]"`                | `<div name="yo">hey</div>`       |
| `assert_select "a[href=?]", '/', count: 1`    | `<a href="/">foo</a>`            |
| `assert_select "a[href=?]", '/', text: "foo"` | `<a href="/">foo</a>`            |

Pro ověření, že náš nový test projde, stačí spustit sadu integračních testů následujícím příkazem:

```
$ rails test:integration
```

Pokud vše proběhlo v pořádku, plné testování by mělo také proběhnout na zelenou.

```
$ rails test
```

S přidaným integračním testem pro odkazy layoutu si nyní můžeme odchytávat případné chyby daleko rychleji.



## Registrace uživatelů: První krok

Jako završení naší práce na layoutu a směrování si vytvoříme i cestu pro registrační stránku, což bude zahrnovat i nový ovladač. Jde o první důležitý krok k umožnění uživatelské registrace, záhy se ale dostaneme i k tvorbě modelu a dokončení.

### Uživatelský ovladač (user controller)

Po ovladači statických stránek je na čase si vytvořit náš druhý ovladač, a to uživatelský. Opět použijeme "generate" pro vytvoření nejjednoduššího ovladače, který splňuje naše potřeby (jmenovitě s pahýlem registrační stránky pro nové uživatele). Jak je v rámci REST architektury zvykem, pojmenujeme akci pro nové uživatele "new", čehož dosáhneme předáním parametru funkci "generate". Výsledek vypadá následovně:

```
$ rails generate controller Users new
      create  app/controllers/users_controller.rb
       route  get 'users/new'
      invoke  erb
      create    app/views/users
      create    app/views/users/new.html.erb
      invoke  test_unit
      create    test/controllers/users_controller_test.rb
      invoke  helper
      create    app/helpers/users_helper.rb
      invoke    test_unit
      invoke  assets
      invoke    coffee
      create      app/assets/javascripts/users.coffee
      invoke    scss
      create      app/assets/stylesheets/users.scss
```

Ovladač s akcí "new" vypadá následovně:

```
class UsersController < ApplicationController

  def new
  end
end
```

Pohled "new" pro uživatele ("app/views/users/new.html.erb"):

```
<h1>Users#new</h1>
<p>Find me in app/views/users/new.html.erb</p>
```

Krom ovladače nám Rails vytvořil i základ testu ("test/controllers/users_controller_test.rb") pro novou uživatelskou stránku:

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  test "should get new" do
    get users_new_url
    assert_response :success
  end
end
```

V současné chvíli by měl být test zelený:

```
$ rails test
```

### Registrační URL

Díky předchozímu kroku máme k dispozici funkční stránku na /users/new, ale chtěli jsme ji přece na adrese /signup. Použijeme tedy postup, který jsme se naučili a přidáme do souboru s cestami ("config/routes.rb") potřebné pravidlo:

```
Rails.application.routes.draw do
  root 'static_pages#home'
  get  '/help',    to: 'static_pages#help'
  get  '/about',   to: 'static_pages#about'
  get  '/contact', to: 'static_pages#contact'
  get  '/signup',  to: 'users#new'
end
```

Bude také zapotřebí aktualizovat test ("test/controllers/users_controller_test.rb") novou cestou pro registraci:

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  test "should get new" do
    get signup_path
    assert_response :success
  end
end
```

Dále použijeme nově definovanou jmennou cestu k přidání vhodného odkazu k tlačítku na domovské stránce ("app/views/static_pages/home.html.erb"). Jako u ostatních cest, i zde jsme díky "get 'signup'" dostali k dispozici jmennou cestu "signup_path", kterou použijeme následovně:

```
<div class="center jumbotron">
  <h1>Vítejte v aplikaci</h1>

  <h2>
    Toto je domovská stránka naší tréningové aplikace.
  </h2>

  <%= link_to "Zaregistrovat se", signup_path, class: "btn btn-lg btn-primary" %>
</div>

<%= link_to image_tag("rails.png", alt: "Rails logo"),
            'http://rubyonrails.org/' %>
```

Nakonec si přidáme vlastní pahýl pohledu pro registrační stránku ("app/views/users/new.html.erb"):

```
<% provide(:title, 'Sign up') %>
<h1>Registrace</h1>
<p>Zde bude registrační stránka pro nové uživatele.</p>
```

Dokud nepřidáme cestu pro přihlašování, máme, co se odkazů a jmenných cest týče, hotovo.

// obr. 5.11 Nová registrační stránka na /signup.



## Shrnutí

V této části jsme konečně dali našemu aplikačnímu layoutu formu a vyřesili i problematiku trasování. Dále si budeme přidávat registraci, přihlášení a odhlášení uživatelů, mikroposty a nakonec i možnost sledovat ostatní uživatele.

Opět je dobrá chvíle pro spojení změn z vedlejší větve do hlavní:

```
$ git add -A
$ git commit -m "Dodelani layoutu a tras"
$ git checkout master
$ git merge filling-in-layout
```

Zbývá jen odeslat na Bitbucket (a předem samozřejmě test):

```
$ rails test
$ git push
```

Výsledek by měl být fungující aplikace na produkčním serveru (obr. 5.12).

// obr. 5.12  Aplikace v produkční fázi.

### Co jsme se naučili

- Díky použití HTML5 můžeme definovat layout stránky s logem, hlavičkou, patičkou a hlavním obsahem těla stránky.
- Partials v Rails se používají k umístění markupu do oddělených souborů, kvůli praktičnosti a přehlednosti.
- CSS nám umožňuje nastylovat layout stránky na základě tříd a id.
- Bootstrapový framework podstatně zjednodušuje tvorbu rychlého a pěkného designu stránky.
- Sass a asset pipeline nám umožňují eliminovat duplicitu v CSS a výsledky zabalí vhodným způsobem pro produkční nasazení.
- Rails nám umožňuje definovat vlastní pravidla tras a tímpádem i jmenné (pojmenované) cesty.
- Integrační testy dokážou efektivně simulovat chování prohlížeče, který přechází klikáním ze stránky na stránku.

