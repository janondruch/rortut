# Resetování hesla

S dokončením aktivace účtu (a mimojiné i ověřením, že uživatelova emailová adresa je platná) nám zbývá už jen zaimplementovat reset hesla, čímž ošetříme běžný problém, kdy uživatelé své heslo zapomenou. Hodně kroků bude podobných jako u aktivace účtu. Začátek bude poněkud jiný; narozdíl od aktivace vyžaduje implementování resetu hesla jak změnu jednoho z našich pohledů, tak dva nové formuláře (pro zpracování emailu a odeslání nového hesla).

Než začneme se samotným psaním kódu, načrtneme si předpokládaný postup pro resetování hesla. Začneme tedy s odkazem "zapomenuté heslo" u přihlašovacího formuláře (obr. 12.1). Odkaz bude směřovat na stránku s formulářem, která na emailovou adresu odešle email obsahující odkaz k resetování hesla (obr. 12.2). Resetovací odkaz povede k formuláři pro resetování hesla i s jeho potvrzením (obr. 12.3).

// obr. 12.1 Mockup odkazu "zapomenuté heslo".

// obr. 12.2 Mockup formuláře "zapomenuté heslo".

// obr. 12.3 Mockup formuláře pro resetování hesla.

Z minula máme vytvořený i mailer pro reset hesla, takže zbývá jen doplnit vhodný zdroj a datový model. Samotný reset zaimplementujeme nakonec.

Jako u aktivací účtu náš obecný plán spočívá ve vytvoření zdroje resetů hesla (Password Resets), kde každý reset bude sestávat z resetového tokenu a souvisejícího resetového digestu. Primární sekvence bude vypadat následovně:

1. Když uživatel zažádá o reset hesla, najdi uživatele podle odeslané emailové adresy.
2. Pokud emailová adresa existuje v databázi, vygeneruj reset token a související reset digest.
3. Ulož digest do databáze a odešli email uživateli s odkazem, který bude obsahovat reset token a uživatelovu emailovou adresu.
4. Když uživatel klikne na odkaz, najdi uživatele podle emailu a ověř token porovnáním s reset digestem.
5. Pokud dojde k autentifikaci, nabídni uživateli formulář pro změnu hesla.



## Zdroj resetů hesla (Password Reset resource)

Podobně, jako u sezení a aktivací účtu pojmeme resety hesla jako zdroj, i když nejsou asociovány s modelem Active Record. Namísto toho zahrneme související data (včetně reset tokenu) do samotného uživatelského modelu.

Jelikož budeme vnímat resety hesla jako zdroj, budeme s nimi i komunikovat skrze standardní REST URL. Narozdíl od aktivačního odkazu, který vyžadoval pouze akci "edit", budeme tentokráte vykreslovat jak formulář "new", tak "edit" pro práci s resety hesel, včetně jejich vytváření a aktualizování, a tedy nakonec použijeme celkem čtyři RESTful cesty.

Jako obvykle si vytvoříme novou větev:

```
$ git checkout -b password-reset
```

### Ovladač resetů hesla

Naším prvním krokem bude vygenerování ovladače pro zdroj resetů hesla, včetně vytvoření obou akcí "new" a "edit":

```
$ rails generate controller PasswordResets new edit --no-test-framework
```

Zahrnuli jsme i parametr pro přeskočení generování testů. To proto, že nebudeme potřebovat testy ovladače, ale spíše budeme stavět na integračním testu z minulé kapitoly.

Jelikož potřebujeme formuláře jak pro vytvoření, tak pro aktualizaci hesel, budeme potřebovat cesty pro "new", "create", "edit" a "update". Použijeme tedy opět řádek "resources" v souboru "config/routes.rb":

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
end
```

Tím si nastavíme RESTful cesty zobrazené v tabulce. Konkrétně pak první cesta v tabulce poskytuje odkaz k formuláři "zapomenuté heslo" skrze

```
new_password_reset_path
```

jak je vidno v následujícím výpisu (soubor "app/views/sessions/new.html.erb") a na obrázku (obr. 12.4).

| **HTTP požadavek** | **URL**                                   | **Akce** | **Jmenná cesta**                 |
| ------------------ | ----------------------------------------- | -------- | -------------------------------- |
| `GET`              | /password_resets/new                      | `new`    | `new_password_reset_path`        |
| `POST`             | /password_resets                          | `create` | `password_resets_path`           |
| `GET`              | http://ex.co/password_resets/<token>/edit | `edit`   | `edit_password_reset_url(token)` |
| `PATCH`            | /password_resets/<token>                  | `update` | `password_reset_path(token)`     |

```
<% provide(:title, "Log in") %>
<h1>Log in</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(:session, url: login_path) do |f| %>

      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= link_to "(forgot password)", new_password_reset_path %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.label :remember_me, class: "checkbox inline" do %>
        <%= f.check_box :remember_me %>
        <span>Zapamatuj si mě na tomto počítači</span>
      <% end %>

      <%= f.submit "Log in", class: "btn btn-primary" %>
    <% end %>

    <p>Nový uživatel? <%= link_to "Zaregistrujte se!", signup_path %></p>
  </div>
</div>
```

// obr. 12.4 Přihlašovací stránka s odkazem "zapomenuté heslo".



### Nové resety hesla

Pro vytvoření nových resetů hesla budeme potřebovat nejprve zadefinovat datový model, který bude podobný tomu z aktivací účtů. Opět použijeme dvojici v podobě virtuálního resetovacího tokenu pro použití v emailu a související resetový digest pro získání uživatele. Pokud bychom uložili nezahašovaný token, útočník s přístupem do databáze by mohl poslat resetový požadavek na emailovou adresu uživatele a použít token a email pro navštívení souvisejícího resetového odkazu, čímž by získal nad daným účtem kontrolu. Použít digest je tedy nutnost. Navíc, jako dodatečný bezpečnostní prvek, zaimplementujeme vypršení platnosti resetovacího odkazu po několika hodinách a bude tedy třeba zaznamenat, kdy byl reset odeslán. Výsledné atributy "reset_digest" a "reset_sent_at" jsou na obrázku (obr. 12.5).

// obr. 12.5 Uživatelský model s přidanými atributy pro reset hesla.

Migrace atributů z obrázku pak vypadá následovně:

```
$ rails generate migration add_reset_to_users reset_digest:string \
> reset_sent_at:datetime
```

(Znak zobáčku > na druhém řádku je opět znakem pro "pokračování řádku" a generuje ho automaticky shell, a tedy ho nepíšeme.) Po této úpravě stačí spustit samotnou migraci:

```
$ rails db:migrate
```

Pro vytvoření pohledu pro nové resety hesla použijeme navazující posloupnost z předchozího formuláře pro vytvoření nového, ne-Active Record zdroje, tedy přihlašovacího formuláře (který je pro srovnání na následujícím výpisu, soubor "app/views/sessions/new.html.erb").

```
<% provide(:title, "Log in") %>
<h1>Log in</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(:session, url: login_path) do |f| %>

      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.label :remember_me, class: "checkbox inline" do %>
        <%= f.check_box :remember_me %>
        <span>Zapamatuj si mě na tomto počítači</span>
      <% end %>

      <%= f.submit "Log in", class: "btn btn-primary" %>
    <% end %>

    <p>Nový uživatel? <%= link_to "Zaregistrujte se!", signup_path %></p>
  </div>
</div>
```

Nový formulář pro reset hesla má s tímto hodně společného; nejpodstatnější rozdíly jsou použití jiného zdroje a URL ve volání "form_for" a opomenutí atributu pro heslo. Výsledný formulář bude vypadat následovně (soubor "app/views/password_resets/new.html.erb"):

```
<% provide(:title, "Forgot password") %>
<h1>Zapomenuté heslo</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(:password_reset, url: password_resets_path) do |f| %>
      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.submit "Submit", class: "btn btn-primary" %>
    <% end %>
  </div>
</div>
```

// obr. 12.6 Formulář "zapomenuté heslo".

### Akce "create" resetů hesla

Po odeslání formuláře z předchozí části potřebujeme najít uživatele podle emailové adresy a aktualizovat její atributy tokenem pro reset hesla a timestampem pro čas, kdy byl požadavek odeslán. Poté přesměrujeme na kořenovou adresu s informativní flash zprávou. V případě neplatného odeslání překreslíme stránku "new" se zprávou "flash.now", jako u přihlášení. Ovladač tedy bude vypadat následovně (soubor "app/controllers/password_resets_controller.rb"):

```
class PasswordResetsController < ApplicationController

  def new
  end

  def create
    @user = User.find_by(email: params[:password_reset][:email].downcase)
    if @user
      @user.create_reset_digest
      @user.send_password_reset_email
      flash[:info] = "Email s instrukcemi pro reset hesla byl odeslán"
      redirect_to root_url
    else
      flash.now[:danger] = "Emailová adresa nenalezena"
      render 'new'
    end
  end

  def edit
  end
end
```

Následující kód v uživatelském modelu (soubor "app/models/user.rb") je podobný metodě "create_activation_digest" použité v callbacku "before_create" z minulé kapitoly:

```
class User < ApplicationRecord
  attr_accessor :remember_token, :activation_token, :reset_token
  before_save   :downcase_email
  before_create :create_activation_digest
  .
  .
  .
  # Aktivuje ucet.
  def activate
    update_attribute(:activated,    true)
    update_attribute(:activated_at, Time.zone.now)
  end

  # Posle aktivacni email.
  def send_activation_email
    UserMailer.account_activation(self).deliver_now
  end

  # Nastavi atributy pro reset hesla.
  def create_reset_digest
    self.reset_token = User.new_token
    update_attribute(:reset_digest,  User.digest(reset_token))
    update_attribute(:reset_sent_at, Time.zone.now)
  end

  # Odesle email pro reset hesla.
  def send_password_reset_email
    UserMailer.password_reset(self).deliver_now
  end

  private

    # Prevede email do malych pismen.
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

Jak vidno na obrázku (obr. 12.7), v současné chvíli je chování aplikace v rámci neplatné emailové adresy funkční. Aby vše fungovalo i po odeslání platné emailové adresy, musíme zadefinovat metodu maileru pro reset hesla.

// obr. 12.7 Formulář "zapomenuté heslo" pro neplatnou emailovou adresu.



##Emaily resetů hesla

Předchozí sekci jsme zanechali s téměř funkční akcí "create" v ovladači resetů hesla. Jediná chybějící věc je metoda pro doručení platných emailů pro reset hesla.

Základní metodu "password_reset" z minulé kapitoly máme vygenerovánu v soboru "app/mailers/user_mailer.rb". Podíváme se, jak by bylo vhodné ji upravit.

### Mailer a šablony resetů hesla

Minule jsme aplikovali stejný princip, jako v rámci refaktorování kódu v předchozí kapitole a vložili uživatelský mailer přímo do modelu:

```
UserMailer.password_reset(self).deliver_now
```

Metoda maileru restu hesla, kterou potřebujeme zprovoznit, je téměř identická s mailerem pro aktivace účtů z minulé kapitoly. Nejprve vytvoříme metodu "password_reset" v uživatelském maileru (soubor "app/mailers/user_mailer.rb") a zadefinujeme šablony pohledu pro textový (soubor "app/views/user_mailer/password_reset.text.erb") a HTML (soubor "app/views/user_mailer/password_reset.html.erb") email.

```
class UserMailer < ApplicationMailer

  def account_activation(user)
    @user = user
    mail to: user.email, subject: "Account activation"
  end

  def password_reset(user)
    @user = user
    mail to: user.email, subject: "Password reset"
  end
end
```

```
To reset your password click the link below:

<%= edit_password_reset_url(@user.reset_token, email: @user.email) %>

This link will expire in two hours.

If you did not request your password to be reset, please ignore this email and
your password will stay as it is.
```

```
<h1>Password reset</h1>

<p>To reset your password click the link below:</p>

<%= link_to "Reset password", edit_password_reset_url(@user.reset_token,
                                                      email: @user.email) %>

<p>This link will expire in two hours.</p>

<p>
If you did not request your password to be reset, please ignore this email and
your password will stay as it is.
</p>
```

Stejně, jako u předchozích emailů si můžeme i v tomto případě prohlédnout náhledy emailů za pomocí Rails previews (soubor "test/mailers/previews/user_mailer_preview.rb"):

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
    user = User.first
    user.reset_token = User.new_token
    UserMailer.password_reset(user)
  end
end
```

Po úpravě kódu naše připravené emaily vypadají jako na obrázcích (obr. 12.8 a 12.9).

// obr. 12.8 Náhled HTML verze emailu pro reset hesla.

// obr. 12.9 Náhled textové verze emailu pro reset hesla.

Odeslání samotné adresy teď bude vypadat, jako na obrázku (obr. 12.10). Související výpis ze serveru by měl vypadat zhruba následovně:

// obr. 12.10 Výsledek odeslání správného emailu.

```
Sent mail to michael@michaelhartl.com (66.8ms)
Date: Mon, 06 Jun 2016 22:00:41 +0000
From: noreply@example.com
To: michael@michaelhartl.com
Message-ID: <5407babbee139_8722b257d04576a@mhartl-rails-tutorial-953753.mail>
Subject: Password reset
Mime-Version: 1.0
Content-Type: multipart/alternative;
 boundary="--==_mimepart_5407babbe3505_8722b257d045617";
 charset=UTF-8
Content-Transfer-Encoding: 7bit


----==_mimepart_5407babbe3505_8722b257d045617
Content-Type: text/plain;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

To reset your password click the link below:

https://rails-tutorial-mhartl.c9users.io/password_resets/3BdBrXeQZSWqFIDRN8cxHA/
edit?email=michael%40michaelhartl.com

This link will expire in two hours.

If you did not request your password to be reset, please ignore this email and
your password will stay as it is.
----==_mimepart_5407babbe3505_8722b257d045617
Content-Type: text/html;
 charset=UTF-8
Content-Transfer-Encoding: 7bit

<h1>Password reset</h1>

<p>To reset your password click the link below:</p>

<a href="https://rails-tutorial-mhartl.c9users.io/
password_resets/3BdBrXeQZSWqFIDRN8cxHA/
edit?email=michael%40michaelhartl.com">Reset password</a>

<p>This link will expire in two hours.</p>

<p>
If you did not request your password to be reset, please ignore this email and
your password will stay as it is.
</p>
----==_mimepart_5407babbe3505_8722b257d045617--
```

### Testy emailu

Stejně, jako u aktivačního maileru si i teď napíšeme test pro metodu resetu hesla (soubor "test/mailers/user_mailer_test.rb"):

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

  test "password_reset" do
    user = users(:michael)
    user.reset_token = User.new_token
    mail = UserMailer.password_reset(user)
    assert_equal "Password reset", mail.subject
    assert_equal [user.email], mail.to
    assert_equal ["noreply@example.com"], mail.from
    assert_match user.reset_token,        mail.body.encoded
    assert_match CGI.escape(user.email),  mail.body.encoded
  end
end
```

Testy by měly proběhnout zeleně:

```
$ rails test
```

## Samotné resetování hesla

Když už máme k dispozici správně vygenerovaný email, napíšeme si akci "edit" v ovladači resetů hesla která reálně resetuje uživatelovo heslo. A stejně, jako dříve, si i teď zahrneme integrační test.

### Akce "edit" resetu

Emaily tedy obsahují odkazy v následující podobě:

```
https://example.com/password_resets/3BdBrXeQZSWqFIDRN8cxHA/edit?email=fu%40bar.com
```

Aby ale samotné odkazy fungovaly, musíme přidat i formulář pro resetování hesel. Jde o podobný úkon, jako když jsme aktualizovali uživatele skrze pohled, ale v tomto případě bude obsahovat pouze pole pro heslo a jeho potvrzení.

Je tu ještě dodatečná komplikace: předpokládáme, že najdeme uživatele podle emailové adresy, což znamená, že tuto hodnotu musíme mít k dispozici jak v akci "edit", tak "update". V akci "edit" sice email bude automaticky k dispozici díky přítomnosti výše zmíněného odkazu, ale jakmile odešleme formulář, hodnota bude ztracena. Řešení tkví v použití skrytého pole (hidden field) pro umístění (ale ne zobrazení) emailu na stránce, čímž se odešle spolu se zbytkem informací ve formuláři. Výsledná úprva (soubor "app/views/password_resets/edit.html.erb") vypadá následovně:

```
<% provide(:title, 'Reset password') %>
<h1>Reset password</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(@user, url: password_reset_path(params[:id])) do |f| %>
      <%= render 'shared/error_messages' %>

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

Používáme pomocníka pro tagy ve formuláři

```
hidden_field_tag :email, @user.email
```

namísto

```
f.hidden_field :email, @user.email
```

jelikož resetovací odkaz umístí email do "params[:email]", kdežto druhý zmíněný by ho uložil do "params\[:user]\[:email]".

Pro vykreslení formuláře potřebujeme definovat proměnnou "@user" v akci "edit" ovladače resetů hesla. Jako u aktivací, i zde to zahrnuje nalezení uživatele podle emailové adresy v "params[:email]". Poté potřebujeme ověřit, že je uživatel platný (tedy že existuje, je aktivní a autentifikovaný v rámci reset tokenu z "params[:id]"). Jelikož je existence platné proměnné "@user" potřeba jak v akci "edit", tak "update", umístíme kód pro nalezení do before filtrů (soubor "app/controllers/password_resets_controller.rb"):

```
class PasswordResetsController < ApplicationController
  before_action :get_user,   only: [:edit, :update]
  before_action :valid_user, only: [:edit, :update]
  .
  .
  .
  def edit
  end

  private

    def get_user
      @user = User.find_by(email: params[:email])
    end

    # Confirms a valid user.
    def valid_user
      unless (@user && @user.activated? &&
              @user.authenticated?(:reset, params[:id]))
        redirect_to root_url
      end
    end
end
```

Všimněme si použití

```
authenticated?(:reset, params[:id])
```

ku

```
authenticated?(:remember, cookies[:remember_token])
```

v případě minulé lekce a

```
authenticated?(:activation, params[:id])
```

tamtéž. Dohromady tyto tři použití tvoří autentifikační metody z tabulky v předchozí kapitole.

S upraveným kódem bude výše zmíněný odkaz schopen vykreslit formulář pro reset hesla (obr. 12.11).

// obr. 12.11 Formulář pro reset hesla.

### Aktualizace resetu

Narozdíl od aktivační metody "edit", která prostě přepne uživatele z "neaktivní" na "aktivní", metoda "edit" pro reset hesla je formulář, který musí být odeslán na související akci "update". Pro její zadefinování potřebujeme zvážit čtyři případy:

1. Reset hesla vypršel
2. Aktualizace selhala kvůli neplatného hesla
3. Aktualizace selhala (i když se to nejprve tváří obráceně) kvůli prázdného hesla a potvrzení
4. Aktualizace proběhla úspěšně

Případy 1, 2 a 4 jsou poměrně přímočaré, ale případ 3 není tak úplně zřejmý a vysvětlíme si ho.

Případ 1 se vztahuje jak na akce "edit", tak "update", a tedy patří do before filtru:

```
before_action :check_expiration, only: [:edit, :update]    # Case (1)
```

Což bude vyžadovat zadefinování soukromé metody "check_expiration":

```
# Overi platnost (vyprseni) tokenu.
def check_expiration
  if @user.password_reset_expired?
    flash[:danger] = "Platnost resetu hesla vypršela."
    redirect_to new_password_reset_url
  end
end
```

V metodě "check_expiration" jsme odložili expirační ověření instanční metodě "password_reset_expired?", která je poněkud ošemetná, ale záhy si ji zadefinujeme.

Následující (velký) výpis shrnuje implementaci těchto filtrů, spolu s akcí "update", která řeší případy 2 až 4. Případ 2 je ošetřen aktualizací, která selhala, s chybovou hláškou ze sdíleného partialu a zobrazuje se automaticky při překreslení formuláře "edit". Případ 4 souvisí s úspěšnou změnou, a výsledek je podobný úspěšnému přihlášení.

Jediné neošetřené selhání, které neošetří případ 2 je situace, kdy je heslo prázdné, což momentálně umožňuje náš uživatelský model a bude ji třeba odchytit a ošetřit explicitně. Jde o případ 3 zmíněný výše. Náš postup bude spočívat v přidání chyby přímo do chybových zpráv objektu "@user" za pomocí "errors.add":

```
@user.errors.add(:password, :blank)
```

V takovém případě bude použita základní zpráva pro prázdný obsah, když heslo nebude vyplněno.

Celková úprava akce "update" tedy bude vypadat následovně (soubor "app/controllers/password_resets_controller.rb") a ošetřuje všechny čtyři případy.

```
class PasswordResetsController < ApplicationController
  before_action :get_user,         only: [:edit, :update]
  before_action :valid_user,       only: [:edit, :update]
  before_action :check_expiration, only: [:edit, :update]    # Case (1)

  def new
  end

  def create
    @user = User.find_by(email: params[:password_reset][:email].downcase)
    if @user
      @user.create_reset_digest
      @user.send_password_reset_email
      flash[:info] = "Email sent with password reset instructions"
      redirect_to root_url
    else
      flash.now[:danger] = "Email address not found"
      render 'new'
    end
  end

  def edit
  end

  def update
    if params[:user][:password].empty?                  # Case (3)
      @user.errors.add(:password, "can't be empty")
      render 'edit'
    elsif @user.update_attributes(user_params)          # Case (4)
      log_in @user
      flash[:success] = "Password has been reset."
      redirect_to @user
    else
      render 'edit'                                     # Case (2)
    end
  end

  private

    def user_params
      params.require(:user).permit(:password, :password_confirmation)
    end

    # Before filters

    def get_user
      @user = User.find_by(email: params[:email])
    end

    # Confirms a valid user.
    def valid_user
      unless (@user && @user.activated? &&
              @user.authenticated?(:reset, params[:id]))
        redirect_to root_url
      end
    end

    # Checks expiration of reset token.
    def check_expiration
      if @user.password_reset_expired?
        flash[:danger] = "Password reset has expired."
        redirect_to new_password_reset_url
      end
    end
end
```

Použili jsme metodu "user_params" a povolili jak atribut hesla, tak jeho potvrzení.

Jak jsme zmínili dříve, tato implementace převádí booleanský test pro vypršení resetu hesla do uživatelského modelu kódem

```
@user.password_reset_expired?
```

Aby fungoval, musíme zadefinovat metodu "password_reset_expired?". Platnost budeme považovat za vypršenou tehdy, pokud byl požadavek zaslán před více, než dvěma hodinami, což lze v Ruby vyjádřit následovně:

```
reset_sent_at < 2.hours.ago
```

Tento zápis může působit krapet matoucím dojmem pokud čteme < jako "méně, než", což vyzní jako "reset hesla byl odeslán před méně, než dvěma hodinami", což je opakem toho, co chceme. V tomto kontextu je lepší číst < jako "dříve, než", tedy v celém znění "reset byl zaslán dříve, než před dvěma hodinami". Což je to, co jsme chtěli. Výsledná definice funkce "password_reset_expired?" v souboru "app/models/user.rb" vypadá následovně:

```
class User < ApplicationRecord
  .
  .
  .
  # Vrati true pokud reset hesla vyprsel.
  def password_reset_expired?
    reset_sent_at < 2.hours.ago
  end

  private
    .
    .
    .
end
```

S touto úpravou už bude akce "update" fungovat. Výsledky platného a neplatného odeslání jsou na obrázcích (obr. 12.12 a 12.13).

// obr. 12.12 Neúspěšný reset hesla.

// obr. 12.13 Úspěšný reset hesla.

### Test resetu hesla

V této části si napíšeme integrační test pro platné a neplatné odeslání. Začneme vygenerováním testovacího souboru pro resety hesla:

```
$ rails generate integration_test password_resets
      invoke  test_unit
      create    test/integration/password_resets_test.rb
```

Postup v rámci testu resetů je podobný testům pro aktivace účtu z minulé kapitoly, i když jsou rozdíly v rozložení: nejprve navštívíme formulář "zapomenuté heslo" a odešleme neplatnou, a pak platnou adresu, kde druhá jmenovaná vytvoří reset token a odešle resetový email. Pak navštívíme odkaz z emailu a znovu odešleme neplatné a platné informace, čímž ověříme žádoucí chování v obou případech. Test samotný (soubor "test/integration/password_resets_test.rb") je mimojiné skvělým cvičením ve čtení kódu:

```
require 'test_helper'

class PasswordResetsTest < ActionDispatch::IntegrationTest

  def setup
    ActionMailer::Base.deliveries.clear
    @user = users(:michael)
  end

  test "password resets" do
    get new_password_reset_path
    assert_template 'password_resets/new'
    # Invalid email
    post password_resets_path, params: { password_reset: { email: "" } }
    assert_not flash.empty?
    assert_template 'password_resets/new'
    # Valid email
    post password_resets_path,
         params: { password_reset: { email: @user.email } }
    assert_not_equal @user.reset_digest, @user.reload.reset_digest
    assert_equal 1, ActionMailer::Base.deliveries.size
    assert_not flash.empty?
    assert_redirected_to root_url
    # Password reset form
    user = assigns(:user)
    # Wrong email
    get edit_password_reset_path(user.reset_token, email: "")
    assert_redirected_to root_url
    # Inactive user
    user.toggle!(:activated)
    get edit_password_reset_path(user.reset_token, email: user.email)
    assert_redirected_to root_url
    user.toggle!(:activated)
    # Right email, wrong token
    get edit_password_reset_path('wrong token', email: user.email)
    assert_redirected_to root_url
    # Right email, right token
    get edit_password_reset_path(user.reset_token, email: user.email)
    assert_template 'password_resets/edit'
    assert_select "input[name=email][type=hidden][value=?]", user.email
    # Invalid password & confirmation
    patch password_reset_path(user.reset_token),
          params: { email: user.email,
                    user: { password:              "foobaz",
                            password_confirmation: "barquux" } }
    assert_select 'div#error_explanation'
    # Empty password
    patch password_reset_path(user.reset_token),
          params: { email: user.email,
                    user: { password:              "",
                            password_confirmation: "" } }
    assert_select 'div#error_explanation'
    # Valid password & confirmation
    patch password_reset_path(user.reset_token),
          params: { email: user.email,
                    user: { password:              "foobaz",
                            password_confirmation: "foobaz" } }
    assert is_logged_in?
    assert_not flash.empty?
    assert_redirected_to user
  end
end
```

Většinu prvků už jsme dříve použili; jediný opravdů nový je test tagu "input":

```
assert_select "input[name=email][type=hidden][value=?]", user.email
```

Tento řádek kontroluje, že je přítomen tag "input" se správným jménem, typem (hidden) a emailovou adresou:

```
<input id="email" name="email" type="hidden" value="michael@example.com" />
```

Testy samotné by měly proběhnout zeleně:

```
$ rails test
```

## Email v produkčním nasazení (podruhé)



# OPET PORESIT, JESTLI ZAHRNEME, ZATIM DAVAM JEN POSLEDNI CAST S BRANCH MERGE



Teď je i ideální čas pro spojení vývojové větve s hlavní:

```
$ rails test
$ git add -A
$ git commit -m "Pridan reset hesla"
$ git checkout master
$ git merge password-reset
```



## Shrnutí

Z vyssich mist zakazano, kdyztak pripisu.