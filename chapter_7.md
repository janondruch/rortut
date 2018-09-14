# Registrace

Jelikož už máme funkční uživatelský model, můžeme si přidat prvek, bez kterého se obejde jen málo stránek: registraci uživatelů. Použijeme HTML formulář k odeslání registračních informací naší aplikaci, které pak budou použity pro vytvoření nového uživatele a uložení jeho atributů do databáze. Na konci registračního procesu je důležité i přidat stránku profilu, kde budou informace o uživateli vypsány. Začneme ale paradoxně právě onou stránkou pro zobrazování uživatelů, jelikož bude sloužit jako první krok k implementaci REST architektury pro uživatele. Během toho si přidáme i nedílné integrační testy.

Zakládat budeme na validacích uživatelského modelu z předchozí kapitoly pro zvýšení šance, že noví uživatelé budou mít platnou emailovou adresu. Ke konci výuky přidáme i aktivaci účtu, abychom si mohli být platností jistí.

I když se snažíme věci pojmout jednoduše, ale zároveň profesionálně, nevyhneme se postupnému nárůstu obtížnosti témat. Zopakovat si někdy i celou kapitolu znovu tedy není žádná ostuda a může naopak jít o velmi praktické cvičení.

## Zobrazování uživatelů

V této části uděláme první kroky směrem k finálnímu profilu tím, že vytvoříme stránku ke zobrazení uživatelského jména a profilového obrázku (viz mockup na obr. 7.1). Našim konečným cílem ohledně profilových stránek bude zobrazení profilového obrázku uživatele, základních uživatelských informací a seznam mikropostů. K poslednímu zmiňovanému se ale dostaneme až úplně nakonec, čímž aplikaci i završíme.

Vytvoříme si tedy vedlejší větev na Gitu:

```
$ git checkout -b sign-up
```

// obr. 7.1 Mockup uživatelského profilu, který si teď vytvoříme.

// obr. 7.2 Mockup finálního vzhledu profilu.



### Debug v prostředí Rails

Profily v této sekci budou prvními skutečně dynamickými stránkami v naší aplikaci. I když bude pohled stále existovat jako jedna samostatná stránka kódu, každý profil bude upraven na základě informací získaných z databáze. V rámci přípravy na přidání dynamických stránek je teď dobrá chvíle i na přidání debugových (de-bug, metodika "odchytávání bugů", tedy vzniklých problémů během vývoje) informací do našeho layoutu ("app/views/layouts/application.html.erb"). Díky zabudované metodě "debug" a proměnné "params" jde o užitečný nástroj ke zjištění informací o každé stránce.

```
<!DOCTYPE html>
<html>
  .
  .
  .
  <body>
    <%= render 'layouts/header' %>
    <div class="container">
      <%= yield %>
      <%= render 'layouts/footer' %>
      <%= debug(params) if Rails.env.development? %>
    </div>
  </body>
</html>
```

Jelikož nechceme zobrazovat debugové informace uživatelům, použijeme

```
if Rails.env.development?
```

k zobrazení debugových informací pouze ve vývojovém prostředí, což je jedno ze tří prostředí, která Rails definuje. Ostatně, "Rails.env.development?" je "true" pouze v případě vývojového prostředí, takže zapouzdřené Ruby

```
<%= debug(params) if Rails.env.development? %>
```

nebude vloženo do produkčních aplikací nebo testů.

#### Prostředí v Rails

Rails má k dispozici tři prostředí: "test" (testovací), "development" (vývojové) a "production" (produkční, ostré nasazení). Základní prostředí pro konzoli Rails je "development":

```
  $ rails console
  Loading development environment
  >> Rails.env
  => "development"
  >> Rails.env.development?
  => true
  >> Rails.env.test?
  => false
```

Jak vidno, Rails poskytuje objektu "Rails" atribut "env" a z něj vyplývající booleanské metody, takže, například, "Rails.env.test?" vrací "true" pokud je prostředí testovací, ale jinak vrátí "false".

Když potřebujeme konzoli spustit v jiném prostředí (třeba v rámci debugu testu), lze odeslat prostředí jako parametr skriptu "console":

```
  $ rails console test
  Loading test environment
  >> Rails.env
  => "test"
  >> Rails.env.test?
  => true
```

Stejně jako u konzole je "development" základní prostředí pro server Rails, ale i ten lze spustit v jiném:

```
  $ rails server --environment production
```

Když si chceme prohlédnout aplikaci v produkčním módu, nebude fungovat bez produkční databáze, kterou můžeme vytvořit spuštěním "rails db:migrate" právě v produkčním módu:

```
  $ rails db:migrate RAILS_ENV=production
```

(Bohužel používá verze pro konzoli, server a migraci pokaždé jinou syntaxi, což je důvod, proč jsme si vypsali všechny tři varianty.)

Aby vypadal výpis debugu pěkně, přidáme mu pár pravidel do našich vlastních stylů, které jsme si dříve vytvořili ("app/assets/stylesheets/custom.scss"):

```
@import "bootstrap-sprockets";
@import "bootstrap";

/* mixins, variables, etc. */

$gray-medium-light: #eaeaea;

@mixin box_sizing {
  -moz-box-sizing:    border-box;
  -webkit-box-sizing: border-box;
  box-sizing:         border-box;
}
.
.
.
/* miscellaneous */

.debug_dump {
  clear: both;
  float: left;
  width: 100%;
  margin-top: 45px;
  @include box_sizing;
}
```

Tímto jsme si představili funkci Sass zvanou "mixin", v tomto případě zavolanou na "box_sizing". Mixin umožňuje zabalení skupiny CSS pravidel pro pozdější použití pro vícero prvků, což efektivně převede

```
.debug_dump {
  .
  .
  .
  @include box_sizing;
}
```

na

```
.debug_dump {
  .
  .
  .
  -moz-box-sizing:    border-box;
  -webkit-box-sizing: border-box;
  box-sizing:         border-box;
}
```

Výsledná nastylovaná stránka pak vypadá jako na obrázku (obr. 7.3).

// obr. 7.3 Domovská stránka s debug informacemi.

Výstup debugu nám dává některé potenciálně užitečné informace o způsobu vykreslení stránky:

```
---
controller: static_pages
action: home
```

Takto vypadá YAML ztvárnění "params", což je v podstatě haš, která v tomto případě identifikuje ovladač a akci pro stránku. Další příklad uvidíme záhy.



### Uživatelský zdroj (Users resource)

Abychom vytvořili stránku s profilem uživatele, musíme mít daného uživatele v databázi, což představuje oblíbený problém slepice-vejce: jak může mít stránka uživatele dříve, než funkční registraci? Tento problém jsme naštěstí už vyřesili: vytvořili jsme si přecejen uživatelský záznam ručně za pomocí konzole Rails, a ten by tedy měl být v databázi:

```
$ rails console
>> User.count
=> 1
>> User.first
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 20:36:46", updated_at: "2016-05-23 20:36:46",
password_digest: "$2a$10$xxucoRlMp06RLJSfWpZ8hO8Dt9AZXlGRi3usP3njQg3...">
```

Z výpisu konzole vidíme, že uživatel má id 1, a naším momentálním cílem je tvorba stránky pro zobrazení informací o tomto uživateli. Budeme následovat konvence architektury REST, které aplikace v Rails upřednostňují, což znamená představování dat jako "zdrojů", které mohou být vytvořeny, zobrazeny, aktualizovány nebo ničeny - čtyři akce, které korespondují se čtyřmi základními [HTTP operacemi](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) "POST", "GET", "PATCH" a "DELETE".

Když následujeme principy REST, na zdroje se obvykle odkazuje za použití jména zdroje a unikátního identifikátoru. V kontextu uživatelů (na které tedy budeme pohlížet jako na "uživatelský zdroj", nebo "zdroj uživatelů") to znamená, že uživatele s id 1 si prohlédneme odesláním GET požadavku na URL "/users/1". Akce "show" je implicitní v typu požadavku - jakmile jsou prvky REST v Rails aktivní, požadavky GET jsou automaticky obslouženy akcí "show".

Ze začátku jsme zjistili, že stránka pro uživatele s id 1 má URL "/users/1". Bohužel, pokud danou URL navštívíme teď, dočkáme se akorat chybového výpisu serveru (obr. 7.4).

// obr. 7.4 Výpis chyby serveru pro /users/1.

Chybějící směrování můžeme jednoduše přidat do našeho souboru s cestami ("config/routes.rb"):

```
resources :users
```

Výsledek vypadá následovně:

```
Rails.application.routes.draw do
  root 'static_pages#home'
  get  '/help',    to: 'static_pages#help'
  get  '/about',   to: 'static_pages#about'
  get  '/contact', to: 'static_pages#contact'
  get  '/signup',  to: 'users#new'
  resources :users
end
```

Ačkoliv je náš momentální cíl tvorba stránky pro zobrazení uživatelů, samotný řádek "resources :users" přidá nejen funkční URL "/users/1", ale umožní aplikaci použít všechny akce potřebné pro zRESTování uživatelského zdroje, spolu velkým množstvím jmenných cest pro generování uživatelských URL. Výsledná korespondence URL, akcí a jmených cest je zobrazena v tabulce. Postupně pokryjeme všechny pojmy v ní zmíněné.

| **HTTP požadavek** | **URL**       | **Akce**  | **Jmenná cesta**       | **Význam**                                          |
| ------------------ | ------------- | --------- | ---------------------- | --------------------------------------------------- |
| `GET`              | /users        | `index`   | `users_path`           | stránka pro zobrazení všech uživatelů               |
| `GET`              | /users/1      | `show`    | `user_path(user)`      | stránka pro zobrazení uživatele                     |
| `GET`              | /users/new    | `new`     | `new_user_path`        | stránka pro vytvoření nového uživatele (registrace) |
| `POST`             | /users        | `create`  | `users_path`           | vytvoření nového uživatele                          |
| `GET`              | /users/1/edit | `edit`    | `edit_user_path(user)` | stránka k úpravě uživatele s id 1                   |
| `PATCH`            | /users/1      | `update`  | `user_path(user)`      | update user aktualizace uživatele                   |
| `DELETE`           | /users/1      | `destroy` | `user_path(user)`      | smazání uživatele                                   |

Po úpravě směrovacího souboru cesty fungují, ale pořád tam neexistuje žádná stránka (viz obr. 7.5). Abychom to napravili, začneme minimalistickou verzí profilové stránky, kterou si záhy poněkud vylepšíme.

// obr 7.5 URL /users/1 se správným směrováním, ale beze stránky.

Začneme standardním Railsovým umístěním pro zobrazení uživatele, což je "app/views/users/show.html.erb". Narozdíl od pohledu "new.html.erb", který jsme si vytvořili generátorem dříve, soubor "show.html.erb" momentálně neexistuje a bude ho tedy třeba vytvořit a naplnit ručně.

```
<%= @user.name %>, <%= @user.email %>
```

Pohled používá zapouzdřené Ruby pro zobrazení uživatelského jména a emailové adresy, přičemž předpokládá existenci instanční proměnné "@user". Samozřejmě bude stránka nakonec vypadat jinak (a rozhodně nebude zobrazovat email veřejně).

Aby náš zobrazovací pohled fungoval, musíme si zadefinovat proměnnou "@user" v související "show" akci v ovladači uživatelů. Pro získání uživatele z databáze jednoduše použijeme metodu "find" na uživatelský model ("app/controllers/users_controller.rb").

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
  end
end
```

Použili jsme "params" pro získání id.  Když vytváříme příslušné požadavky na uživatelský ovladač, "params[:id]" bude uživatel s id 1, takže dopad je stejný jako u metody "find" v podobě "User.find(1)", kterou jsme si ukazovali dříve.

S definovanou akcí a uživatelským pohledem už URL "/users/1" funguje skvěle, jak vidno na obrázku (obr. 7.6). Debugová informace na něm mimojiné potvrzuje hodnotu "params[:id]".

```
---
action: show
controller: users
id: '1'
```

Proto kód

```
User.find(params[:id])
```

najde uživatele s id 1.

// obr. 7.6 Stránka pro zobrazení uživatelů po přidání uživatelského zdroje.

### Debugger

Jelikož nám, jak jsme si ukázali, debugové informace mohou hodně pomoci při snaze pochopit, co se v naší aplikaci děje, použijeme trochu přímočařejší cestu k jejich získání za pomocí gemu "byebug". Stačí přidat řádek se slovem "debugger" do naší aplikace, v tomto případě tedy do ovladače uživatelů ("app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
    debugger
  end

  def new
  end
end
```

Když teď navštívíme "/users/1", Railsový server zobrazí prompt "byebug":

```
(byebug)
```

Můžeme ho vnímat v podstatě jako konzoli Rails a zadávat příkazy, díky kterým zjistíme stav aplikace:

```
(byebug) @user.name
"Example User"
(byebug) @user.email
"example@railstutorial.org"
(byebug) params[:id]
"1"
```

Pro opuštění řádku a pokračování provedení aplikace stačí stisknout Ctrl-D, a následně odstranit "debugger" z akce "show":

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
  end
end
```

Kdykoliv bude něco v aplikaci matoucí, je velmi praktické umístit "debugger" blízko kódu, o kterém se domníváme, že způsobuje problémy. Zjišťování stavu systému skrze "byebug" je silná metoda pro vyhledávání aplikačních chyb a interaktivní odstraňování bugů.



### "Gravatar" a boční lišta

Když máme základní uživatelskou stránku zadefinovanou, trochu ji vyzdobíme profilovým obrázkem pro každého uživatele a nástřelem boční uživatelské lišty. Začneme "globálně uznávaným/rozeznávaným avatarem" ("globally recognized avatar", nebo zkráceně jen [Gravatar](http://gravatar.com/)). Gravatar je bezplatná služba, která umožňuje uživatelům nahrát obrázky a asociovat je s emailovou adresou, kterou disponují. Gravatar je tímpádem praktický způsob, kterým lze zahrnout uživatelský obrázek bez komplikací, které přichází spolu s nahráváním, ořezáváním a ukládáním obrázků. Potřebujeme jen vytvořit URL obrázku za pomocí uživatelovy emailové adresy a související obrázek Gravatar se automaticky objeví.

Chceme si tedy zadefinovat pomocníka "gravatar_for" pro vrácení Gravatar obrázku daného uživatele (soubor "app/views/users/show.html.erb"):

```
<% provide(:title, @user.name) %>
<h1>
  <%= gravatar_for @user %>
  <%= @user.name %>
</h1>
```

Za normálních okolností jsou metody definované v jakémkoliv pomocníkovi automaticky dostupné v jakémkoliv pohledu, ale z důvodu praktičnosti umístíme metodu "gravatar_for" do souboru pro pomocníky, který je asociovaný s ovladačem uživatelů. URL Gravataru je založená na [MD5 haši](http://en.wikipedia.org/wiki/MD5) uživatelovy emailové adresy. V Ruby se algoritmus MD5 implementuje skrze metodu "hexdigest", která je součástí knihovny "Digest":

```
>> email = "JOHNDOE@example.COM"
>> Digest::MD5::hexdigest(email.downcase)
=> "1fda4469bcbec3badf5418269ffc5968"
```

Jelikož nejsou emailové adresy narozdíl od MD5 haší citlivé na velikost písma, použili jsme metodu "downcase" pro ujištění, že parametr pro "hexdigest" bude v malých písmenech. Výsledný pomocník "gravatar_for" v souboru "app/helpers/users_helper.rb" pak bude vypadat následovně:

```
module UsersHelper

  # Returns the Gravatar for the given user.
  def gravatar_for(user)
    gravatar_id = Digest::MD5::hexdigest(user.email.downcase)
    gravatar_url = "https://secure.gravatar.com/avatar/#{gravatar_id}"
    image_tag(gravatar_url, alt: user.name, class: "gravatar")
  end
end
```

Kód nám vrátí tag obrázku pro Gravatar s CSS třídou "gravatar" a alternativním textem v podobě uživatelského jména.

Profilová stránka (obr. 7.7) už tedy obsahuje základní Gravatar obrázek, který vypadá tak, jak vypadá proto, že "user@example.com" není skutečná emailová adresa. (Když jí navštívíme, tak zjistíme, že celá doména "example.com" je určena právě pro tyto příklady.)

// obr. 7.7 Profilová stránka uživatele se základním Gravatarem.

Aby naše aplikace zobrazovala vlastního Gravatara, použijeme "update_attributes" pro změnu uživatelského emailu na nějaký, ke kterému máme přístup:

```
$ rails console
>> user = User.first
>> user.update_attributes(name: "Example User",
?>                        email: "example@railstutorial.org",
?>                        password: "foobar",
?>                        password_confirmation: "foobar")
=> true
```

Přiřadili jsme tedy uživateli adresu, u které je nastavené logo tutorialu Rails (obr. 7.8).

// obr. 7.8 Uživatelská stránka s vlastním Gravatarem.

Poslední potřebný prvek, který potřebujeme přidat, aby náš výtvor alespoň připomínal mockup, který jsme si dali jako předlohu na začátku kapitoly, je prvotní verze boční lišty (sidebar). Implementujeme ji skrze tag "aside", který se používá právě pro obsah, který doprovází zbytek stránky, ale zároveň může "existovat sám o sobě". Zahrneme třídy "row" a "col-md-4", oboje částí Bootstrapu. Kód upravené verze uživatelské stránky (soubor "app/views/users/show.html.erb") vypadá následovně:

```
<% provide(:title, @user.name) %>
<div class="row">
  <aside class="col-md-4">
    <section class="user_info">
      <h1>
        <%= gravatar_for @user %>
        <%= @user.name %>
      </h1>
    </section>
  </aside>
</div>
```

S HTML prvky a CSS třídami na svých místech můžeme stránku nastylovat (včetně lišty a Gravataru) pomocí SCSS z následujícího výpisu (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* sidebar */

aside {
  section.user_info {
    margin-top: 20px;
  }
  section {
    padding: 10px 0;
    margin-top: 20px;
    &:first-child {
      border: 0;
      padding-top: 0;
    }
    span {
      display: block;
      margin-bottom: 3px;
      line-height: 1;
    }
    h1 {
      font-size: 1.4em;
      text-align: left;
      letter-spacing: -1px;
      margin-bottom: 3px;
      margin-top: 0px;
    }
  }
}

.gravatar {
  float: left;
  margin-right: 10px;
}

.gravatar_edit {
  margin-top: 15px;
}
```

// obr. 7.9 Uživatelská stránka s boční lištou a CSS.



## Registrační formulář

Po vytvoření funkční (i když zatím neúplné) stránky uživatelského profilu je na čase vytvořit registrační formulář pro naší stránku. Registrační stránka je momentálně prázdná, což pro registraci nových uživatelů není moc praktické. V této části si tedy vytvoříme registrační formulář podle mockupu (obr. 7.11).



// obr. 7.11 Mockup registrační stránky.



### Použití "form_for"

Srdcem registrační stránky je formulář pro odeslání relevantních informací (jméno, email, heslo a jeho potvrzení). Pomůže nám metoda pomocníka "form_for", která vezme objekt Active Record a vybuduje formulář na základě jeho atributů.

Registrační stránka "/signup" je směrována na akci "new" v uživatelském ovladači, a náš první krok bude tedy vytvoření uživatelského objektu, který bude potřeba jako parametr pro "form_for". Výsledná definice proměnné "@user" vypadá následovně (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end
end
```

Formulář jako takový (soubor "app/views/users/new.html.erb") vypadá následovně:

```
<% provide(:title, 'Sign up') %>
<h1>Sign up</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user) do |f| %>
      <%= f.label :name %>
      <%= f.text_field :name %>

      <%= f.label :email %>
      <%= f.email_field :email %>

      <%= f.label :password %>
      <%= f.password_field :password %>

      <%= f.label :password_confirmation, "Confirmation" %>
      <%= f.password_field :password_confirmation %>

      <%= f.submit "Create my account", class: "btn btn-primary" %>
    <% end %>
  </div>
</div>
```

Probereme si ho záhy podrobněji, ale zatím si ho trochu nastylujeme skrze SCSS (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* forms */

input, textarea, select, .uneditable-input {
  border: 1px solid #bbb;
  width: 100%;
  margin-bottom: 15px;
  @include box_sizing;
}

input {
  height: auto !important;
}
```

Jakmile jsou pravidla aplikována, registrační stránka vypadá jako na obrázku (obr. 7.12).

// obr. 7.12 Registrační formulář.

### HTML registračního formuláře

Abychom lépe pochopili formulář, který jsme si vytvořili, je praktické ho rozdělit do menších částí. Podívejme se první na vnější strukturu, která sestává ze zapouzdřeného Ruby s voláním "form_for":

```
<%= form_for(@user) do |f| %>
  .
  .
  .
<% end %>
```

Přítomnost slova "do" naznačuje, že "form_for" vezme blok s jednou proměnnou, kterou jsme nazvali "f" (jako "form", tedy formulář).

Jak je u pomocníků obvyklé, nemusíme vědět podrobnosti ohledně jejich implementace, ale potřebujeme vědět co objekt "f" dělá: když je zavolán metodou související s [prvkem HTML form](http://www.w3schools.com/html/html_forms.asp), jako je textové pole, radio tlačítko nebo pole pro heslo, "f" vrací kód pro daný element specificky upravený tak, aby v něm byla nastavena hodnota "@user" objektu. Jinak řečeno,

```
<%= f.label :name %>
<%= f.text_field :name %>
```

vytvoří HTML, které je potřebné pro vytvoření prvku označeného textového pole pro nastavení atributu "name" uživatelského modelu.

Pokud si prohlédneme vygenerované HTML v prohlížeči skrze funkci "prozkoumání prvku", zdrojový kód bude vypadat nějak takto:

```
<form accept-charset="UTF-8" action="/users" class="new_user"
      id="new_user" method="post">
  <input name="utf8" type="hidden" value="&#x2713;" />
  <input name="authenticity_token" type="hidden"
         value="NNb6+J/j46LcrgYUC60wQ2titMuJQ5lLqyAbnbAUkdo=" />
  <label for="user_name">Name</label>
  <input id="user_name" name="user[name]" type="text" />

  <label for="user_email">Email</label>
  <input id="user_email" name="user[email]" type="email" />

  <label for="user_password">Password</label>
  <input id="user_password" name="user[password]"
         type="password" />

  <label for="user_password_confirmation">Confirmation</label>
  <input id="user_password_confirmation"
         name="user[password_confirmation]" type="password" />

  <input class="btn btn-primary" name="commit" type="submit"
         value="Create my account" />
</form>
```

Začneme interní strukturou dokumentu. Když si porovnáme předchozí výpisy tak zjistíme, že zapouzdřené Ruby

```
<%= f.label :name %>
<%= f.text_field :name %>
```

vyprodukuje HTML

```
<label for="user_name">Name</label>
<input id="user_name" name="user[name]" type="text" />
```

a kód

```
<%= f.label :email %>
<%= f.email_field :email %>
```

vyprodukuje HTML

```
<label for="user_email">Email</label>
<input id="user_email" name="user[email]" type="email" />
```

a

```
<%= f.label :password %>
<%= f.password_field :password %>
```

vyprodukuje HTML

```
<label for="user_password">Password</label>
<input id="user_password" name="user[password]" type="password" />
```

Jak je vidět na obrázku (obr. 7.13), textová a emailová pole (type="text" a type="email") jednoduše vypíšou svůj obsah, kdežto pole pro heslo (type="password") zahaluje vstup z důvodu bezpečnosti.

// obr. 7.13 Naplněný formulář s poli pro text a heslo.

Jak uvidíme později, klíč pro vytvoření uživatele je speciální atribut "name" v každém tagu "input":

```
<input id="user_name" name="user[name]" - - - />
.
.
.
<input id="user_password" name="user[password]" - - - />
```

Tyto "name" hodnoty umožňují Railsu vytvořit inicializační haši (skrze proměnnou "params") pro vytvoření uživatelů za pomocí hodnot zadaných uživatelem.

Druhý důležitý prvek je samotný tag "form". Rails ho vytvoří pomocí objektu "@user": jelikož každý Ruby objekt zná svou třídu, Rails vyhodnotí, že "@user" je třídy "User"; navíc, jelikož je "@user" nový uživatel, Rails ví, že má vytvořit formulář s metodou "post", což je vhodné sloveso pro vytváření nového objektu:

```
<form action="/users" class="new_user" id="new_user" method="post">
```

Zde jsou atributy "class" a "id" vcelku irelevantní; podstatné je "action="/users"" a "method="post"". Společně dodají instrukce k provedení HTTP požadavku POST na URL /users.



## Neúspěšné registrace

I když jsme si letmo prošli HTML formuláře, nepokryli jsme nic příliš do hloubky, a pro pochopení formuláře nejlépe poslouží kontext selhání registrace. V této části si vytvoříme přihlašovací formulář, který přijímá neplatné vstupy a překreslí registrační stránku se seznamem chyb, které je třeba opravit (viz mockup na obr. 7.14).

Although we’ve briefly examined the HTML for the form in [Figure 7.12](https://www.railstutorial.org/book/sign_up#fig-signup_form) (shown in [Listing 7.17](https://www.railstutorial.org/book/sign_up#code-signup_form_html)), we haven’t yet covered any details, and the form is best understood in the context of *signup failure*. In this section, we’ll create a signup form that accepts an invalid  submission and re-renders the signup page with a list of errors, as  mocked up in [Figure 7.14](https://www.railstutorial.org/book/sign_up#fig-signup_failure_mockup). 

// obr. 7.14 Mockup selhání registrace.

### Funkční formulář

Jak jsme probírali dříve, přidání "resources :users" do souboru "routes.rb" automaticky zajišťuje, že aplikace reaguje na zRESTované URL. Konkrétněji, zajišťuje, že požadavek POST na /users je obsloužen akcí "create". Náš postup pro akci "create" bude spočívat v tom, že použijeme odeslání formuláře pro vytvoření nového uživatelského objektu skrze "User.new", pokusíme se jej uložit a znovu vykreslíme registrační stránku pro případné znovuposlání. Začneme prohlídkou kódu pro registrační formulář:

```
<form action="/users" class="new_user" id="new_user" method="post">
```

Kód odešle POST požadavek na URL /users.

Naším prvníém krokem k funkční registraci bude přidání následujícího kódu (soubor "app/controllers/users_controller.rb"). Zahrnuje druhé použití metody "render", kterou jsme dříve viděli v kontextu partials; jak vidno, "render" funguje i v akcích ovladače. Také jsme zavedli větvící strukturu "if-else", která nám umožňuje ošetřovat případy selhání a úspěchu podle hodnoty "@user.save", která je "true" nebo "false", podle toho, zda uložení uspělo nebo ne.

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(params[:user])    # Not the final implementation!
    if @user.save
      # Handle a successful save.
    else
      render 'new'
    end
  end
end
```

Nejde o konečnou implementaci, ale je to dost na to, abychom se posunuli dále. Nejlepší metodou pro pochopení kódu je odeslat formulář s nějakou formou dat, která nejsou validní. Výsledek je vidět na obrázku (obr. 7.15) a plné debugové informace následují na druhém (obr. 7.16).

// obr. 7.15 Selhání registrace

// obr. 7.16 Debugové informace selhání registrace.

Abychom lépe pochopili, jak Rails zpracovává odeslání, podívejme se blíž na část "user" z haše debugových informací:

```
"user" => { "name" => "Foo Bar",
            "email" => "foo@invalid",
            "password" => "[FILTERED]",
            "password_confirmation" => "[FILTERED]"
          }
```

Tato haš je předána uživatelskému ovladači jakožto součást "params", a ta obsahuje informace o každém požadavku. V případě URL jako /users/1 je hodnota "params[:id]" "id" souvisejícího uživatele (v tomto případě tedy 1). V případě odeslání registračnímu formuláři "params" obsahuje "haši haší", což je také konstrukce, se kterou jsme se seznámili dříve. Debugové informace výše nám napoví, že odeslání formuláře má za výsledek haš "user" s atributy odpovídajícími odeslaným hodnotám, kde jsou klíče odvozeny od atributů "name" tagů "input". Například, hodnota

```
<input id="user_email" name="user[email]" type="email" />
```

se jménem "user[email]" je přesně ten atribut "email" haše "user".

I když se hašové klíče jeví jako řetězce ve výstupu debugu, můžeme k nim přistoupit v ovladači uživatelů jako k symbolům, takže "params[:user]" je haš uživatelských atributů, dokonce přesně těch, které potřebujeme jako parametry pro "User.new". Tedy řádek

```
@user = User.new(params[:user])
```

je nejvíce blízký

```
@user = User.new(name: "Foo Bar", email: "foo@invalid",
                 password: "foo", password_confirmation: "bar")
```

### Silné parametry

Dříve jsme si zmínili myšlenku tzv. masového přiřazování, která zahrnuje inicializaci proměnné Ruby za pomocí haše hodnot, jako třeba v

```
@user = User.new(params[:user])    # Nejde o konecnou implementaci!
```

Komentář naznačuje, že nejde o končnou implementaci. Důvodem je, že inicializování celé haše "params" je extrémně nebezpečné - předá "User.new" všechna data, která uživatel odeslal. Představme si, že jako dodatek k současným atributům by uživatelský model zahrnoval atribut "admin" určený k identifikaci administračních uživatelů stránky. (Což bude něco, co si později naimplementujeme.) Cesta k nastavení takového atributu na "true" je odeslání "admin='1'" jako součást "params[:user]", což je něco, co lze snadno zvládnout v rámci příkazového řádku HTTP klienta, jako třeba "curl". Výsledek by byl, že bychom dovolili jakémukoliv uzivateli stránky získat administrační oprávnění jen tím, že by do webového požadavku přidal "admin='1'".

Předchozí verze Ruby používaly metodu "attr_accessible" ve vrstvě modelu pro vyřešení takového problému, a ve starších aplikacích je to často k vidění. Od verze 4.0 je preferovanou technikou použití takzvaných "silných" parametrů ve vrstvě ovladače. Ta nám umožní specifikovat, které parametry jsou potřeba, a které jsou povolené. Navíc, odeslání takovéto haše "params" způsobí chybu, takže současné aplikace v Rails jsou imunní vůči slabinám souvisejícím s masovým přiřazováním.

V současné chvíli chceme, ať haš "params" potřebuje atribut ":user", a chceme mít atributy "name", "email", "password" a "password confirmation" nastavené jako povolené (ale žádné jiné). Toho můžeme dosáhnout následovně:

```
params.require(:user).permit(:name, :email, :password, :password_confirmation)
```

Tento kód vrací verzi haše "params" jen s povolenými atributy (a zahlášením chyby, když atribut ":user" chybí).

Abychom daných parametrů využili, je běžné použít vnější metodu zvanou "user_params" (která vrátí příslušnou inicializační haši) a použít jí namísto "params[:user]":

```
@user = User.new(user_params)
```

Jelikož bude "user_params" použito jen interně ovladačem Rails a není třeba ho odhalovat externím uživatelům skrze web, upravíme ho na soukromý použitím klíčového slova "private" (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      # Zpracovani uspesneho ulozeni.
    else
      render 'new'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

Mimochodem, extra odsazení metody "user_params" je určeno k lepšímu odlišení, které metody jsou definovány po "private".

V této fázi registrační formulář funguje, alespoň tedy v tom smyslu, že po odeslání nevyplivne chybu. Na druhou stranu, jak vidíme na obrázku (obr. 7.17), nevypíše žádnou zpětnou vazbu v případě neplatných odeslání (s výjimkou vývojářského debugu), což může být matoucí. A také vlastně reálně nevytvoří nového uživatele. Oba problémy si teď postupně vyřešíme.

// obr. 7.17 Registrační formulař odeslaný s neplatnými informacemi.

### Chybové zprávy registrace

Jako poslední krok v rámci ošetření neúspěšného vytvoření uživatele přidáme pomocné chybové zprávy, aby uživatel věděl, proč registrace selhala. Rails užitečně takové zprávy poskytuje na základě validací uživatelského modelu. Kupříkladu, pokud se snažíme vytvořit uživatele s neplatným emailem a krátkým heslem, bude vypadat situace následovně:

```
$ rails console
>> user = User.new(name: "Foo Bar", email: "foo@invalid",
?>                 password: "dude", password_confirmation: "dude")
>> user.save
=> false
>> user.errors.full_messages
=> ["Email is invalid", "Password is too short (minimum is 6 characters)"]
```

Objekt "errors.full_messages" obsahuje pole chybových hlášek.

Podobně jako ve výpisu konzole výše nám teď neúspěšné uložení generuje seznam chybových hlášek asociovaných s objektem "@user". Abychom je zobrazili v prohlížeči, použijeme partial na uživatelskou stránku "new" (soubor "app/views/users/new.html.erb"), přičemž přidáme CSS třídu "form-control" (kvůli Bootstrapu) na každé vstupní pole.

```
<% provide(:title, 'Sign up') %>
<h1>Registrace</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user) do |f| %>
      <%= render 'shared/error_messages' %>

      <%= f.label :name %>
      <%= f.text_field :name, class: 'form-control' %>

      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.label :password_confirmation, "Confirmation" %>
      <%= f.password_field :password_confirmation, class: 'form-control' %>

      <%= f.submit "Create my account", class: "btn btn-primary" %>
    <% end %>
  </div>
</div>
```

Použili jsme "render" partial nazvaný "shared/error_messages", což odráží běžný Railsový postup použití dedikované "shared/" složky pro partials, u kterých se předpokládá použití napříč větším množstvím ovladačů. Vytvoříme si tedy novou složku "app/views/shared" pomocí příkazu "mkdir":

```
$ mkdir app/views/shared
```

Poté bude v dané složce třeba vytvořit partial "_error_messages.html.erb" za pomocí, jako obvykle, textového editoru. Jeho obsah bude vypadat následovně:

```
<% if @user.errors.any? %>
  <div id="error_explanation">
    <div class="alert alert-danger">
      The form contains <%= pluralize(@user.errors.count, "error") %>.
    </div>
    <ul>
    <% @user.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```

Partial nám mimojiné ukáže i použití některých nových technik v Ruby i Rails, včetně dvou metod pro chybové objekty Rails. První metoda je "count", která vrátí počet chyb:

```
>> user.errors.count
=> 2
```

Další metoda je "any?", která (společne s "empty?") je jednou z párových souvisejících metod:

```
>> user.errors.empty?
=> false
>> user.errors.any?
=> true
```

Metoda "empty?", kterou jsme doposud viděli pouze v kontextu řetězců, funguje i na chybové objekty Rails, přičemž vrací "true" pro prázdný objekt a "false" pro neprázdný. Metoda "any?" je opakem "empty?", takže vrací "true" pokud jsou nějaké prvky přítomny, a "false" pokud nejsou.

Další nový prvek je textový pomocník "pluralize", který je k dispozici v konzoli skrze objekt "helper":

```
>> helper.pluralize(1, "error")
=> "1 error"
>> helper.pluralize(5, "error")
=> "5 errors"
```

Jak můžeme vidět, "pluralize" vezme celočíselný parametr a vrátí číslo s vhodně pluralizovanou verzí (množným číslem) druhého parametru.

```
>> helper.pluralize(2, "woman")
=> "2 women"
>> helper.pluralize(3, "erratum")
=> "3 errata"
```

Díky použití "pluralize" bude kód

```
<%= pluralize(@user.errors.count, "error") %>
```

vracet "0 errors", "1 error", "2 errors" a podobně, podle množství přítomných chyb, přičemž zachová gramatickou správnost množného čísla (jako "1 errors").

Použili jsme i CSS id "error_explanation" pro účely stylování chybových zpráv. Navíc, po každém neplatném odeslání Rails automaticky obalí pole s chybami do "div"ů s CSS třidou "field_with_errors". Tyto označení nám pak umožní chybové zprávy nastylovat v SCSS, které si tedy upravíme následovně (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* forms */
.
.
.
#error_explanation {
  color: red;
  ul {
    color: red;
    margin: 0 0 30px 0;
  }
}

.field_with_errors {
  @extend .has-error;
  .form-control {
    color: $state-danger-text;
  }
}
```

Díky předchozím úpravám kódu i SCSS se nyní budou zobrazovat užitečné chybové hlášky při pokusech odeslat nesprávně vyplněný formulář (obr. 7.18). Jelikož jsou zprávy generovány validacemi modelu, automaticky se změní podle jeho případných úprav (například délky hesla). 

// obr. 7.18 Selhání registrace s chybovými hláškami.

### Test pro neplatné odeslání

Než byly k dispozici webové frameworky s automatickými možnostmi testování, museli vývojáři testovat formuláře ručně. Například, pro testování registrace bylo zapotřebí navštívit stránku v prohlížeči a odeslat jak platná, tak neplatná data, pro ověření, že chování aplikace je v obou případech správné. Nemluvě o tom, že v případě změn aplikace bylo třeba obdobným ručním testováním procházet znova, což bylo jak frustrující, tak náchylné k chybám.

Naštěstí můžeme v Rails napsat testy pro automatické testování formulářů. Napíšeme si tedy jeden pro ověření chování po odeslání neplatných informací, a později si přidáme i obdobný pro ověření správně vyplněného testu.

V prvé řadě tedy vygenerujeme soubor s integračním testem pro registraci uživatelů, který nazveme "users_signup":

```
$ rails generate integration_test users_signup
      invoke  test_unit
      create    test/integration/users_signup_test.rb
```

Hlavní účel našeho testu je ověření, že kliknutí na registrační tlačítko nevytvoří nového uživatele, když jsou zadané informace neplatné. Postup spočívá ve zjištění počtu uživatelů, takže naše testy použijí metodu "count", která je k dispozici v každé třídě Active Record, včetně "User":

```
$ rails console
>> User.count
=> 1
```

Použijeme "assert_select" pro otestování HTML prvků relevantních stránek, přičemž se zaměříme pouze na prvky, u kterých je nepravděpodobné, že by se v budoucnu změnily.

Začneme navštívením registrační cesty pomocí "get":

```
get signup_path
```

Abychom otestovali odeslání formuláře, musíme odeslat požadavek "POST" na "users_path", což můžeme udělat skrze funkci "post":

```
assert_no_difference 'User.count' do
  post users_path, params: { user: { name:  "",
                                     email: "user@invalid",
                                     password:              "foo",
                                     password_confirmation: "bar" } }
end
```

Použili jsme haši "params[:user]", kterou předpokládá "User.new" v akci "create". Zabalením "post" do metody "assert_no_difference" s parametrem řetězce "User.count" jsme si požádali o srovnání mezi "User.count" před a po obsahu bloku "assert_no_difference". Je to v podstatě ekvivalentní k uložení počtu uživatelů, odeslání dat a ověření, že je počet stejný:

```
before_count = User.count
post users_path, ...
after_count  = User.count
assert_equal before_count, after_count
```

Ač jsou oba postupy stejné výsledkem, použití "assert_no_difference" je čistší a "správnější" z hlediska použití Ruby.

Výsledný soubor ("test/integration/users_signup_test.rb"), kde jsme mimojiné použili i zavolání "assert_template" pro ověření, že neúspěšná registrace znovu vykreslí akci "new", pak tedy bude vypadat následovně:

```
require 'test_helper'

class UsersSignupTest < ActionDispatch::IntegrationTest

  test "invalid signup information" do
    get signup_path
    assert_no_difference 'User.count' do
      post users_path, params: { user: { name:  "",
                                         email: "user@invalid",
                                         password:              "foo",
                                         password_confirmation: "bar" } }
    end
    assert_template 'users/new'
  end
end
```

Aplikační kód jsme napsali před integračním testem, takže výsledek by měl být zelený:

```
$ rails test
```



## Úspěšná registrace

Když už máme ošetřeny neplatné odeslání formuláře, můžeme registraci dokončit uložením nového (platného) uživatele do databáze. Nejprve se pokusíme uživatele uložit; pokud uložení uspěje, uživatelovy informace se zapíšou do databáze automaticky, a pak přesměrujeme prohlížeč na zobrazení uživatelova profilu, jak je vidět na mockupu (obr. 7.19). Pokud selže, vrátíme se v rámci chování z předchozí části.

// obr. 7.19 Mockup úspěšné registrace.

### Dokončený registrační formulář

Pro dokončení funkčního registračního formuláře musíme naplnit zakomentovanou sekci z minula žádoucím chováním. V současnou chvíli formulář při odeslání platných informací pouze zamrzne, jak je vidět na obrázku (obr. 7.20), kde se změní barva tlačítka pro odeslání. Je tomu tak proto, že základní chování Rails je vykreslit související pohled, a žádnou šablonu nemáme pro "create" akci vytvořenou (obr. 7.21).

// obr. 7.20 Zamrzlá stránka při úspěšném odeslání.

// obr. 7.21 Chybějící šablona ve výpisu serveru.

I když je možné vykreslit šablonu pro akci "create", obvyklý postup je přesměrování na jinou stránku v případě, že je vytvoření úspěšné. Budeme se tedy držet zaběhlého zvyku přesměrování na uživatelův profil poté, co se registrace úspěšne provede, i když by šlo přesměrovat i na kořenovou adresu. Aplikační kód (soubor "app/controllers/users_controller.rb"), který používá metodu "redirect_to" vypadá následovně:

```
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      redirect_to @user
    else
      render 'new'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

Použitá syntaxe je

```
redirect_to @user
```

ačkoliv jsme stejně dobře mohli použít i ekvivalentní

```
redirect_to user_url(@user)
```

Děje se tak proto, že Rails automaticky předpokládá, že když jsme zadali "redirect_to @user", chceme přesměrovat na "user_url(@user)".

### Flash

I když teď naše registrace technicky vzato funguje, přidáme trochu pozlátka, které je v rámci webových aplikací běžné: zpráva, která se objeví na následující stránce (tedy přivítání nového uživatele), a která následně zmizí po refreshi nebo navštívení další stránky.

Railsový způsob pro zobrazování dočasných zpráv je použití speciální metody zvané "flash", ke které se můžeme chovat jako k haši. Rails používá klíč ":success" pro zprávu, která značí úspěšný výsledek (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      flash[:success] = "Vítejte v naší aplikaci!"
      redirect_to @user
    else
      render 'new'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

Díky přiřazení zprávy do "flash" nyní můžeme zobrazit zprávu na první stránce hned po přesměrování. Postupem bude procházení haše "flash" a vypsání všech relevantních zpráv do layoutu stránky. 

Zobrazení celého obsahu "flash" do stránky pak bude vypadat nějak takto:

```
<% flash.each do |message_type, message| %>
  <div class="alert alert-<%= message_type %>"><%= message %></div>
<% end %>
```

Vnořené Ruby

```
alert-<%= message_type %>
```

vytváří CSS třídu, která souvisí s typem zprávy, takže pro ":success" zprávu bude třída vypadat takto:

```
alert-success
```

(I když je ":success" symbol, vnořené Ruby ho automaticky převede do řetězce "success" předtím, než ho vloží do šablony.) Použití ruzných tříd pro klíče nám umožňuje aplikovat různé styly pro různé druhy zpráv (v budoucnu například "flash[:danger]" pro upozornění na neúspěšné přihlášení). Bootstrapové CSS podporuje stylování pro čtyři postupně urgentnější typy zpráv ("success", "info", "warning" a "danger"), a v rámci vývoje se postupně dostaneme ke všem.

Jelikož je zpráva vložena i do šablony, celý HTML výsledek pro

```
flash[:success] = "Vítejte v naší aplikaci!"
```

vypadá následovně:

```
<div class="alert alert-success">Vítejte v naší aplikaci!</div>
```

Vložení vnořeného Ruby zmíněného výše do layoutu pak vypadá takto (soubor "app/views/layouts/application.html.erb"):

```
<!DOCTYPE html>
<html>
  .
  .
  .
  <body>
    <%= render 'layouts/header' %>
    <div class="container">
      <% flash.each do |message_type, message| %>
        <div class="alert alert-<%= message_type %>"><%= message %></div>
      <% end %>
      <%= yield %>
      <%= render 'layouts/footer' %>
      <%= debug(params) if Rails.env.development? %>
    </div>
    .
    .
    .
  </body>
</html>
```

### První registrace

Výsledek našeho snažení si můžeme konečně užít ve formě prvního uživatele naší aplikace. I když předchozí odeslání nefungovaly, jak měly, řádek "user.save" v ovladači uživatelů pracoval, jak měl, takže uživatelé mohli být vytvořeni. Abychom je vyčistili, resetujeme databázi následujícím příkazem:

```
$ rails db:migrate:reset
```

Případně bude nutné restartovat webserver (pomocí Ctrl-C).

Vytvoříme prvního uživatele se jménem Rails Tutorial a emailovou adresou "example@railstutorial.org" (viz obr. 7.22). Výsledná stránka (obr. 7.23) nám demonstruje "přátelskou" zprávu po úspěšné registraci, včetně pěkného zeleného stylování třídy "success", které nám dodalo Bootstrapové CSS. Po znovunačtení stránky zpráva zmizí, jak jsme si prve slíbili (obr. 7.24).

// obr. 7.22 Vyplňování informací první registrace.

// obr. 7.23 Výsledek úspěšné registrace s flash zprávou.

// obr. 7.24 Profilová stránka po znovunačtení bez flash zprávy.

### Test pro platné odeslání

Než postoupíme dále, napíšeme si test pro platné odeslání, abychom si ověřili chování naší aplikace. Podobně jako s testem pro neúspěšné odeslání si i teď ověříme obsah databáze. V tomto případě chceme odeslat platné informace a pak ověřit, že uživatel byl vytvořen. V analogii s předchozím testem, kdy jsme použili

```
assert_no_difference 'User.count' do
  post users_path, ...
end
```

tentokrát použijeme související metodu "assert_difference":

```
assert_difference 'User.count', 1 do
  post users_path, ...
end
```

Stejně jako u "assert_no_difference", první parametr je řetězec "User.count", který porovnáme před blokem "assert_difference" a po něm. Druhý, nepovinný parametr specifikuje velikost rozdílu (v tomto případě tedy 1).

Zahrnutí "assert_difference" do souboru "test/integration/users_signup_test.rb" bude tedy vypadat následovně. Použili jsme mimojiné metodu "follow_redirect!" po odeslání postu uživatelské cestě. Výsledkem je následování přesměrování po odeslání, a tedy vykreslením šablony "users/show".

```
require 'test_helper'

class UsersSignupTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "valid signup information" do
    get signup_path
    assert_difference 'User.count', 1 do
      post users_path, params: { user: { name:  "Example User",
                                         email: "user@example.com",
                                         password:              "password",
                                         password_confirmation: "password" } }
    end
    follow_redirect!
    assert_template 'users/show'
  end
end
```

Ověřujeme také, že po úspěšné registraci následuje vykreslení šablony. Aby test fungoval, je nezbytné, aby uživatelské cesty, uživatelská "show" akce a pohled "show.html.erb" byly v pořádku. Pak je i jediný řádek

```
assert_template 'users/show'
```

citlivý test na prakticky vše, co se týká uživatelské profilové stránky. Velikost pokrytí je právě jedním z důležitých prvků a dobře ilustruje výhody integračních testů jako takových.



## Nasazení na profesionální úrovni



# HEROKU A PODOBNE, SSL, DEPLOY SPOLU S HTTPS, PROBRAT JESTLI TO VUBEC ZAHRNEME



## Shrnutí

Možnost registrovat uživatele je velký milník naší aplikace. I když aplikace jako taková stále nedělá nic užitečného, položili jsme důležité základy pro veškerý budoucí vývoj. V dalších kapitolách dokončíme autentifikaci skrze možnost se přihlásit a odhlásit z aplikace (s volitelným "pamatuj si mě"). Následně umožníme uživatelům upravovat informace o svém účtu a také dáme administrátorům možnost uživatele mazat.

### Co jsme se naučili

- Rails zobrazuje užitečné informace skrze metodu "debug".
- Mixins v Sass umožnují seskupovat pravidla CSS a použít je na více místech.
- Rails má k dispozici tři standardní prostředí: "development" (vývojové), "test" (testovací) a "production" (produkční).
- Můžeme interagovat s uživateli jako se zdrojem (resource) skrze standardní sadu REST URL.
- Gravatar je praktická věc pro zobrazování obrázku k reprezentaci uživatelů.
- Pomocník "form_for" se používá ke generování formulářů, které interagují s objekty Active Record.
- Selhání registrace vykreslí novou uživatelskou stránku a zobrazí chybové hlášky, které jsou automaticky odvozeny díky Active Record.
- Rails poskytuje "flash" jako standardní způsob ke zobrazení dočasných zpráv.
- Úspěšná registrace vytvoří uživatele v databázi, přesměruje ho na jeho stránku a zobrazí uvítací zprávu.
- Integrační testy můžeme použít k ověření chování aplikace po odeslání formuláře a k odchytu nežádoucích situací.



