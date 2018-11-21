# (Ná)sledování uživatelů

V této kapitole si aplikaci dokončíme přidáním sociální vrstvy, která bude umožňovat následovat ostatní uživatele, a tedy každý uživatel bude mít na domovské stránce výpis statusů mikropostů uživatelů, které sleduje. Začneme vysvětlením vztahů mezi uživateli v rámci modelu a následně si vytvoříme samotné webové rozhraní (včetně úvodu do Ajaxu). Kapitolu zakončíme plně funkčním status feedem.

Tato kapitola obsahuje dosud asi nejobtížnější prvky k pochopení, včetně Ruby/SQL triků pro zprovoznění feedu. Skrze tyto příklady si ukážeme, jak Rails zvládá i poměrně složité datové modely, což se bude hodit pro vývoj vlastních aplikací se specifickými požadavky. Ve finále si tedy i uvedeme odkazy na pokročilejší zdroje informací.

Jelikož je samotný obsah této části obvzlášť náročný, projdeme si nejprve samotný interface. Jako dříve si i teď nejprve předvedeme mockupy stránek. Uživatel tedy začne na své profilové stránce (obr. 14.1) a prokliká se na stránku Uživatelé (obr. 14.2) pro výběr uživatele, kterého chce sledovat. Vybere si druhého uživatele (obr. 14.3) a klikne na "Sledovat". Tlačítko se změní ze "Sledovat" na "Přestat sledovat" a počet sledujících druhého uživatele se zvevdne o jedna (obr. 14.4). První uživatel se vrátí na svou domovskou stránku, kde vidí, že sleduje o jednoho uživatele více, a vidí jeho mikroposty (obr. 14.5).

// obr. 14.1 Profil současného uživatele.

// obr. 14.2 Hledání uživatele ke sledování.

// obr. 14.3 Profil uživatele, kterého chceme sledovat (s tlačítkem pro sledování).

// obr. 14.4 Profil s tlačítkem "přestat sledovat" a zvýšeným počtem sledujících.

// obr. 14.5 Domovská stránka se status feedem a zvýšeným počtem sledovaných.

## Vztahový model

Náš první krok bude obnášet vytvoření datového modelu, který není tak jednoduchý, jak by se mohlo zdát. Tedy, že bude stačit vztah "has_many": uživatel "má tolik" (has many) sledovaných uživatelů a sledujících uživatelů. V tomto přístupu, jak zjistíme, je problém, ale naučíme se jej vyřešit pomocí "has_many :through".

Jako obvykle si vytvoříme novou větev v Gitu:

```
$ git checkout -b following-users
```

### Problém s datovým modelem (a jeho řešení)

V rámci prvního kroku konstruování datového modelu pro sledování uživatelů si předvedeme typický příklad. Mějme uživatele, který následuje druhého uživatele, tedy uživatel 1 následuje uživatele 2 a uživatel 2 je sledován uživatelem 1, tedy uživatel 1 je sledující a uživatel 2 sledovaný. Pomocí běžné pluralizační konvence Rails je sada všech uživatelů, kteří daného uživatele sledují uložena v poli "uzivatel2.followers". Naopak to bohužel nefunguje, a budeme k tomu tedy muset přistoupit obráceně (po vzoru Twitteru, kde je "50 sledovaných, 75 sledujících") skrze pole "uzivatel1.following".

Nabízí se tedy použití modelu z obrázku (obr. 14.6), kde je tabulka "following" a asociace "has_many". Jelikož "user.following" má být seznamem uživatelů, každý řádek tabulky "following" musí být uživatel, identifikován skrze "followed_id", spolu s "follower_id" k navázání spřažení. Navíc, jelikož je každý řádek uživatel, budeme muset zahrnout i jeho ostatní atributy, jako jsou jméno, email, heslo a podobně.

// obr. 14.6 Základní implementace sledování uživatelů.

Problém v tomto datovém modelu je ten, že je hodně nadbytečný: každý řádek obsahuje nejen id sledovaného uživatele, ale i jeho další informace, které už máme v tabulce "users". Co je ještě horší, k modelování sledujících uživatele bychom potřebovali tabulku "followers", která by byla obdobně nadbytečná. Nemluvě o tom, že je tento model noční můrou správy: pokaždé, kdy si uživatel změní např. své jméno, musíme aktualizovat nejen jeho záznam v tabulce "users", ale také každý řádek, který uživatele obsahuje v tabulkách "following" a "followers".

Problém je, že nám chybí jistá abstrakce. Jedním ze způsobů, jak najít vhodný model je zvážit, jak bychom zaimplementovali samotné následování ve webové aplikaci. Architektura REST zahrnuje zdroje, které mohou být vytvořeny a zničeny. Vede nás to tedy k myšlence: Když uživatel následuje jiného uživatele, co se vytváří? Když ho uživatel sledovat přestane, co se ničí? V obou případech jde pouze o samotný vztah mezi dvěma uživateli. Uživatel má tedy mnoho vztahů a "following" nebo "followers" má práve skrze tyto vztahy.

Další detail, který je třeba vzít v potaz je ten, že narozdíl od např. Facebooku, kde jde v podstatě o symetrické přátelství (na úrovni datového modelu) je Twitter postavený na asymetrických vztazích, kde uživatel 1 sice může následovat uživatele 2, ale ne nutně naopak. Pro rozlišení těchto případů použijeme terminologii aktivních a pasivních vztahů: pokud uživatel 1 sleduje uživatele 2, ale ne naopak, má uživatel 1 k druhému aktivní vztah, kdežto obráceně mají vztah pasivní.

Zaměříme se nyní na použití aktivních vztahů pro vytvoření seznamu sledovaných uživatelů a posléze se budeme věnovat i pasivní variantě. Na předchozím obrázku (obr. 14.6) je myšlenka, jak problematiku zaimplementovat: jelikož je každý sledovaný uživatel unikátně označený skrze "followed_id", můžeme převést "following" do tabulky "active_relationships", vynechat detaily uživatele a použít "followed_id" pro získání sledovaného uživatele z tabulky "users". Diagram datového modelu je na následujícím obrázku (obr. 14.7).

// obr. 14.7 Model sledovaných uživatelů skrze aktivní vztah.

Jelikož použijeme stejnou tabulku jak pro aktivní, tak pasivní vztahy, použijeme obecný termín "relationships" (vztahy) pro jméno tabulky, se souvisejícím modelem vztahů. Výsledkem je vztahový datový model na obrázku (obr. 14.8). Později si ukážeme, jak použít vztahový model pro simulaci jak aktivních, tak pasivních vztahových modelů.

// obr. 14.8 Vztahový datový model.

Pro začátek si vygenerujeme migraci, která s předchozím obrázkem souvisí:

```
$ rails generate model Relationship follower_id:integer followed_id:integer
```

Jelikož budeme hledat vztahy pomocí "follower_id" a "followed_id", pro efektivitu přidáme index každému z těchto sloupců (soubor "db/migrate/[timestamp]_create_relationships.rb"):

```
class CreateRelationships < ActiveRecord::Migration[5.0]
  def change
    create_table :relationships do |t|
      t.integer :follower_id
      t.integer :followed_id

      t.timestamps
    end
    add_index :relationships, :follower_id
    add_index :relationships, :followed_id
    add_index :relationships, [:follower_id, :followed_id], unique: true
  end
end
```

Vynucujeme si také víceklíčový index, který zaručuje unikátnost kombinace "follower_id" a "followed_id", a tedy uživatel nebude moci následovat jiného více, než jednou. Naše ovládací rozhraní, jak záhy uvidíme, něco takového stejně nedovolí, ale je dobré takové věci ošetřovat i na úrovni databáze.

Pro vytvoření tabulky "relationships" stačí migraci spustit:

```
$ rails db:migrate
```

### Asociace uživatel/vztah

Před samotnou implementací následování uživatelů budeme potřebovat zavést asociaci mezi uživately a vztahy. Uživatel "has_many" (má tolik) vztahů, a, jelikož vztahy zahrnují dva uživatele - vztah "belongs_to" (náleží) jak sledujícímu, tak sledovanému.

Nové vztahy vytvoříme pomocí asociace, tedy kódem jako

```
user.active_relationships.build(followed_id: ...)
```

V současné chvíli se dá předpokládat, že aplikační kód bude podobný, jako v předchozí kapitole (a také je), ale obsahuje dva klíčové rozdíly.

Zaprvé, v případě asociace uživatel/mikropost jsme mohli napsat

```
class User < ApplicationRecord
  has_many :microposts
  .
  .
  .
end
```

Což funguje díky konvenci Rails, kde hledá model mikropostů odpovídající symbolu ":microposts". V současné chvíli ale chceme napsat

```
has_many :active_relationships
```

i když je daný model zvaný Relationship. Musíme tedy Railsu říct jméno třídy modelu, kterou má hledat.

Zadruhé, dříve jsme napsali v modelu mikropostů toto:

```
class Micropost < ApplicationRecord
  belongs_to :user
  .
  .
  .
end
```

Tato syntaxe fungovala díky tomu, že tabulka "microposts" ma atribut "user_id" pro identifikaci uživatele. Id použité v tomto smyslu pro propojení dvou tabulek je známé jako "cizí klíč" (foreign key), a když je cizím klíčem pro objekt uživatelského modelu "user_id", Rails předpokládá asociaci automaticky: resp. Rails očekává cizí klíč ve tvaru "<třída>_id", kde "<třída>" je jméno třídy zapsané malými písmeny. Ale v našem současném případě, i navzdory tomu, že stále pracujeme s uživateli, je daná uživatel, který následuje druhého identifikovaný cizím klíčem "follower_id", a musíme to tedy Railsu sdělit.

Shrnuto a podtrženo, asociace uživatel/vztah bude vypadat následovně (soubory "app/models/user.rb" a "app/models/relationship.rb")

```
class User < ApplicationRecord
  has_many :microposts, dependent: :destroy
  has_many :active_relationships, class_name:  "Relationship",
                                  foreign_key: "follower_id",
                                  dependent:   :destroy
  .
  .
  .
end
```

```
class Relationship < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
end
```

Asociace "followed" zatím není zapotřebí, ale struktura bude čistší, pokud oboje zaimplementujeme najednou. Nově vzniklé vztahy jsou shrnuty v tabulce:

| **Metoda**                                                   | **Význam**                                                   |
| ------------------------------------------------------------ | ------------------------------------------------------------ |
| `active_relationship.follower`                               | Vrátí sledujícího                                            |
| `active_relationship.followed`                               | Vrátí sledovaného                                            |
| `user.active_relationships.create(followed_id: other_user.id)` | Vytvoří aktivní vztah asociovaný s "user"                    |
| `user.active_relationships.create!(followed_id: other_user.id)` | Vytvoří aktivní vztah asociovaný s "user" (s výjimkou při selhání) |
| `user.active_relationships.build(followed_id: other_user.id)` | Vrátí nový vztahový objekt asociovaný s "user"               |

### Validace vztahů

Než se posuneme dále, přidáme několik validací vztahového modelu. Testy (soubor "test/models/relationship_test.rb") a aplikační kód (soubor "app/models/relationship.rb") jsou poměrně přímočaré. Jako u generování uživatelských fixtur i zde musíme odstranit jejich vygenerovaný obsah kvůli unikátnosti (soubor "test/fixtures/relationships.yml").

```
require 'test_helper'

class RelationshipTest < ActiveSupport::TestCase

  def setup
    @relationship = Relationship.new(follower_id: users(:michael).id,
                                     followed_id: users(:archer).id)
  end

  test "should be valid" do
    assert @relationship.valid?
  end

  test "should require a follower_id" do
    @relationship.follower_id = nil
    assert_not @relationship.valid?
  end

  test "should require a followed_id" do
    @relationship.followed_id = nil
    assert_not @relationship.valid?
  end
end
```

```
class Relationship < ApplicationRecord
  belongs_to :follower, class_name: "User"
  belongs_to :followed, class_name: "User"
  validates :follower_id, presence: true
  validates :followed_id, presence: true
end
```

```
# empty
```

Testy by měly proběhnout zeleně:

```
$ rails test
```

### Sledovaní uživatelé

Došli jsme k samotnému jádru asociací vztahů: "following" a "followers". Poprvé použijeme "has_many :through", tedy že uživatel má množství sledujících skrze vztahy. Asociace "has_many :through" v Rails hledá cizí klíč odpovídající jednotné verzi asociace. Jinak řečeno, na kód

```
has_many :followeds, through: :active_relationships
```

bude Rails reagovat tak, že uvidí "followeds" a použije jednotné "followed", přičemž vytvoří kolekci za pomocí "followed_id" v tabulce "relationships". Jak jsme si zmínili dříve, "user.followeds" je přecejen trochu kostrbaté, a proto použijeme "user.following". Rails nám samozřejmě umožňuje takové chování "předrátovat", v tomto případě tedy použít parametr "source" (zdroj), který Rails explicitně sdělí, že zdroj pole "following" je sada "followed" id. Úprava tedy bude vypadat následovně (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  has_many :microposts, dependent: :destroy
  has_many :active_relationships, class_name:  "Relationship",
                                  foreign_key: "follower_id",
                                  dependent:   :destroy
  has_many :following, through: :active_relationships, source: :followed
  .
  .
  .
end
```

Takto definovaná asociace vede k praktické kombinace Active Record a chování, které je podobné poli. Kupříkladu můžeme ověřit, zda kolekce sledovaných uživatelů obsahuje dalšího uživatele pomocí metody "include?", nebo najít objekty skrze asociaci:

```
user.following.include?(other_user)
user.following.find(other_user)
```

Také můžeme přidávat a mazat prvky, jako při práci s poli:

```
user.following << other_user
user.following.delete(other_user)
```

I když můžeme v mnoha případech zacházet s "following" jako s polem, Rails je dostatečně chytrý, aby věděl, jak se k věcem chovat "pod kapotou". Například, kód

```
following.include?(other_user)
```

vypadá, že by měl vytáhnout všechny sledované uživatele z databáze a aplikovat metodu "include?", ale ve skutečnosti (kvůli efektivity) Rails zařídí porovnání přímo v databázi.

Pro práci se vztahy zahrneme metody "follow" a "unfollow", které budeme moci přímo psát (např. "user.follow(jiny_uzivatel)"). Také přidáme související booleanskou metodu "following?" pro testování, zda jeden uživatel sleduje druhého.

Právě v takovéto situaci je dobré si nejprve napsat testy. Důvodem je to, že jsme poměrně daleko od napsání funkčního webového rozhraní pro sledování uživatelů, ale pokračovat dále bez nějakého druhu klienta pro kód, který vyvíjíme, je obtížné. Napíšeme si tedy krátky test pro uživatelský model, ve kterém použijeme "following?" pro ujištění, že uživatel nesleduje jiného, použijeme "follow" pro sledování jiného uživatele, použijeme "following?" pro ověření, že operace uspěla a nakonec "unfollow" a ověření, že fungovalo. Testy tedy budou vypadat následovně (soubor "test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "should follow and unfollow a user" do
    michael = users(:michael)
    archer  = users(:archer)
    assert_not michael.following?(archer)
    michael.follow(archer)
    assert michael.following?(archer)
    michael.unfollow(archer)
    assert_not michael.following?(archer)
  end
end
```

Jelikož se můžeme chovat k asociaci "following" jako k poli, můžeme napsat metody "follow", "unfollow" a "following?" následovně (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  .
  .
  .
  def feed
    .
    .
    .
  end

  # Follows a user.
  def follow(other_user)
    following << other_user
  end

  # Unfollows a user.
  def unfollow(other_user)
    following.delete(other_user)
  end

  # Returns true if the current user is following the other user.
  def following?(other_user)
    following.include?(other_user)
  end

  private
  .
  .
  .
end
```

S tímto kódem by měly testy proběhnout zeleně:

```
$ rails test
```

### Sledující

Poslední kousek vztahové skládačky je přidání metody "user.followers", která půjde ruku v ruce s "user.following". Všechny informace potřebné pro vytažení pole sledujících už existuje v tabulce "relationships" (ke které přistupujeme jako "active_relationships"). Technika je tedy obdobný, jako u sledovaných uživatelů, jen role "follower_id" a "followed_id" jsou prohozeny, a stejně tak "passive_relationships" namísto "active_relationships". Datový model je na obrázku (obr. 14.9).

// obr. 14.9 Model pro sledující uživatele skrze pasivní vztah.

Implementace datového modelu je také obdobná (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  has_many :microposts, dependent: :destroy
  has_many :active_relationships,  class_name:  "Relationship",
                                   foreign_key: "follower_id",
                                   dependent:   :destroy
  has_many :passive_relationships, class_name:  "Relationship",
                                   foreign_key: "followed_id",
                                   dependent:   :destroy
  has_many :following, through: :active_relationships,  source: :followed
  has_many :followers, through: :passive_relationships, source: :follower
  .
  .
  .
end
```

Mimojiné jsme mohli opomenout klíč ":source" pro "followers" a jednoduše použít

```
has_many :followers, through: :passive_relationships
```

To proto, že v případe atributu ":followers" Rails změní "followers" na jednotné číslo a automaticky vyhledá cizí klíč "follower_id". Necháme ho tam ale pro zdůraznění paralely s asociací "has_many :following".

Můžeme také jednoduče otestovat datový model pomocí metody "followers.include?" (soubor "test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "should follow and unfollow a user" do
    michael  = users(:michael)
    archer   = users(:archer)
    assert_not michael.following?(archer)
    michael.follow(archer)
    assert michael.following?(archer)
    assert archer.followers.include?(michael)
    michael.unfollow(archer)
    assert_not michael.following?(archer)
  end
end
```

I když vlastně přidáváme jen jeden řádek oproti předchozímu testu, pořád je hodně věcí, které musí projiít správně, a jde tedy o poměrně citlivý test předchozího kódu.

Testy by nicméně měly proběhnout zeleně:

```
$ rails test
```

## Webové prostředí pro sledování uživatelů

Předchozí část byla poněkud náročná na naše dovednosti datového modelování, a abychom přecejen správně pochopili, co a jak funguje, bude nejrozumnější použít asociace ve webovém rozhraní.

Ze začátku této kapitoly jsme si načrtli vzhled stránek, které budou souviset se sledováním uživatelů. V této části si zaimplementujeme základní rozhraní pro jejich sledování a zrušení sledování. Také vytvoříme samostatné stránky pro zobrazení sledovaných a sledujících. Nakonec naší aplikaci završíme přidáním status feedu.

### Vzorová sledovací data

Opět si naplníme databázi (skrze "rails db:seed") vzorovými daty. Díky tomu můzeme navrhnout vzhled stránek v rámci prvního kroku a problematiku back-endu rešit až posléze.

Kód pro naplnění databáze vypadá následovně (soubor "db/seeds.rb"). První uživatel sleduje uživatele 3 až 51 a následně uživatele 4 až 41 sledují onoho uživatele zpět. Výsledné vztahy pak budou stačit pro vývoj rozhraní aplikace.

```
# Uzivatele
User.create!(name:  "Example User",
             email: "example@railstutorial.org",
             password:              "foobar",
             password_confirmation: "foobar",
             admin:     true,
             activated: true,
             activated_at: Time.zone.now)

99.times do |n|
  name  = Faker::Name.name
  email = "example-#{n+1}@railstutorial.org"
  password = "password"
  User.create!(name:  name,
               email: email,
               password:              password,
               password_confirmation: password,
               activated: true,
               activated_at: Time.zone.now)
end

# Mikroposty
users = User.order(:created_at).take(6)
50.times do
  content = Faker::Lorem.sentence(5)
  users.each { |user| user.microposts.create!(content: content) }
end

# Sledovaci vztahy
users = User.all
user  = users.first
following = users[2..50]
followers = users[3..40]
following.each { |followed| user.follow(followed) }
followers.each { |follower| follower.follow(user) }
```

Pro provedení kódu stačí, jako obvykle, zresetovat a znovu naplnit databázi:

```
$ rails db:migrate:reset
$ rails db:seed
```

### Statistiky a sledovací formulář

Když už naši vzoroví uživatelé mají jak sledující, tak sledované, bude potřeba aktualizovat profilovou a domovskou stránku. Začneme partialem pro zobrazení statistik sledujících a sledovaných právě na profilových a domovských stránkách. Dále přidáme formulář "sledovat/přestat sledovat" a poté dedikované stránky pro zobrazování sledujících a sledovaných. Na obrázku (obr. 14.10) je zobrazeno, jak budou statistiky vypadat.

// obr. 14.10 Mockup partialu statistik.

Statistiky sestávají z počtu uživatelů, které současný uživatel sleduje, a počtu uživatelů, kteří sledují daného uživatele (každý s odkazem na svou samostatnou stránku). Samotné cesty si vytvoříme nyní, jelikož už máme větší zkušenosti se směrováním (soubor "config/routes.rb").

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
  resources :users do
    member do
      get :following, :followers
    end
  end
  resources :account_activations, only: [:edit]
  resources :password_resets,     only: [:new, :create, :edit, :update]
  resources :microposts,          only: [:create, :destroy]
end
```

Dá se uhádnout, že URL pro sledující a sledované budou vypadat ve stylu "/users/1/following" a "/users/1/followers" a to je ostatně přesně to, co předchozí kód dělá. Jelikož budou obě stránky zobrazovat data, vhodné HTTP sloveso je GET požadavek, a tedy použijeme metodu "get" pro vhodnou odezvu daných URL. Metoda "member" zase zařizuje, že cesty budou vhodně reagovat na URL obsahující uživatelovo id. Druhá možnost, "collection", funguje bez id, takže

```
resources :users do
  collection do
    get :tigers
  end
end
```

by reagovala na URL /users/tigers (zřejmě tak, že by zobrazila všechny tygry v naší aplikaci).

Tabulka nově vygenerovaných cest vypadá následovně. Všimněme si jmenných cest, které záhy použijeme.

| **HTTP požadavek** | **URL**            | **Akce**    | **Jmenná cesta**         |
| ------------------ | ------------------ | ----------- | ------------------------ |
| `GET`              | /users/1/following | `following` | `following_user_path(1)` |
| `GET`              | /users/1/followers | `followers` | `followers_user_path(1)` |

Jakmile máme zadefinované cesty, můžeme zadefinovat i samotný partial, což zahrnuje několik odkazů uvnitř tagu <div> (soubor "app/views/shared/_stats.html.erb").

```
<% @user ||= current_user %>
<div class="stats">
  <a href="<%= following_user_path(@user) %>">
    <strong id="following" class="stat">
      <%= @user.following.count %>
    </strong>
    following
  </a>
  <a href="<%= followers_user_path(@user) %>">
    <strong id="followers" class="stat">
      <%= @user.followers.count %>
    </strong>
    followers
  </a>
</div>
```

Jelikož budeme zahrnovat statistiky jak na uživatelské stránce, tak na domovské, první řádek vybere tu správnou za pomocí

```
<% @user ||= current_user %>
```

Jak jsme probírali dříve, kód nedělá nic pokud "@user" není "nil" (tedy jako na profilové stránce), ale když je (jako na stránce domovské), nastaví "@user" na současného uživatele. Samotné výpočty jsou kalkulovány skrze asociace pomocí

```
@user.following.count
```

a

```
@user.followers.count
```

Srovnejme si to se součtem mikropostů, kde jsme napsali

```
@user.microposts.count
```

pro jejich sečtení. I v tomto případě Rails spočítal sumu kvůli efektivitě přímo v databázi.

Poslední drobnost, která stojí za zmínku je přítomnost CSS id na některých prvcích, jako třeba

```
<strong id="following" class="stat">
...
</strong>
```

Využijeme toho v rámci implementace Ajaxu, který přistupuje k prvkům na stránce za pomocí jejich unikátních id.

Když máme partial hotový, je samotné zahrnutí statistik na domovské stránce jednoduchá záležitost (soubor "app/views/static_pages/home.html.erb"):

```
<% if logged_in? %>
  <div class="row">
    <aside class="col-md-4">
      <section class="user_info">
        <%= render 'shared/user_info' %>
      </section>
      <section class="stats">
        <%= render 'shared/stats' %>
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

Pro nastylování statistik použijeme trochu SCSS (soubor "app/assets/stylesheets/custom.scss"). Výsledná podoba domovské stránky je pak na obrázku (obr. 14.11).

```
.
.
.
/* sidebar */
.
.
.
.gravatar {
  float: left;
  margin-right: 10px;
}

.gravatar_edit {
  margin-top: 15px;
}

.stats {
  overflow: auto;
  margin-top: 0;
  padding: 0;
  a {
    float: left;
    padding: 0 10px;
    border-left: 1px solid $gray-lighter;
    color: gray;
    &:first-child {
      padding-left: 0;
      border: 0;
    }
    &:hover {
      text-decoration: none;
      color: blue;
    }
  }
  strong {
    display: block;
  }
}

.user_avatars {
  overflow: auto;
  margin-top: 10px;
  .gravatar {
    margin: 1px 1px;
  }
  a {
    padding: 0;
  }
}

.users.follow {
  padding: 0;
}

/* forms */
.
.
.
```

// obr. 14.11 Domovská stránka s novým partialem.

Partial statistik si vykreslíme i na profilové stránce záhy, ale první si vytvoříme partial pro tlačítko zapnutí/vypnutí sledování (soubor "app/views/users/_follow_form.html.erb"):

```
<% unless current_user?(@user) %>
  <div id="follow_form">
  <% if current_user.following?(@user) %>
    <%= render 'unfollow' %>
  <% else %>
    <%= render 'follow' %>
  <% end %>
  </div>
<% end %>
```

Tento partial nicméně nedělá nic jiného, než že přehazuje reálnou práci na partialy "follow" a "unfollow", které budou potřebovat nové cesty v rámci zdroje vztahů (soubor "config/routes.rb"):

```
Rails.application.routes.draw do
  root                'static_pages#home'
  get    'help'    => 'static_pages#help'
  get    'about'   => 'static_pages#about'
  get    'contact' => 'static_pages#contact'
  get    'signup'  => 'users#new'
  get    'login'   => 'sessions#new'
  post   'login'   => 'sessions#create'
  delete 'logout'  => 'sessions#destroy'
  resources :users do
    member do
      get :following, :followers
    end
  end
  resources :account_activations, only: [:edit]
  resources :password_resets,     only: [:new, :create, :edit, :update]
  resources :microposts,          only: [:create, :destroy]
  resources :relationships,       only: [:create, :destroy]
end
```

Samotné follow/unfollow partialy vypadají následovně (soubory "app/views/users/\_follow.html.erb" a "app/views/users/\_unfollow.html.erb"):

```
<%= form_for(current_user.active_relationships.build) do |f| %>
  <div><%= hidden_field_tag :followed_id, @user.id %></div>
  <%= f.submit "Follow", class: "btn btn-primary" %>
<% end %>
```

```
<%= form_for(current_user.active_relationships.find_by(followed_id: @user.id),
             html: { method: :delete }) do |f| %>
  <%= f.submit "Unfollow", class: "btn" %>
<% end %>
```

Oba formuláře používají "form_for" pro úpravu objektu vztahového modelu; největší rozdílem je mezi nimi to, že follow vytváří nový vztah, kdežto unfollow ho hledá. První jmenovaný pošle POST požadavek na ovladač vztahů pro vytvoření (create) vztahu, kdežto druhý pošle požadavek DELETE pro jeho zničení (destroy). Tyto akce si napíšeme záhy. Samotný formulář také nemá jiný obsah, než tlačítko, ale stále potřebuje odeslat "followed_id" ovladači. Používáme tedy "hidden_field_tag" metodu, která vytvoří HTML v podobě

```
<input id="followed_id" name="followed_id" type="hidden" value="3" />
```

Relevantní informace se tedy na stránce vyskytnou, ale prohlížeč je nezobrazí.

Můžeme nyní zahrnout formulář a statistiky na profilovou stránku uživatele jednoduše pomocí vykreslení partialů (soubor "app/views/users/show.html.erb"). Profily s tlačítky pro zahájení a ukončení sledování jsou na obrázcích (obr. 14.12 a 14.13).

```
<% provide(:title, @user.name) %>
<div class="row">
  <aside class="col-md-4">
    <section>
      <h1>
        <%= gravatar_for @user %>
        <%= @user.name %>
      </h1>
    </section>
    <section class="stats">
      <%= render 'shared/stats' %>
    </section>
  </aside>
  <div class="col-md-8">
    <%= render 'follow_form' if logged_in? %>
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

// obr. 14.12 Uživatelský profil s tlačítkem pro sledování.

// obr. 14.13 Uživatelský profil s tlačítkem pro ukončení sledování.

Samotná tlačítka zprovozníme brzy, ale nejprve dokončíme HTML rozhraní vytvořením stránek pro sledující a sledované.

### Stránky "Sledující" a "Sledovaní"

Stránky pro zobrazení sledujících a sledovaných uživatelů jsou v podstatě hybridem uživatelské profilové stránky a indexem uživatelů, s boční lištou uživatelských informací a seznamem uživatelů. Přidáme také menší uživatelské profilové obrázky s odkazy v boční liště. Mockupy odpovídající těmto požadavkům jsou na obrázcích (obr. 14.14 a 14.15).

// obr. 14.14 Mockup stránky sledovaných.

// obr. 14.15 Mockup stránky sledujících.

První krok bude spočívat ve zprovoznění odkazů "sledující" a "sledovaní". Budeme opět následovat princip Twitteru a vyžadovat přihlášení uživatele. Jako u většiny případů kontroly přístupu budeme i tady psát nejprve testy, kde použijeme i nové jmenné cesty (soubor "test/controllers/users_controller_test.rb"):

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
    @other_user = users(:archer)
  end
  .
  .
  .
  test "should redirect following when not logged in" do
    get following_user_path(@user)
    assert_redirected_to login_url
  end

  test "should redirect followers when not logged in" do
    get followers_user_path(@user)
    assert_redirected_to login_url
  end
end
```

Jediná ošemetná záležitost v rámci implementace je zjištění, že potřebujeme dvě nové akce pro uživatelský ovladač. Podle definovaných cest je potřebujeme nazvat "following" a "followers". Každá akce potřebuje nastavit název, vyhledat uživatele, získat uživatele skrze "@user.following" nebo "@user.followers" (ve stránkované podobě) a poté vykreslit stránku samotnou. Akce tedy budou vypadat následovně (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy,
                                        :following, :followers]
  .
  .
  .
  def following
    @title = "Following"
    @user  = User.find(params[:id])
    @users = @user.following.paginate(page: params[:page])
    render 'show_follow'
  end

  def followers
    @title = "Followers"
    @user  = User.find(params[:id])
    @users = @user.followers.paginate(page: params[:page])
    render 'show_follow'
  end

  private
  .
  .
  .
end
```

Jak jsme viděli hodněkrát dříve, běžná zvyklost Rails spočíváa v implicntím vykreslení šablony, která odpovídá akci, tedy jako vykreslení "show.html.erb" na konci akce "show". Obě naše nově zadefinované akce ale volají "render" explicitně, v tomto případě tedy k vykreslení pohledu zvaného "show_follow", který musíme vytvořit. Důvodem pro běžný pohled je, že ERb je téměř identické pro oba případy, a následující výpis pokrývá oba (soubor "app/views/users/show_follow.html.erb"):

```
<% provide(:title, @title) %>
<div class="row">
  <aside class="col-md-4">
    <section class="user_info">
      <%= gravatar_for @user %>
      <h1><%= @user.name %></h1>
      <span><%= link_to "view my profile", @user %></span>
      <span><b>Microposts:</b> <%= @user.microposts.count %></span>
    </section>
    <section class="stats">
      <%= render 'shared/stats' %>
      <% if @users.any? %>
        <div class="user_avatars">
          <% @users.each do |user| %>
            <%= link_to gravatar_for(user, size: 30), user %>
          <% end %>
        </div>
      <% end %>
    </section>
  </aside>
  <div class="col-md-8">
    <h3><%= @title %></h3>
    <% if @users.any? %>
      <ul class="users follow">
        <%= render @users %>
      </ul>
      <%= will_paginate %>
    <% end %>
  </div>
</div>
```

Akce tedy vykreslují pohled ve dvou kontextech, "following" a "followers" (sledovaní a sledující), kde výsledek vypadá jako na obrázcích (obr 14.16 a 14.17). Nic z předchozího kód nepoužívá současného uživatele, a tedy odkazy budou fungovat i pro jakéhokoliv jiného (viz obr. 14.18).

// obr. 14.16 Zobrazení uživatelů, které daný uživatel sleduje.

// obr. 14.17 Zobrazení uživatelů, kteří daného uživatele sledují.

// obr. 14.18 Zobrazení sledujících jiného uživatele.

Testy by nyní měly proběhnout zeleně (mj. i díky before filtru):

```
$ rails test
```

Pro otestování vykreslení "show_follow" si napíšeme pár krátkých integračních testů pro ověření přítomnosti funkčních stránek sledujících a sledovaných. Budou sloužit jen pro hrubé ověření, jelikož je stejně detailní testování HTML struktury náchylné k problémum a tedy krapet kontraproduktivní. Chceme tedy zjistit, zda je číslo sledujících/sledovaných správně zobrazeno a že se zobrazují správné URL.

Nejprve si tedy vygenerujeme samotný integrační test:

```
$ rails generate integration_test following
      invoke  test_unit
      create    test/integration/following_test.rb
```

Dále budeme potřebovat posbírat nějaká testovací data, která můžeme získat přidáním vztahových fixtur pro vytvoření vztahů. Vzpomeňme si, že lze použít kód jako

```
orange:
  content: "I just ate an orange!"
  created_at: <%= 10.minutes.ago %>
  user: michael
```

pro asociování mikropostu s daným uživatelem. Zvláště můžeme napsat

```
user: michael
```

namísto

```
user_id: 1
```

Ve vztahových fixturách to bude vypadat následovně (soubor "test/fixtures/relationships.yml"):

```
one:
  follower: michael
  followed: lana

two:
  follower: michael
  followed: malory

three:
  follower: lana
  followed: michael

four:
  follower: archer
  followed: michael
```

Pro ověření správného spočítání můžeme použít stejnou "assert_match" metodu, kterou jsme použili pro testování zobrazení počtu mikropostů na profilové stránce uživatele. Test tedy bude vypadat následovně (soubor "test/integration/following_test.rb"):

```
require 'test_helper'

class FollowingTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
    log_in_as(@user)
  end

  test "following page" do
    get following_user_path(@user)
    assert_not @user.following.empty?
    assert_match @user.following.count.to_s, response.body
    @user.following.each do |user|
      assert_select "a[href=?]", user_path(user)
    end
  end

  test "followers page" do
    get followers_user_path(@user)
    assert_not @user.followers.empty?
    assert_match @user.followers.count.to_s, response.body
    @user.followers.each do |user|
      assert_select "a[href=?]", user_path(user)
    end
  end
end
```

Zahrnuli jsme i

```
assert_not @user.following.empty?
```

což ověřuje, že

```
@user.following.each do |user|
  assert_select "a[href=?]", user_path(user)
end
```

není ["vakuově pravdivé"](https://en.wikipedia.org/wiki/Vacuous_truth). Jinak řečeno, pokud by "" bylo true, ani jeden "assert_select" by se ve smyčce nespustil a testy by prošly a daly nám falešný pocit bezpečí.

Sada by nyní měla proběhnout zeleně:

```
$ rails test
```

### Funkční tlačítko pro sledování

Když už se nám povedlo správně zprovoznit pohledy, je na čase dát do pořádku i tlačítko pro sledování. Jelikož samotné zahájení a ukončení sledování zahrnute vytváření a ničení vztahů, bude potřeba vztahový ovladač, který vygenerujeme jako obvykle:

```
$ rails generate controller Relationships
```

I když se zdá, že vynucování kontroly přístupu v akcích vztahového ovladače nebude hrát příliš velkou roli, budeme i tak pokračovat v naší předchozí praxi co nejrychlejší implementace bezpečnostního modelu. Konkrétněji tedy budeme chtít ověřit, že pokusy o přístup k akcím ovladače vyžadují přihlášeného uživatele, ale zároveň nezmení počet vztahů jako takových (soubor "test/controllers/relationships_controller_test.rb")

```
require 'test_helper'

class RelationshipsControllerTest < ActionDispatch::IntegrationTest

  test "create should require logged-in user" do
    assert_no_difference 'Relationship.count' do
      post relationships_path
    end
    assert_redirected_to login_url
  end

  test "destroy should require logged-in user" do
    assert_no_difference 'Relationship.count' do
      delete relationship_path(relationships(:one))
    end
    assert_redirected_to login_url
  end
end
```

Výše vypsaný test projde tehdy, když přidáme before filtr "logged_in_user" (soubor "app/controllers/relationships_controller.rb"):

```
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
  end

  def destroy
  end
end
```

Pro zfunkčnění sledovacího tlačítka budeme potřebovat najít uživatele asociovaného s "followed_id" v souvisejícím formuláři a poté použít vhodnou metodu ("follow" nebo "unfollow"). Samotná implementace tedy vypadá následovně (soubor "app/controllers/relationships_controller.rb"):

```
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
    user = User.find(params[:followed_id])
    current_user.follow(user)
    redirect_to user
  end

  def destroy
    user = Relationship.find(params[:id]).followed
    current_user.unfollow(user)
    redirect_to user
  end
end
```

Jak můžeme vidět, bezpečnostní "hrozba" v případě pokusu o akci nepřihlášeným uživatelem je minimální, jelikož "current_user" bude "nil" a druhý řádek akce vyvolá výjimku, která bude mít za následek chybu, ale nikterak nepoškodí aplikaci nebo její data. Nedá se na to ovšem obecně spoléhat a proto jsme udělali onen krok navíc a přidali další vrstvu bezpečnosti.

Základní follow/unfollow funkcionalita je tedy kompletní a jakýkoliv uživatel může začít a přestat následovat jiného, jak si lze ověřit i kliknutím na dané tlačítko v prohlížeči (obr. 14.19 a 14.20). Integrační test si napíšeme později.

// obr. 14.19 Nesledovaný uživatel.

// obr. 14.20 Výsledek zahájení sledování uživatele.

### Úprava tlačítka skrze Ajax

Ačkoliv máme samotnou implementaci sledování za sebou, zbývá ještě jedna drobnost, kterou by bylo dobré přidat než začneme pracovat na status feedu. Akce "create" i "destroy" v ovladači vztahů jednoduše přesměrovávají zpět na původní profil. Jinak řečeno, uživatel začne na profilové stránce jiného uživatele, dá "sledovat", a je okamžite přesměrován zpátky na původní stránku. Je rozumné se ptát proč by vůbec musel uživatel stránku opouštět.

Tento problém řeší Ajax, který umožňuje webovým stránkam posílat požadavky asynchronně k serveru bez opouštění stránky. Jelikož přidávání Ajaxu do webových formulářů je běžná záležitost, Rails umožňuje jeho snadnou implementaci. Aktualizace samotného formuláře je jednoduchá, stačí změnit

```
form_for
```

na

```
form_for ..., remote: true
```

a Rails "[automagicky](http://catb.org/jargon/html/A/automagically.html)" použije Ajax. Aktualizované partialy jsou na následujících výpisech (soubory "app/views/users/\_follow.html.erb" a "app/views/users/\_unfollow.html.erb").

```
<%= form_for(current_user.active_relationships.build, remote: true) do |f| %>
  <div><%= hidden_field_tag :followed_id, @user.id %></div>
  <%= f.submit "Follow", class: "btn btn-primary" %>
<% end %>
```

```
<%= form_for(current_user.active_relationships.find_by(followed_id: @user.id),
             html: { method: :delete },
             remote: true) do |f| %>
  <%= f.submit "Unfollow", class: "btn" %>
<% end %>
```

HTML vygenerované tímto ERb není nikterak zvlášť relevantní, ale můžeme nakouknout do schématického pohledu (drobnosti se mohou lišit):

```
<form action="/relationships/117" class="edit_relationship" data-remote="true"
      id="edit_relationship_117" method="post">
  .
  .
  .
</form>
```

Uvnitř tagu form je nastavená proměnná "data-remote="true"", což řekne Railsu, aby formulář nechal ovládat JavaScriptem.

Po aktualizaci formuláře nyní potřebujeme nastavit ovladač vztahů tak, aby reagoval na požadavky Ajaxu. Můžeme použít metodu "respond_to", která bude vhodně reagovat podle typu požadavku. Obecný vzorec vypadá následovně:

```
respond_to do |format|
  format.html { redirect_to user }
  format.js
end
```

Syntaxe je potenciálně matoucí a je důležité si uvědomit, že výše zmíněný kód spustí pouze jeden z řádků. (V daném smyslu je "respond_to" spíše if-then-else konstrukce, než série řádků.) Úprava ovladače vztahů bude tedy zahrnovat přidání "respond_to" do akcí "create" a "destroy". Výsledná úprava vypadá následovně (soubor "app/controllers/relationships_controller.rb"):

```
class RelationshipsController < ApplicationController
  before_action :logged_in_user

  def create
    @user = User.find(params[:followed_id])
    current_user.follow(@user)
    respond_to do |format|
      format.html { redirect_to @user }
      format.js
    end
  end

  def destroy
    @user = Relationship.find(params[:id]).followed
    current_user.unfollow(@user)
    respond_to do |format|
      format.html { redirect_to @user }
      format.js
    end
  end
end
```

Pro případ prohlížečů, které mají vypnutý JavaScript musíme udělat ještě jednu drobnou úpravu (soubor "config/application.rb"):

```
require File.expand_path('../boot', __FILE__)
.
.
.
module SampleApp
  class Application < Rails::Application
    .
    .
    .
    # Include the authenticity token in remote forms.
    config.action_view.embed_authenticity_token_in_remote_forms = true
  end
end
```

V případě Ajaxového požadavku Rails automaticky zavolá souborovou variantu "JavaScript embedded Ruby" (".js.erb") se stejným jménem, jako je akce, tedy "create.js.erb" nebo "destroy.js.erb". Takové soubory nám umožní míchat Javascript a embedded Ruby pro provádění akcí na současné stránce. Právě tyto soubory musíme vytvořit a upravit aby docházelo ke správné aktualizaci profilové stránky po zahájení/ukončení sledování.

Uvnitř souboru JS-ERb Rails automaticky poskytne pomocníky [jQuery](http://jquery.com/) pro manipulaci stránky pomocí  [dokumento-objektového modelu (DOM)](http://www.w3.org/DOM/). Knihovna jQuery poskytuje velké množství metod pro manipulaci s DOMem, ale potřebovat budeme jen dvě. V prvé řadě se budeme muset seznámit s "dolarovou syntaxí" pro přístup k prvku DOM podle unikátního CSS id. Například, pro práci s prvkem "follow_form" použijeme syntaxi

```
$("#follow_form")
```

Syntaxe používá, po vzoru CSS, symbol # pro indikaci CSS id. Stejně tak třídy jsou, jako v CSS, označeny tečkou.

Druhá potřebná metoda je "html", která aktualizuje HTML uvnitř souvisejícího prvku obsahem svého parametru. Kupříkladu, pro nahrazení celého formuláře řetězcem "foobar" napíšeme

```
$("#follow_form").html("foobar")
```

Narozdíl od čistých JavaScriptových souborů umožňují JS-ERb soubory použití vnořeného Ruby, které použijeme v souboru "create.js.erb" pro aktualizaci formuláře partialem "unfollow" (který chceme vidět po úspěšném zahájení sledování) a aktualizujeme počet sledujících. Použijeme i metodu "escape_javascript", která je potřeba k ošetření výsledků při vkládání HTML v JavaScriptovém souboru (soubor "app/views/relationships/create.js.erb").

```
$("#follow_form").html("<%= escape_javascript(render('users/unfollow')) %>");
$("#followers").html('<%= @user.followers.count %>');
```

I druhá úprava se nese v obdobném duchu (soubor "app/views/relationships/destroy.js.erb"):

```
$("#follow_form").html("<%= escape_javascript(render('users/follow')) %>");
$("#followers").html('<%= @user.followers.count %>');
```

Nyní si můžeme v prohlížeči ověřit, že lze začít a přestat sledovat uživatele bez nutnosti aktualizovat celou stránku.

### Testy sledování

Jelikož už tlačítka fungují, jak mají, napíšeme si pár jednoduchých testů. Pro sledování uživatele ověříme skrze cestu vztahů, že se počet sledovaných uživatelů zvedl o 1.

```
assert_difference '@user.following.count', 1 do
  post relationships_path, params: { followed_id: @other.id }
end
```

Toto ověřuje standardní implementaci, ale testováí Ajaxové verze je prakticky totožné; jediným rozdílem je přidání možnosti "xhr: true":

```
assert_difference '@user.following.count', 1 do
  post relationships_path, params: { followed_id: @other.id }, xhr: true
end
```

Zkratka "xhr" znamená XmlHttpRequest; nastavení možnosti "xhr" na "true" zapojí v rámci testu i Ajaxový požadavek a blok "respond_to" poté spustí vhodnou JavaScriptovou metodu.

Stejná struktura se aplikuje i na mazání uživatelů, jen s "delete" namísto "post". Ověříme, že počet sledovaných uživatelů klesne o 1 a zahrneme vztah a id sledovaného uživatele.

```
assert_difference '@user.following.count', -1 do
  delete relationship_path(relationship)
end
```

a

```
assert_difference '@user.following.count', -1 do
  delete relationship_path(relationship), xhr: true
end
```

Když obě myšlenky spojíme dohromady, vyjdou nám z toho následující testy (soubor "test/integration/following_test.rb"):

```
require 'test_helper'

class FollowingTest < ActionDispatch::IntegrationTest

  def setup
    @user  = users(:michael)
    @other = users(:archer)
    log_in_as(@user)
  end
  .
  .
  .
  test "should follow a user the standard way" do
    assert_difference '@user.following.count', 1 do
      post relationships_path, params: { followed_id: @other.id }
    end
  end

  test "should follow a user with Ajax" do
    assert_difference '@user.following.count', 1 do
      post relationships_path, xhr: true, params: { followed_id: @other.id }
    end
  end

  test "should unfollow a user the standard way" do
    @user.follow(@other)
    relationship = @user.active_relationships.find_by(followed_id: @other.id)
    assert_difference '@user.following.count', -1 do
      delete relationship_path(relationship)
    end
  end

  test "should unfollow a user with Ajax" do
    @user.follow(@other)
    relationship = @user.active_relationships.find_by(followed_id: @other.id)
    assert_difference '@user.following.count', -1 do
      delete relationship_path(relationship), xhr: true
    end
  end
end
```

Testy by nyní měly proběhnout zeleně:

```
$ rails test
```



## Status feed

Dostali jsme se k vrcholu naší aplikace: status feed mikropostů. Tato část bude, celkem pochopitelně, obsahovat nejnáročnější kroky. Stavět budeme na feedu, který jsme si vytvořili minule, a to tak, že sestavíme pole mikropostů od uživatelů, které daný uživatel sleduje, včetně jeho vlastních. Použijeme poněkud pokročilé techniky Railsu, Ruby i SQL.

Důležité bude si předem ujasnit, čeho chceme dosáhnout. Rekapitulace je na obrázku (obr. 14.21).

// obr. 14.21 Mockup uživatelovy domovské stránky se status feedem.

### Směr a postup

Základní myšlenka feedu je jednoduchá. Na obrázku (obr. 14.22) je vzor tabulky "microposts" a výsledného feedu. Význam feedu spočívá ve vytažení mikropostů podle uživatelských id, které odpovídají seznamu sledovaných daného uživatele (a uživatele samotného), jak naznačují šipky na diagramu.

// obr. 14.22 Feed pro uživatele 1, který následuje uživatele 2, 7, 8 a 10.

Ačkoliv zatím feed neumíme zaimplementovat, testy jsou relativně přímočaré, a napíšeme si je tedy jako první. Klíčem je ověření všech tří požadavků pro feed: mikroposty od sledovaných uživatelů a uživatele samotného musí být zahrnuty, kdežto posty od nesledovaných být nesmí.

Na základě předchozích fixtur si převedeme požadavky na assert, a jelikož je "feed" v uživatelském modelu, upravíme přímo tamní test (soubor "test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  .
  .
  .
  test "feed should have the right posts" do
    michael = users(:michael)
    archer  = users(:archer)
    lana    = users(:lana)
    # Posts from followed user
    lana.microposts.each do |post_following|
      assert michael.feed.include?(post_following)
    end
    # Posts from self
    michael.microposts.each do |post_self|
      assert michael.feed.include?(post_self)
    end
    # Posts from unfollowed user
    archer.microposts.each do |post_unfollowed|
      assert_not michael.feed.include?(post_unfollowed)
    end
  end
end
```

Jelikož současná implementace zahrnuje pouze oholený základní feed, testy budou červené:

```
$ rails test
```

### První implementace feedu

Jelikož máme požadavky shrnuty v testu, můžeme se pustit do samotného psaní feedu. Jelikož je poslední krok poněkud komplikovaný, budeme stavět postupně, po kouscích. Prvním krokem je zamyslet se, jaký druh dotazu budeme potřebovat. Chceme vybrat všechny mikroposty z tabulky "microposts" s id, které odpovídá uživatelům, kteří jsou následováni daným uživatelem (a uživateli samotnému). Schématicky to můžeme zapsat takto:

```
SELECT * FROM microposts
WHERE user_id IN (<list of ids>) OR user_id = <user id>
```

SQL podporuje slovo IN, které nám umožňuje testovat inkluzi sady.

Z prvotního feedu víme, že Active Record používá metodu "where" pro docílení takovéhoto výběru. Náš výběr byl tehdy velmi jednoduchý, prostě jsme vzali všechny mikroposty s uživatelským id daného uživatele:

```
Micropost.where("user_id = ?", id)
```

Teď ovšem předpokládáme něco komplikovanějšího, v duchu:

```
Micropost.where("user_id IN (?) OR user_id = ?", following_ids, id)
```

Z těchto podmínek vidíme, že budeme potřebovat pole id odpovídajících sledovaným uživatelům. Můžeme použít Ruby metodu "map", kterou lze použít na jakýkoliv počítatelný objekt, tedy objekt (jako pole nebo haš) který sestává ze sady prvků. Jako příklad si můžeme uvést použítí "map" pro převedení pole celočíselných hodnot na pole řetězců:

```
$ rails console
>> [1, 2, 3, 4].map { |i| i.to_s }
=> ["1", "2", "3", "4"]
```

Jelikož jde o relativně běžnou záležitost, existuje pro zavolání na každý prvek sady i zkrácený zápis ve formě ampersandu (&) a symbolu, který metodě odpovídá:

```
>> [1, 2, 3, 4].map(&:to_s)
=> ["1", "2", "3", "4"]
```

Pomocí metody "join" můžeme vytvořit řetězec sestávající z id pomocí jejich spojení na základě čárky s mezerou:

```
>> [1, 2, 3, 4].map(&:to_s).join(', ')
=> "1, 2, 3, 4"
```

Výše zmíněnou metodou můžeme vytvořit potřebné pole sledovaných uživatelů zavoláním "id" na každý prvek v "user.following". Například, pro prvního uživatele v databázi vypadá takové pole následovně:

```
>> User.first.following.map(&:id)
=> [3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22,
23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41,
42, 43, 44, 45, 46, 47, 48, 49, 50, 51]
```

Jelikož je taková konstrukce velmi užitečná, Active Record ji přímo poskytuje:

```
>> User.first.following_ids
=> [3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22,
23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41,
42, 43, 44, 45, 46, 47, 48, 49, 50, 51]
```

Metoda "following_ids" je syntetizována skrze Active Record na základě asociace "has_many :following"; ve výsledku jen potřebujeme připojit "_ids" ke jménu asociace pro získání id odpovídajících sadě "user.following". Řetězec id sledovaných uživatelů pak vypadá následovně:

```
>> User.first.following_ids.join(', ')
=> "3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 17, 18, 19, 20, 21, 22,
23, 24, 25, 26, 27, 28, 29, 30, 31, 32, 33, 34, 35, 36, 37, 38, 39, 40, 41,
42, 43, 44, 45, 46, 47, 48, 49, 50, 51"
```

Když ale vkládáme do SQL řetězce, nepotřebujeme to zapisovat takto; překlad skrze "?" se o to postará za nás (a odstraní některé inkompatibility podle typu databáze). Můžeme tedy použít jen holé "following_ids". Původní odhad ve tvaru

```
Micropost.where("user_id IN (?) OR user_id = ?", following_ids, id)
```

ve skutečnosti funguje! Výsledná úprava modelu vypadá tedy následovně (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  .
  .
  .
  # Returns true if a password reset has expired.
  def password_reset_expired?
    reset_sent_at < 2.hours.ago
  end

  # Returns a user's status feed.
  def feed
    Micropost.where("user_id IN (?) OR user_id = ?", following_ids, id)
  end

  # Follows a user.
  def follow(other_user)
    following << other_user
  end
  .
  .
  .
end
```

Testy by měly proběhnout zeleně:

```
$ rails test
```

Na většinu praktických aplikací může být tato implementace dostatečná, ale ještě nás čekají úpravy. Ostatně, co kdyby náš uživatel sledoval dalších 5000 uživatelů?

### Podvýběry (subselects)

Podle posledního náznaku se dá soudit, že současná implementace nenese úplně dobře větší počet mikropostů a bude tedy třeba reimplementovat feed takovým způsobem, aby se s větším počtem uživatelů lépe škáloval.

Problém předchozího kódu je ten, že "following_ids" vytáhne id všech sledovaných uživatelů do paměti a vytvoří pole o plné délce. Jelikož podmínka v dotazu na databázi jen ověřuje zahrnutí sady, musí být efektivnější způsob, jak toho docílit, a SQL je přeci na takové věci optimalizované. Řešení bude spočívat ve vyhledání id v databázi pomocí podvýběru.

Začneme refaktorováním feedu lehce upraveným kódem (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  .
  .
  .
  # Returns a user's status feed.
  def feed
    Micropost.where("user_id IN (:following_ids) OR user_id = :user_id",
                    following_ids: following_ids, user_id: id)
  end
  .
  .
  .
end
```

V rámci příprav na další krok jsme nahradili

```
Micropost.where("user_id IN (?) OR user_id = ?", following_ids, id)
```

ekvivalentním

```
Micropost.where("user_id IN (:following_ids) OR user_id = :user_id",
                following_ids: following_ids, user_id: id)
```

Syntaxe s otazníkem je v pořádku, ale když chceme tu samou proměnnou vložit na více míst, druhá syntaxe je praktičtější.

A my právě chceme přidat druhý "user_id" do SQL dotazu. Tedy konkrétněji, můžeme nahradit Ruby kód

```
following_ids
```

střípkem SQL

```
following_ids = "SELECT followed_id FROM relationships
                 WHERE  follower_id = :user_id"
```

Tento kód obsahuje SQL podvýběr a interně přeloženo by celý výběr pro uživatele jedna vypadal zhruba takto:

```
SELECT * FROM microposts
WHERE user_id IN (SELECT followed_id FROM relationships
                  WHERE  follower_id = 1)
      OR user_id = 1
```

Tento podvýběr předává celou výběrovou logiku ke zpracování databázi, což je mnohem efektivnější.

Na tomto základu budeme schopni zaimplementovat daleko efektivnější feed (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  .
  .
  .
  # Returns a user's status feed.
  def feed
    following_ids = "SELECT followed_id FROM relationships
                     WHERE  follower_id = :user_id"
    Micropost.where("user_id IN (#{following_ids})
                     OR user_id = :user_id", user_id: id)
  end
  .
  .
  .
end
```

I když je kód samotný hybridem Rails, Ruby a SQL, dělá svou práci více, než dobře:

```
$ rails test
```

Samozřejmě ani podvýběr nepůjde škálovat do nekonečna. U větších stránek by bylo zřejmě potřeba vygenerovat feed asynchronně v pozadí, ale to už je téma na jindy.

S touto úpravou je náš status feed kompletní a domovská stránka ho obsahuje v plném rozsahu.

// obr. 14.23 Domovská stránka s funkčním status feedem.

V této chvíli už jen zbývá spojit změny s hlavní větví:

```
$ rails test
$ git add -A
$ git commit -m "Add user following"
$ git checkout master
$ git merge following-users
```



## Shrnutí + seznam dalších zdrojů

zahrnout?