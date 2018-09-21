# Základní přihlášení

Když už se naši uživatelé mohou zaregistrovat, je na čase jim dát i možnost, jak se přihlásit a odhlásit. V této kapitole zaimplementujem základní, ale plně fungující přihlašovací systém: aplikace si udrží přihlášený stav až do doby, kdy uživatel zavře prohlízeč. Výsledný autentifikační systém nám umožní úpravy stránek a implementaci autorizačního modelu založeného na statusu přihlášení a identitě současného uživatele. Kupříkladu budeme moci upravit hlavičku stránky odkazy na přihlášení/odhlášení a také odkazem k profilu.

Systém, který si vytvoříme v této kapitole bude také sloužit jako základ pro následné pokročilejší úpravy. Už v příští kapitole si prohlížeč bude uživatele automaticky pamatovat i po svém zavření, posléze i pomocí zaškrtnutí kolonky "pamatuj si mě". Pokryjeme tím tři nejběžnější způsoby pojetí přihlašovacích systému na webu.

## Sezení (sessions)

[HTTP](http://en.wikipedia.org/wiki/Hypertext_Transfer_Protocol) je ["bezestavový" protokol](https://en.wikipedia.org/wiki/Stateless_protocol), takže každý požadavek vnímá jako nezávislou transakci, která nemůže použít informace z předchozích požadavků. HTTP jako takové si tedy nemůže pamatovat uživatelovu identitu v rámci stránek; tedy, webové aplikace požadující přihlášení uživatele musí použit sezení (angl. [session](http://en.wikipedia.org/wiki/Session), což je polo-permanentní spojení mezi dvěmi počítači (např. klientským, který použivá webový prohlížeč a serverem, na kterém běží Rails).

Nejběžnější technika pro implementaci sezení v Rails zahrnuje použití [cookies](http://en.wikipedia.org/wiki/HTTP_cookie), což jsou malé kousky textu umístěné v uživatelově prohlížeči. Jelikož cookies přetrvávají napříč stránkami, mohou udržovat informace (jako uživatelovo id), které mohou být použity aplikací k získání přihlášeného uživatele z databáze. Prozatím použijeme metodu Rails zvanou "session" k vytvoření dočasných sezení, které automaticky vyprší spolu se zavřením prohlížeče. Posléze si ukážeme i trvanlivější metodu v podobě "cookies".

Je praktické modelovat sezení jako "RESTful" zdroj: návštěva přihlašovací stránky vykreslí formulář pro nové sezení, přihlášení vytvoří sezení a odhlášení ho zničí. Narozdil od zdroje uživatelů, který používá databázi pro uchování dat, zdroj sezení bude používáat cookies, a většina práce ohledně přihlašování pochází právě z vytvoření autentifikační mechaniky, která je založena na cookies. V této a další části připravíme ovladač sezení, přihlašovací formulář a související akce ovladače. Poté přidáme nezbytný kód pro manipulaci se sezeními.

Opět si tedy vytvoříme vývojovou větev a na konci ji spojíme s hlavní:

```
$ git checkout -b basic-login
```

### Ovladač sezení (Sessions controller)

Prvky přihlašování a odhlašování korespondují s akcemi REST ovladače sezení: přihlašovací formulář je obsloužen akcí "new" (kterou si osvětlíme nyní), přihlášení jako takové je obslouženo zasláním požadavku "POST" akci "create" a odhlášení je obslouženo zasláním požadavku "DELETE" akci "destroy".

Pro začátek si vygenerujeme ovladač sezení s akcí "new":

```
$ rails generate controller Sessions new
```

(Zahrnutí akce "new" ve skutečnosti vygeneruje i pohledy, díky čemuž nemusíme zahrnovat akce jako "create" a "destroy" které s pohledy nesouvisí.) Obdobným postupem jako v předchozí kapitole si i teď chceme vytvořit přihlašovací formulář pro vytvoření nových sezení, jak je načrtnuto na mockupu (obr. 8.1).

// obr. 8.1 Mockup přihlašovacího formuláře.

Narozdíl od uživatelského zdroje, který používá speciální metodu "resources" ke zjištění plné sady RESTful cest automaticky, zdroj sezení bude používat pouze jmenné cesty, přičemž bude obsluhovat požadavky "GET" a "POST" skrze cestu "login" a požadavek "DELETE" skrze "logout". Výsledná úprava "config/routes.rb" tedy vypadá následovně (došlo i ke smazání nepotřebných cest vygenerovaných skrze "rails generate controller"):

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
end
```

Bude třeba upravit novou přihlašovací cestou i vygenerovaný test (soubor "test/controllers/sessions_controller_test.rb"):

```
require 'test_helper'

class SessionsControllerTest < ActionDispatch::IntegrationTest

  test "should get new" do
    get login_path
    assert_response :success
  end
end
```

Cesty opět odpovídají URL a akcím podobně, jako u uživatelů:

| **HTTP požadavek** | **URL** | **Jmenná cesta** | **Akce**  | **Význam**                           |
| ------------------ | ------- | ---------------- | --------- | ------------------------------------ |
| `GET`              | /login  | `login_path`     | `new`     | stránka pro nové sezení (přihlášení) |
| `POST`             | /login  | `login_path`     | `create`  | vytvoření nového sezení (přihlášení) |
| `DELETE`           | /logout | `logout_path`    | `destroy` | smazání sezení (odhlášení)           |

Jelikož jsme si přidali několik vlastních jmenných cest, bude dobré se podívat na kompletní seznam cest pro naší aplikaci, čehož docílíme příkazem "rails routes":

```
$ rails routes
   Prefix Verb   URI Pattern               Controller#Action
     root GET    /                         static_pages#home
     help GET    /help(.:format)           static_pages#help
    about GET    /about(.:format)          static_pages#about
  contact GET    /contact(.:format)        static_pages#contact
   signup GET    /signup(.:format)         users#new
    login GET    /login(.:format)          sessions#new
          POST   /login(.:format)          sessions#create
   logout DELETE /logout(.:format)         sessions#destroy
    users GET    /users(.:format)          users#index
          POST   /users(.:format)          users#create
 new_user GET    /users/new(.:format)      users#new
edit_user GET    /users/:id/edit(.:format) users#edit
     user GET    /users/:id(.:format)      users#show
          PATCH  /users/:id(.:format)      users#update
          PUT    /users/:id(.:format)      users#update
          DELETE /users/:id(.:format)      users#destroy
```

Není třeba rozumět výsledkům příliš podrobně, ale poskytuje nám to dobrý náhled na akce, které naše aplikace podporuje.

### Přihlašovací formulář

Když máme zadefinovaný ovladač a cestu, můžeme naplnit pohled pro nová sezení, tedy přihlašovací formulář. Vzhledem je podobný registračnímu formuláři, jen namísto čtyř polí má jen dvě.

Opět si budeme chtít přihlašovací formulář překreslit v případě, že jsou vstupní informace neplatné (obr. 8.2). Dříve jsme použili partial pro zobrazení daných zpráv, ale také jsme zjistili, že je poskytuje automaticky Active Record. Což ovšem v případě tvorby sezení nepůjde, ostatně, nejde o objekt Active Record, a tedy si ho překreslíme za pomocí flash zprávy.

// obr. 8.2 Mockup selhání přihlášení.

Registrační formulář používá pomocníka "form_for", který bere jako parametr instanční proměnnou uživatele "@user":

```
<%= form_for(@user) do |f| %>
  .
  .
  .
<% end %>
```

Hlavní rozdíl mezi formulářem sezení a registračním je ten, že nemáme žádný model sezení, a tímpádem žádnou obdobu proměnné "@user". Budeme tedy muset "form_for" dodat poněkud více informací; především, kde

```
form_for(@user)
```

umožňuje Rails vyvodit, že "action" formuláře má být ve formě "POST" na URL /users, v případě sezení musíme indikovat jméno zdroje a související URL:

```
form_for(:session, url: login_path)
```

Se správným "form_for" po ruce už je jednoduché vytvořit přihlašovací formulář, který bude odpovídat předchozímu mockupu. Upravíme si tedy soubor "app/views/sessions/new.html.erb" a naplníme takto:

```
<% provide(:title, "Log in") %>
<h1>Přihlášení</h1>

<div class="row">
  <div class="col-md-6 col-md-offset-3">
    <%= form_for(:session, url: login_path) do |f| %>

      <%= f.label :email %>
      <%= f.email_field :email, class: 'form-control' %>

      <%= f.label :password %>
      <%= f.password_field :password, class: 'form-control' %>

      <%= f.submit "Log in", class: "btn btn-primary" %>
    <% end %>

    <p>Jste nový uživatel? <%= link_to "Zaregistrujte se!", signup_path %></p>
  </div>
</div>
```

Přidali jsme i odkaz na registraci pro případné nové uživatele. Přihlašovací formulář tedy bude vypadat jako na obrázku (obr. 8.3). Jelikož jsme si ještě nepřidali odkaz do navigace, bude třeba adresu URL /login vyplnit ručně, ale to si brzy opravíme.

// obr. 8.3 Přihlašovací formulář.

Vygenerované HTML formuláře vypadá takto:

```
<form accept-charset="UTF-8" action="/login" method="post">
  <input name="utf8" type="hidden" value="&#x2713;" />
  <input name="authenticity_token" type="hidden"
         value="NNb6+J/j46LcrgYUC60wQ2titMuJQ5lLqyAbnbAUkdo=" />
  <label for="session_email">Email</label>
  <input class="form-control" id="session_email"
         name="session[email]" type="text" />
  <label for="session_password">Password</label>
  <input id="session_password" name="session[password]"
         type="password" />
  <input class="btn btn-primary" name="commit" type="submit"
       value="Log in" />
</form>
```

Když si formulář srovnáme s registračním, jde dobře odhadnout, že výsledkem odeslání je haše "params" kde "params[:session]\[:email]" a "params[:session]\[:password]" odpovídají emailu a heslu.

### Nalezení a autentifikování uživatele

Podobně jako v případě vytváření uživatelů (registrace) bude první krok při vytváření sezení (přihlašování) ošetření neplatného vstupu. Začneme kontrolou toho, co se stane, když je formulář odeslán, a připravíme pomocné chybové zprávy pro případ selhání přihlášení (viz mockup na obr. 8.2). Pak položíme základ pro úspěšné přihlášení ověřením validity každého odeslání přihlášení na základě emailu a hesla.

Začneme minimalistickou akcí "create" pro ovladač sezení, spolu s novými prázdnými akcemi "new" a "destroy" (soubor "app/controllers/sessions_controller.rb"). Akce "create" pouze vykreslí pohled "new", ale pro začátek nám to stačí. Odeslání formuláře na /sessions/new pak vypadá jako na obrázku (obr. 8.4).

```
class SessionsController < ApplicationController

  def new
  end

  def create
    render 'new'
  end

  def destroy
  end
end
```

// obr. 8.4 Základní neúspěšný login s akcí "create".

Prohlídnou debugových informací zjistíme, že odeslání mělo za výsledek haši "params", která obsahuje email a heslo pod klíčem "session", což (ve zkrácené podobě) vypadá následovně:

```
---
session:
  email: 'user@example.com'
  password: 'foobar'
commit: Log in
action: create
controller: sessions
```

Podobně jako u registrace tyto parametry tvoří "vnořenou" haši. Konkétněji obsahuje "params" toto:

```
{ session: { password: "foobar", email: "user@example.com" } }
```

Což znamená, že

```
params[:session]
```

je haší samo o sobě:

```
{ password: "foobar", email: "user@example.com" }
```

Ve finále je pak

```
params[:session][:email]
```

odeslaná emailová adresa a

```
params[:session][:password]
```

je odeslané heslo.

Jinak řečeno, v akci "create" má haš "params" všechny informace potřebné k autentifikaci uživatelů skrze email a heslo. Ne náhodou máme k dispozici přesně ty metody, které potřebujeme: metoda "User.find_by", kterou poskytuje Active Record a metoda "authenticate", kterou poskytuje "has_secure_password". Jak víme, "authenticate" vrací "false" v případě neúspěšné autentifikace, a tedy náš postup pro přihlášení lze shrnout takto (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      # Prihlas uzivatele a presmeruj ho na profil.
    else
      # Vypis chybovou hlasku.
      render 'new'
    end
  end

  def destroy
  end
end
```

První řádek po "create" vytáhne uživatele z databáze pomocí zadané emailové adresy. Další řádek může být poněkud matoucí, ale v rámci Rails jde o poměrně běžnou věc:

```
user && user.authenticate(params[:session][:password])
```

Za pomocí "&&" (logické "a") zjišťujeme, zda je výsledný uživatel platný. V booleanském kontextu je každý objekt, který není "nil" nebo "false" považován za "true" (možné kombinace tohoto případu jsou v tabulce), a tedy celá "if" větev se úspěšně provede pouze tehdy, kdy je dodán uživatel spolu se správným heslem, kde oboje existuje v databázi.

| **Uživatel** | **Heslo**     | **a && b**                  |
| ------------ | ------------- | --------------------------- |
| neexistující | cokoliv       | (nil && [cokoliv]) == false |
| platný       | špatné heslo  | (true && false) == false    |
| platný       | správné heslo | (true && true) == true      |

### Vykreslení s flash zprávou

Minule jsme zobrazovali registrační chyby za pomocí uživatelských chybových hlášení. Tyto chyby jsou spojeny s konkrétním objektem Active Record, ale jelikož sezení není modelem Active Record, tento postup fungovat nebude. Namísto toho vložíme flash zprávu ke zobrazení při neúspěšném přihlášení. První, poněkud nesprávná verze, vypadá takto (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      # Prihlas uzivatele a presmeruj ho na profil.
    else
      flash[:danger] = 'Invalid email/password combination' # Ne uplne idealni verze!
      render 'new'
    end
  end

  def destroy
  end
end
```

Díky zobrazení flash zpráv v layoutu se zpráva automaticky zobrazí a díky Bootstrapovému CSS je dokonce i pěkně nastylovaná (obr. 8.5).

// obr. 8.5 Flash zpráva pro neúspěšné přihlášení.

Bohužel, jak je napsáno v komentu, takový kód není tak úplně správně. Stránka ale vypadá v pořádku, tak v čem je problém? Stkví totiž v tom, že obsah flash zprávy vydrží po dobu jednoho požadavku, ale, narozdíl od přesměrování, které jsme použili dříve, překreslení šablony skrze "render" se jako požadavek nepočítá. Výsledkem je, že zpráva přetrvá o jeden požadavek déle, než chceme. Kdybychom například odeslali neplatné přihlášení a pak se vrátili na domovskou stránku, flash zpráva se zobrazí znova (obr. 8.6). Musíme si tedy tento kaz opravit.

// obr. 8.6 Příklad vytrvalé flash zprávy.

### Flash test

Nesprávné chování flashe je naštěstí jen malý bug. Podle pouček, které jsme si stanovili dříve, je tohle idealní situace pro napsání testu k odchytu chyby. Napíšeme si tedy krátký integrační test pro odeslání přihlašovacího formuláře, než budeme pokračovat dále. Krom zdokumentování chyby a zabránění jejímu opakování nám to dá i dobrý základ pro budoucí integrační testy pro přihlášení a odhlášení.

Začneme generováním integračního testu pro chování naší aplikace při přihlášení:

```
$ rails generate integration_test users_login
      invoke  test_unit
      create    test/integration/users_login_test.rb
```

Dále budeme potřebovat test pro zachycení posloupnosti zobrazené na předešlých obrázcích (obr. 8.5 a 8.6). Základní kroky budou vypadat takto:

1. Navštiv adresu pro přihlášení.
2. Ověř, že se formulář nového sezení vykresluje správně.
3. Odešli na adresu sezení požadavek s neplatnou haší "params".
4. Ověř, že se nové sezení překreslí znova a objeví se flash zpráva.
5. Navštiv jinou stránku (např. domovskou).
6. Ověř, že se flash zpráva na nové stránce neobjeví.

Test, který tyto kroky implementuje pak bude vypadat následovně (soubor "test/integration/users_login_test.rb"):

```
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  test "login with invalid information" do
    get login_path
    assert_template 'sessions/new'
    post login_path, params: { session: { email: "", password: "" } }
    assert_template 'sessions/new'
    assert_not flash.empty?
    get root_path
    assert flash.empty?
  end
end
```

Když si ho zkusíme spustit, bude červený:

```
$ rails test test/integration/users_login_test.rb
```

(Tímto jsme se mimojiné naučili, jak spustit pouze jeden test za pomocí "rails test" a plnou cestou k souboru.)

Aby test prošel na zelenou, musíme nahradit "flash" variantou "flash.now", která je specificky určená pro zobrazování flash zpráv na vykreslených stránkách. Narozdíl od obsahu "flash", obsah "flash.now" zmizí okamžitě poté, co je odeslán další požadavek, což je přesně chování, které jsme testovali. S touto úpravou bude aplikační kód (soubor "app/controllers/sessions_controller.rb") vypadat takto:

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      # Prihlas uzivatele a presmeruj ho na profil.
    else
      flash.now[:danger] = 'Neplatná kombinace emailu/hesla'
      render 'new'
    end
  end

  def destroy
  end
end
```

Můžeme si pak ověřit, že jak integrační test, tak celá sada projde na zelenou:

```
$ rails test test/integration/users_login_test.rb
$ rails test
```



## Přihlášení

Když už náš přihlašovací formulář zvládá neplatné formuláře, bude dalším krokem umožnit skutečné přihlášení skrze ty platné. V této části si uživatele přihlásíme za pomocí dočasné cookie v rámci sezení, která automaticky vyprší po zavření prohlížeče. V další kapitole pak přidáme sezení, které přetrvají i to.

Implementace sezení bude zahrnovat definování velkého množství souvisejících funkcí, které bude používat vícero ovladačů a pohledů. Ruby poskytuje moduly právě pro takové úhledné balení souvisejících funkcí na jednom místě. Dokonce se nám vygeneroval pomocný modul sezení v rámci vytvoření ovladače sezení. Takoví pomocnící jsou automaticky zahrnuti v pohledech Railsu; zahrnutím modulu do základní třídy všech ovladačů (aplikační ovladač, Application controller) se nám zpřístupní i v našich ovladačích (soubor "app/controllers/application_controller.rb"):

```
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception
  include SessionsHelper
end
```

S touto úpravou nastavení můžeme konečne připravit kód pro regulérní přihlašování.

### Metoda "log_in"

Přihlášení uživatele je díky metodě "session", kterou Rails definuje, jednoduché. Můžeme se k ní chovat jako k haši a přiřazovat ji tedy obsah následovně:

```
session[:user_id] = user.id
```

Takový kód vloží do uživatelského prohlížeče dočasnou cookie, která obsahuje zašifrovanou verzi uživatelského id, což nám umožňuje id zpět získat na následujících stránkách skrze "session[:user_id]". Narozdíl od vytrvalé cookie vytvořené skrze metodu "cookie" (ke které se dostaneme později), dočasná cookie vytvořená metodou "session" vyprší okamžitě při zavření prohlížeče (ostatně, proto název "sezení").

Jelikož budeme chtít použít stejnou techniku přihlašování na různých místech, zadefinujeme si metodu "log_in" v pomocníku sezení (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper

  # Prihlasi uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end
end
```

Jelikož jsou dočasné cookies vytvořené pomocí metody "session" automaticky zašifrované, je předchozí kód bezpečný, a pro útočníka neexistuje možnost, jak použít informace ze sezení k přihlášení se pod daným uživatelem. Toto ovšem platí pouze pro dočasné sezení vytvořené skrze metodu "session", nikoliv pro dlouhodobé sezení vytvořené pomocí metody "cookies". Permanentní cookies jsou zranitelné vůči útoku "unesení sezení" (session hijacking), takže v příští kapitole budeme muset být obzvlášť opatrní ohledně informací, ktere do prohlížeče umísťujeme.

Po zadefinování metody "log_in" můžeme dokončit akci sezení "create" přihlášením uživatele a přesměrováním na jeho profilovou stránku. Výsledek vypadá následovně (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      redirect_to user
    else
      flash.now[:danger] = 'Neplatná kombinace emailu/hesla'
      render 'new'
    end
  end

  def destroy
  end
end
```

Všimněme si kompaktního přesměrování

```
redirect_to user
```

které jsme viděli už dříve. Rails ho automaticky převede na cestu k uživatelskému profilu:

```
user_url(user)
```

Po zadefinování "create" akce už by měl fungovat formulář s přihlášením. Jeho dopad na zobrazení čehokoliv sice zatím neuvidíme, takže (krom prozkoumání obsahu sezení prohlížeče) není způsob, jak zjistit, že jsme ve skutečnosti přihlášení. Začneme se tedy věnovat viditelnějším změnám, včetně změn odkazů a URL k uživatelskému profilu.

### Současný uživatel

Jakmile máme uživatelovo id bezpečně umístěno v dočasném sezení, můžeme ho získávat zpět na dalších stránkách naší aplikace, což uděláme pomocí definování metody "current_user" k získání uživatele z databáze, který odpovídá id ze sezení. Účelem "current_user" je umožnit konstrukce jako

```
<%= current_user.name %>
```

a

```
redirect_to current_user
```

Pro zjištění současného uživatele lze použít metodu "find", jako na profilové stránce:

```
User.find(session[:user_id])
```

Nicméně, "find" ohlásí výjimku v momentě, kdy dané uživatelské id neexistuje. Takové chování je vhodné pro uživatelskou stránku, jelikož k tomu dojde pouze tehdy, kdy je id neplatné, ale v současné chvíli bude "session[:user_id]" často "nil" (v případě nepřihlášených uživatelů). Abychom tento problém ošetřili, použijeme stejnou metodu "find_by" jako jsme použili k hledání podle emailové adresy v metodě "create", jen s "id" namísto "email":

```
User.find_by(id: session[:user_id])
```

Místo výjimky metoda vráti "nil" (když uživatel neexistuje) v případě, kdy je id neplatné.

Můžeme si tedy metodu "current_user" zadefinovat takto:

```
def current_user
  if session[:user_id]
    User.find_by(id: session[:user_id])
  end
end
```

Což by sice fungovalo dobře, ale dotazovalo databázi zbytečně často (obzvlášť, když budeme volání metody mít na stránce víckrát). Budeme tedy následovat běžnou zvyklost Ruby tím, že uložíme výsledek "User.find_by" do instanční proměnné, která se databáze dotáže pouze jednou, a při dalších voláních pouze vrací vytažený výsledek:

```
if @current_user.nil?
  @current_user = User.find_by(id: session[:user_id])
else
  @current_user
end
```

Pomocí operátoru || můžeme tento kód zapsat i následovně:

```
@current_user = @current_user || User.find_by(id: session[:user_id])
```

Jelikož uživatelský objekt je "true" v booleanském kontextu, volání "find_by" se provede pouze tehdy, kdy "@current_user" ještě nebyl zadefinován.

I když by daný kód fungoval, nejde o idiomaticky správné Ruby; správný způsob zápisu přiřazení "@current_user" by vypadalo takto:

```
@current_user ||= User.find_by(id: session[:user_id])
```

Což používá potenciálně matoucí, ale často užívaný operátor ||= ("nebo se rovná", angl. "or equals", v praxi přiřadí proměnné hodnotu jen tehdy, kdy je "false", tedy v tomto případě "nil").

Když aplikujeme poznatky výše na našeho pomocníka (soubor "app/helpers/sessions_helper.rb"), bude metoda "current_user" vypadat následovně (drobnou repetici si odstraníme později):

```
module SessionsHelper

  # Prihlasi uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end

  # Vrati v soucasnosti prihlaseneho uzivatele (pokud je).
  def current_user
    if session[:user_id]
      @current_user ||= User.find_by(id: session[:user_id])
    end
  end
end
```

S funkční metodou "current_user" můžeme konečně v naší aplikaci udělat změny, které budou vázány na stav přihlášení.

### Změna odkazů v layoutu

První praktické využití přihlášení spočívá ve změnách odkazů v layoutu na základě stavu přihlášení. Jak je vidět na mockupu (obr. 8.7), přidáme odkazy pro odhlášení, uživatelská nastavení, výpis všech uživatelů a pro profilovou stránku současného uživatele. Na mockupu je použito tzv. "dropdown" (vysunovací) menu pro účet, které se záhy naučíme vytvářet.

// obr. 8.7 Mockup uživatelského profilu po úspěšném přihlášení.

V téhle chvíli by v běžné praxi bylo rozumné zvážit napsání integračního testu pro zachycení chování, které jsme si právě popsali. Jelikož pro to ale bude potřeba použít pár nových poznatků, budeme se tomu věnovat až o něco později.

Způsob, jakým změnit odkazy v layoutu stránky spočívá v podmínečné if-else konstrukci uvnitř vnořeného Ruby. Jednoduše zobrazíme určité odkazy pro přihlášené, a jiné pro nepřihlášené:

```
<% if logged_in? %>
  # Odkazy pro prihlasene
<% else %>
  # Odkazy pro neprihlasene
<% end %>
```

Takový kód ale vyžaduje existenci booleanské metody "logged_in?", kterou si nyní zadefinujeme.

Uživatel je přihlášen tehdy, kdy existuje v sezení, tedy kdy "current_user" není "nil". Pro ověření použijeme operátor "not", označený vykřičníkem (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper

  # Prihlasi uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end

  # Vrati v soucasnosti prihlaseneho uzivatele (pokud je).
  def current_user
    if session[:user_id]
      @current_user ||= User.find_by(id: session[:user_id])
    end
  end

  # Vrati true pokud je uzivatel prihlasen, jinak false.
  def logged_in?
    !current_user.nil?
  end
end
```

S touto novinkou si už můžeme přidat odkazy podmíněné přihlášeným uživatelem. Budou dohomady čtyři, z čehož dvě zatím necháme jako pahýl a dokončíme později:

```
<%= link_to "Users",    '#' %>
<%= link_to "Settings", '#' %>
```

Odhlašovací odkaz používá cestu, kterou jsme si definovali na začátku:

```
<%= link_to "Log out", logout_path, method: :delete %>
```

Všimněme si, že odhlašovací odkaz posíla jako parametr haši, která indikuje, že má odeslat HTTP  požadavek "DELETE". Přidáme také odkaz na profil:

```
<%= link_to "Profile", current_user %>
```

Což by šlo zapsat i jako

```
<%= link_to "Profile", user_path(current_user) %>
```

ale jako obvykle nám Rails umožňuje přimý odkaz na uživatele skrze automatické převedení "current_user" v daném kontextu na "user_path(current_user)". A nakonec, když uživatel přihlášený není, použijeme adresu pro přihlášení k vytvoření odkazu na přihlašovací formulář:

```
<%= link_to "Log in", login_path %>
```

Pohromadě pak vypada upravený partial hlavičky (soubor "app/views/layouts/_header.html.erb") takto:

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
              <li><%= link_to "Settings", '#' %></li>
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

Využili jsme schopnosti Bootstrapu přidat vysunovací (dropdown) menu, a to pomocí tříd jako "dropdown" a "dropdown-menu". Abychom ho ale reálně aktivovali, musíme zahrnout knihovnu Bootstrap JavaScriptu a jQuery do souboru "app/assets/javascripts/application.js":

```
//= require jquery
//= require bootstrap
//= require rails-ujs
//= require turbolinks
//= require_tree .
```

V této chvíli si stači funkcionalitu prozkoušet skrze přihlášení pod skutečným uživatelem a ověřit, zda jsou menu i odkazy na správném místě a fungují (viz obr. 8.8). Také lze vyzkoušet zavřením prohlížeče, že skončením sezení aplikace zapomene stav přihlášení a bude vyžadovat nové.

// obr. 8.8 Přihlášený uživatel s novými odkazy a dropdown menu.

### Testování změn layoutu

I když jsme ověřili ručně, že aplikace po přihlášení funguje, jak má, je dobré si připravit i integrační test. Postavíme ho na testu předchozím, a to v následujících krocích:

1. Navštiv adresu pro přihlášení.
2. Odešli platné informace pro sezení.
3. Ověř, že přihlašovací odkaz zmizel.
4. Ověř, že se odhlašovací odkaz zobrazil.
5. Ověř, že se zobrazil i odkaz na profil.

Abychom dané změny viděli, náš test se musí přihlásit jako dříve registrovaný uživatel, což znamená, že musí existovat v databázi. Rails na toto používá tzv. fixtury, což je způsob organizování dat k nahrání do testovací databáze. Dříve jsme zjistili, že musíme základní fixtury smazat, aby prošel náš test unikátnosti emailů. Teď už si ale můžeme fixtury naplnit dle vlastního uvážení.

V současné chvíli potřebujeme jen jednoho uživatele, jehož informace by měly sestávat z platného jména a emailové adresy. Jelikož potřebujeme, ať se uživatel přihlásí, musíme zahrnout i platné heslo pro srovnání s heslem odeslaným akci "create" ovladače sezení. To bude obnášet vytvoření atributu "password_digest" pro uživatelskou fixturu, čehož docílíme definováním naší vlastní "digest" metody.

Zpracování (digest) hesla je docíleno skrze bcrypt (potažmo "has_secure_password"), takže musíme vytvořit heslo ve fixtuře stejným způsobem. Prozkoumáním [zdrojového kódu bezpečného hesla](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/secure_password.rb) zjistíme, že jde o metodu

```
BCrypt::Password.create(string, cost: cost)
```

kde "string" je řetězec určený k zahašování a "cost" je parametr náročnosti pro spočítání haše. Použití vysoké náročnosti výrazně snižuje šanci zjištění původního hesla, což je důležitý bezpečnostní prvek v produkčním prostředí, ale v testech chceme, ať je metoda "digest" co nejrychlejší. I pro to má zdrojový kód řádek:

```
cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                              BCrypt::Engine.cost
```

Tento poněkud tajemný kód, který není třeba pochopit do detailu, upravuje přesně chování, o kterém byla řeč: v testech používá nejméně náročný výpočet, kdežto v produkčním prostředí pravý opak.

Existuje několik míst, kam bychom mohli umístit výslednou metodu "digest", ale jelikož ji v budoucnu budeme chtít použít i v uživatelském modelu, umístíme ji do souboru "app/models/user.rb". Jelikož nemusíme mít nezbytně přístup k uživatelskému objektu ve chvíli, kdy se bude zpracování počítat, umístíme metodu "digest" do uživatelské třídy (User class) samotné, což z ní udělá třídní metodu.

```
class User < ApplicationRecord
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # Returns the hash digest of the given string.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end
end
```

S připravenou "digest" metodou nyní můžeme vytvořit uživatelskou fixturu pro platného uživatele (soubor "test/fixtures/users.yml"):

```
michael:
  name: Michael Example
  email: michael@example.com
  password_digest: <%= User.digest('password') %>
```

Fixtura podporuje i vnořené Ruby, díky čemuž můžeme použít

```
<%= User.digest('password') %>
```

pro vytvoření platného zpracovaného hesla pro testovacího uživatele.

Ačkoliv jsme si zadefinovali atribut "password_digest", který vyžaduje "has_secure_password", je občas praktické se odkazovat i na obyčejné (virtuální) heslo. To bohužel ve fixturách udělat nelze, a přidání atributu "password" do předchozího výpisu způsobí, že si Rails postežuje, že takový sloupec v databázi neexistuje (což je pravda). Obejdeme to tak, že všichni uživatelé v rámci fixtur budou používat stejné heslo ("password").

Když máme vytvořenu fixturu s platným uživatelem, můžeme k ní přistoupit v rámcí testu následovně:

```
user = users(:michael)
```

V tomto kódu "users" odpovídá názvu souboru fixtury "users.yml", kde symbol ":michaels" odkazuje na uživatele, kterého jsme si zadefinovali.

S vytvořeným pseudo-uživatelem můžeme připravit test, podle kroků, které jsme si vytyčili (soubor "test/integration/users_login_test.rb").

```
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "login with valid information" do
    get login_path
    post login_path, params: { session: { email:    @user.email,
                                          password: 'password' } }
    assert_redirected_to @user
    follow_redirect!
    assert_template 'users/show'
    assert_select "a[href=?]", login_path, count: 0
    assert_select "a[href=?]", logout_path
    assert_select "a[href=?]", user_path(@user)
  end
end
```

Použili jsme

```
assert_redirected_to @user
```

pro ověření, že přesměrování proběhlo na správný cíl a

```
follow_redirect!
```

pro reálné navštívení dané stránky. Ověřujeme také, že přihlašovací odkaz zmizel tak, že se ujišťujeme, že na stránce není žádný odkaz na přihlášení:

```
assert_select "a[href=?]", login_path, count: 0
```

Zahrnutím možnosti "count: 0" řekneme "assert_select", že očekáváme nula odkazů, které odpovídají danému vzorci.

Jelikož aplikační kód už dříve fungoval, test by měl být zelený:

```
$ rails test test/integration/users_login_test.rb
```

### Přihlášení po registraci

I když náš autentifikační systém funguje, nově registrovaní uživatelé mohou být zmatení z toho, že nejsou po registraci automaticky přihlášeni. Jelikož by bylo divné nutit uživatele k přihlášení hned po registraci, provedeme to automaticky v rámci registračního procesu. V podstatě nám stačí přidat volání na "log_in" v akci "create" uživatelského ovladače (soubor "app/controllers/users_controller.rb"):

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

  private

    def user_params
      params.require(:user).permit(:name, :email, :password,
                                   :password_confirmation)
    end
end
```

Pro ověření správného chování pouze přidáme řádek do staršího testu, abychom ověřili, že je uživatel přihlášen. V dané situaci je praktické si zadefinovat pomocníka "is_logged_in?" jako odraz pomocníka "logged_in?", který vrací true pokud je uživatelské id v (testovacím) sezení, a jinak vrací false. (Jelikož v testech pomocníci nejsou k dispozici, nemůžeme použít "current_user", ale metoda "session" je k dispozici, takže použijeme ji.) Použijeme tedy "is_logged_in?" namísto "logged_in?" aby metody v testu a sezení měly jiné názvy, což zabraňuje jejich případně záměně. Upravíme si tedy soubor "test/test_helper.rb":

```
ENV['RAILS_ENV'] ||= 'test'
.
.
.
class ActiveSupport::TestCase
  fixtures :all

  # Vraci true pokud je testovaci uzivatel prihlasen.
  def is_logged_in?
    !session[:user_id].nil?
  end
end
```

S touto úpravou můžeme předpokládat, že je uživatel po registraci přihlášen a ověřit to i v rámci testu (soubor "test/integration/users_signup_test.rb"):

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
    assert is_logged_in?
  end
end
```

Testy by tedy měly proběhnout zeleně:

```
$ rails test
```



## Odhlašování

Jak jsme probírali dříve, náš autentifikační model spočívá v udržení uživatelů v přihlášeném stavu, dokud se explicitně neodhlásí. V této části pro to přidáme nezbytnou odhlašovací funkcionalitu. Jelikož je samotný odkaz pro odhlášení už zadefinován, stačí nám jen napsat platnou akci ovladače pro ničení uživatelských sezení.

V rámci RESTful konvencí tedy použijeme akci "destroy" pro mazání sezení, tedy odhlášení. Narozdíl od funkcionality přihlášení se budeme odhlašovat pouze na jednom místě, a tedy příslušný kód umístíme přimo do akce "destroy". Jak uvidíme v budoucnu, takový přístup nám bude umožnovat i snažší testování.

Odhlášení zahrnuje odstranění důsledků metody "log_in", tedy smazání uživatelského id ze sezení. Použijeme tedy metodu "delete" následovně:

```
session.delete(:user_id)
```

Nastavíme také současného uživatele na "nil", i když na tom v současné chvíli nesejde, díky okamžitému přesměrování na kořenovou URL. Stejně jako u "log_in" a obdobných metod, i "log_out" umístíme do modulu pomocníků sezení (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper

  # PRihlasi daneho uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end
  .
  .
  .
  # Odhlasi soucasneho uzivatele.
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
end
```

Metodu "log_out" pak umístíme do ovladače sezení, potažmo do jeho akce "destroy" (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      redirect_to user
    else
      flash.now[:danger] = 'Invalid email/password combination'
      render 'new'
    end
  end

  def destroy
    log_out
    redirect_to root_url
  end
end
```

Abychom odhlašování i náležite otestovali, přidáme pár kroků do přihlašovacího testu uživatelů. Po přihlášení použijeme "delete" pro odeslání DELETE požadavku na odhlašovací adresu a ověříme, že odhlašovací a profilové odkazy zmizely (soubor "test/integration/users_login_test.rb"):

```
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest
  .
  .
  .
  test "login with valid information followed by logout" do
    get login_path
    post login_path, params: { session: { email:    @user.email,
                                          password: 'password' } }
    assert is_logged_in?
    assert_redirected_to @user
    follow_redirect!
    assert_template 'users/show'
    assert_select "a[href=?]", login_path, count: 0
    assert_select "a[href=?]", logout_path
    assert_select "a[href=?]", user_path(@user)
    delete logout_path
    assert_not is_logged_in?
    assert_redirected_to root_url
    follow_redirect!
    assert_select "a[href=?]", login_path
    assert_select "a[href=?]", logout_path,      count: 0
    assert_select "a[href=?]", user_path(@user), count: 0
  end
end
```

(Přihodili jsme si i bonusový řádek "assert is_logged_in?" jen tak pro úplnost, když už jej máme k dispozici.)

S akcí sezení "destroy" náležite zaimplementovanou a protestovanou máme hotový základní triumvirát registrace/přihlášení/odhlášení a testovací sada by měla být tedy zelená:

```
$ rails test
```



## Shrnutí

Díky této kapitolé má naše aplikace plně funkční přihlašovací a autentifikační systém. V té další se budeme věnovat přidání možnosti zapamatování uživatelů na delší dobu, než je jedno sezení prohlížeče.

Než postoupíme dále, spojíme změny s hlavní větví:

```
$ rails test
$ git add -A
$ git commit -m "Zaimplementovano zakladni prihlasovani"
$ git checkout master
$ git merge basic-login
```

### Co jsme se naučili

- Rails umí udržet "stavy" i po přechodech mezi stránkami díky dočasným cookies a metodě "session".
- Přihlašovací formulář je navrhnut tak, aby vytvořil nové sezení pro přihlášení uživatele.
- Metoda "flash.now" se používá pro zobrazování zpráv na vykreslených stránkách.
- Metodika vývoje "testování předem" je praktická při odstraňování bugů tím, že si v testu bug zreprodukujeme.
- Použitím metody "session" můžeme bezpečně uložit uživatelské id v browseru pro vytvoření dočasného sezení.
- Můžeme změnit prvky jako jsou odkazy v layoutu na základě stavu přihlášení.
- Integrační testy mohou ověřit správné cesty, aktualizace databáze a správnost změn v layoutu.