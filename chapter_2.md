#První aplikace

V této kapitole se budeme věnovat vývoji jednoduché zkušební aplikace, abychom zjistili, co všechno Rails dokáže. Díky použití tzv. "scaffold generators", které umí vygenerovat velké množství funkcionality automaticky, získáme náhled do programování v Rails (a webového vývoje obecně). Další kapitoly se budou věnovat opačnému přístupu, kdy budeme postupně nabalovat funkcionalitu v situacích, kdy vyvstane její potřeba, ale pro jednoduchý přehled a objasnění je praktičtější použít právě scaffolding (angl. "lešení", používá se přeneseně). Výsledná aplikace nám umožní interakci skrze URL a pochopíme i její strukturu, včetně prvního příkladu architektury REST, kterou Rails preferuje.

Aplikace bude sestávat z uživatelů a jejich asociovaných "mikropostů" (v podstatě tedy něco, jako minimalistický Twitter). Funkcionalita bude poměrně primitivní a hodně kroků bude působit jako kouzlo, ale netřeba obav, jelikož si od další kapitoly vysvětlíme, jak vyvinout podobnou aplikaci od nuly. Do té doby je třeba mít trpělivost - nakonec si totiž odneseme hlubší a detailnější pochopení toho, jak Rails funguje.



##Plánování aplikace

V této sekci si vytyčíme plán pro naši novou aplikaci. Podobně jako dříve začneme vytvořením kostry skrze příkaz "rails new" se specifickým číslem verze:

```
$ rails _5.1.4_ new toy_app
$ cd toy_app/
```

Dále si upravíme soubor Gemfile:

```
source 'https://rubygems.org'

gem 'rails',        '5.1.4'
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

group :production do
  gem 'pg', '0.20.0'
end

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```

Opět si nainstalujeme potřebné gemy s parametrem "--without production":

```
$ bundle install --without production
```

Nakonec necháme celou aplikaci verzovat Gitem:

```
$ git init
$ git add -A
$ git commit -m "Inicializace repozitare"
```

Zároveň je dobré [vytvořit nový repozitář](https://bitbucket.org/repo/create) kliknutím na tlačítko “Create” v Bitbucketu (viz obr. 2.1), a použít příkaz push na vzdálený repozitář:

```
$ git remote add origin git@bitbucket.org:<uzivatelske_jmeno>/toy_app.git
$ git push -u origin --all 
```

// obr. 2.1 Vytvoření repozitáře pro aplikaci na Bitbucketu.



# DEPLOY ZALEZITOSTI, ZREVIDOVAT U REALIZACE



Teď už máme k dispozici vše potřebné pro tvorbu aplikace samotné. Obvyklý první krok při tvorbě webové aplikace je vytvoření datového modelu, což je reprezentace struktur, které bude aplikace využívat. V našem případě půjde o mikroblog podobný Twitteru, pouze s uživateli a krátkými mikroposty. Začneme tedy modelem pro uživatele (users) aplikace a následně přidáme model pro mikroposty (microposts).

###Model pro uživatele

Pro model uživatelských dat existuje mnoho formátů a různých registračních formulářů, ale pro jednoduchost se vydáme čistě minimalistickým přístupem. Uživatelé naší aplikace budou mít unikátní identifikátor, nazvaný "id" (datového typu "integer"), veřejně zobrazitelné jméno "name" (datového typu "string") a emailovou adresu "email" (taktéž datového typu "string"), která bude sloužit jako pojistka unikátního uživatelského jména. Jednoduchost je dobře vidět na shrnujícím obr. 2.2.

// obr 2.2 Datový model pro uživatele.

Později si ukážeme i to, jak jednotlivé názvy korespondují. Označení "users" má shodné jméno s tabulkou databáze, a atributy "id", "name" a "email" jsou jednotlivé sloupce.

### Model pro mikroposty

Jádro datového modelu mikropostů je ještě jednodušší, než u uživatelů: mikropost má pouze atributy "id" (opět datový typ "integer") a "content" (datový typ "text"). Jelikož ale chceme asociovat jednotlivé mikroposty s konkrétním uživatelem, musíme přidat i atribut "user_id" (datový typ "integer"), ve kterém budeme ukládat identifikační číslo vlastníka postu. Shrnutí je opět na obr. 2.3.

// obr 2.3 Datový model pro mikroposty.

Později si vysvětlíme, jak nám atribut "user_id" umožňuje jednoduše vyřešit problém, že jeden uživatel může mít potenciálně mnoho asociovaných mikropostů.



## Uživatelé

V této sekci si naimplementujeme uživatelský datový model, společně s jeho webovým rozhraním. Tato kombinace bude představovat tzv. "Users resource" (z angl. "resource" - zdroj, ve smyslu suroviny, přeneseně), díky kterému můžeme vnímat uživatele jako objekty, které lze vytvářet, číst, aktualizovat a mazat skrze webový [HTTP protokol](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol). Jak jsme si ujasnili na začátku, tentokrát nám Users resource vytvoří program pro generování scaffoldingu, který je běžně k dispozici u každého projektu v Rails. Nicméně, v současné chvíli není rozumné blíže prozkoumávat vygenerovaný kód; zbytečně by nás to zmátlo.



Scaffolding v Rails je generován příkazem "scaffold" skriptu "rails generate". Parametr "scaffold" se předává ve tvaru jednotného čísla jména zdroje (v tomto případě tedy "User"), spolu s dalšími nepovinnými parametry, které slouží k nastavení atributů datového modelu:

```
$ rails generate scaffold User name:string email:string
      invoke  active_record
      create    db/migrate/20160515001017_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml
      invoke  resource_route
       route    resources :users
      invoke  scaffold_controller
      create    app/controllers/users_controller.rb
      invoke    erb
      create      app/views/users
      create      app/views/users/index.html.erb
      create      app/views/users/edit.html.erb
      create      app/views/users/show.html.erb
      create      app/views/users/new.html.erb
      create      app/views/users/_form.html.erb
      invoke    test_unit
      create      test/controllers/users_controller_test.rb
      invoke    helper
      create      app/helpers/users_helper.rb
      invoke      test_unit
      invoke    jbuilder
      create      app/views/users/index.json.jbuilder
      create      app/views/users/show.json.jbuilder
      invoke  assets
      invoke    coffee
      create      app/assets/javascripts/users.coffee
      invoke    scss
      create      app/assets/stylesheets/users.scss
      invoke  scss
      create    app/assets/stylesheets/scaffolds.scss
```

Zahrnutím parametrů "name:string" a "email:string" jsme si nastavili uživatelský model tak, aby odpovídal tabulce z předchozí části. (Parametr "id" není potřeba specifikovat, jelikož je vytvořen automaticky vždy, jakožto primární klíč databáze.)

Abychom postoupili dále, je nejprve třeba provést migraci databáze příkazem "rails db:migrate":

```
$ rails db:migrate
==  CreateUsers: migrating ====================================================
-- create_table(:users)
   -> 0.0017s
==  CreateUsers: migrated (0.0018s) ===========================================
```

Díky předchozímu příkazu máme naši databázi obohacenou o náš nový datový model "users". O pokročilejšich záležitostech v oblasti migrací databází si ještě povíme později.

Po úspěšné migraci si už můžeme spustit místní webserver v samostatné konzoli:

```
$ rails server
```



### Prohlídka uživatelů

Když navštívíme kořenovou adresu naší stránky, uvidíme pouze základní "hello world" stránku jako v předchozí kapitole. Ovšem díky tomu, že jsme si vygenerovali Users resource, máme nyní k dispozici i velké množství stránek určených k práci s uživateli. Například, stránka pro výpis všech uživatelů je pod /users, a stránka pro vytvoření nového uživatele je pod /users/new. Zbytek této části bude věnován rychloprohlídce těchto uživatelských stránek. V následující tabulce je dobře vidět provázanost jednotlivých stránek a jejich URL adres.

| **URL**       | **Akce** | **Účel**                               |
| ------------- | -------- | -------------------------------------- |
| /users        | index    | stránka se seznamem všech uživatelů    |
| /users/1      | show     | stránka, která zobrazí uživatele 1     |
| /users/new    | new      | stránka pro vytvoření nového uživatele |
| /users/1/edit | edit     | stránka pro úpravu uživatele 1         |

Začneme stránkou s výpisem všech uživatelů naší aplikace, nazvanou "index" a umístěnou v "/users". Jak se dá čekat, zatím tady není ani jeden uživatel.

// obr. 2.4 Stránka index pro uživatele (/users).

Pro vytvoření nového uživatele stačí navštívit stránku "new", umístěnou v "/users/new".

// obr. 2.5 Stránka pro tvorbu uživatelů (/users/new).

Tady už jen vyplníme jméno a email a kliknutím na "Create User" uživatele zaregistrujeme. Výsledkem je stránka "show" v "users/1", viz obr. 2.6. Za povšimnutí stojí, že hodnota "1" v URL značí uživatelovu hodnotu "id" v databázi.

// obr 2.6 Stránka pro zobrazení uživatele (/users/1). 

Pro změnu informací o uživateli navštívíme stránku "edit", umístěnou v "/users/1/edit". Upravením informací a kliknutím na tlačítko "Update User" odešleme provedené změny do databáze.

// obr 2.7 Stránka pro úpravu uživatele (/users/1/edit). 

// obr 2.8 Uživatel s aktualizovanými údaji.

Na stránce "new" (/users/new) si vytvoříme ještě jednoho uživatele a zadáme mu jeho vlastní údaje. Výsledná stránka "index" uživatelů je pak zobrazena na obr. 2.9.

// obr 2.9 Stránka index pro uživatele (/users) s nově přidaným uživatelem. 

Když už teď zvládáme uživatele vytvářet, zobrazovat a upravovat, zbývá už jen se naučit, jak je mazat. Po kliknutí na Destroy by si měl prohližeč vyžádat potvrzení akce - pokud se tak nestane, máte pravděpodobně zakázaný JavaScript, ktery v tomto případě Rails pro smazání uživatele vyžaduje.

// obr 2.10 Smazání uživatele.

### MVC v akci

Po absolvování rychlokurzu ohledně uživatelů se zaměříme na jednu konkrétní část v kontextu Model-View-Controller pojetí z předchozí kapitoly. Vysvětlíme si výsledky typické akce prohlížeče, a to návštěvy stránky se seznamem uživatelů v "/users", viz obr. 2.11.

// obr 2.11 Podrobný diagram MVC v Rails.

Shrnutí kroků z předchozího obrázku vypadá takto:

1. Prohlížeč si zažádá o URL /users. 
2. Rails přesměruje /users na akci "index" v ovladači (controller) Users.
3. Akce "index" požádá model Users o vrácení hodnot všech uživatelů ("User.all").
4. Model Users vytáhne všechny uživatele z databáze.
5. Model Users vrátí seznam uživatelů ovladači.
6. Ovladač zahrne uživatele do proměnné "@users", kterou předá "index" view (pohled). 
7. Pohled (skrze Ruby) vykreslí stránku ve formě HTML.
8. Ovladač předá HTML kód zpátky prohlížeči.

Podíváme se na jednotlivé kroky podrobněji. Začneme požadavkem, který zadal prohlížeč, což je obvyklý výsledek napsaní URL do adresního řádku, nebo kliknutí na odkaz. Požadavek doputuje k směrovači Rails (Rails router), který ho odešle příslušné akci ovladače, právě na základě URL. Kód pro spřažení je zobrazen v následujícím výpisu (soubor "config/routes.rb") - a jde v podstatě o princip tvorby páru URL a akcí, jak je vidno v tabulce na začátku rychlokurzu ohledně uživatelů.

```
Rails.application.routes.draw do
  resources :users
  root 'application#hello'
end
```

Když už si prohlížíme soubor routes.rb, můžeme si nastavit i kořenovou cestu z praktických důvodů na /users. Jelikož chceme použít akci "index" v ovladači "Users", upravíme si obsah souboru následovně:

```
Rails.application.routes.draw do
  resources :users
  root 'users#index'
end
```

Ovladač obsahuje související akce, a stránky z uživatelského rychlokurzu tomu odpovídají. Ovladač vygenerovaný scaffoldingem (soubor "app/controllers/users_controller.rb") je schématicky zobrazen v následujícím výpisu:

```
class UsersController < ApplicationController
  .
  .
  .
  def index
    .
    .
    .
  end

  def show
    .
    .
    .
  end

  def new
    .
    .
    .
  end

  def edit
    .
    .
    .
  end

  def create
    .
    .
    .
  end

  def update
    .
    .
    .
  end

  def destroy
    .
    .
    .
  end
end
```

Za povšimnutí stojí, že soubor obsahuje více akcí, než stránek; akce "index", "show", "new" a "edit" odpovídají stránkám, ale navíc se zde nachází i akce "create", "update" a "destroy". Tyto akce obvykle nevykreslují stránky (ačkoliv mohou), nýbrž jejich hlavní účel je modifikování informací o uživatelích v databázi. Výpis akcí v následující tabulce reprezentuje implementaci architektury REST ("REpresentational State Transfer") v Rails. Zajímavé je i to, že se některé URL překrývají; například uživatelské akce "show" i "update" odpovídají URL "/users/1". Rozdíl mezi nimi je tvořen [metodou požadavku HTTP](http://en.wikipedia.org/wiki/HTTP_request#Request_methods), na kterou reagují. Více si o HTTP požadavcích povíme v další kapitole.

| **HTTP požadavek | **URL**       | Akce      | Účel                                   |
| ---------------- | ------------- | --------- | -------------------------------------- |
| `GET`            | /users        | `index`   | stránka se seznamem všech uživatelů    |
| `GET`            | /users/1      | `show`    | stránka, která zobrazí uživatele 1     |
| `GET`            | /users/new    | `new`     | stránka pro vytvoření nového uživatele |
| `POST`           | /users        | `create`  | vytvoření nového uživatele             |
| `GET`            | /users/1/edit | `edit`    | stránka pro úpravu uživatele 1         |
| `PATCH`          | /users/1      | `update`  | aktualizace uživatele 1                |
| `DELETE`         | /users/1      | `destroy` | smazání uživatele 1                    |

Abychom správně pochopili vztah mezi Users ovladačem a User modelem, zaměříme se na zjednodušenou verzi akce "index":

```
class UsersController < ApplicationController
  .
  .
  .
  def index
    @users = User.all
  end
  .
  .
  .
end
```

Akce "index" obsahuje řídek "@users = User.all" (krok 3 v MVC diagramu), který požádá User model o seznam všech uživatelů z databáze (krok 4), a vloží je do proměnné "@users" (výslovnost "at-users") (krok 5).  Model sám o sobě je v následujícím výpisu (soubor "app/models/user.rb"), a i když se zdá poněkud prázdný, obsahuje ve skutečnosti množství funkcionality díky principu dědičnosti, o které bude také řeč později.

```
class User < ApplicationRecord
end
```

Jakmile je proměnná "@users" zadefinována, ovladač zavolá pohled (view) (krok 6). Proměnné, které začínají znakem "@", nazývané též "instanční proměnné", jsou automaticky k dispozici pohledům; v tomto případě pohled "index.html.erb" (soubor "app/views/users/index.html.erb") v následujícím výpisu iteruje skrz seznam "@users" a vypíše pro každého řádek HTML. (Pamatujme, že v současné chvíli není nutné celý kód chápat. Je vypsán hlavně pro ilustrační účely.)

```
<h1>Listing users</h1>

<table>
  <thead>
    <tr>
      <th>Name</th>
      <th>Email</th>
      <th colspan="3"></th>
    </tr>
  </thead>

<% @users.each do |user| %>
  <tr>
    <td><%= user.name %></td>
    <td><%= user.email %></td>
    <td><%= link_to 'Show', user %></td>
    <td><%= link_to 'Edit', edit_user_path(user) %></td>
    <td><%= link_to 'Destroy', user, method: :delete,
                                     data: { confirm: 'Are you sure?' } %></td>
  </tr>
<% end %>
</table>

<br>

<%= link_to 'New User', new_user_path %>
```

Pohled následně převede svůj obsah do HTML (krok 7), který je pak ovladačem vrácen prohlížeči ke zobrazení (krok 8).



###Pro a proti Users resource

I když je praktický pro obecné pochopeni Railsu, Users resource vytvořený skrz scaffolding má zásadní slabiny.

- **Žádná validace (ověřování) dat.** Náš User model přijímá i data, jako jsou prázdná jména nebo neplatné emailové adresy, aniž by mu to vadilo.
- **Žádná autentifikace.** Nemáme možnost přihlašování a odhlašování, a žádný mantinel pro to, co uživatel může a nemůže udělat.
- **Žádné stylování nebo rozvržení.** Nemáme žádné konzistentní stylování stránky, ani navigaci.
- **Žádné hlubší pochopení.** Pokud rozumíte kódu, ktery scaffolding vytvoří, zřejmě Vám tyto články nebudou příliš k užitku.



## Microposts resource

Poté, co jsme si úspěšně vygenerovali a prozkoumali Users resource, můžeme naši pozornost obrátit na související Microposts resource. V průběhu této sekce je rozumné porovnávat prvky Microposts resource s obdobnými prvky u uživatelů v sekci předchozí; můžeme si totiž všimnout mnoha vzájemných paralel. Struktura REST aplikací v Rails je nejlépe pochopena a vstřebána formou repetice, a správné pochopení podobnosti obou zdrojů (resources) je jedním z hlavních účelů této kapitoly.



### Prohlídka mikropostů

Stejně, jako u Users resource, nás čeká vygenerování kódu pro mikroposty skrze příkaz "rails generate scaffold", jen v tomto případě použijeme datový model právě pro mikroposty:

```
$ rails generate scaffold Micropost content:text user_id:integer
      invoke  active_record
      create    db/migrate/20160515211229_create_microposts.rb
      create    app/models/micropost.rb
      invoke    test_unit
      create      test/models/micropost_test.rb
      create      test/fixtures/microposts.yml
      invoke  resource_route
       route    resources :microposts
      invoke  scaffold_controller
      create    app/controllers/microposts_controller.rb
      invoke    erb
      create      app/views/microposts
      create      app/views/microposts/index.html.erb
      create      app/views/microposts/edit.html.erb
      create      app/views/microposts/show.html.erb
      create      app/views/microposts/new.html.erb
      create      app/views/microposts/_form.html.erb
      invoke    test_unit
      create      test/controllers/microposts_controller_test.rb
      invoke    helper
      create      app/helpers/microposts_helper.rb
      invoke      test_unit
      invoke    jbuilder
      create      app/views/microposts/index.json.jbuilder
      create      app/views/microposts/show.json.jbuilder
      invoke  assets
      invoke    coffee
      create      app/assets/javascripts/microposts.coffee
      invoke    scss
      create      app/assets/stylesheets/microposts.scss
      invoke  scss
   identical    app/assets/stylesheets/scaffolds.scss
```



Pro aktualizaci naší databáze novým datovým modelem zbývá jen obdobně spustit migraci:

```
$ rails db:migrate
==  CreateMicroposts: migrating ===============================================
-- create_table(:microposts)
   -> 0.0023s
==  CreateMicroposts: migrated (0.0026s) ======================================
```

Nyní můžeme vytvářet mikroposty stejným způsobem, jako uživatele v předchozí části. Scaffold generátor aktualizoval směrovací soubor Railsu pravidlem pro Microposts resource, jak je vidno z následujícího výpisu (soubor "config/routes.rb"). Jako v případě uživatelů, směrovací pravidlo "resources :microposts" mapuje jednotlivé URL k akcím ovladače mikropostů, viz tabulka níže.

```
Rails.application.routes.draw do
  resources :microposts
  resources :users
  root 'users#index'
end
```

| HTTP požadavek | **URL**            | **Akce**  | **Účel**                                |
| -------------- | ------------------ | --------- | --------------------------------------- |
| `GET`          | /microposts        | `index`   | stránka pro výpis všech mikropostů      |
| `GET`          | /microposts/1      | `show`    | stránka ke zobrazení mikropostu 1       |
| `GET`          | /microposts/new    | `new`     | stránka pro vytvoření nového mikropostu |
| `POST`         | /microposts        | `create`  | vytvoření nového mikropostu             |
| `GET`          | /microposts/1/edit | `edit`    | stránka pro úpravu mikropostu 1         |
| `PATCH`        | /microposts/1      | `update`  | aktualizace mikropostu 1                |
| `DELETE`       | /microposts/1      | `destroy` | smazání mikropostu 1                    |

Ovladač Microposts (soubor "app/controllers/microposts_controller.rb") máme opět vyobrazen níže. Můžeme si všimnout, že s výjimkou názvu třídy ("MicropostsController" namísto "UsersController") je identický s ovladačem Users resource - důvodem je právě použití REST architektury.

```
class MicropostsController < ApplicationController
  .
  .
  .
  def index
    .
    .
    .
  end

  def show
    .
    .
    .
  end

  def new
    .
    .
    .
  end

  def edit
    .
    .
    .
  end

  def create
    .
    .
    .
  end

  def update
    .
    .
    .
  end

  def destroy
    .
    .
    .
  end
end
```



Na příslušné stránce (/microposts/new) si vytvoříme pár mikropostů; opět stačí jednoduše zadat příslušné údaje (viz obr. 2.12).

// obr 2.12 Stránka pro vytváření mikropostů (/microposts/new).

Dbáme jen na to, aby uživatel 1 měl alespoň jeden mikropost. Výsledek by měl vypadat podobně, jako na obr. 2.13.

// obr 2.13 Stránka se seznamem mikropostů (/microposts). 

### Aby mikroposty byly opravdu "mikro"

Abychom dostáli jména mikropostů, budeme potřebovat nějaký způsob, kterým omezíme délku jednotlivých příspěvků. Implementace tohoto omezení je v Rails jednoduše řešitelná pomocí ověřování/validace ("validation"). Abychom omezili délku na maximálně 140 znaků (á la Twitter), použijeme ověření na délku ("length validation"). Otevřeme si tedy soubor "app/models/micropost.rb" v textovém editoru a naplníme ho následujícím obsahem:

```
class Micropost < ApplicationRecord
  validates :content, length: { maximum: 140 }
end
```

Kód se může jevit lehce záhadně - ostatně, o validacích si blíže povíme až později - ale jeho efekt je okamžite vidět; stačí si opět otevřít vkládání nových mikropostů a pokusit se vložit text delší, než 140 znaků. Jak je vidno z obr. 2.14, Rails vypíše chybovou zprávu, že je obsah příliš dlouhý.

// obr 2.14 Chybová zpráva při neúspěšném vytváření mikropostu.

### Kolik má uživatel mikropostů?

Jedna z nejužitečnějších dovedností Rails je schopnost tvořit asociace mezi různými datovými modely. V případě našeho User modelu může mít uživatel potenciálně mnoho mikropostů. Upravíme si tedy soubory "app/models/user.rb" a "app/models/micropost.rb" aby vypadaly takto:

"app/models/user.rb":

```
class User < ApplicationRecord
  has_many :microposts
end
```

"app/models/micropost.rb":

```
class Micropost < ApplicationRecord
  belongs_to :user
  validates :content, length: { maximum: 140 }
end
```

Díky sloupci "user_id" v tabulce "microposts", který jsme si právě pro účel asociace prve vytvořili, Rails (opět za pomocí Active Record) dokáže usoudit, které mikroposty jsou asociované se kterým uživatelem.

V budoucnu využijeme právě této asociace, abychom si vytvořili obdobu Twitter feedu, tedy v podstate seznamu postů od daného uživatele. Prozatím můžeme alespoň nahlédnout do provázanosti skrze Rails konzoli, kterou si jednoduše zavoláme v příkazovém řádku příkazem "rails console". Pak už si stačí jen vypsat prvního uživatele do proměnné použitím "User.first":

```
$ rails console
>> first_user = User.first
...
>> first_user.microposts
...
>> micropost = first_user.microposts.first
...
>> micropost.user
...
>> exit
```

Sérií příkazů jsme si první zpřístupnili mikroposty uživatele skrze "first_user.microposts". Active Record nám pak vrátil všechny mikroposty s hodnotou "user_id" rovnou hodnotě "first_user" (což je v našem případě 1).(Příkaz "exit" na posledním řádku je zahrnutý jen pro demonstraci, jak opustit konzoli. Ve většině systémů ale stejně tak funguje i zkratka Ctrl+D.)

### Hierarchie dědičnosti

Kapitolu naší zkušební aplikace uzavřeme stručným popisem hierarchií tříd ovladače a modelu v Railsu. Ti, kteří nemají alespoň nějakou zkušenost s objektově-orientovaným programováním (OOP), zejména ohledně tříd, nemusí věšet hlavu, protože se k tomuto tématu ještě podrobněji vrátíme v budoucnu.

Nejprve si popíšeme strukturu dědičnosti pro modely. Srovnáním následujících souborů ("app/models/user.rb" a "app/models/micropost.rb") zjistíme, že oba modely dědí (skrze ostrou závorku vlevo "<") z "ApplicationRecord", který zase dědí po "ActiveRecord::Base", což je základní třída pro modely, kterou poskytuje Active Record; diagram shrnující tento vztah je na obr. 2.18. Právě díky dědění od "ActiveRecord::Base" má náš model schopnost komunikovat s databází, pracovat s jejími sloupci jako s Ruby atributy, a podobně.

```
class User < ApplicationRecord
  .
  .
  .
end
```

```
class Micropost < ApplicationRecord
  .
  .
  .
end
```

// obr. 2.18 Hierarchie dědičnosti pro modely User a Micropost.

Struktura dědičnosti pro ovladače je v podstatě stejná, jako pro modely. Opět si můžeme porovnat soubory "app/controllers/users_controller.rb" a "app/controllers/microposts_controller.rb", kde je dobře vidět, že oba ovladače dědí po ovladači aplikací. 

```
class UsersController < ApplicationController
  .
  .
  .
end
```

```
class MicropostsController < ApplicationController
  .
  .
  .
end
```

Při dalším průzkumu zjistíme, že i samotný "ApplicationController" (soubor "app/controllers/application_controller.rb") dědí z "ActionController:Base", což je základní třída pro ovladače poskytnutá Railsovou knihovnou "Action Pack". 

```
class ApplicationController < ActionController::Base
  .
  .
  .
end
```

Jejich vztah je opět načrtnutý na obr. 2.19.

// obr. 2.19 Hierarchie dědičnosti pro ovladače User a Micropost.

Podobně jako u modelů, ovladače Users i Microposts ziskávají většinu své funkcionality děděním ze základních tříd (v tomto případě "ActionController::Base"), včetně schopnosti pracovat s objekty modelů, filtrování příchozích HTTP požadavků a vykreslování pohledů ve formě HTML. Jelikož všechny ovladače v Rails dědí z "ApplicationController", pravidla zadefinovaná v aplikačním ovladači se automaticky aplikují na všechny akce v aplikaci. V budoucnu díky tomu například přidáme pomocníky pro přihlašování a odhlašování.



## Závěry

Dostali jsme se k závěru prohlídky aplikace v Rails. Aplikace, kterou jsme si vyvinuli, má několik kladů, ale spoustu nedostatků.

**Pozitiva**

- Obecný přehled Railsu 
- Seznámení s MVC
- První ochutnávka architektury REST
- Začátky modelování dat
- Webová aplikace s podporou databáze

**Negativa**

- Žádné vlastní stylování nebo rozvržení
- Žádné statické stránky (jako "Domů")
- Žádná uživatelské hesla
- Žádné uživatelské obrázky
- Žádné přihlašování
- Žádná bezpečnost
- Žádná automatická asociace uživatelů a mikropostů
- Žádná možnost aktivního sledování uživatelů
- Žádné smysluplné testování
- Chybějící hlubší pochopení

Zbytek lekcí se budeme věnovat právě posilování kladů a odstraňování záporů.



### Co jsme se v kapitole naučili

- Scaffolding automaticky vytváří kód k práci s modely dat a jejich interakci skrze web.
- Scaffolding je dobrý pro rychlý začátek, ale nevhodný k hlubšímu pochopení fungování.
- Rails používá Model-View-Controller (MVC) vzor pro strukturování webových aplikací. 
- Architektura REST v Rails zahrnuje standardní sadu akcí pro ovladače a jejich interakci s datovými modely.
- Rails podporuje validaci dat a tím umožňuje omezovat vložené hodnoty atributů datového modelu.
- Rails má zabudované funkce pro definování asociací mezi ruznými datovými modely. 
- Můžeme s aplikací v Rails interagovat pomocí příkazového řádku skrze konzoli Rails. 

