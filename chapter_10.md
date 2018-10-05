# Aktualizování, zobrazování a mazání uživatelů

V této kapitole dokončíme REST akce pro uživatelský zdroj přidáním akcí "edit", "update", "index" a "destroy". Začneme přidáním možnosti si aktualizovat vlastní profil, kde dalším logicky navazujícím krokem bude upevnění autorizačního modelu. Poté uděláme seznam všech uživatelů (kde bude autentifikace také vyžadována), kde nás čeká i práce s vzorovými daty a stránkováním. Nakonec přidáme i možnost uživatele likvidovat, tedy ve smyslu vymazání z databáze. Jelikož takovou schopnost nemůže mít každý řadový uživatel, vytvoříme i speciální třídu pro administrátory, kteří budou moci mazat ostatní.



## Aktualizování uživatelů

Následující kroky budou v podstatě podobné situaci, kdy jsme nové uživatele vytvářeli. Namísto akce "new", která vykresluje pohled pro nové uživatele ale použijeme akci "edit", která vykreslí pohled k jejich upravování; namísto "create", které by odpovídalo na POST požadavek budeme mít akci "update" reagující na požadavek PATCH. Hlavní rozdíl je ten, že i když se může registrovat kdokoliv, upravovat si profil bude moct jen současný uživatel a to jen v rámci svého profilu.

Začneme tedy pracovat na nové větvi:

```
$ git checkout -b updating-users
```

### Editační formulář

Začneme editačním formulářem, který je na mockupu (obr. 10.1). Bude třeba naplnit jak akci "edit" v uživatelském ovladači, tak pohled uživatelské editace. Akce "edit" bude tedy první krok a ta bude potřebovat vytáhnout souvisejícího uživatele z databáze. Jak víme z dřívějška, správná URL pro editační stránku uživatele je /users/1/edit (za předpokladu, že uživatelovo id je "1"). Samotné uživatelovo id je dostupné v proměnné "params[:id]", což znamená, že ho můžeme najít následovně (soubor "app/controllers/users_controller.rb"):

// obr. 10.1 Mockup editační stránky uživatelů.

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      log_in @user
      flash[:success] = "Vítejte v aplikaci!"
      redirect_to @user
    else
      render 'new'
    end
  end

  def edit
    @user = User.find(params[:id])
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

Odpovídající uživatelský pohled vypadá následovně (soubor "app/views/users/edit.html.erb", který ale bude třeba ručně vytvořit):

```
<% provide(:title, "Edit user") %>
<h1>Aktualizujte svůj profil</h1>

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

      <%= f.submit "Save changes", class: "btn btn-primary" %>
    <% end %>

    <div class="gravatar_edit">
      <%= gravatar_for @user %>
      <a href="http://gravatar.com/emails" target="_blank">změnit</a>
    </div>
  </div>
</div>
```

Použili jsme i sdílený partial "error_messages". Mimochodem, použití target="_blank" je praktický trik díky kterému prohlížeč otevře stránku v novém okně (nebo panelu), což je obvykle žádoucí chování, pokud odkazujeme mimo aplikaci.

S instanční proměnnou "@user" by se editační stránka měla vykreslit správně, jako na obrázku (obr. 10.2). Pole jména a emailu také ukazují, jak šikovně Rails automaticky předvyplní možnosti za pomocí atributů existující "@user" proměnné.

// obr. 10.2 Základní editační stránka uživatele s předvyplněným jménem a emailem.

Když se podíváme na HTML zdroj předchozího obrázku, vídíme, dle očekávání, tag "form":

```
<form accept-charset="UTF-8" action="/users/1" class="edit_user"
      id="edit_user_1" method="post">
  <input name="_method" type="hidden" value="patch" />
  .
  .
  .
</form>
```

Zajímavé je skryté vstupní pole:

```
<input name="_method" type="hidden" value="patch" />
```

Jelikož webové prohlížeče neumí nativně odesílat PATCH požadavky (jak vyžadují konvence REST), Rails problém obejde odesláním požadavku POST a skrytým "input" polem.

Ještě je tu ale jedna drobnost: kód "form_for(@user)" z editačního pohledu je úplně stejný, jako když jsme vytvářeli uživatele - takže jak Rails ví, kdy použít POST požadavek pro nové uživatele a PATCH pro jejich editaci? Je to jednoduše díky tomu, že lze zjistit, zda uživatel již existuje v databázi skrze metodu Active Record "new_record?":

```
$ rails console
>> User.new.new_record?
=> true
>> User.first.new_record?
=> false
```

Při vytváření formuláře skrze "form_for(@user)" použije Rails POST, pokud je "@user.new_record" o hodnotě "true" a PATCH pokud je "false".

Jako poslední drobnost naplníme URL nastavení do navigace stránky. Za pomocí jmenné cesty "edit_user_path" spolu s praktickou pomocnickou metodou "current_user" to nebude problém:

```
<%= link_to "Settings", edit_user_path(current_user) %>
```

Plný aplikační kód tedy bude vypadat následovně (soubor "app/views/layouts/_header.html.erb"):

```
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", root_path, id: "logo" %>
    <nav>
      <ul class="nav navbar-nav navbar-right">
        <li><%= link_to "Home", root_path %></li>
        <li><%= link_to "Help", help_path %></li>
        <% if logged_in? %>
          <li><%= link_to "Users", '#' %></li>
          <li class="dropdown">
            <a href="#" class="dropdown-toggle" data-toggle="dropdown">
              Account <b class="caret"></b>
            </a>
            <ul class="dropdown-menu">
              <li><%= link_to "Profile", current_user %></li>
              <li><%= link_to "Settings", edit_user_path(current_user) %></li>
              <li class="divider"></li>
              <li>
                <%= link_to "Log out", logout_path, method: :delete %>
              </li>
            </ul>
          </li>
        <% else %>
          <li><%= link_to "Log in", login_path %></li>
        <% end %>
      </ul>
    </nav>
  </div>
</header>
```

### Neúspěšné úpravy

Podobně, jako jsme nejprve vyřešili neúspěšné registrace, si i teď jako první ošetříme neúspěšné úpravy. Začneme vytvořením akce "update", která použije "update_attributes" k úpravě uživatele na základě odeslané haše "params". S neplatnými informacemi pokus o aktualizace vrátí "false", aby "else" část větve vykreslila editační stránku. Tento vzorec už jsme dříve viděli, ostatně, tato struktura je hodně podobná první verzi akce "create". Upravíme si tedy soubor "app/controllers/users_controller.rb":

```
class UsersController < ApplicationController

  def show
    @user = User.find(params[:id])
  end

  def new
    @user = User.new
  end

  def create
    @user = User.new(user_params)
    if @user.save
      log_in @user
      flash[:success] = "Vítejte v aplikaci!"
      redirect_to @user
    else
      render 'new'
    end
  end

  def edit
    @user = User.find(params[:id])
  end

  def update
    @user = User.find(params[:id])
    if @user.update_attributes(user_params)
      # Osetri uspesnou aktualizaci.
    else
      render 'edit'
    end
  end

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

V rámci volání "update_attributes" jsme použili "user_params", které používá silné parametry kvůli prevence zranitelnosti masového přiřazování (jak jsme si popsali dříve).

Díky existujícím validacím a partialu s řadou chybových hlášek má odeslání neplatných informací za výsledek spoustu praktických chybových zpráv.

// obr. 10.3 Chybové zprávy po odeslání aktualizačního formuláře.

### Testování neúspěšných úprav

Podle zavedeného postupu nás teď čeká napsání integračního testu pro odchyt případných chyb:

```
$ rails generate integration_test users_edit
      invoke  test_unit
      create    test/integration/users_edit_test.rb
```

Po vytvoření si otestujeme neúspěšnou editaci. Test v následujícím výpisu (soubor "test/integration/users_edit_test.rb") ověřuje správnost chování ověřením, že editační šablona je vykreslena po získání stránky a překreslena po odeslání neplatných informací. Pro odeslání PATCH požadavku jsme použili metodu "patch", podobně jako u "get", "post" a "delete".

```
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end

  test "unsuccessful edit" do
    get edit_user_path(@user)
    assert_template 'users/edit'
    patch user_path(@user), params: { user: { name:  "",
                                              email: "foo@invalid",
                                              password:              "foo",
                                              password_confirmation: "bar" } }

    assert_template 'users/edit'
  end
end
```

V této fázi by měla být naše sada testů zelená:

```
$ rails test
```

### Úspěšné úpravy (s pomocí "testování předem")

Je pomalu čas náš editační formulář zprovoznit. Díky outsourcování obrázku avatara (skrze Gravatar) máme vyřešený profilový obrázek; k jejich změně pak slouží tlačítko "změnit" (viz obr. 10.4). Zbytek funkcionality si tedy doplníme teď.

// obr. 10.4 Ořezávací prostředí Gravataru.

Čím víc si na testování zvykneme, tím praktičtěji pak zní i myšlenka, že vyjde líp si napsat integrační test před aplikačním kódem, namísto opačného postupu. V tomto kontextu jsou podobné testy občas nazývané jako "přijímací testy" (acceptance tests), jelikož rozhodují, zda bude daný prvek přijat jakožto kompletní. Použijeme tedy metodiku TDD ("testování předem") na prvek uživatelských úprav.

Otestujeme správnost chování aktualizace uživatelů pomocí testu, který bude podobný jako test na neúspěšné editace, jen tentokrát odešleme informace platné. Pak ověříme přítomnost neprázdné flash zprávy a úspěšné přesměrování na stránku profilu, přičemž ověříme i úspěšnou změnu uživatelových informací v databázi. Upravíme si tedy soubor "test/integration/users_edit_test.rb" podle následujícího výpisu. Heslo a jeho potvrzení jsme nechali prázdné, což je praktické pro uživatele, kteří si nechtějí měnit heslo pokaždé, kdy si mění jméno nebo email. Všimněme si i použití "@user.reload" pro znovunahrání uživatelských hodnot z databáze pro ověření, že byly úspěšně aktualizovány.

```
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "successful edit" do
    get edit_user_path(@user)
    assert_template 'users/edit'
    name  = "Foo Bar"
    email = "foo@bar.com"
    patch user_path(@user), params: { user: { name:  name,
                                              email: email,
                                              password:              "",
                                              password_confirmation: "" } }
    assert_not flash.empty?
    assert_redirected_to @user
    @user.reload
    assert_equal name,  @user.name
    assert_equal email, @user.email
  end
end
```

Akce "update" potřebná pro projití testů je podobná konečné formě akce "create" (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  .
  .
  .
  def update
    @user = User.find(params[:id])
    if @user.update_attributes(user_params)
      flash[:success] = "Profil aktualizován"
      redirect_to @user
    else
      render 'edit'
    end
  end
  .
  .
  .
end
```

Testy proběhnou nicméně červeně, což je důsledek ověření délky hesla, resp. jeho selhání právě kvůli prázdnému heslu a potvrzení. Aby testy proběhly zeleně, musíme připravit výjimku na ověření hesla pokud je prázdné. Stačí předat možnost "allow_nil: true" do "validates", což vypadá následovně (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }, allow_nil: true
  .
  .
  .
end
```

Zároveň nemusíme mít obavu, že by se noví uživatelé registrovali s prázdným heslem, jelikož "has_secure_password" zahrnuje vlastní ověření přítomnosti které specificky odchytává hesla typu "nil".

Díky těmto úpravám bude editační stránka fungovat, což si můžeme ověřit i opětovným spuštěním testů, které by měly být zelené:

```
$ rails test
```



## Autorizace

V kontextu webových aplikací nám autentifikace umožňuje identifikovat uživatele naší stránky a autorizace mít pod kontrolou, co mohou dělat. Díky předchozí implementaci autentifikace můžeme nyní snadno implementovat i autorizaci.

I když jsou akce úprav a aktualizací funkčně vzato kompletní, trpí obrovskou bezpečnostní dírou: umožňují komukoliv (i nepřihlášeným uživatelům) přistupovat k oběma akcím a upravovat informace o jakémkoliv uživateli. V této části se tedy budeme věnovat bezpečnostnímu modelu, který bude vyžadovat přihlášení uživatelů a nenechá je upravit kohokoliv jiného, než sebe.

Ošetříme si také nepřihlášené (ale registrované) uživatele v případech, kdy se snaží přistoupit na stránky, ke kterým by po přihlášení měli normální přístup. Přesměrujeme je na přihlašovací stránku spolu s pomocnou zprávou (viz mockup na obr. 10.6). A naopak, uživatelé, kteří se budou snažit dostat na stránku, ke které by přístup nikdy mít neměli (jako přihlášený uživatel, který se snaží upravit jiného) budou přesměrování na kořenovou adresu aplikace.

// obr. 10.6 Mockup výsledku pokusu o přístup na chráněnou stránku.

### Vyžadování přihlášení uživatelů

Pro implementaci chování z předchozího obrázku použijeme tzv. "before filter" v ovladači uživatelů. Tyto filtry používají příkaz "before_action" pro zavolání metody před danou akcí. Abychom mohli vyžadovat přihlášení, zadefinujeme si metodu "logged_in_user" a zavoláme jí skrze "before_action :logged_in_user", jak je vidět na následujícím výpisu (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end

    # Before filtry

    # Potvrdi prihlaseneho uzivatele.
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Přihlaste se, prosím."
        redirect_to login_url
      end
    end
end
```

Nativně se before filtry aplikují na každou akci v ovladači, takže jsme náš filtr omezili pouze na akce ":edit" a ":update" skrze haš možností "only:".

Dopad filtru můžeme vidět tak, že se odhlásíme a pokusíme se přistoupit k editační stránce "/users/1/edit" (viz obr. 10.7).

// obr. 10.7 Přihlašovací formulář po pokusu o přístup ke chráněné stránce.

Testy budou nyní vycházet červeně:

```
$ rails test
```

Je tomu tak proto, že akce "edit" a "update" nyní vyžadují přihlašeného uživatele, ale v daných testech žádný uživatel přihlášen není.

Opravíme naší sadu testů přihlášením uživatele před jakoukoliv upravovací akcí. Díky pomocníkovi "log_in_as" je to naštěstí jednoduché (soubor "test/integration/users_edit_test.rb"):

```
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end

  test "unsuccessful edit" do
    log_in_as(@user)
    get edit_user_path(@user)
    .
    .
    .
  end

  test "successful edit" do
    log_in_as(@user)
    get edit_user_path(@user)
    .
    .
    .
  end
end
```

Testy by měly vyjít zeleně:

```
$ rails test
```

Ačkoliv naše sada testů momentálně prochází, nedokončili jsme náš filtr, jelikož testy stále prochází zeleně i po odstranění našeho bezpečnostního modelu, což si lze ověřit jeho zakomentováním. Pochopitelně to není dobré, jelikož testy mají sloužit k odchytávání podobných věcí, a masivní bezpečnostní díra mezi ně rozhodně spadá a předchozí kód by měl vyjít červeně. Upravíme si tedy testy tak, ať tomu odpovídají.

Jelikož filtr pracuje na bázi "jednou-za-akci", umístíme související testy do testu ovladače uživatelů. Plán spočívá ve spuštění "edit"a "update" akcí se správnými požadavky a ověření, že je nastavený flash a uživatel je přesměrován na přihlašovací adresu. Vhodné požadavky jsou GET a PATCH, takže použijeme metody "get" a "patch" i uvnitř testů (soubor "test/controllers/users_controller_test.rb"):

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "should redirect edit when not logged in" do
    get edit_user_path(@user)
    assert_not flash.empty?
    assert_redirected_to login_url
  end

  test "should redirect update when not logged in" do
    patch user_path(@user), params: { user: { name: @user.name,
                                              email: @user.email } }
    assert_not flash.empty?
    assert_redirected_to login_url
  end
end
```

Druhý test zahrnuje použití metody "patch" pro odeslání PATCH požadavku na "user_path(@user)". Takovýto požadavek je přesměrován na akci "update" v uživatelském ovladači.

Testy by nyní měly proběhnout zeleně:

```
$ rails test
```

Jakékoliv odhalení editačních metod neautorizovaným uživatelům bude nyní odchyceno sadou testů.



### Vyžadování správného uživatele

Samozřejmě, samotné vyžadování přihlášení stačit nebude; uživatelé by měli mít možnost upravovat pouze své vlastní informace. Jak jsme si všimli posledně, je až přiliš snadné nechat testy přehlédnout důležité bezpečnostní riziko, takže budeme postupovat metodikou testování předem, abychom naimplementovali náš bezpečnostní model správně. Přidáme tedy testy do testu ovladače uživatelů, které doplňují testy předchozí.

Abychom se mohli ujistit, že uživatelé nemohou upravovat cizí informace, musíme mít možnost se přihlásit jako druhý uživatel. Přidáme si tedy druhého uživatele do souboru s fixturami (soubor "test/fixtures/users.yml"):

```
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>
```

Použitím metody "log_in_as" můžeme otestovat akce "edit" a "update" následujícím způsobem (soubor "test/controllers/users_controller_test.rb"). Mimochodem, předpokládáme přesměrování uživatele na kořenovou adresu namísto přihlašovací, jelikož uživatel pokoušející se upravit jiného už přihlášený bude.

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  def setup
    @user       = users(:michael)
    @other_user = users(:archer)
  end
  .
  .
  .
  test "should redirect edit when logged in as wrong user" do
    log_in_as(@other_user)
    get edit_user_path(@user)
    assert flash.empty?
    assert_redirected_to root_url
  end

  test "should redirect update when logged in as wrong user" do
    log_in_as(@other_user)
    patch user_path(@user), params: { user: { name: @user.name,
                                              email: @user.email } }
    assert flash.empty?
    assert_redirected_to root_url
  end
end
```

K přesměrování použijeme druhou metodu zvanou "correct_user" spolu s before filtrem k jejímu zavolání. Filtr "correct_user" definuje proměnnou "@user", takže můžeme v následujícím výpisu odstranit přiřazení "@user" v akcích "edit" a "update" (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user,   only: [:edit, :update]
  .
  .
  .
  def edit
  end

  def update
    if @user.update_attributes(user_params)
      flash[:success] = "Profile updated"
      redirect_to @user
    else
      render 'edit'
    end
  end
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end

    # Before filtry

    # Potvrdi prihlaseneho uzivatele.
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Přihlaste se, prosím."
        redirect_to login_url
      end
    end

    # Potvrdi spravneho uzivatele.
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless @user == current_user
    end
end
```

Testy by měly být zelené:

```
$ rails test
```

V rámci refaktorace kódu ještě přejmeme běžnou zvyklost a zadefinujeme si booleanskou metodu "current_user?" pro použití ve filtru "correct_user" (soubor "app/helpers/sessions_helper.rb"). Použijeme ji pro nahrazení kódu jako

```
unless @user == current_user
```

poněkud lepším tvarem

```
unless current_user?(@user)
```

```
module SessionsHelper

  # Prihlasi daneho uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end

  # Zapamatuje si uzivatele v trvalem sezeni.
  def remember(user)
    user.remember
    cookies.permanent.signed[:user_id] = user.id
    cookies.permanent[:remember_token] = user.remember_token
  end

  # Vrati true pokud je dany uzivatel soucasny uzivatel.
  def current_user?(user)
    user == current_user
  end

  # Vrati uzivatele, ktery odpovida zapamatovanemu tokenu v cookie.
  def current_user
    .
    .
    .
  end
  .
  .
  .
end
```

Nahrazení přímého srovnání booleanskou metodou nám umožní použít následující kód (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user,   only: [:edit, :update]
  .
  .
  .
  def edit
  end

  def update
    if @user.update_attributes(user_params)
      flash[:success] = "Profile updated"
      redirect_to @user
    else
      render 'edit'
    end
  end
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end

    # Before filters

    # Potvrdi prihlaseneho uzivatele.
    def logged_in_user
      unless logged_in?
        flash[:danger] = "Přihlaste se, prosím."
        redirect_to login_url
      end
    end

    # Potvrdi spravneho uzivatele.
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless current_user?(@user)
    end
end
```



### Přátelské přesměrování

I když je naše autorizace hotová, zbývá vyřešit malý neduh. Pokud se uživatel pokouší přistoupit na chráněnou stránku, je přesměrován na svůj profil nehledě na to, kam chtěl jít. Jinak řečeno, pokud se nepřihlášený uživatel pokusí jít na editační stránku, bude po přihlášení přesměrován na /users/1 namísto /users/1/edit. Bylo by přecejen milé ho přesměrovat tam, kam chtěl původně jít.

Aplikační kód bude relativně komplikovaný, ale můžeme napsat test na přátelské přesměrování velmi jednoduše, a to obrácením pořadí přihlášení a navštívení editační stránky. Jak vidno v následujícím výpisu, test se pousí navštívít editační stránku, poté se přihlásí a ověří, že je uživatel přesměrován na editační stránku namísto běžné profilové (soubor "test/integration/users_edit_test.rb"):

```
require 'test_helper'

class UsersEditTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "successful edit with friendly forwarding" do
    get edit_user_path(@user)
    log_in_as(@user)
    assert_redirected_to edit_user_url(@user)
    name  = "Foo Bar"
    email = "foo@bar.com"
    patch user_path(@user), params: { user: { name:  name,
                                              email: email,
                                              password:              "",
                                              password_confirmation: "" } }
    assert_not flash.empty?
    assert_redirected_to @user
    @user.reload
    assert_equal name,  @user.name
    assert_equal email, @user.email
  end
end
```

Když máme hotový test pro selhání, můžeme naimplementovat přátelské přesměrování. Abychom uživatele posunuli tam, kam chtěli jít, musíme cílovou destinaci někam uložit a následně na ní přesměrovat. Použijeme pár metody, konkrétně "store_location" a "redirect_back_or", a obě si zadefinujeme do pomocníka sezení (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper
  .
  .
  .
  # Presmeruje na ulozenou lokaci (nebo na zakladni).
  def redirect_back_or(default)
    redirect_to(session[:forwarding_url] || default)
    session.delete(:forwarding_url)
  end

  # Ulozi URL ke ktere se dotycny pokusil pristoupit.
  def store_location
    session[:forwarding_url] = request.original_url if request.get?
  end
end
```

Ukládací mechanismus pro URL je stejný "session" princip, který jsme použili pro přihlášení uživatele. Používáme také objekt "request" (skrze "request.original_url") pro zjištění URL požadované stránky.

Metoda "store_location" uloží požadovanou URL do proměnné "session" pod klíč ":forwarding_url", ale pouze pro GET požadavek. To zabrání uložení URL pokud uživatel, řekněmě, odešle formulář když není přihlášen (což se v krajním případě může stát pokud uživatel smazal cookies sezení ručně před odesláním formuláře). V takovém případě bude následující přesměrování zadávat požadavek GET na URL, která čeká POST, PATCH nebo DELETE, což způsobí chybu. Zahrnutí "if request.get?" tomu předejde.

Abychom reálně využili "store_location", musíme ho přidat do before filtru "logged_in_user" (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:edit, :update]
  before_action :correct_user,   only: [:edit, :update]
  .
  .
  .
  def edit
  end
  .
  .
  .
  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end

    # Before filtry

    # Potvrdi prihlaseneho uzivatele.
    def logged_in_user
      unless logged_in?
        store_location
        flash[:danger] = "Přihlaste se, prosím."
        redirect_to login_url
      end
    end

    # Potvrdi spravneho uzivatele.
    def correct_user
      @user = User.find(params[:id])
      redirect_to(root_url) unless current_user?(@user)
    end
end
```

Pro implementaci samotného přesměrování použijeme metodu "redirect_back_or" pro přesměrování na danou URL, pokud existuje, nebo na nativní URL, když neexistuje. Přidáme jí do akce "create" v ovladači sezení pro přesměrování po úspěšném přihlášení. Metoda "redirect_back_or" používá operátor || ve formě

```
session[:forwarding_url] || default
```

Což vyhodnotí jako volbu "session[:forwarding_url]" pokud není "nil", a pokud je, použije dodanou nativní URL. Je pak samozřejmě třeba i přesměrovací URL odstranit (skrze "session.delete(:forwarding_url)"), jinak by postupné pokusy o přihlášení měly za výsledek přesměrování na chráněnou stránku dokud by uživatel nezavřel prohlížeč. Mazání sezení je provedeno i když je přesměrování napsáno dříve, jelikož samotné přesměrování se neprovede dokud metoda nedojde do konce, nebo není zavoláno "return" (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController
  .
  .
  .
  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      params[:session][:remember_me] == '1' ? remember(user) : forget(user)
      redirect_back_or user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end
  .
  .
  .
end
```

Integrační test přesměrování by měl nyní projít, a základní uživatelská autentifikace a ochrana stránek je tím hotová. Jako obvykle je dobré si to ověřit skrze spuštění testů:

```
$ rails test
```



## Zobrazení seznamu všech uživatelů

V této částí si přidáme předposlední uživatelskou akci, "index", kterou si lze zobrazit seznam všech uživatelů, namísto jen jednoho. Během toho se naučíme i jak naplnit databázi vzorovými uživateli a jak vyřešit stránkování výstupu uživatelů, aby jich index stránka zvládla vypsat i potenciálně velké množství. Mockup výsledku, tedy uživatelů, stránkování a navigačního odkazu na seznam je na obrázku (obr. 10.8). V další části si přidáme ještě administrativní rozhraní k seznamu uživatelů, abychom je mohli snadno i smazat.

// obr. 10.8 Mockup stránky se seznamem uživatelů.



### Seznam uživatelů

Jako první krok v rámci seznamu uživatelů si zaimplementujeme bezpečnostní model. I když budou jednotlivé uživatelské "show" stránky přístupné všem návštěvníkům, "index" uživatelů bude omezen pro přihlášené uživatele, abychom měli pod kontrolou kolik toho mohou neregistrovaní uživatelé vidět.

Pro ochranu stránky "index" před neautorizovaným přístupem si přidáme krátký test, který ověří, že je akce "index" správně přesměrována (soubor "test/controllers/users_controller_test.rb").

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  def setup
    @user       = users(:michael)
    @other_user = users(:archer)
  end

  test "should redirect index when not logged in" do
    get users_path
    assert_redirected_to login_url
  end
  .
  .
  .
end
```

Pak už jen stačí přidat akci "index" a zahrnout ji do seznamu akcí, které jsou chráněny before filtrem "logged_in_user" (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update]
  before_action :correct_user,   only: [:edit, :update]

  def index
  end

  def show
    @user = User.find(params[:id])
  end
  .
  .
  .
end
```

Pro zobrazení samotných uživatelů budeme potřebovat proměnnou, která obsahuje všechny uživatele stránky a každého z nich vypíšeme skrze iteraci v pohledu indexu. Pro vytažení všech uživatelů z databáze můžeme použít "User.all" a přiřadit je do instanční proměnné "@users" pro použití v pohledu (soubor "app/controllers/users_controller.rb"). (Pokud zní zobrazování všech uživatelů najednou jako špatný nápad, zní to tak oprávněně, ale v další části si výpis upravíme vhodněji.)

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update]
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

Pro vytvoření skutečné seznamové stránky si vytvoříme pohled (jehož soubor si také musíme vytvořit) který iteruje skrze uživatele a každého "zabalí" do "li" tagu. Použijeme metodu "each", zobrazíme Gravatara a jméno každého uživatele a celé to úhledně zabalíme do tagu "ul" (soubor "app/views/users/index.html.erb").

```
<% provide(:title, 'All users') %>
<h1>Všichni uživatelé</h1>

<ul class="users">
  <% @users.each do |user| %>
    <li>
      <%= gravatar_for user, size: 50 %>
      <%= link_to user.name, user %>
    </li>
  <% end %>
</ul>
```

Využijeme i pomocníka Gravataru pro specifikaci velikosti jednotlivých obrázků (což se předává jako parametr) (soubor "app/helpers/users_helper.rb"):

```
module UsersHelper

  # Vrati Gravatara daneho uzivatele.
  def gravatar_for(user, options = { size: 80 })
    gravatar_id = Digest::MD5::hexdigest(user.email.downcase)
    size = options[:size]
    gravatar_url = "https://secure.gravatar.com/avatar/#{gravatar_id}?s=#{size}"
    image_tag(gravatar_url, alt: user.name, class: "gravatar")
  end
end
```

Přidáme si taky trochu CSS (resp. SCSS), aby náš seznam trochu vypadal (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* Users index */

.users {
  list-style: none;
  margin: 0;
  li {
    overflow: auto;
    padding: 10px 0;
    border-bottom: 1px solid $gray-lighter;
  }
}
```

Nakonec přidáme i URL do hlavní navigace stránky v hlavičce za pomocí "users_path", čímž využijeme i poslední jmennou cestu (soubor "app/views/layouts/_header.html.erb"):

```
<header class="navbar navbar-fixed-top navbar-inverse">
  <div class="container">
    <%= link_to "sample app", root_path, id: "logo" %>
    <nav>
      <ul class="nav navbar-nav navbar-right">
        <li><%= link_to "Home", root_path %></li>
        <li><%= link_to "Help", help_path %></li>
        <% if logged_in? %>
          <li><%= link_to "Users", users_path %></li>
          <li class="dropdown">
            <a href="#" class="dropdown-toggle" data-toggle="dropdown">
              Účet <b class="caret"></b>
            </a>
            <ul class="dropdown-menu">
              <li><%= link_to "Profile", current_user %></li>
              <li><%= link_to "Settings", edit_user_path(current_user) %></li>
              <li class="divider"></li>
              <li>
                <%= link_to "Log out", logout_path, method: :delete %>
              </li>
            </ul>
          </li>
        <% else %>
          <li><%= link_to "Log in", login_path %></li>
        <% end %>
      </ul>
    </nav>
  </div>
</header>
```

Uživatelský index je teď plně funkční a všechny testy by měly být zelené:

```
$ rails test
```

Na druhou straný je výpis (obr. 10.9) poněkud skromný, takže ho trochu obohatíme.

// obr. 10.9 Seznam uživatelů s pouze jedním uživatelem.

### Vzoroví uživatelé

V této části přidáme našemu osamocenému uživateli trochu společnosti. Pochopitelně bychom mohli použít prohlížeč, otevřít registrační stránku a po jednom uživatele vytvořit, ale na takové problémy má Ruby skvělé řešení a uživatele vytvoří za nás.

V prvé řadě přidáme do Gemfile gem "Faker", který nám umožní vytvořit vzorové uživatele s polorealistickými jmény a emaily. (Za normálních okolností by se gem "faker" omezoval na vývojové prostředí, ale v našem případě ho budeme chtít použít i v produkčním.)

```
source 'https://rubygems.org'

gem 'rails',          '5.1.6'
gem 'bcrypt',         '3.1.12'
gem 'faker',          '1.7.3'
.
.
.
```

Poté, jako obvykle, spustíme instalaci:

```
$ bundle install
```

Dále přidáme program Ruby pro zaplnění databáze vzorovými uživateli, pro které Rails používá standardní soubor "db/seeds.rb". (Kód je poněkud pokročilý, ale nemusíme se jím zatím zaobírat do detailu.)

```
User.create!(name:  "Example User",
             email: "example@railstutorial.org",
             password:              "foobar",
             password_confirmation: "foobar")

99.times do |n|
  name  = Faker::Name.name
  email = "example-#{n+1}@railstutorial.org"
  password = "password"
  User.create!(name:  name,
               email: email,
               password:              password,
               password_confirmation: password)
end
```

Kód vytvoří vzorového uživatele se jménem a adresou podobnou předchozímu uživateli a pak jich vytvoří dalších 99. Metoda "create!" je podobná jako "create", ale v případě neplatného uživatele pouze vyvolá výjimku namísto vrácení "false". Toto chování podstatně zjednodušuje proces odchytu bugů tím, že přechází "tiché" chyby.

Teď už nám stačí pouze restartovat databázi a zavolat zaplňovací úkol pomocí příkazu "db:seed":

```
$ rails db:migrate:reset
$ rails db:seed
```

Naplňení databáze může chvíli trvat, až několik minut u pomalejších systémů. Existují podle všeho i případy, kdy příkaz "reset" nešel spustit pokud server Rails běžel, takže ho možná bude třeba nejprve zastavit.

Po dokončení úkolu by naše aplikace měla mít 100 vzorových uživatelů (obr. 10.10).

// obr. 10.10 Stránka seznamu uživatelů se 100 uživateli.

###Stránkování (pagination)

Náš původní uživatel už samotou netrpí, ale vznikl nám právě opačný problém: příliš velké množství společníků, kteří jsou zobrazeni na té samé stránce. Momentálně je jich sto, což je už samo o sobě poměrně velké číslo, a na reálné stránce to může jít do tisíců. Řešení spočívá ve stránkování, tedy že se v jednu chvíli na jedné stránce objeví (například) jen třicet uživatelů.

V Rails existuje několik postupů jak vyřešit stránkování; použijeme jednu z těch jednoduchých a zároveň robustních, zvanou [will_paginate](http://wiki.github.com/mislav/will_paginate/). Aby fungovala jak má, musíme zahrnout jak "will_paginate" gem, tak "bootstrap-will_paginate", který nakonfiguruje will_paginate pro použití stránkovacích stylů Bootstrapu. Aktualizovaný soubor Gemfile vypadá následovně:

```
source 'https://rubygems.org'

gem 'rails',                   '5.1.6'
gem 'bcrypt',                  '3.1.12'
gem 'faker',                   '1.7.3'
gem 'will_paginate',           '3.1.6'
gem 'bootstrap-will_paginate', '1.0.0'
.
.
.
```

Poté spustíme "bundle install":

```
$ bundle install
```

Je také rozumné restartovat webserver pro ujištění, že se nové gemy správně nahrály.

Aby stránkování fungovalo, musíme přidat trochu kódu do pohledu indexu, aby Rails věděl, že má uživatele stránkovat. Také bude třeba nahradit "User.all" v akci "index" objektem, který se stránkováním umí pracovat. Začneme přidáním speciální metody "will_paginate" do pohledu (soubor "app/views/users/index.html.erb"); za chvíli uvidíme, proč se kód vyskytuje jak nad, tak pod seznamem uživatelů.

```
<% provide(:title, 'All users') %>
<h1>Seznam uživatelů</h1>

<%= will_paginate %>

<ul class="users">
  <% @users.each do |user| %>
    <li>
      <%= gravatar_for user, size: 50 %>
      <%= link_to user.name, user %>
    </li>
  <% end %>
</ul>

<%= will_paginate %>
```

Metoda "will_paginate" je poněkud kouzelná; uvnitř pohledu "users" automaticky hledá objekt "@users", načež zobrazí stránkovací odkazy pro přístup k ostatním stranám. Pohled ale zatím nefunguje, jelikož v současné chvíli "@users" obsahuje výsledky z "User.all", ale "will_paginate" vyžaduje nastránkování výsledků explicitně za pomocí metody "paginate":

```
$ rails console
>> User.paginate(page: 1)
  User Load (1.5ms)  SELECT "users".* FROM "users" LIMIT 30 OFFSET 0
   (1.7ms)  SELECT COUNT(*) FROM "users"
=> #<ActiveRecord::Relation [#<User id: 1,...
```

Všimněme si, že "paginate" bere jako parametr haši s klíčem ":page" a hodnotou rovnou požadované straně. "User.paginate" vytáhne uživatele z databáze po částech (nativně po třiceti) podle parametru ":page". Takže, například, stránka 1 jsou uživatelé 1-130, stránka 2 jsou uživatelé 31-60, atd. Pokud je "page" o hodnotě "nil", "paginate" prostě vrátí první stránku.

Použitím metody "paginate" můžeme stránkovat uživatele v aplikaci tak, že použijeme "paginate" namísto "all" v akci "index". Parametr "page" je brán z "params[:page]", který je generován automaticky díky "will_paginate" (soubor "app/controllers/users_controller.rb").

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update]
  .
  .
  .
  def index
    @users = User.paginate(page: params[:page])
  end
  .
  .
  .
end
```

Seznam uživatelů by měl nyní fungovat, jak je vidět na obrázku (obr. 10.11), ačkoliv bude možná třeba restartovat server Rails. Jelikož jsme zahrnuli "will_paginate" jak nad, tak pod seznam, stránkovací odkazy se zobrazují na obou místech.

// obr. 10.11 Seznam uživatelů se stránkováním.

Stačí kliknout na "2" nebo "Next" a zobrazí se další strana (obr. 10.12).

// obr. 10.12 Druhá stránka seznamu.

### Test seznamu uživatelů

Když už náš seznam uživatelů funguje, napíšeme si pro něj odlehčený test, který bude zahrnovat i minimální otestování stránkování z předchozí části. Postup bude spočívat v přihlášení se, navštívení adresy seznamu, ověření, že první stránka uživatelů je přítomna a potvrzení, že je na místě i stránkování. Pro poslední dva kroky budeme potřebovat dostatek uživatelů i v testovací databázi pro vyvolání stránkování, tedy více, než 30.

Ve fixturách jsme si vytvořili druhého uživatele už dříve, ale 30 je přecejen velké číslo na ruční vytváření. Naštěstí, jak jsme viděli u atributu "password_digest", soubory fixtur podporují vnořené Ruby, což znamená, že můžeme vytvořit dodatečných 30 uživatelů jednoduše (soubor "test/fixtures/users.yml", vytvořili jsme si i pár pojmenovaných uživatelů pro budoucí účely):

```
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>

lana:
  name: Lana Kane
  email: hands@example.gov
  password_digest: <%= User.digest('password') %>

malory:
  name: Malory Archer
  email: boss@example.gov
  password_digest: <%= User.digest('password') %>

<% 30.times do |n| %>
user_<%= n %>:
  name:  <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com" %>
  password_digest: <%= User.digest('password') %>
<% end %>
```

Zbývá napsat test pro seznam uživatelů. Nejprve si ho tedy vygenerujeme:

```
$ rails generate integration_test users_index
      invoke  test_unit
      create    test/integration/users_index_test.rb
```

Test samotný zahrnuje ověření, že je přítomen "div" s vyžadovanou třídou "pagination" a ověřením, že je přítomna i první strana uživatelů (soubor "test/integration/users_index_test.rb").

```
require 'test_helper'

class UsersIndexTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end

  test "index including pagination" do
    log_in_as(@user)
    get users_path
    assert_template 'users/index'
    assert_select 'div.pagination'
    User.paginate(page: 1).each do |user|
      assert_select 'a[href=?]', user_path(user), text: user.name
    end
  end
end
```

Sada testů by měla proběhnout zeleně:

```
$ rails test
```

### Refaktorování partials

Ostránkovaný seznam uživatelů je nyní hotový, ale ještě existuje jedno vylepšení, které by bylo škoda opomenout: Rails má skvělé nástroje pro tvorbu kompaktních pohledů, a v této sekci si refaktorujeme (přepracujeme) stránku seznamu tak, aby je patřičně využila. Jelikož je náš kód poctivě otestovaný, můžeme bez obav kód přepracovat aniž bychom se museli bát, že rozbijeme funkcionalitu naší stránky.

Prvním krokem bude nahrazení uživatelského tagu "li" voláním "render" (soubor "app/views/users/index.html.erb"):

```
<% provide(:title, 'All users') %>
<h1>Seznam uživatelů</h1>

<%= will_paginate %>

<ul class="users">
  <% @users.each do |user| %>
    <%= render user %>
  <% end %>
</ul>

<%= will_paginate %>
```

Nevoláme "render" na řetězec se jménem partialu, ale raději na proměnnou "user" třídy "User"; v daném kontextu Rails automaticky vyhledá partial nazvaný "\_user.html.erb", který musíme vytvořit (umístění "app/views/users/\_user.html.erb"):

```
<li>
  <%= gravatar_for user, size: 50 %>
  <%= link_to user.name, user %>
</li>
```

I když je už tohle jednoznačné zlepšení, můžeme to hnát ještě o třídu výš: zavolat "render" přímo na proměnnou "@users" (soubor "app/views/users/index.html.erb"):

```
<% provide(:title, 'All users') %>
<h1>Seznam uživatelů</h1>

<%= will_paginate %>

<ul class="users">
  <%= render @users %>
</ul>

<%= will_paginate %>
```

Rails předpokládá, že "@users" je seznam "User" objektů, ba co víc, když je zavolán seznamem uživatelů, Rails automaticky iteruje skrz každého z nich pomocí paritalu "_user.html.erb" (kde odvozuje jméno partialu podle jména třídy). Výsledkem je právě tento úžasně úsporný zápis kódu.

Jako po každém refaktorování je i teď záhodno ověřit, zda testy projdou na zelenou:

```
$ rails test
```



## Mazání uživatelů

Když máme seznam uživatelů hotový, zbývá pouze poslední klasická REST akce: "destroy". V této části si přidáme odkazy pro mazání uživatelů (viz. mockup na obr. 10.13) a zadefinujeme akci "destroy" potřebnou pro uskutečnění smazání. V prvé řadě si ale vytvoříme třídu administrátorů (též zkráceně adminů), kteří budou mít pro takovou operaci oprávnění.

// obr. 10.13 Mockup seznamu uživatelů s mazacími odkazy.

### Administrátoři

Administrátorská oprávnění označíme u uživatelů pomocí booleanského atributu "admin" v uživatelském modelu, který povede k booleanské metodě "admin?" pro zjištění administátorského statusu. Výsledný datový model je na obrázku (obr. 10.14).

// obr. 10.14 Uživatelský model s přidaným atributem "admin".

Jako obvykle si atribut "admin" přidáme skrze migraci a nezapomeneme zahrnout typ "boolean" v příkazovém řádku:

```
$ rails generate migration add_admin_to_users admin:boolean
```

Migrace přidá sloupec "admin" do tabulky "users". Použijeme i parametr "default: false" u "add_column", což znamená, že uživatelé nebudou automaticky vytvořeni s oprávněním administrátorů. (Bez parametru "default: false" by byl "admin" o hodnotě "nil", což je sice pořád "false", ale tímto způsobem dáváme více explicitně najevo, jaké máme úmysly. To se vyplácí nejen z pohledu Railsu, který pak často zvládá lépe odhadovat, co se snažíme udělat, ale i pro případné další čtenáře našeho kódu.)

Upravíme si tedy migraci (soubor "db/migrate/[timestamp]_add_admin_to_users.rb") následovně:

```
class AddAdminToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :admin, :boolean, default: false
  end
end
```

Pak už ji jen spustíme:

```
$ rails db:migrate
```

Rails si správně dovtípil booleanskou podstatu atributu "admin" a automaticky přidal i metodu "admin?":

```
$ rails console --sandbox
>> user = User.first
>> user.admin?
=> false
>> user.toggle!(:admin)
=> true
>> user.admin?
=> true
```

Otestovali jsme si i metodu "toggle!" pro přehození atributu "admin" z "false" na "true".

Jako poslední krok ještě upravíme naše vyplňovací data tak, aby byl první uživatel automaticky admin (soubor "db/seeds.rb"):

```
User.create!(name:  "Example User",
             email: "example@railstutorial.org",
             password:              "foobar",
             password_confirmation: "foobar",
             admin: true)

99.times do |n|
  name  = Faker::Name.name
  email = "example-#{n+1}@railstutorial.org"
  password = "password"
  User.create!(name:  name,
               email: email,
               password:              password,
               password_confirmation: password)
end
```

A zresetujeme databázi:

```
$ rails db:migrate:reset
$ rails db:seed
```

### Silné parametry podruhé

V rámci zaplňovacích dat jsme si mohli všimnout, že systém udělá z uživatele admina zahrnutím "admin: true" v inicializační haši. Podtrhuje to problematiku vystavování našich objektů "divokému" internetu: pokud bychom prostě jen předali inicializační haši z běžného webového požadavku, případný útočník by mohl poslat požadavek PATCH v následující podobě:

```
patch /users/17?admin=1
```

Čímž by nastavil uživateli číslo 17 administrátorská práva, což je pochopitelně obrovské bezpečnostní riziko.

Kvůli tohoto nebezpečí je nutné aktualizovat pouze atributy, které je bezpečné editovat skrze web. Jak jsme probírali dříve, od toho existují tzv. silné parametry, které se volají skrze "require" a "permit" na haši "params":

```
    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
```

Za povšimnutí stojí hlavně to, že "admin" není v seznamu povolených atributů. Právě to brání běžným uživatelům v získání administračních oprávnění v naší aplikaci.

### Akce "destroy"

Posledním krokem pro dokončení uživatelského zdroje je přidání mazacích odkazů a akce "destroy". Začneme přidáním odkazu ke každému uživateli na seznamu uživatelů a přístup k němu omezíme na uživatele s administrativním oprávněním. Výsledné odkazy "delete" budou zobrazeny jen tehdy, když je současný uživatel administrátor.

Upravíme si tedy soubor "app/views/users/_user.html.erb":

```
<li>
  <%= gravatar_for user, size: 50 %>
  <%= link_to user.name, user %>
  <% if current_user.admin? && !current_user?(user) %>
    | <%= link_to "delete", user, method: :delete,
                                  data: { confirm: "You sure?" } %>
  <% end %>
</li>
```

Všimněme si parametru "method: :delete", který řekne odkazu, ať zahrne potřebný DELETE požadavek. Každý odkaz jsme také zabalili do "if" podmínky, takže je uvidí jen administrátoři. Výsledek bude pro administrátora vypadat jako na obrázku (obr. 10.15).

Webové prohlížeče neumí nativně odesílat DELETE požadavky, takže je Rails imituje skrze JavaScript. Mazací odkazy tedy nebudou fungovat pokud má uživatel JavaScript vypnutý. V případě, že bychom museli podporovat i prohlížeče bez JavaScriptu lze předstírat DELETE požadavek za pomocí formuláře a požadavku POST, což funguje i bez JavaScriptu.

// obr. 10.15 Seznam uživatelů s mazacími odkazy.

Aby mazací odkazy fungovaly, musíme přidat akci "destroy", která najde příslušného uživatele a "zničí" (smaže) ho pomocí Active Record metody "destroy", načež přesměruje zpět na seznam uživatelů. Jelikož uživatelé musí být pro mazání přihlášeni, přidáme i ":destroy" do "logged_in_user" before filtru (soubor "app/controllers/users_controller.rb":

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy]
  before_action :correct_user,   only: [:edit, :update]
  .
  .
  .
  def destroy
    User.find(params[:id]).destroy
    flash[:success] = "Uživatel smazán"
    redirect_to users_url
  end

  private
  .
  .
  .
end
```

Akce "destroy" použivá zřetězení metod pro zkombinování "find" a "destroy" do jednoho řádku:

```
User.find(params[:id]).destroy
```

Administrátoři tedy mohou mazat uživatele skrze web díky tomu, že jsou jediní, kdo vidí mazací odkazy, ale stále nás trápí bezpečnostní díra: sofistikovanější útočník může skrze příkazový řádek zadat požadavek DELETE přímo a smazat jakéhokoliv uživatele. Abychom tedy stránku patřičně zabezpečili, musíme zavést kontrolu přístupu k akci "destroy", aby uživatele mohli mazat pouze administrátoři.

Použijeme opět before filtr, tentokrát pro omezení přístupu k akci "destroy" pouze administrátorům. Výsledný "admin_user" before filtr vypadá následovně (soubor "app/controllers/users_controller.rb"):

```
class UsersController < ApplicationController
  before_action :logged_in_user, only: [:index, :edit, :update, :destroy]
  before_action :correct_user,   only: [:edit, :update]
  before_action :admin_user,     only: :destroy
  .
  .
  .
  private
    .
    .
    .
    # Potvrdi, ze je uzivatel admin.
    def admin_user
      redirect_to(root_url) unless current_user.admin?
    end
end
```

### Testy mazání uživatelů

S něčím tak nebezpečným, jako je mazání uživatelů je třeba mít dobrou sadu testů pro očekávané chování. Začneme úpravou jedné z fixtur tak, že z ní uděláme administrátora (soubor "test/fixtures/users.yml"):

```
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
  admin: true

archer:
  name: Sterling Archer
  email: duchess@example.gov
  password_digest: <%= User.digest('password') %>

lana:
  name: Lana Kane
  email: hands@example.gov
  password_digest: <%= User.digest('password') %>

malory:
  name: Malory Archer
  email: boss@example.gov
  password_digest: <%= User.digest('password') %>

<% 30.times do |n| %>
user_<%= n %>:
  name:  <%= "User #{n}" %>
  email: <%= "user-#{n}@example.com" %>
  password_digest: <%= User.digest('password') %>
<% end %>
```

Podobně jako dříve umístíme testy akcí omezení přístupu do testů uživatelského ovladače. Jako u testu pro odhlášení použijeme "delete" pro zavolání požadavku DELETE přímo akci "destroy". Musíme ověřit dva případy: uživatelé, kteří nejsou přihlášení by měli být přesměrováni na přihlašovací stránku a uživatelé, kteří přihlášení jsou, ale nejsou administrátoři by měli být přesměrováni na stránku domovskou (soubor "test/controllers/users_controller_test.rb").

```
require 'test_helper'

class UsersControllerTest < ActionDispatch::IntegrationTest

  def setup
    @user       = users(:michael)
    @other_user = users(:archer)
  end
  .
  .
  .
  test "should redirect destroy when not logged in" do
    assert_no_difference 'User.count' do
      delete user_path(@user)
    end
    assert_redirected_to login_url
  end

  test "should redirect destroy when logged in as a non-admin" do
    log_in_as(@other_user)
    assert_no_difference 'User.count' do
      delete user_path(@user)
    end
    assert_redirected_to root_url
  end
end
```

Ujišťujeme se také, že počet uživatelů se nezmění pomocí metody "assert_no_difference".

Testy ověřují chování v případě neautorizovaného (ne-admina) uživatele, ale také chcem ověřit že admin může použit mazací odkaz k úspěšnému smazání uživatele. Jelikož se mazací odkazy objevují na seznamu uživatelů, přidáme tyto testy do příslušného testu seznamu uživatelů z dřívějška. Jediná ošemetná část je ověření, že je uživatel smazán poté, co admin klikne na mazací odkaz, čehož dosáhneme následovně:

```
assert_difference 'User.count', -1 do
  delete user_path(@other_user)
end
```

Tentokrát použijeme metodu "assert_difference" tak, že ověříme, že je uživatel smazán pomocí ověření, že se "User.count" sníží o 1 po zadání "delete" požadavku na příslušnou adresu.

Když dáme všechno dohromady, máme test, který kontroluje stránkování a mazání pro adminy i normální uživatele (soubor "test/integration/users_index_test.rb"):

```
require 'test_helper'

class UsersIndexTest < ActionDispatch::IntegrationTest

  def setup
    @admin     = users(:michael)
    @non_admin = users(:archer)
  end

  test "index as admin including pagination and delete links" do
    log_in_as(@admin)
    get users_path
    assert_template 'users/index'
    assert_select 'div.pagination'
    first_page_of_users = User.paginate(page: 1)
    first_page_of_users.each do |user|
      assert_select 'a[href=?]', user_path(user), text: user.name
      unless user == @admin
        assert_select 'a[href=?]', user_path(user), text: 'delete'
      end
    end
    assert_difference 'User.count', -1 do
      delete user_path(@non_admin)
    end
  end

  test "index as non-admin" do
    log_in_as(@non_admin)
    get users_path
    assert_select 'a', text: 'delete', count: 0
  end
end
```

Ověřujeme tedy i správné mazací odkazy, ale test přeskakujeme pokud je uživatel administrátor (který mazací odkaz nemá).

Mazací kód máme dobře protestovaný a sada by měla proběhnout zeleně:

```
$ rails test
```



## Shrnutí

Od prvního zahrnutí uživatelského ovladače už uběhlo hodně vody. Při zavedení se uživatelé ani nemohli registrovat; teď se mohou jak registrovat, tak přihlásit, odhlásit, prohlížet své profily, upravovat svá nastavení a vidět seznam všech uživatelů. A někteří mohou uživatele i mazat.

Aplikace má tedy solidní základy pro jakoukoliv stránku, která by vyžadovala uživatele s autentifikací a autorizací. V budoucnu nás čeká přidání drobností jako aktivační odkaz pro nově zaregistrované uživatele (čímž se ověřuje i platná emailová adresa) a reset hesla pro případ, že by někdo to své zapomněl.

Než postoupíme dále, sloučíme změny s hlavní větví:

```
$ git add -A
$ git commit -m "Dokoncena editace, aktualizace, seznam a mazani uzivatelu"
$ git checkout master
$ git merge updating-users
$ git push
```

### Co jsme se naučili

- Uživatelé mohou být aktualizováni pomocí editačního formuláře, který odešle PATCH požadavek akci "update".
- Bezpečné aktualizování skrze web se provádí skrze silné parametry.
- "Before" (před) filtry umožňují spouštět metody před specifickou akcí ovladače.
- Autorizaci implementujeme skrze before filtry.
- Autorizační testy použivají jak nízkoúrovňové příkazy pro odeslání specifických HTTP požadavků přímo akcím ovladače, tak vysokoúrovňové integrační testy.
- Přátelské přesměrování přesměruje uživatele tam, kam chtěli jít poté, co se přihlásí.
- Stránka se seznamem uživatelů  zobrazuje všechny uživatele.
- Rails používá standardní soubor "db/seeds.rb" pro zaplnění databáze vzorovými daty za pomocí příkazu "rails db:seed".
- Spuštění "render @users" automaticky zavolá "_user.html.erb" partial na každého uživatele v dané kolekci.
- Booleanský atribut "admin" zavolaný na uživatelský model automaticky vytvoří booleanskou metodu "admin?" pro uživatelské objekty.
- Administrátoři mohou mazat uživatele skrze web kliknutím na mazací odkaz, který odešle DELETE požadavek na akci "destroy" uživatelského ovladače.
- Můžeme vytvořit velké množství testovacích uživatelů za pomocí vnořeného Ruby uvnitř fixtur.