# Uživatelské mikroposty

V rámci tvorby jádra naší aplikace jsme se setkali se čtyřmi zdroji: uživatelé, sezení, aktivace účtů a resety hesel, ale pouze první jmenovaný se opírá o Active Record model s tabulkou v databázi. Konečně je na čase přidat i druhý takový zdroj: uživatelské mikroposty, což jsou krátké zprávy asociované s daným uživatelem. Prve jsme se s nimi setkali hned na začátku, ale tentokrát z nich uděláme plnohodnotnou verzi vybudováním datového modelu mikropostů, která bude souviset s uživatelským modelem pomocí metod "has_many" a "belongs_to". Následně vytvoříme formuláře a partials pro úpravu a zobrazování výsledků. Nakonec si i přidáme možnost uživatele sledovat, abychom viděli, co přidávají, ale to až příště.

## Model mikropostů

Vytváření zdroje mikropostů zahájíme pomocí tvorby stejnojmenného modelu, který pokryje základní prvky mikropostů. Bude zahrnovat jak validace dat, tak asociaci s uživatelským modelem, přičemž bude plně otestován a bude obsahovat i prvky jako základní seřazování a automatické mazání souvisejících postů v případě, že bude rodičovský uživatel (parent user) zlikvidován.

Opět si vytvoříme novou větev v Gitu:

```
$ git checkout -b user-microposts
```

### Základní model

Model mikropostů potřebuje pouze dva atributy: "content" pro udržování obsahu postu a "user_id" pro spřažení s uživatelem, který ho vytvořil. Výsledkem je model mikropostů se strukturou, která je na obrázku (obr. 13.1).

// obr. 13.1 Datový model mikropostů.

Pro obsah postů budeme použivávat datový typ "text" (namísto "string"), jelikož je schopen udržet daleko větší množství znaků (i navzdory tomu, že budou naše posty omezeny na 140 symbolů). Typ "text" totiž lépe vystihuje podstatu mikropostů, což je v podstatě blok textu. Stějne tak i posléze použijeme formát "text area" namísto "text field" pro jejich odesílání. Také se nám tím otevřou další možnosti, jako zvětšení limitu znaků (například v rámci internacionalizace). V produkčním nasazení navíc nebude žádný rozdíl ve smyslu výkonu, takže není proč se tímto směrem nevydat.

Jako v případě uživatelského modelu si i teď vygenerujeme model mikropostů pomocí "generate model":

```
$ rails generate model Micropost content:text user:references
```

Výsledek bude vypadat následovně (soubor "app/models/micropost.rb"). Krom obvyklého podědění od "ApplicationRecord" vygenerovaný model zahrnuje i řádek, který naznačuje, že mikropost patří ("belongs_to") uživateli, a do generate byl zahrnut díky použití parametru "user:references".

```
class Micropost < ApplicationRecord
   belongs_to :user
end
```

Příkaz "generate" také vytvoří migraci pro tvorbu tabulky "microposts" v databázi; můžeme ji srovnat s předchozí tabulkou "users". Největší rozdíl je v použití "references", které automaticky přidá sloupec "user_id" (spolu s indexem a referencí na cizí klíč) pro použití v asociaci uživatel-mikropost. Jako u uživatelského modelu i migrace modelu mikropostů automaticky zahrnuje řádek "t.timestamps", který přidá kouzelné sloupce "created_at" a " updated_at" (viz. předchozí obr. 13.1). Migrace tedy vypadá následovně (soubor "db/migrate/[timestamp]_create_microposts.rb"):

```
class CreateMicroposts < ActiveRecord::Migration[5.0]
  def change
    create_table :microposts do |t|
      t.text :content
      t.references :user, foreign_key: true

      t.timestamps
    end
    add_index :microposts, [:user_id, :created_at]
  end
end
```

Jelikož předpokládáme, že budeme chtít získat všechny mikroposty asociované s daným uživatelem v obráceném pořadí podle vytvoření, následující řádek přidává index na sloupce "user_id" a "created_at":

```
add_index :microposts, [:user_id, :created_at]
```

Zahrnutím obou sloupců v poli Railsu řekneme, že chceme vytvořit index o více klíčích (multiple key index), což znamená, že Active Record používá oba klíče zároveň.

S touto úpravou můžeme spustit samotnou migraci:

```
$ rails db:migrate
```

### Validace mikropostů

Po vytvoření základního modelu přidáme pár validací pro upevnění požadoavného designu. Jeden z nezbytných prvků modelu mikropostů je přítomnost uživatelského id pro označení uživatele, který mikropost vytvořil. Idiomaticky správný postup spočívá v použití asociací Active Record, které si naimplementujeme později, ale nyní nás čeká práce se samotným modelem "Micropost".

Prvotní testy mikropostů jsou podobné, jako u uživatelského modelu. V kroku "setup" vytvoříme nový mikropost, který bude asociován s platným uživatelem z fixtur, a poté ověříme, že je samotný výsledek validní. Jelikož musí mít každý mikropost uživatelské id, přidáme i test na přítomnost "user_id". Bude to tedy vypadat následovně (soubor "test/models/micropost_test.rb"):

```
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase

  def setup
    @user = users(:michael)
    # This code is not idiomatically correct.
    @micropost = Micropost.new(content: "Lorem ipsum", user_id: @user.id)
  end

  test "should be valid" do
    assert @micropost.valid?
  end

  test "user id should be present" do
    @micropost.user_id = nil
    assert_not @micropost.valid?
  end
end
```

Jak naznačuje komentář v metodě "setup", kód pro vytvoření mikropostu není idiomaticky správný, ale to opravíme záhy.

Jako u původního testu uživatelského modelu je i zde první test jen jakýmsi "testem příčetnosti", kdežto druhý už testuje přítomnost uživatelského id, pro který přidáme samotnou validaci v souboru "app/models/micropost.rb":

```
class Micropost < ActiveRecord::Base
  belongs_to :user
  validates :user_id, presence: true
end
```

Testy by měly být (zatím stále) zelené:

```
$ rails test:models
```

Dále si přidáme validace atributu "content" (obsah). Podobně, jako v případě "user_id", i atribut "content" musí být přítomen, a navíc je třeba ho omezit na 140 znaků (což je právě to "mikro" v mikropostech).

Validaci obsahu přidáme za pomocí metodiky testování předem. Výsledné testy následují příkladu z validačních testů v uživatelském modelu (soubor "test/models/micropost_test.rb"):

```
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase

  def setup
    @user = users(:michael)
    @micropost = Micropost.new(content: "Lorem ipsum", user_id: @user.id)
  end

  test "should be valid" do
    assert @micropost.valid?
  end

  test "user id should be present" do
    @micropost.user_id = nil
    assert_not @micropost.valid?
  end

  test "content should be present" do
    @micropost.content = "   "
    assert_not @micropost.valid?
  end

  test "content should be at most 140 characters" do
    @micropost.content = "a" * 141
    assert_not @micropost.valid?
  end
end
```

Opět používáme násobení řetězce pro otestování validace délky:

```
$ rails console
>> "a" * 10
=> "aaaaaaaaaa"
>> "a" * 141
=> "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
```

Související aplikační kód (soubor "app/models/micropost.rb") je mimochodem prakticky totožný s validací "name" v uživatelích:

```
class Micropost < ApplicationRecord
  belongs_to :user
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

V danou chvíli by měly testy proběhnout zeleně:

```
$ rails test
```
### Spřažení mikropostů a uživatelů

Při tvorbě datových modelů pro webové aplikace je nutné, abychom byli schopni vytvářet asociace (spřažení) mezi jednotlivými modely. V současném případě je každý mikropost asociován s jedním uživatelem, a každý uživatel je asociován s (potenciálně) mnoha mikroposty (viz obr. 13.2 a 13.3). Jako součást implementace těchto asociací si napíšeme testy pro model mikropostů a přidáme i několik testů uživatelskému modelu.

// obr. 13.2 Vztah "belongs_to" (náleží) mezi mikropostem a asociovaným uživatelem.

// obr. 13.3 Vztah "has_many" (kolik má) mezi uživatelem a jeho mikroposty.

Za použití těchto asociací Rails vytvoří metody, které jsou vypsány v tabulce níže. Zajímavé je, že namísto

```
Micropost.create
Micropost.create!
Micropost.new
```

máme

```
user.microposts.create
user.microposts.create!
user.microposts.build
```

Tyto metody reprezentují idiomaticky správný způsob pro vytvoření mikropostu, tedy doslova skrze asociaci s uživatelem. Pokud je tímto způsobem mikropost vytvořen, jeho "user_id" je automaticky nastaveno na správnou hodnotu. Můžeme tedy kód

```
@user = users(:michael)
# This code is not idiomatically correct.
@micropost = Micropost.new(content: "Lorem ipsum", user_id: @user.id)
```

z minula nahradit tímto:

```
@user = users(:michael)
@micropost = @user.microposts.build(content: "Lorem ipsum")
```

(Stejně jako "new", "build" vrací objekt v paměti ale neupravuje databázi.) Jakmile zadefinujeme správné asociace, výsledná proměnná "@micropost" bude mít automaticky atribut "user_id" nastavený správně, tedy podle id daného uživatele.

| **Metoda**                       | **Význam**                                                   |
| -------------------------------- | ------------------------------------------------------------ |
| `micropost.user`                 | Vrací uživatelský objekt asociovaný s mikropostem            |
| `user.microposts`                | Returns a collection of the user’s microposts Vrací souhrn uživatelových mikropostů |
| `user.microposts.create(arg)`    | Vytvoří mikropost asociovaný s uživatelem                    |
| `user.microposts.create!(arg)`   | Vytvoří mikropost asociovaný s uživatelem (s výjimkou při selhání) |
| `user.microposts.build(arg)`     | Vrací nový mikropost asociovaný s uživatelem                 |
| `user.microposts.find_by(id: 1)` | Najde mikropost s id "1" a "user_id" o hodnotě rovné "user.id" |

Aby kód typu "" fungoval, musíme aktualizovat model mikropostů a uživatelů. První část asociací byla přidána migrací skrze "belongs_to :user" (soubor "app/models/micropost.rb")

```
class Micropost < ApplicationRecord
  belongs_to :user
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

ale druhou část, tedy "has_many :microposts" musíme přidat ručně (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  has_many :microposts
  .
  .
  .
end
```

Když máme hotové asociace, můžeme aktualizovat metodu "setup" idiomaticky správným způsobem pro vytvoření nového mikropostu (soubor "test/models/micropost_test.rb"):

```
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase

  def setup
    @user = users(:michael)
    @micropost = @user.microposts.build(content: "Lorem ipsum")
  end

  test "should be valid" do
    assert @micropost.valid?
  end

  test "user id should be present" do
    @micropost.user_id = nil
    assert_not @micropost.valid?
  end
  .
  .
  .
end
```

I po tomto drobném refaktorování by měly testy vycházet zeleně:

```
$ rails test
```

### Odlazení mikropostů

V této části si přidáme několik drobných ladících úprav do asociace uživatel/mikropost. Konkrétněji zařídíme, aby uživatelovy mikroposty byly získávány ve specifickém pořadí a také upravíme mikroposty tak, ať jsou na uživatelích závislé, a tedy budou automaticky smazány v případě smazání jejich vlastníka.

#### Základní rozsah

Za normálních okolností metoda "user.microposts" nijak neřeší pořadí postů, ale (právě podle zvyklostí blogů a Twitteru) chceme, ať se mikroposty vypisují v opačném pořadí jejich vytvoření, a tedy nejnovější budou vypsány jako první. Toho docílíme pomocí tzv. základního rozsahu (default scope).

Přesně takovýto prvek aplikace může snadno vést k falešnému testu (tedy že test projde i navzdory tomu, že je aplikační kód špatně) a použijeme tedy metodiku testování předem, abychom si mohli být jistí, že testujeme správnou věc. Napíšeme si tedy test pro ověření, že první mikropost v databázi je stejný jako fixturový mikropost který nazveme "most_recent" (nejnovější) (soubor "test/models/micropost_test.rb"):

```
require 'test_helper'

class MicropostTest < ActiveSupport::TestCase
  .
  .
  .
  test "order should be most recent first" do
    assert_equal microposts(:most_recent), Micropost.first
  end
end
```

Jelikož je tento test závislý na fixturách, bude třeba si je zadefinovat (soubor "test/fixtures/microposts.yml"):

```
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>

tau_manifesto:
  content: "Check out the @tauday site by @mhartl: http://tauday.com"
  created_at: <%= 3.years.ago %>

cat_video:
  content: "Sad cats are sad: http://youtu.be/PKffm2uI4dk"
  created_at: <%= 2.hours.ago %>

most_recent:
  content: "Writing a short test"
  created_at: <%= Time.zone.now %>
```

Explicitně jsme si nastavili sloupec "created_at" za pomocí vnořeného Ruby. Jelikož jde o "kouzelný" sloupec, který Rails automaticky aktualizuje, jeho nastavování ručně není obvykle možné, ale ve fixturách to jde. V praxi to ale nebývá nutné a ve spoustě systémů jsou fixtury vytvořeny v daném pořadí. V tomto případě je poslední fixtura v souboru i vytvořena jako poslední (a tímpádem je nejnovější), ale bylo by bláhové se na toto chování spoléhat, jelikož může byt dost dobře závislé i na konkrétním systému.

Testy jsou nyní červené:

```
$ rails test test/models/micropost_test.rb
```

Zezelenáme je právě pomocí metody "default_scope", která může být mimojiné použita i na řazení prvků při jejich získávání z databáze. Abychom si vynutili specifické pořadí, zahrneme parametr "order", který nám umožní seřazovat podle sloupce "created_at":

```
order(:created_at)
```

Bohužel, takovéto řazení je vzestupné, tedy od nejmenšího po největší, což by znamenalo, že nejstarší mikroposty budou řazeny jako první. Pro opačné pořadí musíme jít ještě o úroveň níže a zahrnout trochu surového SQL:

```
order('created_at DESC')
```

Parametr "DESC" znamená "descending" (sestupné), tedy od nejnovějšího po nejstarší. Ve starších verzích Ruby toto byla jediná možnost, jak docílit požadovaného chování, ale od Rails 4 a výše už můžeme použít i přirozenější Ruby syntaxi:

```
order(created_at: :desc)
```

Zahrnutí těchto poznatků do základního rozsahu (default scope) pak nese následující úpravu (soubor "app/models/micropost.rb"):

```
class Micropost < ApplicationRecord
  belongs_to :user
  default_scope -> { order(created_at: :desc) }
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

Poprvé je zde zahrnutá i syntaxe pro volání procedury ("Proc", anonymní funkce beze jména, která vezme vstup ve formě bloku a vrací Proc, kterou lze zavolat metodou "call"). Můžeme si ji vyzkoušet zavolat i v konzoli:

```
>> -> { puts "foo" }
=> #<Proc:0x007fab938d0108@(irb):1 (lambda)>
>> -> { puts "foo" }.call
foo
=> nil
```

(Jde o poměrně pokročilejší koncept Ruby, takže nám nemusí dávat okamžitě smysl.)

Po úpravách už budou testy vycházet zeleně:

```
$ rails test
```

#### Závislost: zničení

Krom správného řazení chceme mikropostům přidat ještě jednu praktickou vlastnost. Vzpomeňme si, že administrátoři mají možnost smazat uživatele. Dává tedy smysl, že když bude uživatel smazán, budou smazány i jeho mikroposty.

Docílíme toho za pomocí předání parametru asociační metodě "has_many" (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  has_many :microposts, dependent: :destroy
  .
  .
  .
end
```

Parametr "dependent: :destroy" nastavuje chování tak, že když je uživatel zničen, jsou zničeny i jeho mikroposty. V databázi tedy nebudou zůstávat osiřelé mikroposty bez uživatele, když se administrátor rozhodne ho odstranit.

Můžeme si ověřit, že předchozí kód funguje pomocí testu pro uživatelský model. Stačí nám uložit uživatele (aby získal id) a vytvořit související mikropost. Pak jen ověříme, že se po zničení uživatele sníží celkový počet mikropostů o 1. Test tedy vypadá následovně (soubor "test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com",
                     password: "foobar", password_confirmation: "foobar")
  end
  .
  .
  .
  test "associated microposts should be destroyed" do
    @user.save
    @user.microposts.create!(content: "Lorem ipsum")
    assert_difference 'Micropost.count', -1 do
      @user.destroy
    end
  end
end
```

Pokud kód funguje správně, testy budou pořád zelené:

```
$ rails test
```



## Zobrazování mikropostů

Ačkoliv zatím nemáme způsob, jak přidat mikroposty skrze webové prostředí (ale do konce této kapitoly ho mít budeme), pořád je můžeme zobrazovat (a samotné zobrazení testovat). Po vzoru Twitteru budeme chtít zobrazovat uživatelovy mikroposty na jeho samostatné "show" stránce (viz. obr. 13.4). Začneme s jednoduchými ERb šablonami pro přidání zobrazení mikropostů na uživatelském profilu a poté přidáme vzorová data mikropostů, abychom měli reálně co zobrazovat.

// obr. 13.4 Mockup profilové stránky s mikroposty.

### Vykreslování mikropostů

V plánu tedy máme zobrazovat mikroposty každého uživatele na jejich vlastních profilových stránkách (soubor "show.html.erb"), spolu s celkovým součtem mikropostů, které vytvořili. Jak uvidíme, spousta prvků je podobných jako při vypisování všech uživatelů.

Ačkoliv zatím nebudeme potřebovat ovladač mikropostů, budeme potřebovat adresář pohledů, a tedy si ovladač vygenerujeme teď:

```
$ rails generate controller Microposts
```

V této části si budeme chtít především vykreslit všechny mikroposty každého uživatele. U vypisování uživatelů jsme viděli, že kód

```
<ul class="users">
  <%= render @users %>
</ul>
```

automaticky vykresli každého uživatele v proměnné "@users" pomocí partialu "_user.html.erb". Zadefinujeme si obdobný partial "micropost.html.erb", abychom mohli stejným způsobem vypsat seznam mikropostů:

```
<ol class="microposts">
  <%= render @microposts %>
</ol>
```

Používáme tentokrát tag "ol", což je "ordered list" (řazený seznam), narozdíl od původního neřazeného "ul", jelikož jsou mikroposty vypisovány ve specifickém pořadí. Partial tedy bude vypadat následovně (soubor "app/views/microposts/_micropost.html.erb"):

```
<li id="micropost-<%= micropost.id %>">
  <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
  <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
  <span class="content"><%= micropost.content %></span>
  <span class="timestamp">
    Posted <%= time_ago_in_words(micropost.created_at) %> ago.
  </span>
</li>
```

Používáme zde skvělou pomocnickou metodu "time_ago_in_words" ("před jakou dobou - slovně"), jejíž efekt uvidíme záhy. Také jsme jednotlivým mikropostům přiřadili CSS id za pomocí

```
<li id="micropost-<%= micropost.id %>">
```

Což je obecně užitečná věc, jelikož otevírá možnosti manipulace s jednotlivými mikroposty (např. za pomocí JavaScriptu).

Dalším krokem je vyřešení zobrazení potenciálně velkého množství mikropostů. Podobně, jako u uživatelů zde použijeme stránkování, resp. metodu "will_paginate":

```
<%= will_paginate @microposts %>
```

Když to srovnáme s uživateli, dříve stačilo použít pouze

```
<%= will_paginate %>
```

Tento přístup mohl fungovat díky tomu, že v kontextu uživatelského ovladače "will_paginate" předpokládá existenci instanční proměnné zvané "@users". V současné chvíli jsme ale stále v uživatelském ovladači a chceme stránkovat mikroposty, proto je třeba explicitně zmínit proměnnou "@microposts". Samozřejmě budeme muset takovou proměnnou definovat v uživatelské akci "show" (soubor "app/controllers/users_controller.rb").

```
class UsersController < ApplicationController
  .
  .
  .
  def show
    @user = User.find(params[:id])
    @microposts = @user.microposts.paginate(page: params[:page])
  end
  .
  .
  .
end
```

Je zajímavé, jak chytře "paginate" funguje, tedy skrze asociaci mikropostů až do tabulky "microposts", kde vytáhne potřebnou stránku.

Naším posledním úkonem bude zobrazit celkový seznam mikropostů každého uživatele, což můžeme udělat za pomocí metody "count":

```
user.microposts.count
```

Jako u "paginate", můžeme použít metodu "count" skrze asociaci. Metoda "count" naštěstí nevytahuje všechny mikroposty z databáze, aby pak zavolala "length" na výsledné pole (což by bylo ohromně neefektivní, když by počet mikropostů narostl), ale provede kalkulaci přímo v databázi, které se zeptá na počet mikropostů s daným "user_id" (což je operace, na kterou jsou všechny databáze vysoce optimalizované).

Když si všechny tyto poznatky shrneme dohromady, můžeme přidat mikroposty na profilovou stránku následující úpravou (soubor "app/views/users/show.html.erb"). Mimochodem, používáme i konstrukci "if @user.microposts.any?", díky které se ujistíme, že pokud uživatel nemá mikroposty, nebudeme seznam vůbec zobrazovat.

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
  <div class="col-md-8">
    <% if @user.microposts.any? %>
      <h3>Microposts (<%= @user.microposts.count %>)</h3>
      <ol class="microposts">
        <%= render @microposts %>
      </ol>
      <%= will_paginate @microposts %>
    <% end %>
  </div>
</div>
```

Můžeme se podívat na aktualizovanou profilovou stránku uživatele (obr. 13.5), která je ale přecejen krapet chudá na obsah, jelikož zatím žádné mikroposty neexistují. Je na čase s tím něco udělat.

// obr. 13.5 Uživatelská stránka s kódem pro mikroposty (i když bez samotných mikropostů).

### Vzorové mikroposty

I když bylo zakončení předchozí části poněkud slabé, přidání mikropostů do vzorových dat pořád může dojem zachránit.

Nicméně, přidání vzorových mikropostů všem uživatelům by zabralo poměrně hodně času, a vybereme si tedy pouze prvních šest uživatelů za pomocí metody "take":

```
User.order(:created_at).take(6)
```

Zavolání skrze "order" zajišťuje, že se vybere prvních šest uživatelů, které jsme vytvořili.

Pro každého z vybraných uživatelů vytvoříme 50 mikropostů (což je dost na překročení limitu stránkování 30). Pro vytvoření vzorového obsahu pro každý mikropost použijeme praktickou metodu "Lorem.sentence", kterou disponuje gem Faker (soubor "db/seeds.rb"):

```
.
.
.
users = User.order(:created_at).take(6)
50.times do
  content = Faker::Lorem.sentence(5)
  users.each { |user| user.microposts.create!(content: content) }
end
```

Zbývá jen znovu naplnit vývojovou databázi:

```
$ rails db:migrate:reset
$ rails db:seed
```

Také je vhodné restartovat samotný vývojový server Rails.

Předběžný výsledek by měl vypadat obdobně, jako na obrázku (obr. 13.6), tedy zobrazení informací o každém mikropostu.

// obr. 13.6 Profilová stránka s mikroposty bez stylování.

Jak vidno, bude třeba přidat trochu CSS, aby byl výsledek poněkud více líbivý (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* microposts */

.microposts {
  list-style: none;
  padding: 0;
  li {
    padding: 10px 0;
    border-top: 1px solid #e8e8e8;
  }
  .user {
    margin-top: 5em;
    padding-top: 0;
  }
  .content {
    display: block;
    margin-left: 60px;
    img {
      display: block;
      padding: 5px 0;
    }
  }
  .timestamp {
    color: $gray-light;
    display: block;
    margin-left: 60px;
  }
  .gravatar {
    float: left;
    margin-right: 10px;
    margin-top: 5px;
  }
}

aside {
  textarea {
    height: 100px;
    margin-bottom: 5px;
  }
}

span.picture {
  margin-top: 10px;
  input {
    border: 0;
  }
}
```

Následující obrázky zachycují profilovou stránku prvního uživatele (obr. 13.7), profilovou stránku druhého uživatele (obr. 13.8) a nakonec druhou stránku mikropostů prvního uživatele, spolu s odkazy stránkování vespod (obr. 13.9). Ve všech případech si můžeme všimnout, že každý mikropost zobrazuje dobu, od které byl vytvořen (což je právě výsledek metody "time_ago_in_words"). Když si po několika minutách stránku znovu načteme, bude i zobrazení času náležitě aktualizováno.

// obr. 13.7 Uživatelův profil s mikroposty (/users/1).

// obr. 13.8 Profil jiného uživatele s mikroposty (/users/5).

// obr. 13.9 Stránkovací odkazy mikropostů (/users/1?page=2).

### Testy profilových mikropostů

Jelikož jsou nově aktivovaní uživatelé přesměrováni na své profilové stránky, máme už pro takový případ vytvořený test. V této části si napíšeme krátký integrační test pro pár dalších prvků profilové stránky, včetně těch, které jsme přidali jako poslední. Začneme vygenerováním integračního testu pro profily našich uživatelů:

```
$ rails generate integration_test users_profile
      invoke  test_unit
      create    test/integration/users_profile_test.rb
```

Pro otestování zobrazení mikropostů na profilu potřebujeme asociovat fixturové mikroposty s uživatelem. Rails disponuje praktickým způsobem pro vytvoření asociací ve fixturách:

```
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael
```

Identifikováním uživatele ("user") jako "michael" Railsu oznamujeme, že má asociovat mikropost se souvisejícím uživatelem ve fixtuře:

```
michael:
  name: Michael Example
  email: michael@example.com
  .
  .
  .
```

Pro otestování stránkování mikropostů také vygenerujeme pár dodatečných mikropostových fixtur za pomocí stejné techniky vnořeného Ruby, jako když jsme vytvářeli vzorové uživatele:

```
<% 30.times do |n| %>
micropost_<%= n %>:
  content: <%= Faker::Lorem.sentence(5) %>
  created_at: <%= 42.days.ago %>
  user: michael
<% end %>
```

Jedno s druhým povede k následující úpravě fixtur (soubor "test/fixtures/microposts.yml"):

```
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael

tau_manifesto:
  content: "Check out the @tauday site by @mhartl: http://tauday.com"
  created_at: <%= 3.years.ago %>
  user: michael

cat_video:
  content: "Sad cats are sad: http://youtu.be/PKffm2uI4dk"
  created_at: <%= 2.hours.ago %>
  user: michael

most_recent:
  content: "Writing a short test"
  created_at: <%= Time.zone.now %>
  user: michael

<% 30.times do |n| %>
micropost_<%= n %>:
  content: <%= Faker::Lorem.sentence(5) %>
  created_at: <%= 42.days.ago %>
  user: michael
<% end %>
```

Když máme testovací data připravena, je samotný test už poměrně přímočarý: navštívíme profilovou stránku uživatele a ověříme název stránky a uživatelské jméno, Gravatara, součet mikropostů a nastránkované mikroposty. Použijeme i pomocníka "full_title" (kterého máme k dispozici díky zahrnutím modulu Application Helper) právě pro ověření názvu stránky. Test bude tedy vypadat následovně (soubor "test/integration/users_profile_test.rb"):

```
require 'test_helper'

class UsersProfileTest < ActionDispatch::IntegrationTest
  include ApplicationHelper

  def setup
    @user = users(:michael)
  end

  test "profile display" do
    get user_path(@user)
    assert_template 'users/show'
    assert_select 'title', full_title(@user.name)
    assert_select 'h1', text: @user.name
    assert_select 'h1>img.gravatar'
    assert_match @user.microposts.count.to_s, response.body
    assert_select 'div.pagination'
    @user.microposts.paginate(page: 1).each do |micropost|
      assert_match micropost.content, response.body
    end
  end
end
```

Pro součet mikropostů používáme "response.body", které, navzdory jménu, obsahuje celý HTML zdroj stránky. Záleží nám tedy pouze na tom, že se někde na stránce nachází text součtu mikropostů, který vyhledáme takto:

```
assert_match @user.microposts.count.to_s, response.body
```

Jde o daleko méně specifické hledání než v případe "assert_select" a tedy nemusíme určovat, který HTML tag přesně hledáme.

Také jsme použili vnořovací syntaxi pro "assert_select":

```
assert_select 'h1>img.gravatar'
```

Tento řádek hledá tag "img" se třídou "gravatar" uvnitř tagu heading ("h1").

Jelikož aplikační kód fungoval, testy by měly být zelené:

```
$ rails test
```

## Manipulace s mikroposty

Jelikož máme hotové jako modelování dat, tak zobrazovací šablony pro mikroposty, je na čase se zaměřit na rozhraní pro jejich tvorbu skrze web. V této části také uvidíme první náznak tzv. status feed, tedy sledování postů ostatních uživatelů, ale plně tuto funkcionalitu rozvedeme až příště. Také zpřístupníme možnost mazání mikropostů skrze web jako takových.

Jedna změna oproti předchozím zvyklostem stojí za zmínku: rozhraní zdroje mikropostů poběží v podstatě skrze domovskou stránku a stránku profilu, takže nebudeme potřebovat akce jako "new" a "edit" v ovladači mikropostů; potřeba bude jen "create" a "destroy". Úprava směrování tedy bude vypadat jako v následujícím výpisu. Jde v podstatě o malou podmnožinu běžných "RESTful" cest z dřívějška (soubor "config/routes.rb").

```
Rails.application.routes.draw do
  root   'static_pages#home'
  get    '/help',    to: 'static_pages#help'
  get    '/about',   to: 'static_pages#about'
  get    '/contact', to: 'static_pages#contact'
  get    '/signup',  to: 'users#new'
  get    '/login',   to: 'sessions#new'
  post   '/login',   to: 'sessions#create'
  delete '/logout',  to: 'sessions#destroy'
  resources :users
  resources :account_activations, only: [:edit]
  resources :password_resets,     only: [:new, :create, :edit, :update]
  resources :microposts,          only: [:create, :destroy]
end
```

| **HTTP požadavek** | **URL**       | **Akce**  | **Jmenná cesta**            |
| ------------------ | ------------- | --------- | --------------------------- |
| `POST`             | /microposts   | `create`  | `microposts_path`           |
| `DELETE`           | /microposts/1 | `destroy` | `micropost_path(micropost)` |

### Kontrola přístupu mikropostů (access control)

Vývoj zdroje mikropostů započneme kontrolou přístupu v ovladači mikropostů. Konkrétněji tedy, jelikož přistupujeme k mikropostům skrze jejich asociované uživatele, jak akce "create" tak "destroy" budou vyžadovat přihlášení.

Testy pro vynucení přihlášeného stavu jsou podobné, jako v případě uživatelského ovladače. Jednoduše zadáme správný požadavek pro každou akci a potvrdíme, že se počet mikropostů nezměnil a výsledek je přesměrován na přihlašovací URL (soubor "test/controllers/microposts_controller_test.rb"):

```
require 'test_helper'

class MicropostsControllerTest < ActionDispatch::IntegrationTest

  def setup
    @micropost = microposts(:orange)
  end

  test "should redirect create when not logged in" do
    assert_no_difference 'Micropost.count' do
      post microposts_path, params: { micropost: { content: "Lorem ipsum" } }
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy when not logged in" do
    assert_no_difference 'Micropost.count' do
      delete micropost_path(@micropost)
    end
    assert_redirected_to login_url
  end
end
```

Napsání aplikačního kódu, který je potřeba, aby testy prošly vyžaduje nejprve trochu refaktorování. Dříve jsme použili before filtr který zavolal metodu "logged_in_user". Metodu jsme potřebovali pouze v ovladači uživatelů, ale nyní ji potřebujeme i v ovladači mikropostů, a tedy ji přesuneme do aplikačního ovladače, což je základní třída všech ovladačů. Výsledná úprava vypadá následovně (soubor "app/controllers/application_controller.rb"):

```
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  include SessionsHelper

  private

    # Confirms a logged-in user.
    def logged_in_user
      unless logged_in?
        store_location
        flash[:danger] = "Please log in."
        redirect_to login_url
      end
    end
end
```

Abychom se vyhnuli duplnicnímu kódu, bude třeba odstranit i "logged_in_user" z ovladače uživatelů (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy]
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end

    # Before filters

    # Confirms the correct user.
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless current_user?(@user)
    end

    # Confirms an admin user.
    def admin_user
      redirect_to(root_url) unless current_user.admin?
    end
end
```

Díky této úpravě je metoda "logged_in_user" k dispozici i v ovladači mikropostů, což znamená, že můžeme přidat akce "create" a "destroy" a následně omezit přístup k nim skrze before filtr (soubor "app/controllers/microposts_controller.rb"):

```
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]

  def create
  end

  def destroy
  end
end
```

Testy by měly proběhnout zeleně:

```
$ rails test
```

### Vytváření mikropostů

Když jsme implementovali registraci uživatelů, použili jsme HTML formulář, který zadal HTTP požadavek "POST" akci "create" v uživatelském ovladači. Implementace tvorby mikropostů bude podobná; hlavní rozdíl je ten, že namísto použití samostatné stránky /microposts/new použijeme formulář přímo na domovské stránce (tedy kořenová cesta /), jak je vidět na obrázku (obr. 13.10).

// obr. 13.10 Mockup domovské stránky s formulářem pro tvorbu mikropostů.

Naposledy jsme opustili domovskou stránku jen s tlačítkem "Zaregistrujte se!" uprostřed. Jelikož formulář tvorby mikropostů má smysl pouze v kontextu přihlášeného uživatele, jeden z cílů této části bude spočívat v rozvětvění zobrazených verzí podle stavu přihlášení.

Začneme akcí "create" pro mikroposty, která je podobná, jako v případě uživatelů; základní rozdíl spočívá v použití asociace uživatel/mikropost pro tvorbu ("build") nového mikropostu, jak je vidno v následující úpravě (soubor "app/controllers/microposts_controller.rb"). Mimojiné zase používáme silné parametry skrze "micropost_params", které umožňují úpravu atributu "content" pouze skrze web.

```
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]

  def create
    @micropost = current_user.microposts.build(micropost_params)
    if @micropost.save
      flash[:success] = "Micropost created!"
      redirect_to root_url
    else
      render 'static_pages/home'
    end
  end

  def destroy
  end

  private

    def micropost_params
      params.require(:micropost).permit(:content)
    end
end
```

Pro vytvoření formuláře pro tvorbu mikropostů použijeme následující kód, který zobrazuje rozdílné HTML na základě stavu přihlášení uživatele (soubor "app/views/static_pages/home.html.erb"):

```
<% if logged_in? %>
  <div class="row">
    <aside class="col-md-4">
      <section class="user_info">
        <%= render 'shared/user_info' %>
      </section>
      <section class="micropost_form">
        <%= render 'shared/micropost_form' %>
      </section>
    </aside>
  </div>
<% else %>
  <div class="center jumbotron">
    <h1>Welcome to the Sample App</h1>

    <h2>
      This is the home page for the
      <a href="http://www.railstutorial.org/">Ruby on Rails Tutorial</a>
      sample application.
    </h2>

    <%= link_to "Sign up now!", signup_path, class: "btn btn-lg btn-primary" %>
  </div>

  <%= link_to image_tag("rails.png", alt: "Rails logo"),
              'http://rubyonrails.org/' %>
<% end %>
```

Aby takto zadefinovaná stránka fungovala, musíme vytvořit a naplnit několik partialů. Prvním je boční lišta domovské stránky (home page sidebar), který vypadá následovně (soubor "app/views/shared/_user_info.html.erb"):

```
<%= link_to gravatar_for(current_user, size: 50), current_user %>
<h1><%= current_user.name %></h1>
<span><%= link_to "view my profile", current_user %></span>
<span><%= pluralize(current_user.microposts.count, "micropost") %></span>
```

Boční lišta mimojiné zobrazuje celkový počet mikropostů daného uživatele. Zobrazení samotné se také poněkud větví a mění se podle počtu mikropostů (za pomocí metody "pluralize").

Formulář samotný je podobný registraci a bude vypadat následovně (soubor "app/views/shared/_micropost_form.html.erb"):

```
<%= form_for(@micropost) do |f| %>
  <%= render 'shared/error_messages', object: f.object %>
  <div class="field">
    <%= f.text_area :content, placeholder: "Compose new micropost..." %>
  </div>
  <%= f.submit "Post", class: "btn btn-primary" %>
<% end %>
```

Než bude samotný formulář funkční, bude třeba ještě dvou úprav. V prvé řadě zadefinovat "@micropost", což opět můžeme udělat skrze asociaci:

```
@micropost = current_user.microposts.build
```

Úprava tedy bude vypadat takto (soubor "app/controllers/static_pages_controller.rb"):

```
class StaticPagesController < ApplicationController

  def home
    @micropost = current_user.microposts.build if logged_in?
  end

  def help
  end

  def about
  end

  def contact
  end
end
```

Pochopitelně "current_user" existuje pouze tehdy, kdy je uživatel přihlášený, takže proměnná "@micropost" bude definována jen v tomto případě.

Druhá změna spočívá v předefinování partialu chybových zpráv, aby následující kód fungoval:

```
<%= render 'shared/error_messages', object: f.object %>
```

Z dřívějška si můžeme vzpomenout, že partial pro chybové zprávy odkazuje na proměnnou "@user" explicitně, ale v současné chvíli nás zajímá proměnná "@micropost". Abychom případy sjednotili, předáme formulářovou proměnnou "f" partialu a přistoupíme k asociovanému objektu skrze "f.object", takže v

```
form_for(@user) do |f|
```

bude "f.object" "@user" a v

```
form_for(@micropost) do |f|
```

bude "f.object" naopak "@micropost", atd.

Pro předání objektu partialu použijeme haš o stejné hodnotě, jako má objekt a klíč, který odpovídá požadovanému jménu proměnné v partialu. Jinak řečeno, "object: f.object" vytvoří proměnnou "object" v partialu "error_messages", kterou lze použít pro vytvoření upravené chybové hlášky, viz. následující úprava (soubor "app/views/shared/_error_messages.html.erb"):

```
<% if object.errors.any? %>
  <div id="error_explanation">
    <div class="alert alert-danger">
      The form contains <%= pluralize(object.errors.count, "error") %>.
    </div>
    <ul>
    <% object.errors.full_messages.each do |msg| %>
      <li><%= msg %></li>
    <% end %>
    </ul>
  </div>
<% end %>
```

V tuto chvíli bychom si měli ověřit, že jsou testy červené:

```
$ rails test
```

Musíme totiž aktualizovat ostatní použití partialu chybových zpráv, který jsme použili při registraci a úpravě uživatelů a resetování hesel. Upravené verze vypadají následovně (soubory "app/views/users/new.html.erb", "app/views/users/edit.html.erb" a "app/views/password_resets/edit.html.erb"):

```
<% provide(:title, 'Sign up') %>
<h1>Sign up</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user) do |f| %>
      <%= render 'shared/error_messages', object: f.object %>
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

```
<% provide(:title, "Edit user") %>
<h1>Update your profile</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user) do |f| %>
      <%= render 'shared/error_messages', object: f.object %>

      <%= f.label :name %>
      <%= f.text_field :name, class: 'form-control' %>

      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.label :password_confirmation, "Confirmation" %>
      <%= f.password_field :password_confirmation, class: 'form-control' %>

      <%= f.submit "Save changes", class: "btn btn-primary" %>
    <% end %>

    <div class="gravatar_edit">
      <%= gravatar_for @user %>
      <a href="http://gravatar.com/emails">change</a>
    </div>
  </div>
</div>
```

```
<% provide(:title, 'Reset password') %>
<h1>Password reset</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user, url: password_reset_path(params[:id])) do |f| %>
      <%= render 'shared/error_messages', object: f.object %>

      <%= hidden_field_tag :email, @user.email %>

      <%= f.label :password %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.label :password_confirmation, "Confirmation" %>
      <%= f.password_field :password_confirmation, class: 'form-control' %>

      <%= f.submit "Update password", class: "btn btn-primary" %>
    <% end %>
  </div>
</div>
```

Teď už by testy měly proběhnout zeleně:

```
$ rails test
```

Všechno HTML v této sekci by se také už mělo správně vykreslovat. Na obrázku je vidět formulář samotný (obr. 13.11) a verze s chybným odesláním (obr. 13.12).

// obr. 13.11 Domovská stránka s formulářem pro mikroposty.

// obr. 13.12 Domovská stránka s chybou formuláře.

### Prvotní verze feedu

Ačkoliv je formulář pro mikroposty funkční, uživatelé nemohou ihned vidět výsledky úspěšného odeslání, jelikož současná domovská stránka žádné mikroposty nezobrazuje. Můžeme si ověřit, že formulář funguje odesláním platných dat a poté proklikáním se na profilovou stránku, kde post uvidíme, ale to je poněkud neohrabané. Bylo by daleko lepší mít feed (doslova "krmení", tedy odběr obsahu) mikropostů který zahrnuje i vlastní uživatelovy posty, jak je vidět na mockupu na obrázku (obr. 13.13).

// obr. 13.13 Mockup domovské stránky s prvotní verzí feedu.

Jelikož by každý uživatel měl mít svůj feed, jsme přirozeně vedeni metodě "feed" v uživatelském modelu, která (zatím) jen vybere všechny mikroposty, které danému uživateli náleží. Toho docílíme pomocí metody "where" na model "Micropost" (soubor "app/models/user.rb "):

```
class User < ApplicationRecord
  .
  .
  .
  # Defines a proto-feed.
  # See "Following users" for the full implementation.
  def feed
    Micropost.where("user_id = ?", id)
  end

    private
    .
    .
    .
end
```

Otazník v

```
Micropost.where("user_id = ?", id)
```

zajišťuje, že je "id" náležitě ošetřeno před zahrnutím do následujícího SQL dotazu, čímž se vyhneme nebezpečí zneužití pomocí tzv. [SQL injection](http://en.wikipedia.org/wiki/SQL_injection). Atribut "id" je pouze integer (celočíselná hodnota), takže i když v tomto případě reálně nehrozí SQL injection, rozhodně neuškodí se naučit vždy ošetřovat proměnné, které jsou vloženy do SQL požadavků.

Abychom mohli feed v aplikaci použít, přidáme instanční proměnnou "@feed_items" do (nastránkovaného) feedu současného uživatele a také samotný partial status feedu do domovské stránky. Jelikož už máme dva řádky, které je potřeba spustit pouze když je uživatel přihlášen, řádek

```
@micropost = current_user.microposts.build if logged_in?
```

se změní na

```
  if logged_in?
    @micropost  = current_user.microposts.build
    @feed_items = current_user.feed.paginate(page: params[:page])
  end
```

Přidání instanční proměnné do akce "home" tedy bude vypadat takto (soubor "app/controllers/static_pages_controller.rb"):

```
class StaticPagesController < ApplicationController

  def home
    if logged_in?
      @micropost  = current_user.microposts.build
      @feed_items = current_user.feed.paginate(page: params[:page])
    end
  end

  def help
  end

  def about
  end

  def contact
  end
end
```

Samotný partial bude vypadat následovně (soubor "app/views/shared/_feed.html.erb"):

```
<% if @feed_items.any? %>
  <ol class="microposts">
    <%= render @feed_items %>
  </ol>
  <%= will_paginate @feed_items %>
<% end %>
```

Partial feedu odkládá vykreslování partialu mikropostů, který jsme si definovali dříve:

```
<%= render @feed_items %>
```

Rails ví, že má zavolat partial mikropostů jelikož každý prvek "@feed_items" má třídu "Micropost". Rails tedy vyhledá partial se souvisejícím jménem v adresáři pohledů daného zdroje:

```
app/views/microposts/_micropost.html.erb
```

Přidat feed do domovské stránky pak můžeme jednoduše (obr. 13.14), pomocí vykreslení partialu (soubor "app/views/static_pages/home.html.erb"):

```
<% if logged_in? %>
  <div class="row">
    <aside class="col-md-4">
      <section class="user_info">
        <%= render 'shared/user_info' %>
      </section>
      <section class="micropost_form">
        <%= render 'shared/micropost_form' %>
      </section>
    </aside>
    <div class="col-md-8">
      <h3>Micropost Feed</h3>
      <%= render 'shared/feed' %>
    </div>
  </div>
<% else %>
  .
  .
  .
<% end %>
```

// obr. 13.14 Domovská stránka se základním feedem.

V tuto chvíli už vytváření nových mikropostů funguje podle očekávání, jak je vidět na obrázku (obr. 13.15). Zbývá ale jeden neduh: při neúspěšném odeslání mikropostu domovská stránka očekává instanční proměnnou "@feed_items", a kód se na tomto problému zasekne. Nejjednodušší řešení je potlačit celý feed přiřazením prázdného pole (soubor ""):

// obr. 13.15 Domovská stránka po vytvoření nového mikropostu.

```
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]

  def create
    @micropost = current_user.microposts.build(micropost_params)
    if @micropost.save
      flash[:success] = "Micropost created!"
      redirect_to root_url
    else
      @feed_items = []
      render 'static_pages/home'
    end
  end

  def destroy
  end

  private

    def micropost_params
      params.require(:micropost).permit(:content)
    end
end
```

### Mazání mikropostů

Posledním typem funkcionality, kterou přidáme do zdroje mikropostů je možnost posty mazat. Podobně, jako u mazaní uživatelů toho i zde dosáhneme skrze odkazy "smazat" (viz. obr. 13.16). Narozdíl od minulého případu, kdy mohl uživatele mazat pouze administrátor, tyto odkazy budou funkční pouze pro jejich tvůrce.

// obr. 13.16 Mockup základního feedu s odkazy pro mazání postů.

Prvním krokem bude přidání mazacího odkazu do partialu mikropostů (soubor "app/views/microposts/_micropost.html.erb"): 

```
<li id="<%= micropost.id %>">
  <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
  <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
  <span class="content"><%= micropost.content %></span>
  <span class="timestamp">
    Posted <%= time_ago_in_words(micropost.created_at) %> ago.
    <% if current_user?(micropost.user) %>
      <%= link_to "delete", micropost, method: :delete,
                                       data: { confirm: "You sure?" } %>
    <% end %>
  </span>
</li>
```

Dalším krokem bude zadefinování akce "destroy" v ovladači mikropostů, což bude podléhat stejnému principu, jako v případě uživatelů. Základní rozdíl je ten, že namísto použití proměnné "@user" s before filtrem "admin_user" najdeme mikroposty skrze asociaci, což automaticky selže v případě, že se uživatel snaží mazat post někoho jiného. Výsledný "find" umístíme do "correct_user" before filtru, což ověří, že daný uživatel opravdů má mikropost s daným id. Výsledek tedy bude vypadat takto (soubor "app/controllers/microposts_controller.rb"):

```
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]
  before_action :correct_user,   only: :destroy
  .
  .
  .
  def destroy
    @micropost.destroy
    flash[:success] = "Micropost deleted"
    redirect_to request.referrer || root_url
  end

  private

    def micropost_params
      params.require(:micropost).permit(:content)
    end

    def correct_user
      @micropost = current_user.microposts.find_by(id: params[:id])
      redirect_to root_url if @micropost.nil?
    end
end
```

Metoda "destroy" přesměrovává na URL

```
request.referrer || root_url
```

Použili jsme metodu "request.referrer", která je příbuzná proměnné "request.original_url", kterou jsme použili v přátelském přesměrování, a obsahuje pouze předchozí URL (v tomto případě domovskou stránku). Je to praktická věc, jelikož mikroposty se objevují jak na domovské, tak na profilové stránce, a tedy použitím "request.referrer" zařídíme přesměrování zpět na stránku, která zadala mazací požadavek v obou případěch. Pokud by byla odkazující URL "nil" (což je případ některých testů), bude použita "root_url" (kořenová adresa) díky operátoru ||.

S těmito úpravami pak vypadá výsledek smazání druhého nejnovějšího postu jako na obrázku (obr. 13.17).

// obr. 13.17 Domovská stránka po smazání druhého nejnovějšího mikropostu.

### Testy mikropostů

S předchozími úpravami je hotový jak model, tak rozhraní mikropostů. Zbývá jen napsat krátký test ovladače mikropostů pro ověření autorizace a integrační test pro všechny úpravy zároveň.

Začneme přidáním několika mikropostů s různými vlastníky do fixtur (soubor "test/fixtures/microposts.yml"):

```
.
.
.
ants:
  content: "Oh, is that what you want? Because that's how you get ants!"
  created_at: <%= 2.years.ago %>
  user: archer

zone:
  content: "Danger zone!"
  created_at: <%= 3.days.ago %>
  user: archer

tone:
  content: "I'm sorry. Your words made sense, but your sarcastic tone did not."
  created_at: <%= 10.minutes.ago %>
  user: lana

van:
  content: "Dude, this van's, like, rolling probable cause."
  created_at: <%= 4.hours.ago %>
  user: lana
```

Dále napíšeme krátky test pro ujištění, že uživatel nemůže mazat posty jiného, a také ověříme správné přesměrování (soubor "test/controllers/microposts_controller_test.rb"):

```
require 'test_helper'

class MicropostsControllerTest < ActionDispatch::IntegrationTest

  def setup
    @micropost = microposts(:orange)
  end

  test "should redirect create when not logged in" do
    assert_no_difference 'Micropost.count' do
      post microposts_path, params: { micropost: { content: "Lorem ipsum" } }
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy when not logged in" do
    assert_no_difference 'Micropost.count' do
      delete micropost_path(@micropost)
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy for wrong micropost" do
    log_in_as(users(:michael))
    micropost = microposts(:ants)
    assert_no_difference 'Micropost.count' do
      delete micropost_path(micropost)
    end
    assert_redirected_to root_url
  end
end
```

Nakonec si napíšeme integrační test pro přihlášení, ověření funkčnosti stránkování mikropostů, odeslání neplatných dat, odeslání platných dat, mazání postu a navštívění jiné uživatelské stránky pro ujištění, že neobsahuje žádné mazací odkazy. Test si vygenerujeme stejně, jako obvykle:

```
$ rails generate integration_test microposts_interface
      invoke  test_unit
      create    test/integration/microposts_interface_test.rb
```

Jeho obsah si pak upravíme následovně (soubor "test/integration/microposts_interface_test.rb"):

```
require 'test_helper'

class MicropostsInterfaceTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end

  test "micropost interface" do
    log_in_as(@user)
    get root_path
    assert_select 'div.pagination'
    # Invalid submission
    assert_no_difference 'Micropost.count' do
      post microposts_path, params: { micropost: { content: "" } }
    end
    assert_select 'div#error_explanation'
    # Valid submission
    content = "This micropost really ties the room together"
    assert_difference 'Micropost.count', 1 do
      post microposts_path, params: { micropost: { content: content } }
    end
    assert_redirected_to root_url
    follow_redirect!
    assert_match content, response.body
    # Delete post
    assert_select 'a', text: 'delete'
    first_micropost = @user.microposts.paginate(page: 1).first
    assert_difference 'Micropost.count', -1 do
      delete micropost_path(first_micropost)
    end
    # Visit different user (no delete links)
    get user_path(users(:archer))
    assert_select 'a', text: 'delete', count: 0
  end
end
```

Jelikož jsme si napsali nejprve funkční aplikační kód, sada by měla proběhnout zeleně:

```
$ rails test
```

## Obrázky v mikropostech

Jelikož máme hotovou podporu pro všechny relevantní akce mikropostů, přidáme v této sekci možnost, že krom textu budou moci obsahovat mikroposty i obrázky. Začneme se základní verzí (dostatečně dobrou pro vývojové použití) a následně přidáme sadu vylepšení pro produkční využití.

Přidání možnosti nahrávání obrázku zahrnuje dva viditelné prvky: formulářové pole pro nahrání obrázku a samotné obrázky v mikropostech. Mockup výsledného tlačítka "Nahrát obrázek" a mikropostu s fotkou je na obrázku (obr. 13.18).

// obr. 13.18 Mockup nahrání obrázku v mikropostu (včetně už nahraného obrázku).

### Základní nahrání obrázku

Pro zpracování nahraného obrázku a jeho asociaci s modelem mikropostů použijeme nahrávač obrázků [CarrierWave](https://github.com/carrierwaveuploader/carrierwave). Pro začátek bude třeba zahrnout gemy "carrierwave" a "mini_magick" do souboru Gemfile. Pro úplnost zahrnujeme i gem "fog", který bude potřeba pro nahrání obrázku v produkčním nasazení.

```
source 'https://rubygems.org'

gem 'rails',                   '5.1.6'
gem 'bcrypt',                  '3.1.12'
gem 'faker',                   '1.7.3'
gem 'carrierwave',             '1.2.2'
gem 'mini_magick',             '4.7.0'
gem 'will_paginate',           '3.1.5'
gem 'bootstrap-will_paginate', '1.0.0'
.
.
.
group :production do
  gem 'pg',  '0.20.0'
  gem 'fog', '1.42'
end
.
.
.
```

Jako obvykle spustíme instalaci:

```
$ bundle install
```

CarrierWave přidá generátor Railsu pro tvorbu nahrávače obrázků, který použijeme pro vytvoření nahrávače pro obrázek zvaný "picture":

```
$ rails generate uploader Picture
```

Obrázky nahrané skrze CarrierWave by měly být asociovány se souvisejícím atributem v modelu Active Record, který pak obsahuje jméno souboru obrázku v řetězci. Výsledný vylepšený datový model pro mikroposty pak vypadá následovně (obr. 13.19).

// obr. 13.19 Datový model mikropostů s atributem "picture".

Pro přidání potřebného atributu "picture" do modelu mikropostů si vygenerujeme migraci a zmigrujeme samotnou vývojovou databázi:

```
$ rails generate migration add_picture_to_microposts picture:string
$ rails db:migrate
```

Způsob, jakým řekneme CarrierWave ať si asociuje obrázek s modelem spočívá v použití metody "mount_uploader", který bere jako parametr symbol reprezentující atribut a jméno třídy generovaného nahrávače:

```
mount_uploader :picture, PictureUploader
```

Přidání nahrávače do modelu mikropostů bude vypadat následovně (soubor "app/models/micropost.rb"):

```
class Micropost < ApplicationRecord
  belongs_to :user
  default_scope -> { order(created_at: :desc) }
  mount_uploader :picture, PictureUploader
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
end
```

Na některých systémech bude možná třeba restartovat server Rails, aby testy proběhly zeleně.

Pro zahrnutí nahrávače na domovské stránce potřebujeme zahrnout tag "file_field" ve formuláři mikropostů (soubor "app/views/shared/_micropost_form.html.erb"):

```
<%= form_for(@micropost) do |f| %>
  <%= render 'shared/error_messages', object: f.object %>
  <div class="field">
    <%= f.text_area :content, placeholder: "Compose new micropost..." %>
  </div>
  <%= f.submit "Post", class: "btn btn-primary" %>
  <span class="picture">
    <%= f.file_field :picture %>
  </span>
<% end %>
```

Nakonec ještě potřebujeme přidat "picture" do seznamu atributů, které jsou povoleny skrze web upravovat. Opět to bude zahrnovat použití metody "micropost_params" (soubor "app/controllers/microposts_controller.rb"):

```
class MicropostsController < ApplicationController
  before_action :logged_in_user, only: [:create, :destroy]
  before_action :correct_user,   only: :destroy
  .
  .
  .
  private

    def micropost_params
      params.require(:micropost).permit(:content, :picture)
    end

    def correct_user
      @micropost = current_user.microposts.find_by(id: params[:id])
      redirect_to root_url if @micropost.nil?
    end
end
```

Jakmile byl obrázek nahrán, můžeme ho vykreslit použitím pomocníka "image_tag" v partialu mikropostů. Použijeme i metodu "picture?" pro ujištění, že nezobrazujeme tag obrázku tam, kde žádný není. Metoda samotná je vygenerována díky CarrierWave na základě jména atributu. Výsledek pak bude vypadat jako na obrázku (obr. 13.20). Úprava samotná vypadá následovně (soubor "app/views/microposts/_micropost.html.erb"):

```
<li id="micropost-<%= micropost.id %>">
  <%= link_to gravatar_for(micropost.user, size: 50), micropost.user %>
  <span class="user"><%= link_to micropost.user.name, micropost.user %></span>
  <span class="content">
    <%= micropost.content %>
    <%= image_tag micropost.picture.url if micropost.picture? %>
  </span>
  <span class="timestamp">
    Posted <%= time_ago_in_words(micropost.created_at) %> ago.
    <% if current_user?(micropost.user) %>
      <%= link_to "delete", micropost, method: :delete,
                                       data: { confirm: "You sure?" } %>
    <% end %>
  </span>
</li>
```

// obr. 13.20 Výsledek odeslání mikropostu s obrázkem.

### Validace obrázku

Samotný nahrávač je sice dobrým začátkem, ale trpí na výrazná omezení. Konkrétněji nevynucuje jakékoliv omezení na nahraný soubor, což může způsobit problémy, pokud se uživatelé pokusí nahrát velké soubory, nebo dokonce neplatné typy souborů. Abychom tento problém ošetřili, zahrneme validace na formát a velikost obrázku, a to jak na serveru, tak v prohlížeči.

První validace, která omezuje nahrávání pouze na obrázky, je už přímo zahrnuta v CarrierWave nahrávači. Výsledný kód (který je v souboru "app/uploaders/picture_uploader.rb" jako zakomentovaný návrh) ověřuje, že jméno obrázku končí na platnou obrázkovou příponu (PNG, GIF a obě varianty JPEG):

```
class PictureUploader < CarrierWave::Uploader::Base
  storage :file

  # Override the directory where uploaded files will be stored.
  # This is a sensible default for uploaders that are meant to be mounted:
  def store_dir
    "uploads/#{model.class.to_s.underscore}/#{mounted_as}/#{model.id}"
  end

  # Add a white list of extensions which are allowed to be uploaded.
  def extension_whitelist
    %w(jpg jpeg gif png)
  end
end
```

Druhá validace, která ovládá velikost obrázku, bude přímo v modelu mikropostů. Narozdíl od předchozích validací Rails nemá pro tuto funkcionalitu zabudovaný validátor. Budeme si tedy muset napsat vlastní validaci, kterou nazveme "picture_size" a zadefinujeme ji následovně (soubor "app/models/micropost.rb"):

```
class Micropost < ApplicationRecord
  belongs_to :user
  default_scope -> { order(created_at: :desc) }
  mount_uploader :picture, PictureUploader
  validates :user_id, presence: true
  validates :content, presence: true, length: { maximum: 140 }
  validate  :picture_size

  private

    # Validates the size of an uploaded picture.
    def picture_size
      if picture.size > 5.megabytes
        errors.add(:picture, "should be less than 5MB")
      end
    end
end
```

Vlastní validace zavolá metodu, která odpovídá danému symbolu (:picture_size). V samotném "picture_size" přidáme vlastní chybovou hlášku do seznamu "errors", tedy v tomto případě limit na 5 megabytů.

Zároveň si přidáme dvě ověření na straně klienta. V prvé řadě tedy validaci formátu za pomocí parametru "accept" v input tagu "file_field":

```
<%= f.file_field :picture, accept: 'image/jpeg,image/gif,image/png' %>
```

Platný formát sestává z tzv. [MIME types](https://en.wikipedia.org/wiki/Internet_media_type).

Dále zahrneme trochu JavaScriptu (konkrétněji tedy [jQuery](http://jquery.com/)) pro odeslání varování v případě, že se uživatel pokouší nahrát obrázek, který je příliš velký:

```
$('#micropost_picture').bind('change', function() {
  var size_in_megabytes = this.files[0].size/1024/1024;
  if (size_in_megabytes > 5) {
    alert('Maximum file size is 5MB. Please choose a smaller file.');
  }
});
```

I když se na jQuery primárně nezaměřujeme, dá se odvodit, že kód výše kontroluje prvek na stránce s CSS id "micropost_picture" (podle mřížky #), což je id mikropostového formuláře. Když se prvek s daným id změní, jQuery funkce se spustí a odešle metodu "alert" v případě, že je soubor příliš velký. Celková úprava tedy bude vypadat následovně (soubor "app/views/shared/_micropost_form.html.erb"):

```
<%= form_for(@micropost) do |f| %>
  <%= render 'shared/error_messages', object: f.object %>
  <div class="field">
    <%= f.text_area :content, placeholder: "Compose new micropost..." %>
  </div>
  <%= f.submit "Post", class: "btn btn-primary" %>
  <span class="picture">
    <%= f.file_field :picture, accept: 'image/jpeg,image/gif,image/png' %>
  </span>
<% end %>

<script type="text/javascript">
  $('#micropost_picture').bind('change', function() {
    var size_in_megabytes = this.files[0].size/1024/1024;
    if (size_in_megabytes > 5) {
      alert('Maximum file size is 5MB. Please choose a smaller file.');
    }
  });
</script>
```

Jak můžeme vidět při pokusu o nahrání příliš velkého souboru, samotný kód na straně klienta nevyčistí vstupní pole souboru, a uživatel jej tedy může odkliknout a dál se pokoušet soubor nahrát. S větším zaměřením na jQuery bychom pravděpodobně pole vyčistili, ale důležitá pointa je ta, že front-endový kód jako je tento nemůže uživateli zabránit v pokusu odeslat soubor, který je příliš velký. I kdybychom pole vyčistili, uživatel pořád může editovat JavaScript skrze prohlížeč a odeslat si vlastní POST požadavek. Proto je nutné primárně používat ověřování na straně serveru.

### Změna velikosti obrázku

### Nahrávání obrázku v produkční fázi

# ZAHRNOUT? POKUD JO, DOPISU



## Shrnutí

opet podle pokynu vynechano, pripadne dopisu