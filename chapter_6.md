# Modelování uživatelů

V předchozí části jsme si připravili základ stránky pro vytváření nových uživatelů. Během několika následujících kapitol si celý tento systém plně zprovozníme. Pro začátek uděláme důležitý krok v podobě vytvoření datového modelu pro uživatele naší stránky, společně se způsobem, jak budeme data ukládat.

## Uživatelský model

I když je v nejbližší době naším cílem vytvořit stránku pro registraci uživatelů (viz mockup na obr. 6.1), momentálně by nám takový krok nebyl příliš platný: nemáme daná data kde uložit. Prvním krokem v rámci registrací bude tedy tvorba datové struktury, která pojme a uloží informace o uživatelích.

// obr. 6.1 Mockup registrační stránky.

Základní datová struktura pro datové modely se v Rails nazývá, vcelku přirozeně, model (tedy ono "M" v MVC architektuře). Obvyklé řešení problematiky přetrvávání dat v Rails je použití databáze, a základní knihovna pro interakci s databází se nazývá "Active Record". Active Record má k dispozici pestrou škálu metod pro vytváření, ukládání a nalézání datových objektů i bez potřeby použít strukturovaný dotazovací jazyk (structured query language, známé jako SQL), který používají [relační databáze](https://en.wikipedia.org/wiki/Relational_database). Dokonce Ruby obsahuje příjemnou funkcionalitu v podobě migrací, které umožňují psaní datových definicí přímo v Ruby, aniž by se člověk musel učit SQL jazyk. Díky tomu nás Rails poměrně dobře izoluje od přilišných detailů databází. Pro vývoj budeme používat verzi SQLite.

Jako obvykle si vytvoříme novou větev na Gitu:

```
$ git checkout -b modeling-users
```

### Migrace databází

Dřívě jsme si už vytvářeli vlastní třídu User s atributy "name" a "email". Posloužila nám jako praktický příklad, ale postrádala důležitou vlastnost persistence, tedy přetrvání: když jsme vytvořili objekt User v konzoli Rails, zmizel v momentě, kdy jsme konzoli opustili. V této části si vytvoříme model pro uživatele, který jen tak lehko nezmizí.

Začneme opět s modelem se dvěma atributy, "name" a "email", kde druhá jmenovaná poslouží jako unikátní uživatelské jméno. (Atribut pro hesla si přidáme posléze.) Původně jsme použili metodu "attr_accessor":

```
class User
  attr_accessor :name, :email
  .
  .
  .
end
```

Když ale použijeme pro modelování uživatelů Rails, nemusíme označovat atributy explicitně. Rails používá v základu relační databázi, která sestává z tabulek složených z datových řádků, kde každý řádek má sloupce datových atributů. Kupříkladu, pro uložení uživatelů se jmény a emailovými adresami vytvoříme tabulku "users" se sloupci "name" a "email" (kde každý řádek bude odpovídat jednomu uživateli). Příklad takové tabulky je na obrázku (obr. 6.2), a za ním následuje i náhled datového modelu pro srování (obr. 6.3). Tím, že pojmenujeme sloupce "name" a "email" necháme Active Record možnost se dovtípit, jak budou vypadat uživatelské objektové atributy.

// obr. 6.2 Náhledový diagram dat v tabulce "users".

// obr. 6.3 Náhled datového modelu uživatelů (User data model).



Ovladač jsme si minule vytvořili za pomocí příkazu:

```
$ rails generate controller Users new
```

Obdobný příkaz pro vytvoření modelu je "generate model", který použijeme pro vytvoření uživatelského modelu s atributy "name" a "email":

```
$ rails generate model User name:string email:string
      invoke  active_record
      create    db/migrate/20160523010738_create_users.rb
      create    app/models/user.rb
      invoke    test_unit
      create      test/models/user_test.rb
      create      test/fixtures/users.yml
```

(Jména ovladačů se v rámcí konvencí vždy označují množným číslem, ale modely jsou v čísle jednotném: "Users controller", "User model".) Předáním volitelných parametrů "name:string" a "email:string" sdělíme Railsu, že bychom rádi, aby dané atributy přidal, spolu s definováním jejich datových typů (v tomto případě "string", tedy řetězec).

Jedním z výsledků příkazu "generate" je nový soubor zvaný "migration". Migrace nabízí možnost pozměnění struktury databáze tak, že se náš datový model zvládne adaptovat na měnící se povahu požadavků. V případě modelu Users je migrace vytvořena automaticky skriptem pro generování modelu; vytvoří tabulku "users" se dvěma sloupci, "name" a "email", jak je vidět na výpisu níže (soubor "db/migrate/[timestamp]_create_users.rb"):

```
class CreateUsers < ActiveRecord::Migration[5.0]
  def change
    create_table :users do |t|
      t.string :name
      t.string :email

      t.timestamps
    end
  end
end
```

Jméno migračního souboru dostane automaticky předponu ve formě tzv. "timestamp" (doslova "časová známka"), která se vytváří z času, kdy byla migrace vygenerována. V dřevních dobách migrací se jako předpona připojovalo jen postupně narůstající číslo, díky čemuž ale docházelo ke konfliktům v týmech, kdy vícero programátorů mohlo mít migrace se stejným číslem. V současné chvíli stačí, ať nejsou migrace vytvořené ve stejné vteřině a k podobným problémům nemůže dojít.

Migrace samotná sestává z metody "change", která určuje změnu, která má být v databázi provedena. V našem případě použíje "change" metodu Rails zvanou "create_table" pro vytvoření tabulky v databázi k ukládání uživatelů. Metoda "create_table" přijímá blok s jednou blokovou proměnnou, v tomto případě "t" (zkráceně "table", tedy tabulka). Uvnitř bloku metoda "create_table" použije objekt "t" pro vytvoření sloupců "name" a "email" v databázi, oba typu "string". Jméno tabulky je v množném čísle ("users"), i když je jméno modelu v čísle jednotném ("User"), což odráží lingvistické zvyklosti, které Rails dodržuje: model reprezentuje jednoho uživatele, kdežto databázová tabulka sestává z mnoha uživatelů. Poslední řádek bloku, "t.timestamps", je speciální příkaz, který vytvoří dva "kouzelné sloupce" "created_at" a "updated_at", což jsou timestampy které se automaticky zaznamenávají v moment, kdy je daný uživatel vytvořen nebo aktualizován. Plný datový model reprezentovaný migrací je k vidění na obrázku (obr. 6.4).

// obr. 6.4 Uživatelský datový model vytvořený migrací.

Migraci jako takovou pak spustíme příkazem "db:migrate":

```
$ rails db:migrate
```

// obr. 6.6 Struktura tabulky "users" po migraci.

### Modelový soubor

Vytvořili jsme si tedy uživatelský (User) model, vygenerovali si a nechali proběhnout migraci, a prohlédli jsme si i výsledek: byl aktualizován soubor "development.sqlite3" vytvořením tabulky "users" se sloupci "id", "name", "email", "created_at" a "updated_at". Zkusíme tomu všemu teď i lépe porozumět.

Začneme kódem uživatelského modelu (soubor "app/models/user.rb"), který je, kulantně řečeno, hodně úsporný:

```
class User < ApplicationRecord
end
```

Jak jsme se naučili dříve, syntaxe "class User < ApplicationRecord" znamená, že třída User dědí ze třídy "ApplicationRecord", která zase dědí od "ActiveRecord::Base", takže náš uživatelský model má automaticky veškerou funkcionalitu třídy "ActiveRecord::Base". Tahle informace nám ale není příliš platná dokud nezjistíme, co vlastně "ActiveRecord::Base" dovede, takže si ukážeme pár příkladů.

### Vytváření uživatelských objektů

Ideální volbou pro prozkoumávání datových modelů pro nás zase bude konzole Rails. Jelikož ale (zatím) nechceme v databázi vykonávat trvalé změny, spustíme konzoli v tzv. "sandbox" (dosl. pískoviště) módu:

```
$ rails console --sandbox
Loading development environment in sandbox
Any modifications you make will be rolled back on exit
>>
```

Jak nám napovídá zpráva "Any modifications you make will  be rolled back on exit", pokud spustíme konzoli v sandbox módu, budou při jejím zavření všechny provedené změny navráceny zpět.

Jelikož si konzole automaticky nahraje prostředí Rails, model už máme zadefinovaný. Nový uživatelský objekt tímpádem vytvoříme jednoduše:

```
>> User.new
=> #<User id: nil, name: nil, email: nil, created_at: nil, updated_at: nil>
```

Takto vypadá v konzoli základní uživatelský objekt.

Pokud zavoláme "User.new" bez parametrů, vrátí nám objekt s atributy "nil". Pro inicializaci ale díky Active Record můžeme použít obdobný postup, jako dříve, a naplnit parametry haší:

```
>> user = User.new(name: "John Doe", email: "johndoe@example.com")
=> #<User id: nil, name: "John Doe", email: "johndoe@example.com",
created_at: nil, updated_at: nil>
```

Validita (platnost) je důležitým prvkem modelových objektů Active Record, a blíže si ji probereme později. Prozatím si můžeme ověřit, že námi vytvořený objekt je validní skrze metodu "valid?":

```
>> user.valid?
true
```

Zatím jsme se databáze ani nedotkli: "User.new" pouze vytvoří objekt v paměti, a "user.valid?" jednoduše ověří, zda je objekt validní. Abychom objekt uložili do databáze, musíme zavolat metodu "save" na proměnnou "user":

```
>> user.save
   (0.1ms)  SAVEPOINT active_record_1
  SQL (0.8ms)  INSERT INTO "users" ("name", "email", "created_at",
  "updated_at") VALUES (?, ?, ?, ?)  [["name", "John Doe"],
  ["email", "johndoe@example.com"], ["created_at", 2016-05-23 19:05:58 UTC],
  ["updated_at", 2016-05-23 19:05:58 UTC]]
   (0.1ms)  RELEASE SAVEPOINT active_record_1
=> true
```

Metoda "save" vrací při úspěchu "true" a "false" pokud se něco pokazí. (Momentálně není důvod, aby ukládání neuspělo, jelikož neprobíhá žádné ověřování; uvidíme pár takových případů záhy.) Pro úplnost konzole také zobrazí SQL příkaz, který odpovídí "user.save" metodě (jmenovitě tedy "INSERT INTO "users"...").

Náš nový objekt měl pro sloupec "id" a magické sloupce "created_at" a "updated_at" hodnoty "nil". Podíváme se, jestli se něco s uložením změnilo:

```
>> user
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 19:05:58", updated_at: "2016-05-23 19:05:58">
```

Můžeme vidět, že hodnota "id" je nyní 1, přičemž magické sloupce dostaly hodnotu současného data a času.

Stejně jako u naší User třídy dříve, i instance uživatelského modelu umožňují přístup k jejich atributům skrze "tečkovou notaci":

```
>> user.name
=> "John Doe"
>> user.email
=> "johndoe@example.com"
>> user.updated_at
=> Mon, 23 May 2016 19:05:58 UTC +00:00
```

Jak uvidíme později, je často praktický postup vytváření a ukládání modelu ve dvou krocích, jako jsme to teď udělali, ale Active Record umožňuje i jejich skloubení do jednoho příkazu "User.create":

```
>> User.create(name: "A Nother", email: "another@example.org")
#<User id: 2, name: "A Nother", email: "another@example.org", created_at:
"2016-05-23 19:18:46", updated_at: "2016-05-23 19:18:46">
>> foo = User.create(name: "Foo", email: "foo@bar.com")
#<User id: 3, name: "Foo", email: "foo@bar.com", created_at: "2016-05-23
19:19:06", updated_at: "2016-05-23 19:19:06">
```

Namísto "true" nebo "false" vrací "User.create" objekt jako takový, který můžeme přiřadit do proměnné (jako jsme to udělali s proměnnou "foo").

Obrácenou funkcí "create" je "destroy":

```
>> foo.destroy
   (0.1ms)  SAVEPOINT active_record_1
  SQL (0.2ms)  DELETE FROM "users" WHERE "users"."id" = ?  [["id", 3]]
   (0.1ms)  RELEASE SAVEPOINT active_record_1
=> #<User id: 3, name: "Foo", email: "foo@bar.com", created_at: "2016-05-23
19:19:06", updated_at: "2016-05-23 19:19:06">
```

Stejně jako "create" vrací i "destroy" zmiňovaný objekt, i když v praxi sotva kdy chce člověk vracenou hodnotu "destroy" nějak využít. Navíc zůstává zničený objekt v paměti:

```
>> foo
=> #<User id: 3, name: "Foo", email: "foo@bar.com", created_at: "2016-05-23
19:19:06", updated_at: "2016-05-23 19:19:06">
```

Jak tedy víme, že jsme opravdu objekt zničili? A ohledně uložených a nezničených objektů, jak si vytáhneme uživatele z databáze? Bude potřeba se naučit, jak použít Active Record pro hledání uživatelských objektů.

### Hledání uživatelských objektů

Active Record nabízí několik možností pro hledání objektů. Zkusíme si tedy najít našeho prvního vytvořeného uživatele a pak si ověříme, že jsme třetího ("foo") opravdu smazali:

```
>> User.find(1)
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 19:05:58", updated_at: "2016-05-23 19:05:58">
```

Funkci "User.find" jsme předali parametr v podobě hodnoty "id", takže Active Record nám daného uživatele vrátil.

Podívejme se, jestli uživatel s "id" 3 ještě stále existuje v databázi:

```
>> User.find(3)
ActiveRecord::RecordNotFound: Couldn't find User with ID=3
```

Jelikož jsme třetího uživatele zničili (zní to trochu ostře, ale v daném kontextu by šlo použít i mnohem slabší "smazali"), Active Record ho v databázi nenašel. Namísto toho vrátil "find" tzv. výjimku ("exception"), což je způsob vyjádření, že nastala zvláštní situace v rámci provedení programu. V tomto případě tedy neexistující Active Record "id", díky čemuž "find" vrátil výjimku "ActiveRecord::RecordNotFound".

Jako dodatečnou možnost k obecnému "find" umožňuje Active Record nalézt uživatele i na základě specifických vlastností:

```
>> User.find_by(email: "johndoe@example.com")
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 19:05:58", updated_at: "2016-05-23 19:05:58">
```

Jelikož budeme používat emailové adresy jako uživatelská jména, tento způsob použití funkce "find" bude obzvlášť praktický až se naučíme, jak umožňit uživatelům přihlášení.

Existuje i pár dalších obecných způsobů pro vyhledání uživatelů. Předně, "first":

```
>> User.first
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 19:05:58", updated_at: "2016-05-23 19:05:58">
```

Přirozeně vrátí "first" prvního uživatele v databázi. Také existuje "all":

```
>> User.all
=> #<ActiveRecord::Relation [#<User id: 1, name: "John Doe",
email: "johndoe@example.com", created_at: "2016-05-23 19:05:58",
updated_at: "2016-05-23 19:05:58">, #<User id: 2, name: "A Nother",
email: "another@example.org", created_at: "2016-05-23 19:18:46",
updated_at: "2016-05-23 19:18:46">]>
```

Jak je vidět z výpisu konzole, "User.all" vrátí všechny uživatele v databázi jako objekt třídy "ActiveRecord::Relation", což je prakticky vzato pole.

### Aktualizace uživatelských objektů

Jakmile máme vytvořeny objekty, budeme je často chtít i aktualizovat. Existují dva základní způsoby, jak toho dosáhnout. Předně můžeme přiřadit jednotlivé atributy individuálně, jako jsme to udělali už dříve:

```
>> user           # Just a reminder about our user's attributes
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 19:05:58", updated_at: "2016-05-23 19:05:58">
>> user.email = "johndoe@example.net"
=> "johndoe@example.net"
>> user.save
=> true
```

Poslední krok je nutný k tomu, aby se změny zapsaly do databáze. Co se stane bez uložení si můžeme prohlédnout skrze "reload", což je metoda, která nahraje objekt znova na základě informací z databáze:

```
>> user.email
=> "johndoe@example.net"
>> user.email = "foo@bar.com"
=> "foo@bar.com"
>> user.reload.email
=> "johndoe@example.net"
```

Mimochodem, když jsme si uživatele aktualizovali skrze "user.save", magické sloupce už nejsou shodné:

```
>> user.created_at
=> "2016-05-23 19:05:58"
>> user.updated_at
=> "2016-05-23 19:08:23"
```

Druhý způsob pro aktualizaci většího množství atributů je použití "update_attributes":

```
>> user.update_attributes(name: "The Dude", email: "dude@abides.org")
=> true
>> user.name
=> "The Dude"
>> user.email
=> "dude@abides.org"
```

Metoda "update_attributes" přijme haši atributů, a v případě úspěchu provede jak aktualizaci, tak uložení dohromady v jednom kroku (načež vrátí "true" jako oznámení, že uložení proběhlo úspěšně). Pokud ale jakákoliv validace selže, například když je vyžadováno heslo pro uložení záznamu (jak si naimplementujeme později), zavolání "update_attributes" selže. Pokud potřebujeme aktualizovat jen jeden atribut, použití jednotné verze "update_attribute" obejde toto omezení přeskočením validací:

```
>> user.update_attribute(:name, "El Duderino")
=> true
>> user.name
=> "El Duderino"
```



## Ověřování (validace) uživatelů

Uživatelský model, který jsme si vytvořili, má nyní fungující atributy "name" a "email", ale jsou naprosto "obecné": jakýkoliv řetězec (včetně prázdného) je v současné chvíli validní. Přitom jak jména, tak emailové adresy dodržují jistá specifika. Například, "name" nemůže být prázdné a "email" musí odpovídat specifickému formátu emailových adres. A především, jelikož budeme používat emailové adresy jakožto unikátní uživatelská jména pro přihlašování, nemůže jich existovat vícero stejných.

Ve zkratce, nemůžeme nechat "name" a "email" jako jen tak jakékoliv řetězce; musíme zavést jistá omezení na jejich hodnoty. Active Record nám umožňuje takováto omezení uplatňovat skrze validace (ověřování). Probereme si ty nejběžnější případy, jako je validace přítomnosti (ve smyslu existence), délky, formátu a unikátnosti. Nakonec přidáme i poslední běžnou validaci, a to potvrzení. 

### Validační test

I když není "test-driven development" (TDD, metoda psaní testů předem) vždy to nejlepší řešení, modelové validace jsou přesně tím typem problému, na které je TDD ideální. Je přecejen problematické si být jistý, že daná validace děla přesně to, co chceme, aniž bychom měli test, který to potvrdí.

Náš postup bude spočívat v tom, že začneme validním objektem modelu, nastavíme jeden z atributů tak, jak chceme, ať nevypadá, a pak si testem ověříme, že opravdu není validní. Jako záchrannou síť si první napíšeme test pro ujištění, že základní modelový objekt validní je. Díky tomu si můžeme být jistí, že validační test selže z toho správného důvodu (a ne proto, že už původní objekt nemusel být validní).

Použijeme tedy test, který nám (ač momentálně poněkud prázdný) vytvořil na začátku "generate" ("test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase
  # test "the truth" do
  #   assert true
  # end
end
```

Pro napsání testu na validní objekt vytvoříme uživatelský modelový objekt "@user" pomocí speciální metody "setup", která se automaticky spustí před každým testem. Jelikož je "@user" instanční proměnná, je automaticky dostupná ve všech testech, a můžeme její validitu ověřit pomocí metody "valid?". Výsledek vypadá takto:

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end

  test "should be valid" do
    assert @user.valid?
  end
end
```

Používáme metodu "assert", která v tomto případě uspěje pokud "@user.valid?" vrátí "true" a selže, pokud vrátí "false".

Jelikož náš uživatelský model momentálně žádné validace nemá, původní test by měl projít:

```
$ rails test:models
```

Použili jsme variantu "rails test:models" pro spuštění pouze modelových testů (narozdíl například od "rails test:integration" použité dříve).

### Validace přítomnosti

Asi nejzákladnější validací je přítomnost, která prostě ověří, zda je daný atribut přítomen. Například nyní se budeme věnovat ujišťování, že jak "name" tak "email" jsou vyplněny předtím, než se uživatel uloží do databáze.

Začneme testem na přítomnost atributu "name". Jak je v následujícím výpisu (stále soubor "test/models/user_test.rb") vidět, vše co musíme udělat je nastavit atribut "name" proměnné "@user" na prázdný řetězec (potažmo řetězec mezer) a pak ověřit (metodou "assert_not"), že výsledný uživatelský objekt není validní.

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end

  test "should be valid" do
    assert @user.valid?
  end

  test "name should be present" do
    @user.name = "     "
    assert_not @user.valid?
  end
end
```

V této fázi by měly modelové testy proběhnout červeně:

```
$ rails test:models
```

Možnost, jak ověřit přítomnost atributu "name" je použití metody "validates" spolu s parametrem "presence: true". Parametr "presence: true" je jednoprvková haš možností (jak jsme si říkali dříve, složené závorky jsou v tomto případě volitelné). Soubor "app/models/user.rb" si tedy upravíme následovně:

```
class User < ApplicationRecord
  validates :name, presence: true
end
```

I když to vypadá jako kouzlo, "validates" je prostě jen metoda. Ekvivalentní formulace za použití závorek by vypadala následovně:

```
class User < ApplicationRecord
  validates(:name, presence: true)
end
```

Zkusíme si tedy v konzoli, jak dopadlo naše validační snažení:

```
$ rails console --sandbox
>> user = User.new(name: "", email: "johndoe@example.com")
>> user.valid?
=> false
```

Ověříme si validitu proměnné "user" použitím metody "valid?", která nám vrátí "false" když objekt selže u jedné nebo více validací, a "true" pokud všechny validace projdou. V současné chvíli máme pouze jednu validaci, takže víme která selhala, ale vždy se hodí vědět podrobnosti pomocí zavolání vygenerovaného objektu "errors":

```
>> user.errors.full_messages
=> ["Name can't be blank"]
```

Jelikož uživatel není validní, pokus o jeho uložení automaticky selže:

```
>> user.save
=> false
```

Test by měl nyní proběhnout zeleně:

```
$ rails test:models
```

Když už teď víme jak, napsat si test ("test/models/user_test.rb") na přítomnost atributu "email" nebude žádný problém:

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end

  test "should be valid" do
    assert @user.valid?
  end

  test "name should be present" do
    @user.name = ""
    assert_not @user.valid?
  end

  test "email should be present" do
    @user.email = "     "
    assert_not @user.valid?
  end
end
```

A samozřejmě si upravíme i "app/models/user.rb":

```
class User < ApplicationRecord
  validates :name,  presence: true
  validates :email, presence: true
end
```

Ověřování přítomnosti je tímto hotové, a testy by měly proběhnout zeleně:

```
$ rails test
```



### Validace délky

Omezili jsme uživatelský model tak, že každý uživatel vyžaduje jméno, ale měli bychom zajít dál: jelikož budou jména zobrazena na demonstrační stránce, měli bychom jim dát nějaké omezení i na délku. Představu jak na to už máme a neměl by to tedy být problém.

Výběr maximální délky není žádná věda; prostě nastavíme 50 jakožto rozumnou horní hranici, a tedy jména čítající 51 a více znaků už nebudou validní. I když to sotva někdy bude problém, existuje šance, že uživatelská emailová adresa překoná maximální délku řetězců, což je pro mnoho databází 255. I když si ošetříme záhy i formát a něco takového tímpádem ošetřovat nebudeme muset, přidáme toto omezení jen pro úplnost. Test ("test/models/user_test.rb") si tedy upravíme takto:

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "name should not be too long" do
    @user.name = "a" * 51
    assert_not @user.valid?
  end

  test "email should not be too long" do
    @user.email = "a" * 244 + "@example.com"
    assert_not @user.valid?
  end
end
```

Pro usnadnění jsme použili "násobení řetězce" tak, aby byl 51 znaků dlouhý. V konzoli si můžeme vyzkoušet, jak vypadá výstup:

```
>> "a" * 51
=> "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa"
>> ("a" * 51).length
=> 51
```

Ověření emailu je nastaveno tak, aby vytvořilo validní adresu, která je o jeden znak delší:

```
>> "a" * 244 + "@example.com"
=> "aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaaa
aaaaaaaaaaa@example.com"
>> ("a" * 244 + "@example.com").length
=> 256
```

Testy by tedy měly proběhnout červeně:

```
$ rails test
```

Aby prošly, použijeme validační parametr pro omezení délky, tedy "length", zároveň s parametrem "maximum" pro omezení horní hranice. Upravíme si tedy "app/models/user.rb" takto:

```
class User < ApplicationRecord
  validates :name,  presence: true, length: { maximum: 50 }
  validates :email, presence: true, length: { maximum: 255 }
end
```

Testy by měly proběhnout zeleně:

```
$ rails test
```

Když už opět testování probíhá úspěšně, podíváme se na něco komplikovanějšího: formát emailu.

### Validace formátu

Naše validace atributu "name" vynucuje pouze minimální omezení, jakékoliv neprázdné jméno pod 51 znaků je dostatečně dobré, ale atribut "email" je přecejen náročnější záležitost, která musí splňovat vícero kritérií, aby mohla být považována za platnou. Prozatím jsme odmítali pouze prázdné adresy, ale teď už je budeme vyžadovat ve známém formátu "user@example.com".

Testy i validace nebudou nikterak extrémně důsledné, ale prostě jen dostatečně dobré na to, aby přijímaly většinu platných emailových adres a odmítaly neplatné. Začneme několika testy zahrnujícími kolekce validních a invalidních adres. Pro vytvoření těchto kolekcí je praktické znát užitečnou %w[] techniku pro vytváření polí řetězců, jak jsme se dříve naučili v rámci experimentování s konzolí:

```
>> %w[foo bar baz]
=> ["foo", "bar", "baz"]
>> addresses = %w[USER@foo.COM THE_US-ER@foo.bar.org first.last@foo.jp]
=> ["USER@foo.COM", "THE_US-ER@foo.bar.org", "first.last@foo.jp"]
>> addresses.each do |address|
?>   puts address
>> end
USER@foo.COM
THE_US-ER@foo.bar.org
first.last@foo.jp
```

Prošli jsme si prvky pole "addresses" pomocí metody "each". A když už tuhle techniku ovládáme, můžeme si napsat pár základních validačních testů na formát emailu.

Jelikož je taková validace ošemetná a náchylná k chybám, začneme s průchozími testy pro validní adresy, abychom odchytili případné chyby během validace. Jinak řečeno, chceme se ujistit že nebudou odmítnuty jen neplatné adresy jako "user@example,com", ale že i validní adresy jako "user@example.com" přijaty budou, i poté co zavedeme omezení. (Momentálně samozřejmě přijaty budou, protože jsou validní všechny neprázdné adresy.) Výsledek vzorku platných adres vypadá takto (soubor "test/models/user_test.rb"):

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email validation should accept valid addresses" do
    valid_addresses = %w[user@example.com USER@foo.COM A_US-ER@foo.bar.org
                         first.last@foo.jp alice+bob@baz.cn]
    valid_addresses.each do |valid_address|
      @user.email = valid_address
      assert @user.valid?, "#{valid_address.inspect} by mela byt platna"
    end
  end
end
```

Zahrnuli jsme i volitelný druhý parametr s vlastní chybovou zprávou, která v tomto případě identifikuje adresu, díky které test selhal:

```
assert @user.valid?, "#{valid_address.inspect} by mela byt platna"
```

Zahrnutí specifické adresy, která způsobuje selhání je zvlášť praktické v testu se smyčkou "each", jinak by jakékoliv selhání vypsalo jen číslo řádku, které je ovšem pro všechny emailové adresy stejné, a díky tomu bychom těžko zjistili zdroj problému.

Dále si přidáme testy pro invaliditu různých druhů neplatných emailových adres, jako právě "user@example,com" nebo "user_at_foo.org" (kde chybí znak zavináče). Opět si zahrneme i vlastní chybovou hlášku pro identifikaci adresy, která způsobila selhání.

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email validation should reject invalid addresses" do
    invalid_addresses = %w[user@example,com user_at_foo.org user.name@example.
                           foo@bar_baz.com foo@bar+baz.com]
    invalid_addresses.each do |invalid_address|
      @user.email = invalid_address
      assert_not @user.valid?, "#{invalid_address.inspect} by mela byt neplatna"
    end
  end
end
```

V současné chvíli jsou testy červené:

```
$ rails test
```

Aplikační kód pro validace emailového formátu používají validaci "format", která funguje takto:

```
validates :email, format: { with: /<regular expression>/ }
```

Řádek validuje atribut s daným regulárním výrazem (regular expression, zkráceně regex), což je mocný (a často i tajemný) jazyk pro vyhledávání vzorů v řetězcích. Musíme si tedy složit regulární výraz který odpovídá platné emailové adrese a zároveň neodpovídá neplatným.

Existuje plný regex pro platné emailové adresy podle oficiálního emailového standardu, ale je enormní, nepřehledný a v současné chvíli především kontraproduktivní. V rámci výuky použijeme pragmatičtější výraz který se v praxi osvědčil jako robustní. Vypadá takto:

```
VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
```

K lepšímu pochopení jednotlivých částí nám pomůže tabulka:

| **Výraz**                              | **Význam**                                             |
| -------------------------------------- | ------------------------------------------------------ |
| `/\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i` | plný výraz                                             |
| `/`                                    | začátek výrazu                                         |
| `\A`                                   | označení začátku řetězce                               |
| `[\w+\-.]+`                            | alespoň jednopísmenné slovo, plus, mínus nebo tečka    |
| `@`                                    | doslovný symbol "at" (na)                              |
| `[a-z\d\-.]+`                          | alespoň jedno písmeno, čislo, mínus nebo tečka         |
| `\.`                                   | doslovná tečka                                         |
| `[a-z]+`                               | alespoň jedno písmeno                                  |
| `\z`                                   | označení konce řetězce                                 |
| `/`                                    | konec výrazu                                           |
| `i`                                    | case-insensitive (nerozhodují velká nebo malá písmena) |

I když nám tabulka napoví hodně, pro daleko lepší pochopení regulárních výrazů je ještě lepší použít interaktivní program jako je [Rubular](http://www.rubular.com/) a v něm jednoduše experimentovat. Stránka má příjemné interaktivní rozhraní a praxi v rámci regulárních výrazů jen čistá teorie nahradit prostě nemůže.

// obr. 6.7 Editor regulárních výrazů Rubular.

Aplikování výrazu na validaci emailu v souboru "app/models/user.rb" pak vypadá následovně:

```
class User < ApplicationRecord
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX }
end
```

Výraz "VALID_EMAIL_REGEX" je konstanta, která se podle zvyklostí zapisuje velkými písmeny. Kód

```
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX }
```

zajišťuje, že jen emailové adresy které odpovídají vzorci budou považovány za validní.

Testy by tímpádem nyní měly být zelené:

```
$ rails test:models
```

Zbývá nám tedy poslední omezení: unikátnost emailu.



### Validace unikátnosti (jedinečnosti)

Pro vynucení unikátnosti emailových adres (abychom je mohli použít jako uživatelská jména) použijeme variantu ":unique" metody "validates". Je v tom ale velký háček, takže tato část zase tak jednoduchá nebude.

Začneme několika krátkými testy. V předchozích modelových testech jsme především používali "User.new", který vytvoří objekt Ruby v paměti, ale pro testy unikátnosti musíme tento záznam vložit do databáze. Původní test ("test/models/user_test.rb") na duplicitní email vypadá takto:

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email addresses should be unique" do
    duplicate_user = @user.dup
    @user.save
    assert_not duplicate_user.valid?
  end
end
```

Postup spočívá v tomto: vytvoříme uživatele se stejným emailem jako "@user" pomocí "@user.dup", což vytvoří duplicitního uživatele se stejnými atributy. Jelikož pak uložíme uživatele "@user", duplicitní uživatel má pak takovou emailovou adresu, která už v databázi existuje, a tímpádem není validní. 

Do "app/models/user.rb" si tedy přidáme parametr "uniqueness: true":

```
class User < ApplicationRecord
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: true
end
```

Nemáme ale ještě vystaráno. Emailové adresy jsou obvykle zpracovány bez ohledu na typ (velikost) použitého písma (case-insensitive, tedy "foo@bar.com" je vnímáno stejně jako "FOO@BAR.COM" nebo "FoO@BAr.coM") a naše validace by to měla zahrnout také. Zahrneme si to tedy do testu ("test/models/user_test.rb") :

```
require 'test_helper'

class UserTest < ActiveSupport::TestCase

  def setup
    @user = User.new(name: "Example User", email: "user@example.com")
  end
  .
  .
  .
  test "email addresses should be unique" do
    duplicate_user = @user.dup
    duplicate_user.email = @user.email.upcase
    @user.save
    assert_not duplicate_user.valid?
  end
end
```

Na řetězce použijeme metodu "upcase". Tento test dělá v podstatě to samé, co původní, ale s "upper-case" variantou emailu, tedy samá velká písmena. Pokud to zní trochu podivně, snad to osvětlí konzole:

```
$ rails console --sandbox
>> user = User.create(name: "Example User", email: "user@example.com")
>> user.email.upcase
=> "USER@EXAMPLE.COM"
>> duplicate_user = user.dup
>> duplicate_user.email = user.email.upcase
>> duplicate_user.valid?
=> true
```

Samozřejmě je "duplicate_user.valid?" na hodnotě "true" proto, že je validace unikátnosti citlivá na velikost písmen, ale chceme, ať je "false". Naštěstí ":uniqueness" přijímá parametr ":case_sensitive" právě pro tento účel (soubor "app/models/user.rb"):

```
class User < ApplicationRecord
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
end
```

Prostě jsme nahradili "true" možností "case_sensitive: false". Rails ostatně vyvozuje, že "uniqueness" by měla být "true".

V současné chvíli naše aplikace, i když stále s podstatným háčkem, vynucuje unikátnost emailu a testy by měly projít:

```
$ rails test
```

Je tu jen jeden drobný problém, a to ten, že validace unikátnosti Active Record negarantuje unikátnost na úrovni databáze. Následující scénář naznačí proč:

1. Alenka se zaregistruje do aplikace s adresou alice@wonderland.com.
2. Alenka omylem klikne na "Odeslat" dvakrát, čímž odešle dva požadavky v rychlém sledu.
3. Nastane následující sekvence: požadavek 1 vytvoří uživatele v paměti který projde validací, požadavek 2 uděla totéž, uživatel požadavku 1 se uloží, uživatel požadavku 2 se uloží
4. Výsledek: dva uživatelské záznamy se stejnou emailovou adresou, navzdory validaci unikátnosti

I navzdory tomu, že to zní nepravděpodobně, je to poměrně častý scénář (obzvláště u stránek s velkým provozem). Naštěstí lze řešení implementovat poměrně jednoduše: prostě vynutíme unikátnost i na úrovni databáze. Náš postup bude spočívat ve vytvoření databázového indexu v emailovém sloupci a následném vyžadování unikátnosti daného indexu.

Když vytváříme v databázi sloupec, musíme zvážit, zda podle něj budeme chtít v budoucnu vyhledávat. Vezměme si například atribut "email" vytvořený migrací dříve. Když v následující kapitole umožníme uživatelům přihlašování, musíme najít záznam korespondující s odeslanou adresou. Bohužel, v základním datovém modelu bychom uživatele na základě emailu našli jen projitím všech záznamů a porovnáním výsledků se zadaným emailem. Něco takového se označuje jako "full-table scan", tedy projití celé tabulky, a jak napovídá selský rozum, zrovna efektivní řešení to není.

Přidání indexu na sloupec emailu tento problém řeší. K pochopení principu databázového indexu je dobré si představit databázi jako knihu. Kdybychom chtěli nalézt řetězec, např. "foobar", museli bychom projít každou jednotlivou stránku a hledat dané slovo. S indexy (tedy záložkami) prohlédneme jen je a v ten moment víme, že daná stránka slovo obsahuje. Databáze funguje obdobně.

Emailový index tedy bude vyžadovat zásah do našich modelů dat, a to v Ruby řeší migrace. V současné chvíli si ale cheme přidat strukturu do existujícího modelu, takže si zavoláme generátor migrací přímo:

```
$ rails generate migration add_index_to_users_email
```

Narozdíl od migrace pro uživatele tato není předdefinovaná ("db/migrate/[timestamp]_add_index_to_users_email.rb") a musíme si ji naplnit:

```
class AddIndexToUsersEmail < ActiveRecord::Migration[5.0]
  def change
    add_index :users, :email, unique: true
  end
end
```

Používáme Rails metodu zvanou "add_index" pro přidání indexu sloupci "email" v tabulce "users". Index sám o sobě nevynucuje unikátnost, ale možnost "unique: true" ano.

Zbývá jen provést migraci samotnou:

```
$ rails db:migrate
```

V danou chvíli budou testy červené, a to díky porušení omezení unikátnosti v tzv. "fixturách", které obsahují ukázková data testovací databáze. Uživatelské fixtury byly vygenerovány automaticky, a když se podíváme do "test/fixtures/users.yml" tak zjistíme, že nejsou unikátní. (Nejsou ani validní, ale data fixtur neprocházi skrze validace.)

```
# Read about fixtures at http://api.rubyonrails.org/classes/ActiveRecord/
# FixtureSet.html

one:
  name: MyString
  email: MyString

two:
  name: MyString
  email: MyString
```

Jelikož zatím fixtury potřebovat nebudeme, prostě je smažeme a necháme "prázdný" soubor:

```
# empty
```

Když jsme vyřesili náš háček ohledně unikátnosti, je ještě jedna věc, kterou musíme vyřešit. Některé databázové adaptéry používají indexy, které jsou citlivé na velikost písmen (case-sensitive), a tímpádem řetězce jako "Foo@ExAMPle.CoM" a "foo@example.com" považuje za odlišné, ačkoliv naše aplikace je vnímá jako stejné. Abychom předešli této nekompatibilitě, převedeme všechny adresy na malá písmena předtím, než je uložíme do databáze. Použíjeme způsob zvaný "[callback](http://en.wikipedia.org/wiki/Callback)", což je metoda která se vyvolá ve specifickém bodě cyklu objektu Active Record. Momentalně jde o bod těsně před uložením objektu, takže použijeme callback "before_save" pro převedení emailů na malá písmena. Výsledek úpravy "app/models/user.rb" vypadá následovně:

```
class User < ApplicationRecord
  before_save { self.email = email.downcase }
  validates :name,  presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
end
```

Kód pošle blok callbacku "before_save" a nastaví uživatelský email na jeho verzi s malými písmeny pomocí metody "downcase".

Mohli jsme použít i variantu zápisu

```
self.email = self.email.downcase
```

(kde "self" odkazuje na momentálního uživatele), ale uvnitř uživatelského modelu je slovo "self" na pravé straně volitelné:

```
self.email = email.downcase
```

Všimli jsme si ostatně dříve, během experimentování s metodou "palindrome", že v přiřazování (levé straně) "self" volitelné není, a tedy

```
email = email.downcase
```

fungovat nebude. K tomuto tématu se ještě vrátíme později.

V této chvíli by scénář s Alenkou dopadl dobře: databáze by uložila uživatelský záznam podle prvního požadavku a odmítla druhé uložení kvůli porušení omezení na unikátnost emailu. Navíc se nám zavedením indexu na sloupci "email" povedlo předejít i onomu plnému prohledávání databáze, což náležitě doceníme později.



## Přidání bezpečného hesla

Když jsme si zadefinovali validace pro jména a emaily, můžeme přidat poslední uživatelský atribut: bezpečné heslo. Budeme požadovat, aby měl každý uživatel heslo (spolu s jeho potvrzením) a pak uložíme jeho zahašovanou podobu do databáze. I když to zní podobně, nikterak to nesouvisí s datovou strukturou Ruby, nýbrž jde o výsledek aplikování nezvratitelné funkce [hash](http://en.wikipedia.org/wiki/Hash_function) na vstupní data. Také si přidáme způsob, jak uživatele na základě hesla ověřit, ale tu samotnou pak použijeme až později v rámci přihlašování.

Postup pro ověřování tedy bude přijetí odeslaného hesla, jeho zahašování a porovnání výsledku se zahašovanou variantou uloženou v databázi. Pokud dojde ke shodě, odeslané heslo je správné a uživatel je ověřen. Porovnáním zahašovaných hodnot namísto čistých hesel budeme moci ověřovat uživatele aniž bychom byli nuceni ukládat hesla jako taková. Pokud by pak byla nějaký způsobem naše databáže kompromitována, hesla uživatelů zůstanou v bezpečí.

### Zahašované (hashed) heslo

Většina mechaniky bezpečného hesla bude implementována skrze jedinou metodu Rails zvanou "has_secure_password", kterou si zahrneme do uživatelského modelu následovně:

```
class User < ApplicationRecord
  .
  .
  .
  has_secure_password
end
```

Po zahrnutí do modelu tato jediná metoda přidá následující funkcionalitu:

- Schopnost uložit bezpečne zahašovaný "password_digest" atribut do databáze
- Dva virtuální atributy ("password" a "password_confirmation") včetně validace přítomnosti po vytvoření objektu a ověření, že se shodují
- Metodu "authenticate" která vrací uživatele pokud je heslo správně (jinak vrátí "false") 

Jediným požadavkem pro "has_secure_password" je, že související model musí mít atribut zvaný "password_digest". V případě modelu uživatelů bude datový model vypadat následovně (viz obr. 6.8).

// obr. 6.8 Datový model uživatelů s přidaným atributem "password_digest".

Pro implementaci tohoto modelu musíme vygenerovat vhodnou migraci pro sloupec "password_digest". Můžeme vybrat jakékoliv jméno migrace, ale je praktické zakončit jméno příponou "to_users", jelikož v tomto případě Rails automaticky vytváří migraci pro přidání sloupcu v tabulce "users". Výsledek bude vypadat takto:

```
$ rails generate migration add_password_digest_to_users password_digest:string
```

Dodali jsme také parametr "password_digest:string" se jménem a typem atributu který chceme vytvořit. Zahrnutím "password_digest:string" jsme dali Railsu dostatek informací k tomu, aby vytvořil celou migraci (soubor "db/migrate/[timestamp]_add_password_digest_to_users.rb") sám:

```
class AddPasswordDigestToUsers < ActiveRecord::Migration[5.0]
  def change
    add_column :users, :password_digest, :string
  end
end
```

Metodou "add_colum" migrace přidá sloupec "password_digest" do tabulky "users". Pro samotnou aplikaci změn provedeme migraci:

```
$ rails db:migrate
```

Pro samotné zpracování hesla používá "has_secure_password" kvalitní hašovací funkci zvanou [bcrypt](http://en.wikipedia.org/wiki/Bcrypt). I kdyby útočník nějakým způsobem sehnal kopii databáze, nebude se moci díky zahašování přihlásit, jelikož z ní nic nevyčte. Abychom ale bcrypt mohli použít, musíme si ho přidat do našeho Gemfile.

```
source 'https://rubygems.org'

gem 'rails',          '5.1.6'
gem 'bcrypt',         '3.1.12'
.
.
.
```

A spustit "bundle install" jako obvykle:

```
$ bundle install
```

### Uživatel má bezpečné heslo

Jakmile má náš uživatelský model k dispozici atribut "password_digest" a nainstalovaný bcrypt, můžeme přidat "has_secure_password" do modelu samotného ("app/models/user.rb"):

```
class User < ApplicationRecord
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
end
```

Testy nyní ale proběhnou červeně:

```
$ rails test
```

Důvodem je, že "has_secure_password" vynucuje validaci virtuálních atributů "password" a "password_confirmation", ale naše testy vytváří pouze uživatele s těmito atributy:

```
def setup
  @user = User.new(name: "Example User", email: "user@example.com")
end
```

Aby tedy test znovu prošel, potřebujeme přidat heslo a jeho potvrzení (soubor "test/models/user_test.rb"):

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
end
```

Testy by měly proběhnout na zelenou:

```
$ rails test
```

Za chvíli se podíváme na výhody přidání "has_secure_password" do uživatelského modelu, ale první přidáme alespoň minimální požadavky na bezpečnost hesla.



### Minimální standardy hesla

V praxi je dobré vynucovat alespon základní standardy při vytváření hesel, aby je nešlo jednoduše uhodnout. Způsobů je mnoho, ale pro jednoduchost použijeme jen minimální délku a vynucení neprázdného hesla. Minimálně šest znaků je rozumná délka a následná úprava "test/models/user_test.rb" pak bude vypadat takto:

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
  test "password should be present (nonblank)" do
    @user.password = @user.password_confirmation = " " * 6
    assert_not @user.valid?
  end

  test "password should have a minimum length" do
    @user.password = @user.password_confirmation = "a" * 5
    assert_not @user.valid?
  end
end
```

Všimněme si i použitého kompaktního vícenásobného přiřazení:

```
@user.password = @user.password_confirmation = "a" * 5
```

Daná hodnota je pak přiřazena nejen jako heslo, ale i jako jeho potvrzení (v tomto případě řetězec o délce 5, opět vytvořen za pomocí násobení řetězce).

Jelikož "maximum" jsme už použili u uživatelského jména, obdobné "minimum" určitě nikoho nepřekvapí:

```
validates :password, length: { minimum: 6 }
```

Spolu s validací "presence" pro zajištění neprázdných hesel bude uživatelský model ("app/models/user.rb") vypadat následovně:

```
class User < ApplicationRecord
  before_save { self.email = email.downcase }
  validates :name, presence: true, length: { maximum: 50 }
  VALID_EMAIL_REGEX = /\A[\w+\-.]+@[a-z\d\-.]+\.[a-z]+\z/i
  validates :email, presence: true, length: { maximum: 255 },
                    format: { with: VALID_EMAIL_REGEX },
                    uniqueness: { case_sensitive: false }
  has_secure_password
  validates :password, presence: true, length: { minimum: 6 }
end
```

Testy by už zase měly být zelené:

```
$ rails test:models
```

### Vytváření a ověřování uživatele

Když máme základní uživatelský model hotový, vytvoříme uživatele v databázi jako přípravu pro stránku, která bude zobrazovat detaily o daném uživateli. Podíváme se také bliže na dopady přidání "has_secure_password" do modelu, včetně prozkoumání důležité metody "authenticate".

Jelikož se uživatelé zatím skrze web nemohou do aplikace přihlásit (to nás čeká příště), použijeme konzoli Rails pro vytvoření nového uživatele ručně. Kvůli praktičnosti použijeme metodu "create", ale v tomto případě nespustíme konzoli v sandbox módu, aby se daný uživatel uložil do databáze. Stačí tedy spustit konzoli klasickým "rails console" a vytvořit uživatele s validním jménem, emailovou adresou a platným heslem i s jeho potvrzením:

```
$ rails console
>> User.create(name: "John Doe", email: "johndoe@example.com",
?>             password: "foobar", password_confirmation: "foobar")
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 20:36:46", updated_at: "2016-05-23 20:36:46",
password_digest: "$2a$10$xxucoRlMp06RLJSfWpZ8hO8Dt9AZXlGRi3usP3njQg3...">
```

Jakmile si uživatele vyhledáme a vypíšeme jeho heslo, je efekt "has_secure_password" patrný:

```
>> user = User.find_by(email: "johndoe@example.com")
>> user.password_digest
=> "$2a$10$xxucoRlMp06RLJSfWpZ8hO8Dt9AZXlGRi3usP3njQg3yOcVFzb6oK"
```

Jde o zahašovanou verzi hesla "foobar", která je použita pro inicializaci uživatelského objektu. Jelikož je vytvořena bcriptem, je výpočetně nepraktické (slušně řečeno) zjišťovat z této verze původní heslo.

Zároveň "has_secure_password" automaticky přidal metodu "authenticate" pro související modelové objekty. Tato metoda určuje, zda je dané heslo validní pro daného uživatele výpočtem haše a porovnáním výsledku s obsahem "password_digest" v databázi. V případě uživatele, kterého jsme právě vytvořili, můžeme vyzkoušet několik neplatných hesel:

```
>> user.authenticate("not_the_right_password")
false
>> user.authenticate("foobaz")
false
```

Jak vidno, "user.authenticate" vrátí "false" kvůli neplatnému heslu. Pokud ale ověření provedeme se správným heslem, "authenticate" vrátí uživatele samotného:

```
>> user.authenticate("foobar")
=> #<User id: 1, name: "John Doe", email: "johndoe@example.com",
created_at: "2016-05-23 20:36:46", updated_at: "2016-05-23 20:36:46",
password_digest: "$2a$10$xxucoRlMp06RLJSfWpZ8hO8Dt9AZXlGRi3usP3njQg3...">
```

Metodu "authenticate" později použijeme pro přihlašování uživatelů na naši stránku. Nebude pro nás důležité, že vrací uživatele jako takového, ale hlavně to, že z pohledu booleanské logiky vrací "true". Syntaxe !! převede objekt do jeho související booleanské hodnoty, a v případě "user.authenticate" nám to bohatě postačí:

```
>> !!user.authenticate("foobar")
=> true 
```

## Shrnutí

V této kapitole jsme si od nuly vytvořili funkční uživatelský model se jménem, emailem a heslem, spolu s validacemi které vynucují důležitá omezení na jejich hodnoty. Navíc už můžeme bezpečně autentifikovat uživatele za pomocí hesla.

V další části se budeme věnovat vytváření funkční registrace pro vytváření nových uživatelů, spolu se stránkou zobrazující jejich detaily. Dále pak vyřešíme i přihlašování samotné.

Opět je dobrá chvíle pro odeslání změn na Git:

```
$ rails test
$ git add -A
$ git commit -m "Vytvoreni zakladniho User modelu (vcetne hesel)"
```

A jejich spojení s hlavní větví a odeslání do vzdáleného repozitáře:

```
$ git checkout master
$ git merge modeling-users
$ git push
```

### Co jsme se naučili

- Migrace umožňují úpravy aplikačního datového modelu.
- Active Record má k dispozici mnoho metod pro vytváření a úpravu datových modelů.
- Validace Active Record nám umožňují použít omezení na data v našich modelech.
- Běžné validace zahrnují přítomnost, délku a formát.
- Regulární výrazy (regex) jsou záhadné, ale mocné.
- Definování databázového indexu výrazně zlepšuje efektivitu vyhledávání a umožňuje dodržovat unikátnost na úrovni databáze.
- Můžeme do modelu přidat bezpečné heslo za pomocí zabudované metody "has_secure_password".

