# Pokročilé přihlášení

Základní přihlašovací systém z předchozí kapitoly je sice plně funkční, ale běžným standardem v dnešní době je funkce "zapamatování si" uživatelů, když stránku znovu navštíví, nehledě na to, jestli mezitím svůj prohlížeč zavřeli. Za pomocí trvalých cookies si toto chování zaimplementujeme. Začneme automatickým zapamatováváním uživatelů poté, co se přihlásí, a posléze přidáme volitelnou možnost skrze zaškrtnutí "pamatuj si mě", což je asi nejběžněji používaný model.

Technicky vzato je tato kapitola volitelná, ale naučit se takovou funkcionalitu zaimplementovat je praktické samo o sobě, nemluvě o tom, že se tím i položí základy pro pochopení kapitol dalších, jako je aktivace účtu a resetování hesla. Krom toho, všichni jsme už viděli tuhle funkcionalitu mockrát a nyní máme šanci zjistit, jak vlastně funguje.



## Zapamatuj si mě

V této části si přidáme možnost zapamatování stavu uživatelova přihlášení i poté, co zavřeli a znovu otevřeli svůj prohlížeč. Toto "pamatuj si mě" chování bude plně automatické, a uživatelé zůstanou přihlášeni až do té doby, než se explicitně sami odhlásí. Posléze zjistíme, že do takového systému už není problém přidat i volitelný checkbox (běžný typ políčka ve formuláři, zaškrtnutí vypadá jako "fajfka") pro zapnutí/vypnutí této funkcionality.

Jako obvykle si vytvoříme vedlejší větev:

```
$ git checkout -b advanced-login
```

### Token k zapamatování a zpracování

V předchozí kapitole jsme použili metodu "session" pro uložení uživatelského id, ale tato informace zmizí, jakmile uživatel zavře svůj prohlížeč. V této části uděláme první krok k vytrvalým sezením pomocí generování tokenu (z angl. "poukaz, žeton") k zapamatování ("remember token"), který bude vhodný pro vytváření trvalých cookies pomocí metody "cookies", spolu s bezpečným zpracováním k zapamatování ("remember digest") pro autentifikaci těchto tokenů.

Jak jsme si řekli dříve, informace uložené skrze "session" jsou automaticky bezpečné, ale pro "cookies" to neplatí. Trvalé cookies jsou obzvláště zranitelné metodou [session hijacking](http://en.wikipedia.org/wiki/Session_hijacking), ve které útočník použije ukradený zapamatovaný token, díky čemuž se pak přihlásí jako daný uživatel. Existuje vícero způsobů, jak cookies ukrást, ale před hlavními nás chrání metodika ukládání zpracovaných (digested) tokenů, použití [SSL](https://en.wikipedia.org/wiki/Transport_Layer_Security) a Rails samotný. Zbývá nám akorát minimalizovat problém, kdy útočník získá fyzický přístup k počítači, na kterém je uživatel přihlášený, čehož docílíme tak, že tokeny změníme pokaždé, kdy se uživatel přihlásí/odhlásí a samozřejmě veškeré informace, které umístíme do prohlížeče [zašifrujeme](https://en.wikipedia.org/wiki/Digital_signature).

Když už máme představu o postupu, náš postup vytváření trvalých sezení bude vypadat následovně:

1. Vytvoření náhodného řetězce čísel, který použijeme jako token.
2. Umístíme token do cookies prohlížeče s expiračním datem daleko v budoucnosti.
3. Uložíme hašové zpracování tokenu do databáze.
4. Umístíme zašifrovanou verzi uživatelského id do cookies prohlížeče.
5. Když aplikace dostane cookie, která obsahuje trvalé uživatelské id, najde uživatele v databázi (za pomocí daného id) a ověří, že token cookie odpovídá souvisejícímu zašifrovanému zpracování z databáze.

Poslední krok je podobný přihlašování, kde jsme vytáhli uživatele podle emailové adresy a pak ověřili (skrze metodu "authenticate"), že odeslané heslo odpovídá jeho zpracované podobě. Naše implementace tedy bude mít shodné prvky s "has_secure_password".

Začneme přidáním požadovaného atributu "remember_digest" do uživatelského modelu (viz obr. 9.1).

// obr 9.1 Uživatelský model s přidaným atributem "remember_digest".

Pro přidání tohoto datového modelu vytvoříme migraci:

```
$ rails generate migration add_remember_digest_to_users remember_digest:string
```

Podobně jako u předchozích migrací jsme použili jméno migrace, které končí na "\_to_users", aby Rails věděl, že je tato migrace určená k úpravě tabulky "users" v databázi. Jelikož jsme zahrnuli i atribut "remember_digest" a typ "string", Rails vygeneruje základní migraci v takovéto podobě (soubor "db/migrate/[timestamp]_add_remember_digest_to_users.rb"):

```
class AddRememberDigestToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :remember_digest, :string
  end
end
```

Jelikož nebudeme chtít uživatele vytahovat skrze digest, nepotřebujeme na sloupec "remember_digest" dávat index. Spustíme si tedy migraci tak, jak je:

```
$ rails db:migrate
```

Teď se musíme rozhodnout, co použijeme jako token k zapamatování. Většina možností je víceméně shodná, v podstatě stačí jakýkoliv náhodný dlouhý řetězec. Metoda "urlsafe_base64" modulu "SecureRandom" ze standardní knihovny Ruby je ideální volba: vrátí náhodný řetězec o délce 22 znaků složený z alfanumerických symbolů, mínusu a podtržítka (což dá dohromady 64 možností pro každý znak, proto "[base64](http://en.wikipedia.org/wiki/Base64)"). Obvyklý base64 řetězec vypadá takto:

```
$ rails console
>> SecureRandom.urlsafe_base64
=> "q5lt38hQDc_959PVoo6b7A"
```

Tak, jako není nutné, aby dva uživatelé měli rozdílné hesla, není ani potřeba, aby tokeny byly unikátní, ačkoliv je bezpečnější, když jsou. Nicméně v případě base64 je šance opravdu zanedbatelně mizivá. Navíc jsou tyto řetězce navrženy tak, aby byly bezpečné pro použití v URL (proto jméno "urlsafe_base64"), takže tento generátor budeme moci použít i pro aktivaci účtu a reset hesla v budoucnu.

Zapamatování uživatelů zahrnuje vytvoření tokenu a uložení digestu tokenu do databáze. Metodu "digest" už jsme si definovali pro použití ve fixturách minule, a výsledky můžeme použít pro vytvoření nové metody "new_token" pro vytvoření nového tokenu. Stejně jako u "digest" ani nová tokenová metoda nepotřebuje uživatelský objekt, takže z ní uděláme třídní metodu. Úprava tedy bude vypadat takto (soubor "app/models/user.rb"):

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

  # Vrati zahasovane zpracovani daneho retezce.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # Vrati nahodny token.
  def User.new_token
    SecureRandom.urlsafe_base64
  end
end
```

Náš implementační plán spočívá ve vytvoření metody "user.remember", která si spojí token s uživatelem a uloží související digest do databáze. Díky migraci už má uživatelský model atribut "remember_digest", ale zatím nemá "remember_token". Potřebujeme způsob, jak token zpřístupnit skrze "user.remember_token" (pro uložení do cookies) aniž bychom ho uložili do databáze. Podobný problém jsme vyřešili s bezpečností hesel dříve, kde jsme spárovali virtuální atribut "password" s bezpečným atributem "password_digest" v databázi. V onom případě byl virtuální atribut "password" vytvořen automaticky skrze "has_secure_password", ale pro "remember_token" si musíme kód napsat sami. Použijeme na to "attr_accessor" pro vytvoření atributu, ke kterému lze přistupovat:

```
class User < ApplicationRecord
  attr_accessor :remember_token
  .
  .
  .
  def remember
    self.remember_token = ...
    update_attribute(:remember_digest, ...)
  end
end
```

Všimněme si způsobu přiřazení v prvním řídku metody "remember". Díky způsobu, kterým Ruby obsluhuje přiřazení uvnitř objektů by bez "self" přiřazení vytvořilo místní proměnnou nazvanou "remember_token", což není to, co chceme. Použití "self" zajišťuje, že přiřazení nastaví uživatelský atribut "remember_token". Druhý řádek "remember" používá metodu "update_attribute" pro aktualizaci digestu k zapamatování (mimojiné tento postup obchází validace, což je v tomto případě nutné, jelikož nemáme přístup k uživatelskému heslu nebo potvrzení).

Po zralé úvaze tedy můžeme vytvořit platný token a související digest vytvořením nového tokenu použitím "User.new_token" a poté aktualizujeme digest výsledkem aplikace metody "User.digest". Výsledek bude vypadat takto (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # Vrati zahasovane zpracovani daneho retezce.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # Vrati nahodny token.
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # Zapamatuje si uzivatele v databazi pro pouziti v trvalych sezenich.
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end
end
```

### Přihlášení se zapamatováním

Jelikož máme hotovou funkční "user.remember" metodu, můžeme si nyní vytvořit trvalé sezení uložením uživatelského (zašifrovaného) id a tokenu ve formě permanentních cookies v prohlížeči. Použijeme metodu "cookies", kterou, podobně jako "session", můžeme vnímat jako haši. Cookie sestává ze dvou informačních prvků: "value" (hodnota) a volitelné "expires" (vyprší). Kupříkladu můžeme vytvořit trvalé sezení vytvořením cookie s hodnotou tokenu, která vyprší za 20 let:

```
cookies[:remember_token] = { value:   remember_token,
                             expires: 20.years.from_now.utc }
```

(Takto mimochodem vypadá jeden z praktických pomocníku Railsu pro práci s časovými údaji.) Postup "nastavení vypršení cookie za 20 let" je tak běžný, že na něj má Rails dokonce speciální metodu, takže stačí napsat pouze

```
cookies.permanent[:remember_token] = remember_token
```

Díky tomuto zápisu Rails automaticky nastaví expiraci na "20.years.from_now".

Pro uložení uživatelského id v cookies můžeme postupovat stejně, jako v případě metody "session", tedy takto:

```
cookies[:user_id] = user.id
```

Jelikož je id nastaveno jako obyčejný text, takovýto postup odhaluje formu aplikačních cookies a případnému útočníkovi ulehčuje kompromitování uživatelských účtů. Abychom tomu předešli, cookie si "podepíšeme" (signed cookie), což v praxi znamená, že se bezpečně zašifruje předtím, než je umístěna do prohlížeče:

```
cookies.signed[:user_id] = user.id
```

Jelikož chceme id spárovat s permanentním tokenem, nastavíme ho taktéž jako permanentní, čehož docílíme zřetězením metod "signed" a "permanent":

```
cookies.permanent.signed[:user_id] = user.id
```

Jakmile jsou cookies nastavené, můžeme je získat na ostatních stránkach uživatelským kódem, který vypadá následovně:

```
User.find_by(id: cookies.signed[:user_id])
```

V tomto zápise nám "cookies.signed[:user_id]" automaticky dešifruje cookie s uživatelským id. Stačí pak použít bcrypt pro ověření, že "cookies[:remember_token]" odpovídá "remember_digest".

Poslední kousek skládačky spočívá v ověření, že daný token odpovídá uživatelskému digestu (zpracování), přičemž v současném kontextu existuje několik ekvivalentních metod použití bcryptu pro ověření shody. Když se podíváme [do zdrojového kódu bezpečného hesla](https://github.com/rails/rails/blob/master/activemodel/lib/active_model/secure_password.rb), najdeme takovéto porovnání:

```
BCrypt::Password.new(password_digest) == unencrypted_password
```

V našem případě by kód vypadal takto:

```
BCrypt::Password.new(remember_digest) == remember_token
```

Když se nad tím zamyslíme, je tento kód hodně zvláštní: vypadá, že porovnává výsledek zpracování hesla bcryptem přímo s tokenem, což by naznačovalo dešifrování zpracování, aby bylo možno použít ==. Ale celý smysl použití bcryptu je ten, že je postup nevratný, takže to tak přeci nemůže být. A vskutku, když si lépe prohlédneme [zdrojový kód gemu bcrypt](https://github.com/codahale/bcrypt-ruby/blob/master/lib/bcrypt/password.rb) tak zjistíme, že operátor == je předefinován, a ve skutečnosti "pod kapotou" vypadá porovnání takto:

```
BCrypt::Password.new(remember_digest).is_password?(remember_token)
```

Namísto == je použita booleanská metoda "is_password?" pro provedení porovnání. Jeliož je její význam poněkud zřejmější, v aplikačním kódu použijeme tuto variantu.

Kontext napovídá, že bude nejvhodnější toto "digest-token porovnání" umístit do uživatelského modelu, který hraje podobnou roli jako metoda "authenticate" skrze "has_secure_password" pro autentifikaci uživatele. Když už máme jasněji v tom, co chceme udělat, upravíme si uživatelský model (soubor "app/models/user.rb") následovně:

```
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # Vrati zahasovane zpracovani daneho retezce.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # Vrati nahodny token.
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # Zapamatuje si uzivatele v databazi pro pouziti v trvalych sezenich.
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end

  # Vrati true pokud dany token odpovida digestu.
  def authenticated?(remember_token)
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end
end
```

Parametr "remember_token" v metodě "authenticated?" není ten samý, jako jsme použili dříve, jde jen o místní proměnnou v rámci metody. Použití atributu "remember_digest", který je obdobný jako "self.remember_digest" (jako u "name" a "email" použitých dříve) je automaticky vytvoření skrze Active Record podle odpovídajícího názvu sloupce.

Můžeme si tedy zaimplementovat zapamatování přihlášeného uživatele, čehož docílíme pomocí přidání "remember" pomocníka (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      remember user
      redirect_to user
    else
      flash.now[:danger] = 'Neplatná kombinace emailu/hesla'
      render 'new'
    end
  end

  def destroy
    log_out
    redirect_to root_url
  end
end
```

Stejně jako s "log_in", i touto úpravou necháme většinu práce dělat pomocníka sezení, kde si zadefinujeme "remember" metodu, která zavolá "user.remember", čímž vygeneruje token a uloží digest do databáze. Poté použije "cookies" pro vytvoření permaneních cookies pro uživatelské id a token, jak jsme si popsali výše. Upravíme si tedy pomocníka (soubor "app/helpers/sessions_helper.rb") takto:

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

  # Vrati momentalne prihlaseneho uzivatele (pokud je).
  def current_user
    if session[:user_id]
      @current_user ||= User.find_by(id: session[:user_id])
    end
  end

  # Vrati true pokud je uzivatel prihlasen, jinak false.
  def logged_in?
    !current_user.nil?
  end

  # Odhlasi soucasneho uzivatele.
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
end
```

Uživatel bude nyní "zapamatován" v tom smyslu, že prohlížeč dostane platný token, ale ještě nám to k ničemu není, jelikož metoda "current_user" definovaná dříve zná jen dočasné sezení:

```
@current_user ||= User.find_by(id: session[:user_id])
```

V případě trvalých sezení chceme získat uživatele z dočasného sezení, pokud "session[:user_id]" existuje, ale jinak se musíme podívat do "cookies[:user_id]" pro získání (a přihlášení) souvisejícího uživatele do trvalého sezení. Můžeme toho dosáhnout následovně:

```
if session[:user_id]
  @current_user ||= User.find_by(id: session[:user_id])
elsif cookies.signed[:user_id]
  user = User.find_by(id: cookies.signed[:user_id])
  if user && user.authenticated?(cookies[:remember_token])
    log_in user
    @current_user = user
  end
end
```

Tento kód sice bude fungovat, ale určitě se chceme zbavit té škaredé duplicity:

```
if (user_id = session[:user_id])
  @current_user ||= User.find_by(id: user_id)
elsif (user_id = cookies.signed[:user_id])
  user = User.find_by(id: user_id)
  if user && user.authenticated?(cookies[:remember_token])
    log_in user
    @current_user = user
  end
end
```

Použili jsme běžnou, i když možná lehce matoucí konstrukci

```
if (user_id = session[:user_id])
```

Navzdory zdání nejde o porovnání (které by použilo dvě rovnítka ==), nýbrž jde o přiřazení. Pokud bychom řádek měli přečíst nahlas, nezněl by jako "Pokud se uživatelské id rovná sezení uživatelského id...", ale spíše jako "Pokud existuje sezení uživatelského id existuje (přičemž nastavujeme uživatelské id sezení uživatelského id)...".

Definování pomocníka "current_user" pak vypadá takto (soubor "app/helpers/sessions_helper.rb"):

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

  # Vrati uzivatele, ktery odpovida cookie tokenu.
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
        log_in user
        @current_user = user
      end
    end
  end

  # Vrati true pokud je uzivatel prihlasen, jinak false.
  def logged_in?
    !current_user.nil?
  end

  # Odhlasi soucasneho uzivatele.
  def log_out
    session.delete(:user_id)
    @current_user = nil
  end
end
```

S touto úpravou už jsou nově přihlášení uživatelé správně zapamatováni, což si můžeme ověřit přihlášením, zavřením prohlížeče, otevřením prohlížeče a opětovným navštívením naší aplikace. Můžeme si i přímo prohlédnout cookies prohlížeče (obr. 9.2).

// obr 9.2 Zapamatovaný token jako cookie v místním prohlížeči.

Je tu jen jeden drobný problém: pokud uživatel v současné chvíli nevymaže cookies svého prohlížeče (nebo nepočká 20 let), nemá se jak odhlásit. Přesně na takové věci slouží naše sada testů, která to ostatně i správně odchytne a výsledek bude červený:

```
$ rails test
```

### Zapomínání uživatelů

Abychom uživatelům umožnili odhlášení, zadefinujeme metody pro zapomenutí uživatelů, podobně, jako už máme metody pro jejich zapamatování. Výsledná metoda "user.forget" jen "odčiní" "user.remember" tak, že aktualizuje digest na "nil" (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  attr_accessor :remember_token
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }

  # Vrati zahasovane zpracovani daneho retezce.
  def User.digest(string)
    cost = ActiveModel::SecurePassword.min_cost ? BCrypt::Engine::MIN_COST :
                                                  BCrypt::Engine.cost
    BCrypt::Password.create(string, cost: cost)
  end

  # Vrati nahodny token.
  def User.new_token
    SecureRandom.urlsafe_base64
  end

  # Zapamatuje si uzivatele v databazi pro pouziti v trvalych sezenich.
  def remember
    self.remember_token = User.new_token
    update_attribute(:remember_digest, User.digest(remember_token))
  end

  # Vrati true pokud dany token odpovida digestu (zpracovani).
  def authenticated?(remember_token)
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end

  # Zapomene uzivatele.
  def forget
    update_attribute(:remember_digest, nil)
  end
end
```

Obdobně můžeme "zapomenout" i trvalé sezení skrze pomocníka "forget", kterého zavoláme z pomocníka "log_out". Pomocník "forget" zavolá "user.forget" a poté vymaže "user_id" a "remember_token" z cookies (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper

  # Prihlasi daneho uzivatele.
  def log_in(user)
    session[:user_id] = user.id
  end
  .
  .
  .
  # Zapomene trvale sezeni.
  def forget(user)
    user.forget
    cookies.delete(:user_id)
    cookies.delete(:remember_token)
  end

  # Odhlasi soucasneho uzivatele.
  def log_out
    forget(current_user)
    session.delete(:user_id)
    @current_user = nil
  end
end
```

Díky těmto úpravám by testy měly procházet na zelenou:

```
$ rails test
```

### Dva drobné bugy

Zbývá se ještě věnovat dvěma drobným problémům. První je ten, že i když se odkaz "Odhlásit se" zobrazuje jen tehdy, kdy je uživatel přihlášen, je možné, že bude mít daný uživatel otevřeno více oken aplikace v prohlížeči najednou. Pokud se v jednom okně odhlásí (a tedy nastaví "current_user" na "nil"), kliknutí na "Odhlásit se" v druhém okné bude mít za důsledek chybu kvůli "forget(current_user)" v metodě "log_out". Vyřešit to lze tak, že uživatele odhlásíme pouze tehdy, když je přihlášen.

Druhá drobnost je ta, že uživatel může být přihlášen (a zapamatován) skrze vícero prohlížečů, jako Chrome a Firefox, což může představovat problém, pokud se uživatel odhlásí z prvního prohlížeče, ale z druhého ne, načež druhý zavře a znovu otevře. Například, předpokládejme, že se uživatel odhlásí ve Firefoxu, čímž nastaví digest na "nil" (skrze "user.forget"). Aplikace bude ve Firefoxu stále fungovat, jelikož metoda "log_out" smaže uživatelovo id a obě vyhodnocení podmínek bude "false":

```
# Vrati uzivatele, ktery odpovida cookie tokenu.
def current_user
  if (user_id = session[:user_id])
    @current_user ||= User.find_by(id: user_id)
  elsif (user_id = cookies.signed[:user_id])
    user = User.find_by(id: user_id)
    if user && user.authenticated?(cookies[:remember_token])
      log_in user
      @current_user = user
    end
  end
end
```

Výsledkem bude, že vyhodnocení vrátí "nil", jak ostatně chceme.

Na druhou stranu, pokud zavřeme Chrome, nastavíme "session[:user_id]" na "nil" (jelikož všechny "session" proměnné vyprší automaticky při zavření prohlížeče), ale cookie "user_id" bude stále přítomna. Daný uživatel bude tedy vytáhnutý z databáze v momentě, kdy se Chrome znovu spustí:

```
# Vrati uzivatele, ktery odpovida cookie tokenu.
def current_user
  if (user_id = session[:user_id])
    @current_user ||= User.find_by(id: user_id)
  elsif (user_id = cookies.signed[:user_id])
    user = User.find_by(id: user_id)
    if user && user.authenticated?(cookies[:remember_token])
      log_in user
      @current_user = user
    end
  end
end
```

Vnitřní "if" bude tedy vyhodnoceno:

```
user && user.authenticated?(cookies[:remember_token])
```

Jelikož "user" není "nil", bude vyhodnoceno druhé vyjádření, což vyvolá chybu. Děje se tak proto, že uživatelův digest byl smazán jako součást odhlášení ve Firefoxu, takže když přistoupíme k aplikaci v Chrome, zavoláme

```
BCrypt::Password.new(remember_digest).is_password?(remember_token)
```

s digestem o hodnotě "nil", což vyvolá výjimku v knihovně bcryptu. Pro opravu bude potřeba upravit "authenticated?" tak, aby v tomto případě vracel "false".

Přesně takové problémy jsou jako dělané pro metodiku testování dopředu, takže si připravíme testy na odchycení těchto dvou chyb předtím, než si je opravíme. V prvé řadě si upravíme integrační test (soubor "test/integration/users_login_test.rb"):

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
    # Simulate a user clicking logout in a second window.
    delete logout_path
    follow_redirect!
    assert_select "a[href=?]", login_path
    assert_select "a[href=?]", logout_path,      count: 0
    assert_select "a[href=?]", user_path(@user), count: 0
  end
end
```

Druhé volání na "delete logout_path" by mělo vyvolat chybu díky chybějícímu "current_user", a test tedy bude červený:

```
$ rails test
```

Aplikační kód bude jednoduše zahrnovat zavolání "log_out" pouze tehdy, když je "logged_in?" vyhodnocen jako "true" (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController
  .
  .
  .
  def destroy
    log_out if logged_in?
    redirect_to root_url
  end
end
```

Druhý případ, zahrnující scnénář se dvěma prohlížeči, je v integračním testu obtížnější nasimulovat, ale lze to jednoduše ověřit v testu uživatelského modelu. Vše, co potřebujeme, je začít s uživatelem, který nemá žádny digest (což odpovídá proměnné "@user" definované v "setup" metodě) a pak zavolat "authenticated?" (soubor "test/models/user_test.rb"):

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
    assert_not @user.authenticated?('')
  end
end
```

Jelikož "BCrypt::Password.new(nil)" vyvolá chybu, sada testů bude nyní červená:

```
$ rails test
```

Pro opravení chyby budeme potřebovat vrátit "false" v případě, kdy je digest "nil" (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  .
  .
  .
  # Vrati true pokud dany token odpovida digestu.
  def authenticated?(remember_token)
    return false if remember_digest.nil?
    BCrypt::Password.new(remember_digest).is_password?(remember_token)
  end

  # Zapomene uzivatele.
  def forget
    update_attribute(:remember_digest, nil)
  end
end
```

Použili jsme klíčové slovo "return" pro okamžitý návrat v případě, že je digest "nil", což je běžný postup, když chceme dát najevo, že zbytek metody může být odignorován. Ekvivalentní kód

```
if remember_digest.nil?
  false
else
  BCrypt::Password.new(remember_digest).is_password?(remember_token)
end
```

by také fungoval, ale explicitnější (a kratší) verze je přecejen výstižnější.

S poslední úpravou by měla být naše sada testů zelená a obě chyby opravené:

```
$ rails test
```



## Checkbox "zapamatuj si mě"

Naše aplikace má nyní kompletní autentifikační systém na profesionální úrovni. Jako poslední krok se budeme věnovat úpravě, díky které bude zapamatování volitelné pomocí checkboxu (políčka pro zaškrtnutí). Mockup na obrázku (obr. 9.3) nastíní, jak pak bude náš přihlašovací formulář vypadat.

// obr. 9.3 Mockup s checkboxem "zapamatuj si mě".

Začneme přidáním samotného checkboxu do přihlašovacího formuláře z minulé kapitoly. Jako u všech štítků, textových polí a odesílacích tlačítek může být checkbox vytvořen Railsovým pomocníkem. Aby stylování vypadalo, jak má, musíme checkbox umístit do štítku, tedy:

```
<%= f.label :remember_me, class: "checkbox inline" do %>
  <%= f.check_box :remember_me %>
  <span>Zapamatuj si mě na tomto počítači</span>
<% end %>
```

Vložení do přihlašovacího formuláře bude vypadat takto (soubor "app/views/sessions/new.html.erb"):

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

Zahrnuli jsme CSS třídy "checkbox" a "inline", které Bootstrap použivá pro umístění checkboxu a textu "Zapamatuj si mě na tomto počítači" na stejný řádek. Ještě ale bude nezbytných pár úprav přímo v našem CSS (soubor "app/assets/stylesheets/custom.scss"):

```
.
.
.
/* forms */
.
.
.
.checkbox {
  margin-top: -10px;
  margin-bottom: 10px;
  span {
    margin-left: 20px;
    font-weight: normal;
  }
}

#session_remember_me {
  width: auto;
  margin-left: 0;
}
```

// obr. 9.4 Přihlašovací formulář s přidaným checkboxem "zapamatuj si mě".

Po úpravě formuláře si můžeme připravit zapamatování uživatelů pouze tehdy, kdy o to budou stát (a jinak je zapomenout). I když je to neuvěřitelné, díky předchozí práci implementace zahrnuje pouze jeden řádek. A to díky tomu, že haše "params" s odeslaným formulářem nyní obsahuje hodnotu založenou na checkboxu. Konkrétně, hodnota

```
params[:session][:remember_me]
```

bude "1" tehdy, kdy je checkbox zaškrtnutý a "0" když nebude.

Ověřením této hodnoty si tedy můžeme uživatele zapamatovat nebo ho zapomenout, podle toho, jak mu to vyhovuje:

```
if params[:session][:remember_me] == '1'
  remember(user)
else
  forget(user)
end
```

Podobná "if - then" struktura jde zapsat skrze tzv. [ternární operátor](https://en.wikipedia.org/wiki/%3F:) a zabere jen jeden řádek:

```
params[:session][:remember_me] == '1' ? remember(user) : forget(user)
```

Pokud jím nahradíme "remember user" v ovladači sezení, resp. jeho "create" metodě, dostaneme úžasně úsporný kód (soubor "app/controllers/sessions_controller.rb"):

```
class SessionsController < ApplicationController

  def new
  end

  def create
    user = User.find_by(email: params[:session][:email].downcase)
    if user && user.authenticate(params[:session][:password])
      log_in user
      params[:session][:remember_me] == '1' ? remember(user) : forget(user)
      redirect_to user
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

S touto úpravou je nás přihlašovací systém kompletní, jak si ostatně můžeme ověřit zaškrtnutím/odškrtnutím checkboxu v prohlížeči.



## Testy zapamatování

I když je naše "zapamatuj si mě" funkcionalita funkční, je důležité napsat i několik testů pro ověření jejího chování. Mimojiné i proto, abychom odchytili implementační chyby, jak si povíme záhy. Ještě důležitější je, že jádro kódu pro uživatelské přetrvávání je momentálně zcela neotestovné. Pro vyřešení této problematiky sice použijeme pár triků, ale výsledkem bude mnohem silnější sada testů.

### Testování checkboxu "zapamatuj si mě"

Představme si, že bychom v rámci předchozí implementace chování checkboxu namísto správného

```
params[:session][:remember_me] == '1' ? remember(user) : forget(user)
```

použili

```
params[:session][:remember_me] ? remember(user) : forget(user)
```

V daném kontextu je "params\[:session]\[:remember_me] " buď "0" nebo "1", což je ale z booleanského pohledu vždy "true", a aplikace by se chovala jako by byl checkbox vždy zaškrtnutý. Přesně takové typy chyb nám pomůžou odchytit testy.

Protože zapamatování uživatelů v prvé řadě vyžaduje, aby byli přihlášení, nás prvni krok bude definování pomocníka pro přihlášení uživatelů v testech. Normálně uživatele přihlašujeme skrze "post" metodu a platnou haši "session", ale je nepraktické to takto dělat pokaždé. Abychom se vyhnuli zbytečnému opakování, napíšeme si metodu pomocníka nazvanou "log_in_as", která pro nás bude přihlašovat.

Naše technika pro přihlašování uživatelů bude záviset na typu testu. Uvnitř testů ovladačů můžeme manipulovat s metodou "session" přímo, přiřazujíce "user.id" klíči ":user_id":

```
def log_in_as(user)
  session[:user_id] = user.id
end
```

Metodu jsme nazvali "log_in_as" abychom předešli záměně s metodou "log_in" apliačního kódu. Její umístění je v třídě "ActiveSupport::TestCase" uvnitř souboru "test_helper", což je stejné umístění, jako u pomocníka "is_logged_in?" z dřívějška:

```
class ActiveSupport::TestCase
  fixtures :all

  # Vrati true pokud je testovaci uzivatel prihlasen.
  def is_logged_in?
    !session[:user_id].nil?
  end

  # Prihlasi se jako specificky uzivatel.
  def log_in_as(user)
    session[:user_id] = user.id
  end
end
```

V této kapitole tuto verzi metody potřebovat nebudeme, ale bude se nám hodit v budoucnu.

Uvnitř integračních testů nemůžeme ovládat "session" přímo, ale můžeme použít "post" na adresu sezení, a metoda "log_in_as" bude tedy vypadat následovně:

```
class ActionDispatch::IntegrationTest

  # Prihlasi se jako specificky uzivatel.
  def log_in_as(user, password: 'password', remember_me: '1')
    post login_path, params: { session: { email: user.email,
                                          password: password,
                                          remember_me: remember_me } }
  end
end
```

Jelikož je umístěna uvnitř třídy "", bude tato verze "log_in_as" zavolána v integračních testech. Používáme stejné jméno metody v obou případech proto, že nám to umožní dělat věci jako použití kódu z testu ovladače v testu integračním, aniž bychom museli udělat nějaké změny v přihlašovací metodě.

Když dáme metody dohromady, budou pomocníci vypadat takto (soubor "test/test_helper.rb"):

```
ENV['RAILS_ENV'] ||= 'test'
.
.
.
class ActiveSupport::TestCase
  fixtures :all

  # Vrati true pokud je testovaci uzivatel prihlasen.
  def is_logged_in?
    !session[:user_id].nil?
  end

  # Prihlasi se jako specificky uzivatel.
  def log_in_as(user)
    session[:user_id] = user.id
  end
end

class ActionDispatch::IntegrationTest

  # Prihlasi se jako specificky uzivatel.
  def log_in_as(user, password: 'password', remember_me: '1')
    post login_path, params: { session: { email: user.email,
                                          password: password,
                                          remember_me: remember_me } }
  end
end
```

Pro maximální flexibilitu přijímá druhá metoda "log_in_as" parametry se základními hodnotami pro heslo (hodnota "password") a checkbox "pamatuj si mě" (hodnota "1").

Abychom ověřili chování checkboxu, napíšeme si dva testy, jeden pro odeslání formuláře s nezaškrtnutým, a druhý se zaškrtnutým checkboxem. Díky pomocníkovi to bude jednoduché, protože budou jednotlivé případy vypadat následovně:

```
log_in_as(@user, remember_me: '1')
```

a

```
log_in_as(@user, remember_me: '0')
```

(Jelikož je "1" základní hodnota "remember_me", můžeme parametr úplně vynechat, ale pro zřejmou paralelu obou případů jsme ho uvedli.)

Po přihlášení můžeme ověřit, zda byl uživatel zapamatován tak, že se porozhlédneme po klíči "remember_token" v "cookies". Ideálně bychom ověřili, zda hodnota cookie je rovná uživatelovu tokenu, ale při současném návrhu se k takovým informacím nema test jak dostat: proměnna "user" v ovladači má atribut tokenu, ale (jelikož je "remember_token" virtuální) proměnná "@user" v testu ho nemá. Důležité pro nás v tomto případě je hlavně to, jestli je relevantní cookie "nil", nebo ne.

Zvláštní je, že z nějakého důvodu uvnitř testů metoda "cookie" nefunguje se symboly jako klíči, a tedy

```
cookies[:remember_token]
```

je vždy "nil". Naštěstí "cookies" funguje s řetězci jakožto klíči, a tedy

```
cookies['remember_token']
```

obsahuje hodnotu, kterou potřebujeme. Výsledný test bude tedy vypadat následovně (soubor "test/integration/users_login_test.rb"):

```
require 'test_helper'

class UsersLoginTest < ActionDispatch::IntegrationTest

  def setup
    @user = users(:michael)
  end
  .
  .
  .
  test "login with remembering" do
    log_in_as(@user, remember_me: '1')
    assert_not_empty cookies['remember_token']
  end

  test "login without remembering" do
    # Log in to set the cookie.
    log_in_as(@user, remember_me: '1')
    # Log in again and verify that the cookie is deleted.
    log_in_as(@user, remember_me: '0')
    assert_empty cookies['remember_token']
  end
end
```

Testy by nyní měly být zelené:

```
$ rails test
```

### Testování zapamatovací větve

Dříve jsme si ručně ověřili, že trvalé sezení, které jsme si zaimplementovali, funguje jak má, ale související větev pro metodu "current_user" je zcela neotestovaná. Oblíbený způsob pro ošetření takových situací je vyvolání výjimky v podezřelém netestovaném bloku kódu: pokud kód není pokrytý, testy projdou; pokud je pokrytý, výsledná chyba identifikuje daný test. Výsledek v tomto případě vypadá následovně (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper
  .
  .
  .
  # Vrati uzivatele podle dane token cookie.
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      raise       # Testy stale projdou, takze je dana vetev neotestovana.
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
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

V této chvíli jsou testy zelené:

```
$ rails test
```

Což je ovšem problém, jelikož je vypsaný kód rozbitý. Nemluvě o tom, že je nepraktické ručně ověřovat trvalá sezení, takže pokud bychom chtěli přepracovat metodu "current_user" (což v budoucnu budeme chtít), je důležité ji otestovat.

Jelikož obě verze pomocnické metody "log_in_as" automaticky nastavují "session[:user_id]" (ať už přímo, nebo skrze odeslání požadavku na adresu přihlášení), testování "zapamatovávací" větve metody "current_user" je v integračním testu obtížné. Naštěstí můžeme obejít toto omezení otestováním metody "current_user" přímo v testu pomocníka sezení, jehož soubor budeme muset vytvořit:

```
$ touch test/helpers/sessions_helper_test.rb
```

Testovací sekvence je jednoduchá:

1. Definuj proměnnou "user" za pomocí fixtur.
2. Zavolej metodu "remember" pro zapamatování daného uživatele.
3. Ověř, že "current_user" je roven danému uživateli.

Jelikož metoda "remember" nenastavuje "session[:user_id]", tato procedura efektivně otestuje onu "zapamatovávací" větev. Výsledek vypadá následovně (soubor "test/helpers/sessions_helper_test.rb"):

```
require 'test_helper'

class SessionsHelperTest < ActionView::TestCase

  def setup
    @user = users(:michael)
    remember(@user)
  end

  test "current_user returns right user when session is nil" do
    assert_equal @user, current_user
    assert is_logged_in?
  end

  test "current_user returns nil when remember digest is wrong" do
    @user.update_attribute(:remember_digest, User.digest(User.new_token))
    assert_nil current_user
  end
end
```

Přidali jsme si i druhý test, který ověřuje, že současný uživatel je "nil" pokud uživatelův digest neodpovídá tokenu, čímž testuje i "authenticated?", které je zanořené v podmínce "if":

```
if user && user.authenticated?(cookies[:remember_token])
```

Mohli bychom napsat i

```
assert_equal current_user, @user
```

a test by fungoval stejně, ale bežná zvyklost pro pořadí parametrů u "assert_equal" je "očekávaný" a "reálný":

```
assert_equal <ocekavany>, <realny>
```

Což v našem případě odpovídá

```
assert_equal @user, current_user
```

Test by měl být červený, jak jsme ostatně chtěli:

```
$ rails test test/helpers/sessions_helper_test.rb
```

Aby výše sepsané testy prošly, stačí odstranit "raise" a obnovit původní "current_user" metodu (soubor "app/helpers/sessions_helper.rb"):

```
module SessionsHelper
  .
  .
  .
  # Vrati uzivatele podle dane token cookie.
  def current_user
    if (user_id = session[:user_id])
      @current_user ||= User.find_by(id: user_id)
    elsif (user_id = cookies.signed[:user_id])
      user = User.find_by(id: user_id)
      if user && user.authenticated?(cookies[:remember_token])
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

Testy by měly být zelené:

```
$ rails test
```

Když teď máme "zapamatovací" větev "current_user" otestovanou, případné problémy budou odchyceny aniž bychom je museli hledat ručně.

## Shrnutí

V posledních třech kapitolách jsme toho stihli hodně a změnili naší slibnou, ale příliš nezformovanou aplikaci do stránky, která má plně funkční registraci a přihlašování. Zbývá akorát omezit přístup na některé stránky podle stavu přihlášení a uživatelské identity. Tuto záležitost vyřešíme v rámci implementace možnosti si upravovat vlastní informace, což je cíl příští kapitoly.

Než postoupíme dále, spojíme změny zpět do hlavní větve:

```
$ rails test
$ git add -A
$ git commit -m "Implementovano pokrocile prihlaseni"
$ git checkout master
$ git merge advanced-login
$ git push
```

### Co jsme se naučili

- Rails umí udržovat stavy z jedné stránky na druhou pomocí trvalých cookies skrze metodu "cookies".
- Každému uživateli jsme přiřadili token pro zapamatování a související digest (zpracování) pro použití v trvalých sezeních.
- Pomocí metody "cookies" můžeme vytvořit trvalé sezení umístěním trvalého tokenu pro zapamatování do prohlížeče.
- Stav přihlášení je odvozen od přítomnosti současného uživatele podle uživatelského id v dočasném sezení, nebo unikátním tokenu v sezení trvalém.
- Aplikace uživatele odhlašuje smazáním uživatelského id ze sezení a odstraněním trvalé cookie z prohlížeče.
- Ternární operátor je úsporný způsob pro napsání jednoduchých if-then podmínek.