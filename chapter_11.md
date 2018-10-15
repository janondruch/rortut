# Aktivace účtu

V současné chvíli mají nově zaregistrovaní uživatele okamžitě plný přístup ke svým účtům; v této kapitole si zaimplementujeme krok aktivace účtu pro ověření, že uživatel má přístup k emailové adrese, kterou pro registraci použil. To bude zahrnovat asociaci aktivačního tokenu a digestu s uživatelem, odeslání emailu uživateli (s odkazem, který bude token obsahovat) a aktivací uživatele po kliknutí na odkaz. V další kapitole na podobném principu dáme uživatelům možnost si resetovat heslo v případě, že ho zapomněli. Oba prvky budou zahrnovat vytvoření nového zdroje, čímž si rozšíříme obzory v rámci ovladačů, směrování a migrací databází. Také se naučíme v Rails odeslat email, a to jak ve vývojovém, tak v produkčním prostředí.

Náš postup ohledně aktivace účtu bude podobný, jako u přihlašování, a to hlavně ve smyslu zapamatovávání uživatelů. Základní posloupnost vypadá takto:

1. Uživatelé jsou vytvářeni v neaktivním stavu.
2. Když se uživatel zaregistruje, vygeneruje se aktivační token a související aktivační digest.
3. Aktivační digest se uloží do databáze a odešle se uživateli email s odkazem, který bude obsahovat aktivační token a emailovou adresu.
4. Jakmile uživatel klikne na odkaz, za pomocí emailu ho systém vyhledá a autentifikuje token porovnáním s aktivačním digestem.
5. Pokud dojde k autentifikaci, změní se uživatelův stav z "neaktivní" na "aktivní".

Díky souvislosti s hesly a tokeny (k zapamatování) budeme moci použít několik podobných principů i pro aktivaci účtu (a posléze i pro reset hesla), včetně metod "User.digest" a "User.new_token" spolu s upravenou verzí metody "user.authenticated?". V následující tabulce je analogie dobře vidět:

| **najdi podle** | **řetězec**        | **digest**          | **autentifikace**                    |
| --------------- | ------------------ | ------------------- | ------------------------------------ |
| `email`         | `password`         | `password_digest`   | `authenticate(password)`             |
| `id`            | `remember_token`   | `remember_digest`   | `authenticated?(:remember, token)`   |
| `email`         | `activation_token` | `activation_digest` | `authenticated?(:activation, token)` |
| `email`         | `reset_token`      | `reset_digest`      | `authenticated?(:reset, token)`      |

V další části si vytvoříme zdroj a datový model pro aktivace účtů a následně přidáme tzv. "mailer" pro odesílání aktivačních emailů. Poté si zaimplementujeme aktivaci jako takovou, včetně zobecněné verze metody "authenticated?", která je zmíněná v tabulce.

## Zdroj aktivací účtu

Podobně jako u sezení pojmeme aktivace účtu jakožto zdroj, i když nebudou spojeny s modelem Active Record. Namísto toho zahrneme relevantní údaje (včetně aktivačního tokenu a aktivačního stavu) v samotném uživatelském modelu.

Jelikož budeme vnímat aktivace účtu jakožto zdroj, budeme s nimi interagovat skrze standardní REST URL. Aktivační odkaz bude upravovat uživatelův aktivační stav, a pro takové úpravy je běžný REST postup odeslání požadavku PATCH akci "update". Aktivační odkaz musí být poslán emailem, a tedy bude zahrnovat normální kliknutí v prohlížeči, což je v praxi požadavek GET, namísto PATCH. Toto omezení nám tedy brání v použití akce "update", ale můžeme postupovat skrze akci "edit", která reaguje na GET požadavky.

Jako obvykle si připravíme vývojovou větev:

```
$ git checkout -b account-activation
```

### Ovladač aktivací účtu

Podobně, jako u uživatelů a sezení, akce (i když v tomto případě jen jedna) aktivací účtu umístíme do ovladače aktivací účtu, ktery vygenerujeme následovně:

```
$ rails generate controller AccountActivations
```

Jak uvidíme v budoucnu, aktivační email bude zarnovat URL ve tvaru

```
edit_account_activation_url(activation_token, ...)
```

což znamená, že budeme potřebovat jmennou cestu pro akci "edit". Upravíme si tedy soubor "config/routes.rb" a přidáme následující RESTful "resources" cestu:

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
end
```

| **HTTP požadavek** | **URL**                                      | **Akce** | **Jmenná cesta**                     |
| ------------------ | -------------------------------------------- | -------- | ------------------------------------ |
| `GET`              | http://ex.co/account_activation/<token>/edit | `edit`   | `edit_account_activation_url(token)` |

Samotnou akci "edit" si zadefinujeme až budeme mít hotový datový model a mailery.



### Datový model aktivací účtu

Jak jsme si řekli v rámci úvodu, budeme potřebovat unikátní aktivační token pro aktivační email. Jedna z možností by byla použít řetězec, který je uložen jak v databázi tak v aktivační URL, ale pokud by byla databáze narušena útočníkem, mohlo by to mít neblahý dopad na bezpečnost. Kupříkladu by mohl útočník s přístupem k databázi okamžitě aktivovat nově vytvořené účty (a tímpádem se i přihlásit jako daný uživatel), načež by stačilo změnit heslo pro získání kontroly.

Abychom podobným scénářům předešli, budeme následovat příkladu hesel a zapamatovacích tokenů pomocí spárování věřejně odhaleného virtuálního atributu spolu s bezpečně zahašovaným digestem, který bude uložen v databázi. Poté můžeme přistoupit k aktivačnímu tokenu skrze

```
user.activation_token
```

a autentifikovat uživatele kódem

```
user.authenticated?(:activation, token)
```

(Což ovšem bude vyžadovat úpravu metody "authenticated?", kterou jsme si definovali dříve.)

Přidáme také booleanský atribut zvaný "activated" do uživatelského modelu, který nám umožní otestovat, zda je uživatel aktivován pomocí stejného druhu automaticky vygenerovaná booleanské metody, kterou jsme viděli už dříve:

```
if user.activated? ...
```

A nakonec, i když to v rámci výuky nebudeme potřebovat, uložíme i čas a datum aktivace pro případ, že bychom takové informace v budoucnu potřebovali. Plný datový model je na obrázku (obr. 11.1).

// obr. 11.1 Uživatelský model s přidanými atributy aktivací účtu.

Migrace, která přidá požadované atributy do datového modelu vypadá následovně:

```
$ rails generate migration add_activation_to_users \
> activation_digest:string activated:boolean activated_at:datetime
```

(Příkazový řádek vkládá znak ">" automaticky jako signalizaci pokračování z předchozího řádku a do samotného příkazu se tedy nepíše.) Podobně, jako u atributu "admin" z minula přidáme i teď nativní hodnotu "false" atributu "activated" (soubor "db/migrate/[timestamp]_add_activation_to_users.rb"):

```
class AddActivationToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :activation_digest, :string
    add_column :users, :activated, :boolean, default: false
    add_column :users, :activated_at, :datetime
  end
end
```

Pak už zbývá jen provést samotnou migraci:

```
$ rails db:migrate
```



### Callback (zpětné volání) aktivačního tokenu

Jelikož každý nově zaregistrovaný uživatel bude potřebovat aktivaci, měli bychom přiřadit aktivační token a digest každému uživatelskému objektu ještě před tím, než bude vytvořen. Něco podobného jsme dělali i dříve, v rámci ukládání emailů, kde jsme prvně potřebovali email převést na malá písmena. Použili jsme "before_save" callback zkombinovaný s metodou "downcase". Callback "before_save" je automaticky zavolán před tím, než je objekt uložen, což zahrnuje jak tvorbu tak aktualizaci objektů, ale v případě aktivačního digestu chceme, ať se callback spouští pouze při vytvoření uživatele. Využijeme tedy callback "before_create", který si zadefinujeme následovně:

```
before_create :create_activation_digest
```

Tento kód se nazývá reference metody a říká Railsu, ať najde metodu zvanou "create_activation_digest" a spustí ji před vytvořením uživatele. Jelikož je samotná metoda "create_activation_digest" použita jen interně v rámci uživatelského modelu, není třeba ji veřejně odhalovat; jak jsme viděli dříve, pro takovéto záležitosti má Ruby klíčové slovo "private":

```
private

  def create_activation_digest
    # Vytvori token a digest.
  end
```

Všechny metody definované v třídě po "private" jsou automaticky skryty, jak je vidět v tomto sezení konzole:

```
$ rails console
>> User.first.create_activation_digest
NoMethodError: private method `create_activation_digest' called for #<User>
```

Účelem callbacku "before_create" je přiřadit token a související digest, čehož můžeme dosáhnout následovně:

```
self.activation_token  = User.new_token
self.activation_digest = User.digest(activation_token)
```

Tento kód jednoduše využije metod pro token a digest, které byly původně použity pro token k zapamatování, což si můžeme ověřit porovnáním s dřívější metodou "remember":

```
# Zapamatuje si uzivatele pro pouziti v trvalych sezenich.
def remember
  self.remember_token = User.new_token
  update_attribute(:remember_digest, User.digest(remember_token))
end
```

Hlavním rozdílem je použití "update_attributes" v druhém případě. Důvod je ten, že tokeny a digesty vytvořené pro uživatele už v databázi existují, kdežto "before_callback" se spouští ještě dříve, něž je uživatel vůbec vytvořen, a tedy neexistuje žádný atribut k aktualizaci. Výsledkem callbacku tedy je, že když je zadefinován nový uživatel skrze "User.new" (v rámci registrace), dostane automaticky atributy "activation_token" a "activation_digest", a jelikož je druhé jmenované asociováno se sloupcem v databázi, bude do ní automaticky zapsáno když se uživatel uloží.

Shrnuto a podtrženo, na základě těchto informací vypadá úprava uživatelského modelu následovně  (soubor "app/models/user.rb"). Jak je vyžadováno z virtuální povahy aktivačního tokenu, přidali jsme do našeho modelu i druhý "attr_accessor". Také jsme upravili úpravu emailu (změnu na malá písmena) referencí metody.

```
class User < ApplicationRecord
  attr_accessor :remember_token, :activation_token
  before_save   :downcase_email
  before_create :create_activation_digest
  validates :name,  presence: true, length: { maximum: 50 }
  .
  .
  .
  private

    # Prevede email na mala pismena.
    def downcase_email
      self.email = email.downcase
    end

    # Vytvori a priradi aktivacni token a digest.
    def create_activation_digest
      self.activation_token  = User.new_token
      self.activation_digest = User.digest(activation_token)
    end
end
```



### Zaplňovací data a uživatelé ve fixturách

Než postoupíme dále, měli bychom aktualizovat naše demonstrační data a fixtury, aby naši vzoroví testovací uživatelé byli aktivní (soubory "db/seeds.rb" a "test/fixtures/users.yml").

```
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
```

```
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
  admin: true
  activated: true
  activated_at: <%= Time.zone.now %>

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

lana:
  name: Lana Kane
  email: hands@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

malory:
  name: Malory Archer
  email: boss@example.gov
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>

<% 30.times do |n| %>
user_<%= n %>:
  name:  <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com" %>
  password_digest: <%= User.digest('password') %>
  activated: true
  activated_at: <%= Time.zone.now %>
<% end %>
```

Pro aplikaci změn stačí restartovat databázi a znovu naplnit testovací data:

```
$ rails db:migrate:reset
$ rails db:seed
```



## Emaily aktivace účtu

Když máme připraven datový model, můžeme přidat samotný kód pro odeslání aktivačního emailu. Postup bude zahrnovat přidání uživatelského maileru za pomocí knihovny Action Mailer, který použijeme v akci "create" uživatelského ovladače k odeslání emailu s aktivačním odkazem. Mailery jsou strukturované podobně, jako akce ovladačů, kde jsou šablony emailu definovány jako pohledy. Tyto šablony budou zahrnovat odkazy s aktivačním tokenem a emailovou adresou účtu, který má být aktivován.

### Šablony maileru

Jako u modelů a ovladačů si můžeme vytvořit mailer pomocí "rails generate":

```
$ rails generate mailer UserMailer account_activation password_reset
```

Krom nezbytné metody "account_activation" si zároveň vygenerujeme i metodu "password_reset", kterou použijeme v příští kapitole.

Příkaz také vygeneruje dvě šablony pohledu pro každý mailer, jeden pro plain-textový (běžný text) email a druhý pro HTML email. Textová i HTML verze šablon vypadají následovně (soubory "app/views/user_mailer/account_activation.text.erb" a "app/views/user_mailer/account_activation.html.erb"):

```
UserMailer#account_activation

<%= @greeting %>, find me in app/views/user_mailer/account_activation.text.erb
```

```
<h1>UserMailer#account_activation</h1>

<p>
  <%= @greeting %>, find me in app/views/user_mailer/account_activation.html.erb
</p>
```

Podíváme se na zoubek přímo jednotlivým mailerům, abychom lépe pochopili, jak fungují (soubory "app/mailers/application_mailer.rb" a "app/mailers/user_mailer.rb"). V prním je základní "from" adresa, kterou mají všechny mailery v aplikaci společnou, a každá metoda v druhém má také adresu příjemce. Vygenerovaný kód zahrnuje také instanční proměnnou (@greeting), která je dostupná v pohledech mailerů víceméně stejným způsobem, jako jsou k dispozici instanční proměnné v ovladačích a běžných pohledech.

```
class ApplicationMailer < ActionMailer::Base
  default from: "from@example.com"
  layout 'mailer'
end
```

```
class UserMailer < ApplicationMailer

  # Subject can be set in your I18n file at config/locales/en.yml
  # with the following lookup:
  #
  #   en.user_mailer.account_activation.subject
  #
  def account_activation
    @greeting = "Hi"

    mail to: "to@example.org"
  end

  # Subject can be set in your I18n file at config/locales/en.yml
  # with the following lookup:
  #
  #   en.user_mailer.password_reset.subject
  #
  def password_reset
    @greeting = "Hi"

    mail to: "to@example.org"
  end
end
```

Pro vytvoření funkčního aktivačního emailu si nejprve upravíme vygenerovanou šablonu (soubor "app/mailers/application_mailer.rb") následovně:

```
class ApplicationMailer < ActionMailer::Base
  default from: "noreply@example.com"
  layout 'mailer'
end
```

Poté vytvoříme instanční proměnnou, která bude obsahovat uživatele (pro použití v pohledu) a odešleme mailem výsledek na "user.email". Metoda "mail" také zvládá brát klíč "subject", jehož obsah bude v emailu vypsán jako předmět zprávy (soubor "app/mailers/user_mailer.rb").

```
class UserMailer < ApplicationMailer

  def account_activation(user)
    @user = user
    mail to: user.email, subject: "Account activation"
  end

  def password_reset
    @greeting = "Hi"

    mail to: "to@example.org"
  end
end
```

Jako u běžných pohledů, i zde můžeme použít vnořené Ruby pro úpravu šablon, v tomto případě pozdravením uživatele jménem a zahrnutím odkazu na vlastní aktivační odkaz. Náš plán spočívá v nalezení uživatele podle emailové adresy a autentifikaci aktivačního tokenu, takže odkaz musí zahrnovat jak email, tak token. Jelikož aktivace t voříme pomocí zdroje Account Activations, token sám o sobě může mít podobu parametru ve jmenné cestě, jako jsme zmiňovali dříve:

```
edit_account_activation_url(@user.activation_token, ...)
```

Vzpomeňme si, že

```
edit_user_url(user)
```

vytvoří URL v podobě

```
http://www.example.com/users/1/edit
```

Takže související aktivační odkaz (resp. jeho základní URL) bude vypadat následovně:

```
http://www.example.com/account_activations/q5lt38hQDc_959PVoo6b7A/edit
```

Kde "" je URL-safe (obsahuje jen znaky, které jdou bez problémů použít v URL) base64 řetězec vygenerovaný metodou "new_token" a hraje stejnou roli, jako uživatelovo id v /users/1/edit. V akci "edit" ovladače aktivací bude token k dispozici v haši params pod klíčem "params[:id]".

Abychom zahrnuli i email, musíme použít tzv. dotazový parametr (query parameter), který bude v URL vypadat jako klasický pár klíč-hodnota za znakem otazníku:

```
account_activations/q5lt38hQDc_959PVoo6b7A/edit?email=foo%40example.com
```

Znak "@" v emailové adrese je přeložen jako %40, tedy je "vyescapován" (escaped out) pro garanci platné URL. Způsob nastavení dotazového parametru v Rails spočívá v zahrnutí haše ve jmenné cestě:

```
edit_account_activation_url(@user.activation_token, email: @user.email)
```

Když používáme jmenné cesty tímto způsobem (pro definování dotazových parametrů), Rails automaticky escapuje jakékoliv speciální znaky. Výsledná emailová adresa bude zase v ovladači navrácena do původní podoby a přístupná skrze "params[:email]".

Se zadefinovanou instanční proměnnou "@user" můžeme vytvořit nezbytné odkazy pomocí specifické jmenné editační cesty a vnořeného Ruby (soubory "app/views/user_mailer/account_activation.text.erb" a "app/views/user_mailer/account_activation.html.erb"). Mimojiné v druhém případě používáme metodu "link_to" pro vytvoření platného odkazu.

```
Hi <%= @user.name %>,

Welcome to the Sample App! Click on the link below to activate your account:

<%= edit_account_activation_url(@user.activation_token, email: @user.email) %>
```

```
<h1>Sample App</h1>

<p>Hi <%= @user.name %>,</p>

<p>
Welcome to the Sample App! Click on the link below to activate your account:
</p>

<%= link_to "Activate", edit_account_activation_url(@user.activation_token,
                                                    email: @user.email) %>
```



### Náhledy emailu

Abychom viděli výsledky našeho snažení ohledně šablon emailu, můžeme použít náhledy (email previews), což jsou speciální URL které Rails poskytuje právě k tomuto účelu. V prvé řadě musíme trochu nakonfigurovat naše vývojové prostředí (soubor "config/environments/development.rb"):

```
Rails.application.configure do
  .
  .
  .
  config.action_mailer.raise_delivery_errors = true
  config.action_mailer.delivery_method = :test
  host = 'localhost:3000' # Pouzivame adresu naseho lokalniho serveru
  config.action_mailer.default_url_options = { host: host, protocol: 'http' }
  .
  .
  .
end
```

Jelikož používáme místní vývojový server, měla by fungovat adresa "localhost:3000" a protokol http.

Po restartování vývojového serveru (kvůli změn konfigurace) si ještě aktualizujeme náhledový soubor maileru, který byl automaticky vygenerován dříve (soubor "test/mailers/previews/user_mailer_preview.rb"):

```
# Nahled vsech emailu je na http://localhost:3000/rails/mailers/user_mailer
class UserMailerPreview < ActionMailer::Preview

  # Nahled tohoto emailu je na
  # http://localhost:3000/rails/mailers/user_mailer/account_activation
  def account_activation
    UserMailer.account_activation
  end

  # Nahled tohoto emailu je na
  # http://localhost:3000/rails/mailers/user_mailer/password_reset
  def password_reset
    UserMailer.password_reset
  end

end
```

Kvůli tomu, že metoda "account_activation" vyžaduje platný uživatelský objekt jako parametr nebude předchozí kód fungovat. Abychom to napravili, zadefinujeme si proměnnou "user" která bude rovna prvnímu uživateli ve vývojové databázi a předáme ji jako parametr metodě "UserMailer.account_activation" (soubor "test/mailers/previews/user_mailer_preview.rb"). Také si přiřadíme hodnotu "user.activation_token", což je potřeba udělat proto, že naše šablony vyžadují aktivační token (který je virtuálním atributem, a uživatel z databáze by tedy žádný neměl).

```
# Nahled vsech emailu je na http://localhost:3000/rails/mailers/user_mailer
class UserMailerPreview < ActionMailer::Preview

  # Nahled tohoto emailu je na
  # http://localhost:3000/rails/mailers/user_mailer/account_activation
  def account_activation
    user = User.first
    user.activation_token = User.new_token
    UserMailer.account_activation(user)
  end

  # Nahled tohoto emailu je na
  # http://localhost:3000/rails/mailers/user_mailer/password_reset
  def password_reset
    UserMailer.password_reset
  end
end
```

S upraveným kódem můžeme navštívit navrhované URL pro představu, jak naše aktivační emaily vypadají.

// obr. 11.2 Náhled HTML verze aktivačního emailu.

// obr. 11.3 Náhled textové verze aktivačního emailu.

### Testy emailů

Jako poslední krok si napíšeme několik testů pro ověření výsledků zobrazených v náhledech emailů. Není to tak těžké, jak by se zdálo, jelikož pro nás Rails už pár užitečných testů sám vygeneroval (soubor "test/mailers/user_mailer_test.rb").

```
require 'test_helper'

class UserMailerTest < ActionMailer::TestCase

  test "account_activation" do
    mail = UserMailer.account_activation
    assert_equal "Account activation", mail.subject
    assert_equal ["to@example.org"], mail.to
    assert_equal ["from@example.com"], mail.from
    assert_match "Hi", mail.body.encoded
  end

  test "password_reset" do
    mail = UserMailer.password_reset
    assert_equal "Password reset", mail.subject
    assert_equal ["to@example.org"], mail.to
    assert_equal ["from@example.com"], mail.from
    assert_match "Hi", mail.body.encoded
  end
end
```

Testy používají mocnou metodu "assert_match", kterou lze použít buďto s řetězcem, nebo regulárním výrazem:

```
assert_match 'foo', 'foobar'      # true
assert_match 'baz', 'foobar'      # false
assert_match /\w+/, 'foobar'      # true
assert_match /\w+/, '$#!*+@'      # false
```

Následující test bude používat "assert_match" pro ověření jména, aktivačního tokenu a escapovaného emailu, resp. jejich přítomnosti v těle emailu. Pro poslední jmenované použijeme

```
CGI.escape(user.email)
```

právě pro escapovaní emailu (soubor "test/mailers/user_mailer_test.rb").

```
require 'test_helper'

class UserMailerTest < ActionMailer::TestCase

  test "account_activation" do
    user = users(:michael)
    user.activation_token = User.new_token
    mail = UserMailer.account_activation(user)
    assert_equal "Account activation", mail.subject
    assert_equal [user.email], mail.to
    assert_equal ["noreply@example.com"], mail.from
    assert_match user.name,               mail.body.encoded
    assert_match user.activation_token,   mail.body.encoded
    assert_match CGI.escape(user.email),  mail.body.encoded
  end
end
```

Test tedy přiřadí aktivační token provizornímu uživateli, který by jinak byl prázdný. (Také si dočasně odstraníme test pro restartování hesla, který si později v upravené podobě vrátíme.)

Aby tento test prošel, musíme nastavit do testovacího souboru i vhodnou doménu (soubor "config/environments/test.rb"):

```
Rails.application.configure do
  .
  .
  .
  config.action_mailer.delivery_method = :test
  config.action_mailer.default_url_options = { host: 'example.com' }
  .
  .
  .
end
```

S těmito úpravami by měl test maileru proběhnout zeleně:

```
$ rails test:mailers
```

### Aktualizace uživatelské akce "update"

Aby mailer správně fungoval, bude třeba ještě přidat několik řádek do akce "create", která se používá pro registraci. Mimojiné si změníme chování přesměrování po registraci jako takové. Dříve jsme přesměrováváli na profilovou stránku uživatele, což už ale nedává smysl, jelikož první budeme vyžadovat aktivaci. Přesměrování tedy proběhne na kořenovou URL (soubor "app/controllers/users_controller.rb").

```
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      UserMailer.account_activation(@user).deliver_now
      flash[:info] = "Pro aktivaci účtu si prosím zkontrolujte email."
      redirect_to root_url
    else
      render 'new'
    end
  end
  .
  .
  .
end
```

Díky změně přesměrování a absenci přihlášení nám testovací sada vyjde červeně, i když aplikace funguje tak, jak chceme. Dočasně si zakomentujeme řádky, které toto selhání způsobují, ale záhy se k nim vrátíme a testy vhodně upravíme (soubor "test/integration/users_signup_test.rb"):

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
    assert_select 'div#error_explanation'
    assert_select 'div.field_with_errors'
  end

  test "valid signup information" do
    get signup_path
    assert_difference 'User.count', 1 do
      post users_path, params: { user: { name:  "Example User",
                                         email: "user@example.com",
                                         password:              "password",
                                         password_confirmation: "password" } }
    end
    follow_redirect!
    # assert_template 'users/show'
    # assert is_logged_in?
  end
end
```

Pokud se nyní pokusíme registrovat jako nový uživatel, měli bychom být přesměrováni (obr. 11.4) a také by měl být vygenerován email, který je podobný, jako ten vypsaný níže. Samotný email nám ve vývojovém prostředí do schránky nepřijde, ale objevi se v záznamech serveru. Později si probereme, jak odeslat email v reálném produkčním prostředí.

```
UserMailer#account_activation: processed outbound mail in 292.4ms
Sent mail to michael@michaelhartl.com (47.3ms)
Date: Mon, 06 Jun 2016 20:17:41 +0000
From: noreply@example.com
To: michael@michaelhartl.com
Message-ID: <5755da6518cb4_f2c9222494c7178e@mhartl-rails-tutorial-3045526.mail>
Subject: Account activation
Mime-Version: 1.0
Content-Type: multipart/alternative;
 boundary="--==_mimepart_5755da6513e89_f2c9222494c71639";
 charset=UTF-8
Content-Transfer-Encoding: 7bit


----==_mimepart_5755da6513e89_f2c9222494c71639
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

Hi Michael Hartl,

Welcome to the Sample App! Click on the link below to activate your account:

https://rails-tutorial-mhartl.c9users.io/account_activations/
-L9kBsbIjmrqpJGB0TUKcA/edit?email=michael%40michaelhartl.com

----==_mimepart_5755da6513e89_f2c9222494c71639
Content-Type: text/html;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

<!DOCTYPE html>
<html>
  <head>
    <meta http-equiv="Content-Type" content="text/html; charset=utf-8" />
    <style>
      /* Email styles need to be inline */
    </style>
  </head>

  <body>
    <h1>Sample App</h1>

<p>Hi Michael Hartl,</p>

<p>
Welcome to the Sample App! Click on the link below to activate your account:
</p>

<a href="https://rails-tutorial-mhartl.c9users.io/account_activations/
-L9kBsbIjmrqpJGB0TUKcA/edit?email=michael%40michaelhartl.com">Activate</a>
  </body>
</html>

----==_mimepart_5755da6513e89_f2c9222494c71639--
```

// obr. 11.4 Domovská stránka s aktivační zprávou po registraci.



## Aktivace účtu

Jelikož už máme správně vygenerovaný email z předchozí části k dispozici, zbývá jen připravit akci "edit" v ovladači aktivací účtu pro reálné aktivování uživatele. Jako obvykle si pro tuto akci napíšeme i test, a jakmile bude kód otestován, refaktorujeme (přepracujeme) si ho a některou funkcionalitu převedeme z ovladače aktivace účtů do uživatelského modelu.

### Zobecnění metody "authenticated?"

Jak jsme si říkali dříve, aktivační token a email jsou k dispozici skrze "params[:id]" a "params[:email]". Když budeme následovat model hesel a tokenů k zapamatování, chtěli bychom autentifikovat uživatele zhruba takovýmto kódem:

```
user = User.find_by(email: params[:email])
if user && user.authenticated?(:activation, params[:id])
```

Kód používá metodu "authenticated?" pro ověření, že digest aktivace účtu odpovídá danému tokenu, ale v současné chvíli fungovat nebude, jelikož je metoda specializovaná pouze na token k zapamatování:

```
# Vrati true pokud dany token odpovida digestu.
def authenticated?(remember_token)
  return false if remember_digest.nil?
  BCrypt::Password.new(remember_digest).is_password?(remember_token)
end
```

V tomto případě je "remember_digest" atribut uživatelského modelu, a přímo v něm to lze přepsat následovně:

```
self.remember_digest
```

Chceme být schopni z toho udělat proměnnou, a zavoláme tedy

```
self.activation_digest
```

namísto předání vhodného parametru metodě "authenticated?".

Řešení zahrnuje první příklad tzv. metaprogramování, což je v podstatě program, který píše program. (Metaprogramování je jedním z nejsilnějších prvků Ruby a spousta "kouzelných" prvků Ruby existuje právě díky použití metaprogramování.) Klíč je v tomto případě v metodě "send", která nás nechá zavolat metodu dle našeho výběru tak, že "pošle zprávu" danému objektu. V následujícím konzolovém sezení například použijeme "send" na nativní objekt Ruby pro zjištění délky pole:

```
$ rails console
>> a = [1, 2, 3]
>> a.length
=> 3
>> a.send(:length)
=> 3
>> a.send("length")
=> 3
```

Předání symbolu ":length" nebo řetězce "length" metodě "send" je ekvivalentní zavolání metody "length" na daný objekt. V rámci druhého příkladu přistoupíme k atributu "activation_digest" prvního uživatele databáze:

```
>> user = User.first
>> user.activation_digest
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> user.send(:activation_digest)
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> user.send("activation_digest")
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
>> attribute = :activation
>> user.send("#{attribute}_digest")
=> "$2a$10$4e6TFzEJAVNyjLv8Q5u22ensMt28qEkx0roaZvtRcp6UZKRM6N9Ae"
```

V posledním příkladu jsme si definovali proměnnou "attribute", která je rovna symbolu ":activation" a používá interpolaci řetězců pro vytvoření správného parametru "send". S řetězcem "'activation'" by to fungovalo také, ale použití symbolu je běžnější, a tak jako tak se z

```
"#{attribute}_digest"
```

stane

```
"activation_digest"
```

ve chvíli, kdy je řetězec interpolován (vložen).

Na základě těchto znalostí můžeme přepsat současnou metodu "authenticated?" následovně:

```
def authenticated?(remember_token)
  digest = self.send("remember_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(remember_token)
end
```

S touto šablonou pak můžeme zobecnit metodu přidáním parametru funkce se jménom digestu a použít interpolaci řetězce následovně:

```
def authenticated?(attribute, token)
  digest = self.send("#{attribute}_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(token)
end
```

(Přejmenovali jsme druhý parametr na "token" pro zdůraznění, že je nyní obecný.) Jelikož jsme uvnitř uživatelského modelu, můžeme opomenout "self", což má za výsledek idiomaticky správnou verzi:

```
def authenticated?(attribute, token)
  digest = send("#{attribute}_digest")
  return false if digest.nil?
  BCrypt::Password.new(digest).is_password?(token)
end
```

Předchozí chování "authenticated?" můžeme nyní vyvolat takto:

```
user.authenticated?(:remember, remember_token)
```

Naše upravená metoda v souboru "app/models/user.rb" bude tedy vypadat takto:

```
class User < ApplicationRecord
  .
  .
  .
  # Vrati true pokud dany token odpovida digestu.
  def authenticated?(attribute, token)
    digest = send("#{attribute}_digest")
    return false if digest.nil?
    BCrypt::Password.new(digest).is_password?(token)
  end
  .
  .
  .
end
```

Testy nicméně vyjdou červeně:

```
$ rails test
```

Důvodem selhání je, že metoda "current_user" a test na zpracování "nil" používají starou verzi "authenticated?", která očekává jeden parametr namísto dvou. Pro opravu stačí aktualizovat tyto dva případy zobecněnou metodou (soubory "app/helpers/sessions_helper.rb" a "test/models/user_test.rb"):

```
module SessionsHelper
  .
  .
  .
  # Vrati momentalne prihlaseneho uzivatele (pokud je).
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(:remember, cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end
  .
  .
  .
end
```

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
  test "authenticated? should return false for a user with nil digest" do
    assert_not @user.authenticated?(:remember, '')
  end
end
```

Testy by už měly vyjít zeleně:

```
$ rails test
```

Přepracovávání kódu je daleko snažší, když máme silnou sadu testů, což je ostatně důvod, proč se tak věnujeme jejich psaní.



### Aktivační akce "edit"

S upravenou metodou "authenticated?" si můžeme napsat akci "edit" která autentifikuje uživatele podle emailu v haši "params". Test validity bude vypadat následovně:

```
if user && !user.activated? && user.authenticated?(:activation, params[:id])
```

Zajímavá je přítomnost "!user.activated?", což je kritérium, které brání našemu kódu v aktivaci uživatelů, kteří již aktivní jsou, a to je důležité proto, že budeme po potvrzení uživatele přihlašovat a nechceme, aby případní útočníci s aktivačním odkazem byli schopni přihlášení pod daným uživatelem.

Pokud je uživatel autentifikován předchozím kódem, musíme ho aktivovat a aktualizovat timestamp "activated_at":

```
user.update_attribute(:activated,    true)
user.update_attribute(:activated_at, Time.zone.now)
```

Připravíme si tedy akci "edit" (soubor "app/controllers/account_activations_controller.rb") následovně (plus si i ošetřujeme případ, kdy by byl aktivační token neplatný): 

```
class AccountActivationsController < ApplicationController

  def edit
    user = User.find_by(email: params[:email])
    if user && !user.activated? && user.authenticated?(:activation, params[:id])
      user.update_attribute(:activated,    true)
      user.update_attribute(:activated_at, Time.zone.now)
      log_in user
      flash[:success] = "Účet byl aktivován!"
      redirect_to user
    else
      flash[:danger] = "Neplatný aktivační odkaz"
      redirect_to root_url
    end
  end
end
```

Můžeme navštívit přislušnou URL pro aktivaci a tím daného uživatele aktivovat:

```
https://rails-tutorial-mhartl.c9users.io/account_activations/
fFb_F94mgQtmlSvRFGsITw/edit?email=michael%40michaelhartl.com
```

Výsledek by měl odpovídat obrázku (obr. 11.5).

// obr. 11.5 Profilová stránka po úspěšné aktivaci.

Samotná aktivace samozřejmě reálně nic nedělá, jelikož jsme zatím neupravili způsob, jakým se uživatelé přihlašují. Aby dostála svého významu, bude třeba povolit uživatelům přihlášení jen pokud byli aktivováni. Postup jednoduše spočívá ve zjištění, zda byl uživatel aktivován skrze "user.activated?" a v případě false přesměrovat na kořenovou adresu s "warning" zprávou (soubor "app/controllers/sessions_controller.rb").

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      if user.activated?
        log_in user
        params[:session][:remember_me] == '1' ? remember(user) : forget(user)
        redirect_back_or user
      else
        message  = "Účet není aktivní. "
        message += "Zkontrolujte email pro aktivační odkaz."
        flash[:warning] = message
        redirect_to root_url
      end
    else
      flash.now[:danger] = 'Neplatná kombinace emailu/hesla'
      render 'new'
    end
  end

  def destroy
    log_out if logged_in?
    redirect_to root_url
  end
end
```

// obr. 11.6 Varovná zpráva pro dosud neaktivovaného uživatele.

Tím je základní funkcionalita aktivace účtů hotová.

### Aktivační testy a refaktorování

V této části si přidáme integrační test pro aktivace účtů. Jelikož už máme test pro registraci s platnými informacemi, přidáme další kroky do něj. Je jich poněkud více, ale jsou vcelku přímočaré (soubor "test/integration/users_signup_test.rb"):

```
require 'test_helper'

class UsersSignupTest < ActionDispatch::IntegrationTest

  def setup
    ActionMailer::Base.deliveries.clear
  end

  test "invalid signup information" do
    get signup_path
    assert_no_difference 'User.count' do
      post users_path, params: { user: { name:  "",
                                         email: "user@invalid",
                                         password:              "foo",
                                         password_confirmation: "bar" } }
    end
    assert_template 'users/new'
    assert_select 'div#error_explanation'
    assert_select 'div.field_with_errors'
  end

  test "valid signup information with account activation" do
    get signup_path
    assert_difference 'User.count', 1 do
      post users_path, params: { user: { name:  "Example User",
                                         email: "user@example.com",
                                         password:              "password",
                                         password_confirmation: "password" } }
    end
    assert_equal 1, ActionMailer::Base.deliveries.size
    user = assigns(:user)
    assert_not user.activated?
    # Try to log in before activation.
    log_in_as(user)
    assert_not is_logged_in?
    # Invalid activation token
    get edit_account_activation_path("invalid token", email: user.email)
    assert_not is_logged_in?
    # Valid token, wrong email
    get edit_account_activation_path(user.activation_token, email: 'wrong')
    assert_not is_logged_in?
    # Valid activation token
    get edit_account_activation_path(user.activation_token, email: user.email)
    assert user.reload.activated?
    follow_redirect!
    assert_template 'users/show'
    assert is_logged_in?
  end
end
```

I když je kódu hodně, jedina opravdová novota je řádek

```
assert_equal 1, ActionMailer::Base.deliveries.size
```

Tento kód ověřuje, že byla doručena přesně jedna zpráva. Jelikož je pole "deliveries" globální, je třeba ho resetovat v metodě "setup" abychom zabránili případnému rozbití jiných testů, které by chtěly odeslat email. (Což bude ostatně případ další kapitoly.) Používáme také poprvé metodu "assigns", díky které můžeme přistoupit k instančním proměnným související akce. Kupříkladu, akce "create" uživatelského ovladače definuje proměnnou "@user", takže k ní můžeme přistoupit použitím "assigns(:user)". Také si obnovujeme řádky zakomentované dříve.

Testy by nyní měly proběhnout zeleně:

```
$ rails test
```

S funkčním upraveným testem můžeme náš kód krapet refaktorovat přesunutím některých prvků práce s uživateli z ovladače do modelu. Konkrétněji tedy připravíme metodu "activate" pro aktualizaci uživatelských aktivačních atributů a "send_activation_email" zase odešle aktivační email. Dané metody jsou zobrazeny v prvním výpisu (soubor "app/models/user.rb") a jejich nová umístění (a přepracování) na dalších dvou (soubory "app/controllers/users_controller.rb" a "app/controllers/account_activations_controller.rb").

```
class User < ApplicationRecord
  .
  .
  .
  # Aktivuje ucet.
  def activate
    update_attribute(:activated,    true)
    update_attribute(:activated_at, Time.zone.now)
  end

  # Odesle aktivacni email.
  def send_activation_email
    UserMailer.account_activation(self).deliver_now
  end

  private
    .
    .
    .
end
```

```
class UsersController < ApplicationController
  .
  .
  .
  def create
    @user = User.new(user_params)
    if @user.save
      @user.send_activation_email
      flash[:info] = "Please check your email to activate your account."
      redirect_to root_url
    else
      render 'new'
    end
  end
  .
  .
  .
end
```

```
class AccountActivationsController < ApplicationController

  def edit
    user = User.find_by(email: params[:email])
    if user && !user.activated? && user.authenticated?(:activation, params[:id])
      user.activate
      log_in user
      flash[:success] = "Account activated!"
      redirect_to user
    else
      flash[:danger] = "Invalid activation link"
      redirect_to root_url
    end
  end
end
```

V rámci úpravy eliminujeme použití "user.", jelikož daném modelu takový proměnná není:

```
-user.update_attribute(:activated,    true)
-user.update_attribute(:activated_at, Time.zone.now)
+update_attribute(:activated,    true)
+update_attribute(:activated_at, Time.zone.now)
```

(Mohli jsme samozřejmě přejit z "user" na "self", ale "self" je v modelu volitelná záležitost.) Ve volání uživatelského maileru "@user" na "self" měníme:

```
-UserMailer.account_activation(@user).deliver_now
+UserMailer.account_activation(self).deliver_now
```

Jde o drobnosti, které právě zvládá výborně odchytávat dobrá sada testů. Která by mimochodem měla proběhnout zeleně:

```
$ rails test
```



## Email v produkčním nasazení



# PROBRAT, JESTLI SE ZAHRNE, PRIPADNE V JAKE PODOBE







Opět si spojíme vývojovou větev s hlavní:

```
$ rails test
$ git add -A
$ git commit -m "Add account activation"
$ git checkout master
$ git merge account-activation
```

A odešleme do repozitáře:

```
$ rails test
$ git push
```



## Shrnutí

Po přidání aktivace účtu je mechanika naší aplikace v podobě registrace, přihlášení a odhlášení téměř kompletní. Jediný podstatný zbývající prvek je umožnit uživatelum resetovat jejich zapomenutá hesla. Naštestí zjistíme, že má tato problematika hodně společného s aktivací účtu a tímpádem prakticky využijeme nově nabyté znalosti.

### Co jsme se naučili

- Stejně jako sezení mohou být aktivace účtu pojaty jako zdroj navzdory tomu, že nejsou objekty Active Record.
- Rails umí vygenerovat akce a pohledy Action Maileru pro odesílání emailů. 
- Action Mailer podporuje maily jak v podobě HTML, tak běžného textu.
- Jako u běžných pohledů a akcí jsou i instanční proměnné definované v akcích maileru dostupně v jeho pohledech.
- Aktivace účtu používají generovaný token pro vytvoření unikátní URL pro aktivaci uživatelů.
- Aktivace účtů používají zpracovaný digest pro bezpečnou identifikaci platných aktivačních požadavků.
- Jak integrační testy, tak testy maileru jsou užitečné pro ověření chování uživatelského maileru.