## Představení

Ruby on Rails (zkráceně pouze Rails nebo RoR) je framework určený k vývoji webu napsaný v programovacím jazyce Ruby. Od svého uvedení v roce 2004 se stal jedním z nejužitečnejších a nejoblíbenějších nástrojů pro tvorbu dymanických webových aplikací. Rails používají různorodé společnosti, jako [Airbnb](http://airbnb.com/), [Basecamp](http://basecamp.com/), [GitHub](http://github.com/), [Kickstarter](http://kickstarter.com/), [Shopify](http://shopify.com/), [Twitter](http://twitter.com/) a další.

Ruby on Rails je 100% open-source, dostupný pod [MIT Licencí](http://www.opensource.org/licenses/mit-license.php). Rails také vděčí za svůj úspěch elegantnímu a kompaktnímu designu; právě pružnost [Ruby](http://ruby-lang.org/), na kterém běží, z něj dělá [doménově specifický jazyk](http://en.wikipedia.org/wiki/Domain_Specific_Language) pro tvorbu webových aplikací. Díky tomu je řešení mnoha běžných webově-programovacích úkonů, jako je generování HTML, tvorba datových modelů a směrování URL s Rails podstatně jednodušší, a výsledný kód je stručný a čitelný.

Rails se také zvládá rychle přizpůsobovat novým způsobům vývoje webových technologií a strukturám frameworků. Rails byl například jedním z prvních frameworků jež zvládl plně pojmout a implementovat styl architektury REST pro strukturování webových aplikací (o kterém bude ještě řeč později).

Ale především za Rails stojí nadšená a podporující komunita. Tisíce open-source [přispivatelů](http://contributors.rubyonrails.org/), zábavné a informativní [konference](http://railsconf.com/), obrovské množství tzv. [gems](https://rubygems.org/) (samostatná uzavřená řešení specifických problémů, jako stránkování nebo upload obrázků), hojnost blogů a spousta diskuzních fór. Velké množství programátorů v Rails také usnadňuje řešení nevyhnutelných aplikačních chyb: oblíbený postup "vygooglit chybovou hlášku" tak prakticky vždy nabídne relevantní blogový příspěvek, nebo diskuzní vlákno na fóru.


## Instalace

Jelikož většinu kódu, který je před námi, budeme spouštět pod Linuxovým serverem, využijeme možnosti si Linux spustit pro vývojové účely lokálně, tedy konkrétně Ubuntu verze 18.04, který lze stáhnout [zde](http://releases.ubuntu.com/18.04/). I když je možné si ho nainstalovat jako samostatný operační systém (a při startu počítače si vybrat, který spustit, skrze tzv. "multiboot"), z praktických důvodů se vydáme cestou virtualizace skrze programy jako [VMware Workstation Player](https://www.vmware.com/sg/products/workstation-player/workstation-player-evaluation.html) (do kterého lze Ubuntu jednoduše nainstalovat a spouštět pak dle libosti).

Ubuntu má z různých distribucí Linuxu jedno z uživatelsky nejpřívějtivějších prostředí, takže ani začátečník se v něm úplně neztratí. Instalace dodatečných komponent a spousta dalších úkonů se ale obvykle provádí přes příkazovou řádku, resp. terminál, takže se budeme věnovat hlavně jemu.

V rámci všech článků jsou příkazy označeny znakem dolaru a mezerou, podobně jako je to právě v konzoli ("$ "), za kterou konkrétní příkaz následuje, aby se nepletly s výpisem konzole, který může být v bloku s kódem rovněž obsažen. Do samotného příkazu se tyto dva znaky ale pochopitelně nepíšou.

Dále je vhodný postup před instalací (obzvlášťě většího množství) aplikací použít příkaz 

```
$ sudo -i
```

díky kterému se konzole přepne na uživatele "root" a administrátorské úkony (jako je právě instalace) nebudou vyžadovat příkaz "sudo", ani ověřování heslem. Na druhou stranu je zřejmé, že je na místě zvýšená opatrnost a takovouto formu konzole používat jen na podobné úkony, ne na běžnou práci.  

V prvé řadě nainstalujeme balíčky Yarn, Node.js a další, které Rails vyžaduje. V terminálu stačí postupně zadat tyto příkazy (jednotlivé příkazy mohou být dlouhé, proto jsou pro lepší orientaci oddělené prázdným řádkem - zároveň je netřeba přepisovat, do konzole jde i vkládat text ze schránky):

```
$ curl -sL https://deb.nodesource.com/setup_8.x | sudo -E bash -

$ curl -sS https://dl.yarnpkg.com/debian/pubkey.gpg | sudo apt-key add -

$ echo "deb https://dl.yarnpkg.com/debian/ stable main" | sudo tee /etc/apt/sources.list.d/yarn.list

$ sudo apt-get update

$ sudo apt-get install git-core zlib1g-dev build-essential libssl-dev libreadline-dev libyaml-dev libsqlite3-dev sqlite3 libxml2-dev libxslt1-dev libcurl4-openssl-dev software-properties-common libffi-dev nodejs yarn
```

Příkazem "curl", což je v podstatě nástroj na přenos souborů, jsme si první stáhli/zinicializovali Yarn, a poté ho skrze příkaz "apt-get" spolu s dalšími nainstalovali.

Nainstalovali jsme si mimojiné i sqlite3, což je podpora stejnojmenného typu databáze, kterou Rails využívá. I když je jednodušší, na začátek bude pro naše potřeby stačit, a na komplexnější typy databází, jako je PostgreSQL nebo MySQL, lze bez problémů přejít později.

Dále nainstalujeme podporu Ruby formou "rbenv", což je běžně používaný nástroj pro správu verzí Ruby.

```
$ cd

$ git clone https://github.com/rbenv/rbenv.git ~/.rbenv

$ echo 'export PATH="$HOME/.rbenv/bin:$PATH"' >> ~/.bashrc

$ echo 'eval "$(rbenv init -)"' >> ~/.bashrc

$ exec $SHELL

$ git clone https://github.com/rbenv/ruby-build.git ~/.rbenv/plugins/ruby-build

$ echo 'export PATH="$HOME/.rbenv/plugins/ruby-build/bin:$PATH"' >> ~/.bashrc

$ exec $SHELL

$ rbenv install 2.5.1

$ rbenv global 2.5.1

$ ruby -v
```

Poté nainstalujeme Bundler:
```
$ gem install bundler
```
a zadáme příkaz:
```
$ rbenv rehash
```


### Rails

A teď, bez dalšího otálení:

```
$ gem install rails -v 5.2.0
```

Jelikož používáme rbenv, čeká nás ještě příkaz

```
$ rbenv rehash
```

A už zbývá jen pro ujištění, že jsme nainstalovali všechno v pořádku, zadat

```
$ rails -v
```

kde by odpověď systému měla být

```
Rails 5.2.0
```

## První aplikace

Jak bývá dobrým zvykem ve výuce programování, začneme jednoduchou aplikací, která vypíše jednoduchý řetězec "hello world" jak ve vývojovém prostředí, tak přímo na výsledné stránce.

Prakticky všechny aplikace v Rails začínají stejně, spuštěním příkazu "rails new", který vytvoří základní kostru aplikace v adresáři dle Vašeho výběru.

Především pro uživatele Windows může být práce s Unixovým příkazovým řádkem neznámá. Základní myšlenka příkazového řádku je jednoduchá: zadáváním jednoduchých příkazů mohou uživatelé provést velké množství operací, jako vytváření složek ("mkdir"), přesouvání a kopírování souborů ("mv" a "cp"), a procházení souborového systému změnou adresáře, ve kterém se uživatel aktuálně nachází ("cd"). Ačkoliv může příkazový řádek na uživatele grafických rozhraní (GUI) působit primitivně, zdání klame: jde o jeden z nejužitečnějších nástrojů pro vývojáře. Většina zkušených vývojářů má obvykle otevřených několik oken s terminály příkazového řádku.

Pro naše potřeby následuje alespoň několik nejběžnějších příkazů.

| Popis                            | Příkaz           | Příklad                 |
| -------------------------------- | ---------------- | ----------------------- |
| vypsat obsah                     | ls               | $ ls -l                 |
| vytvořit složku                  | mkdir <složka>   | $ mkdir environment     |
| přejít do složky                 | cd <složka>      | $ cd environment/       |
| cd o jednu úroveň složky výše    |                  | $ cd ..                 |
| cd do domovské složky            |                  | $ cd                    |
| cd k cíli včetně domovské složky |                  | $ cd ~/environment/     |
| přesunout (přejmenovat) soubor   | mv <zdroj> <cíl> | $ mv foo bar            |
| zkopírovat soubor                | cp <zdroj> <cíl> | $ cp foo bar            |
| odstranit soubor                 | rm <soubor>      | $ rm foo                |
| odstranit prázdnou složku        | rmdir <složka>   | $ rmdir environment/    |
| odstranit neprázdnou složku      | rm -rf <složka>  | $ rm -rf tmp/           |
| zobrazit obsah souboru           | cat <soubor>     | $ cat ~/.ssh/id_rsa.pub |



Dalším krokem je vytvoření aplikace použitím příkazu z předchozí části. Je rozumné si i vytvořit složku, ve které pak budeme všechny projekty skladovat. Opět použijeme parametr pro konkrétní verzi, abychom měli jistotu, že souborová struktura bude stejné verze, jako námi nainstalovaný Rails.

```
$ rails _5.2.0_ new hello_app
```

Všimněte si, kolik souborů a složek příkaz "rails" vytvoří. Tato standardní souborová struktura (obr. 1.7) je jednou z výhod Rails: z ničeho vytvoří fungující, i když minimalistickou aplikaci. Navíc díky tomu, že tuto strukturu sdílí všechny Railsové aplikace, se pak daleko snáze zorientujete v cizím kódu.

// obrazek 1.7 s desc. Souborová struktura nově vytvořené Rails aplikace.

Následující tabulka shrnuje základní prvky struktury. Postupně probereme většinu z nich v následujících článcích.



| Soubor/Složka | Význam                                                       |
| ------------- | ------------------------------------------------------------ |
| app/          | Kód jádra aplikace, včetně modelů (models), pohledů (views), řadičů (controllers) a pomocníků (helpers) |
| app/assets    | Aktiva (assets) aplikace, jako kaskádové stylování (CSS), JavaScript soubory a obrázky |
| bin/          | Spustitelné soubory                                          |
| config/       | Konfigurace aplikace                                         |
| db/           | Soubory databáze                                             |
| doc/          | Dokumentace k aplikaci                                       |
| lib/          | Moduly knihoven                                              |
| lib/assets    | Aktiva (assets) knihoven, jako kaskádové stylování (CSS), JavaScript soubory a obrázky |
| log/          | Záznamové (log) soubory aplikace                             |
| public/       | Data dostupná veřejnosti (obvykle skrze webový prohlížeč), jako např. chybové stránky, atd. |
| bin/rails     | Program ke generování kódu, otevírání konzolových sezení, nebo spouštění místního serveru |
| test/         | Aplikační testy                                              |
| tmp/          | Dočasné soubory                                              |
| vendor/       | Kód třetích stran, jako pluginy a gemy                       |
| vendor/assets | Aktiva (assets) třetích stran, jako kaskádové stylování (CSS), JavaScript soubory a obrázky |
| README.md     | Letmý popis aplikace                                         |
| Rakefile      | Podpůrné (utility) úlohy dostpné skrze příkaz "rake"         |
| Gemfile       | Gemy vyžadované touto aplikací                               |
| Gemfile.lock  | Seznam použitých gemů pro ujištění, že všechny kopie aplikace používají správné verze gemů |
| config.ru     | Konfigurační soubor pro Rack                                 |
| .gitignore    | Typy souborů, které má Git ignorovat                         |

### Bundler

Po vytvoření nové Railsové aplikace je dalším krokem použití nástroje Bundler k instalaci a zahrnutí gemů, které aplikace potřebuje. Bundler se spouští automaticky v rámci příkazu "rails", ale v této sekci v něm provedeme pár změn a spustíme ho znovu. V prvé řadě bude třeba otevřít soubor Gemfile v textovém editoru. I když se mohou konkrétní čísla verzí a jiné drobnosti mírně lišit, obsah by měl vypadat zhruba jako na obr. 1.8. Pokud soubory a složky neodpovídají obrázku, obnovte okno pomocí klávesy F5. Obecně platí, že pokud v souborovém systému někdy něco není na předpokládaném místě, je obnovení (refresh) vždy prvním krokem řešení.

// obrazek 1.8 s desc Základní verze souboru Gemfile v textovém editoru.

Všimněte si, že je mnoho řádků tzv. zakomentovaných (commented out) symbolem mřížky # (tzv. hash). Jsou zde pro snažší pochopení významu a syntaxe gemů, které bývají běžně potřeba. Prozatím nebudeme potřebovat jiné gemy, než ty základní.

Pokud neupřesníte číslo verze, kterou má příkaz "gem" nainstalovat, Bundler se automaticky pokusí o instalaci nejnovější verze gemu. V našem případě je to v kódu řádek

```
gem 'sqlite3'
```
Jsou dva běžné způsoby, jak upřesnit rozsah verzí gemu, což nám umožňuje určitou kontrolu nad verzemi, které Rails používá. První vypadá takto:

```
gem 'uglifier', '>= 1.3.0'
```
Příkaz nainstaluje poslední verzi gemu "uglifier" (který se stará o kompresi souborů aktiv) pokud je větší nebo roven verzi 1.3.0. Například verze 7.2 se tímpádem bez problémů nainstaluje. Druhý způsob vypadá takto:
```
gem 'coffee-rails', '~> 4.0.0'
```

Což nainstaluje gem "coffee-rails", který je novější, než verze 4.0.0, ale _ne_ novější, než verze 4.1.0. Jinak řečeno, operátor ">=" vždy nainstaluje nejnovější gem, kdežto "~> 4.0.0" nainstaluje verzi s rozdílným posledním číslem (např. ze 4.0.0 na 4.0.1) . Bohužel se stává, že i drobné změny ve verzích mohou aplikaci rozbít, a proto budeme všude uvádět plné čísla verzí pro všechny gemy.

Náš upravený Gemfile na konkrétní verze gemů tedy bude vypadat takto:
```
source 'https://rubygems.org'

gem 'rails',        '5.2.0'
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

# Windows does not include zoneinfo files, so bundle the tzinfo-data gem
gem 'tzinfo-data', platforms: [:mingw, :mswin, :x64_mingw, :jruby]
```



Po úspěšné úpravě souboru Gemfile stačí gemy nainstalovat příkazem "bundle install"

```
$ cd hello_app/
$ bundle install
```



Instalace může chvíli trvat, ale dodá nám plně připravenou aplikaci. Pokud by příkaz "bundle install" vyžadoval nejprve spuštění příkazu "bundle update", není důvod mu nevyhovět. Obecně platí, že pokud někdy nejdou věci přesně podle plánu, podobné chybové hlášky mohou obsahovat kompletní postup k jejich řešení, a tedy není důvod k panice.



### rails server



Díky předchozím krokům nyní máme k dispozici aplikaci, kterou můžeme spustit - ale jak? Rails má naštěstí k dispozici skript (série příkazů pro příkazový řádek), který spustí lokální webserver pro usnadnění vývoje naší aplikace. 

```
$ cd hello_app/
$ rails server
```


Obecně vzato je rozumné spustit příkaz "rails server" v novém okně terminálu, aby bylo možné v prvním stále provádět příkazy (viz obr. 1.9). V prohlížeči už stačí jen zadat adresu [http://localhost:3000](http://localhost:3000) a lokální domovská stránka je na světě (viz obr. 1.10).

// obr 1.9 s desc Spuštění serveru Rails v samostatném okně terminálu.

// obr 1.10 s desc Základní stránka Rails.


### Model-View-Controller

Hned ze začátku je dobré mít celkový náhled do toho, jak aplikace v Rails fungují (obr. 1.14). Asi jste si všimli, že standardní aplikační struktura Rails obsahuje složku "app/" s třemi podsložkami: "models" (modely), "views" (pohledy) a "controllers" (řadiče). Rails tedy následuje architekturu [MVC](http://en.wikipedia.org/wiki/Model-view-controller) (Model-View-Controller), která pevně rozděluje data aplikací (jako informace o uživatelích) a kód, který je zobrazuje, což je běžný postup strukturování grafického uživatelského rozhraní (GUI).

Při interakci s aplikací Rails pošle prohlížeč _požadavek_, který přijme webový server a přepošle ho _řadiči_ Rails, který rozhodne, co se bude dít dál. V některých případech si _řadič_ rovnou vykreslí _pohled_, což je šablona převedena do HTML a poslána zpátky prohlížeči. U dynamických stránek je běžnější, že _řadič_ komunikuje s _modelem_, což je objekt v Ruby který reprezentuje prvek stránky (např. uživatele) a je zodpovědný za komunikaci s databází. Po povolání _modelu_ _řadič_ vykreslí _pohled_ a prohlížeč dostane kompletní webovou stránku v HTML.

// obrazek 1.14 s desc Schéma architektury model-view-controller (MVC).



### Ahoj světe!

MVC princip si uvedeme do praxe jednoduchou úpravou, a to přidáním _akce řadiče_ k vypsání řetězce "hello world!" namísto původní stránky Rails (obr. 1.13). Jak název napovídá, akce řadičů jsou definovány uvnitř řadičů. V současné chvíli je aplikační řadič (Application controller) jediný řadič, který máme, což si lze ověřit příkazem
```
$ ls app/controllers/*_controller.rb
```

pro zobrazení aktuálních řadičů. Do aplikačního řadiče (app/controllers/application_controller.rb) si tedy přidáme akci "hello", který použije funkci "vykreslení" (render) a vypíše HTML text "hello world!". Což bude vypadat takto:

```
class ApplicationController < ActionController::Base
  protect_from_forgery with: :exception

  def hello
    render html: "hello world!"
  end
end
```
Akci která vypíše řetězec jsme si zadefinovali a teď stačí Railsu říct, ať ji použije namísto původní stránky z obr. 1.13. Upravíme si tedy "router" (směrovač) Railsu, který operuje mezi řadičem a prohlížečem (obr. 1.14 - pro zjednodušení na obrázku chybí, ale ještě se k němu vrátíme) a rozhoduje, kam poslat požadavky které prohlížeč zašle. Konkrétně chceme změnit základní stránku, tedy "root route", která určuje stránku, kterou prohlížeč zobrazí po zadání tzv. "root URL" ("kořenové", základní adresy stránky). Jelikož je adresa úplně základní, jako http://www.priklad.cz/ (kde za posledním lomítkem nic není), root URL se často zkracuje jako prosté / (tedy lomítko).

Syntaxe (způsob zápisu) v souboru "config/routes.rb" je jednoduchá:
```
root 'nazev_radice#nazev_akce'
```

V našem případě je název řadiče "application" a název akce "hello", takže soubor upravíme aby vypadal následovně:
```
Rails.application.routes.draw do
  root 'application#hello'
end
```

Po úpravách obou souborů už nám konečně základní adresa naší stránky vypíše pozdrav světu (obr. 1.15).

// obrazek 1.15 s desc Zobrazení "hello world!" v prohlížeči.



## Git a verzování

Když máme hotovou funkční "hello world" aplikaci, zastavíme se na chvíli u kroku, který je sice technicky vzato volitelný, ale pro zkušenější vývojáře a reálnou praxi téměř nezbytný: umístění kódu naší aplikace pod kontrolu [verzování](https://en.wikipedia.org/wiki/Version_control). Systémy pro verzování nám poskytují přehled o změnách v kódu, výrazně usnadňují spolupráci a v případě potřeby umožňují návrat k předchozím verzím kódu (například v případě omylem smazaných souborů). Znalost práce s verzovacím systémem (version control system) je požadovaná dovednost pro každého profesionálního vývojáře softwaru.

Existuje mnoho možností verzování, ale jako standard pro komunitu Railsu (a mnoho jiných) je [Git](http://git-scm.com/), distribuovaný systém verzování původně vyvinutý pro jádro Linuxu. I když je Git tématem sám o sobě, budeme se mu věnovat jen okrajově, alespoň pro naučení základů.

Krom výše zmiňovaných výhod nám Git umožní i hned v následujícím článku naši aplikaci "vypustit ven" k používání ("deploy" - čeština bohužel nemá pro tento výraz příliš vhodnou obdobu).


### Instalace a nastavení

Před použitím Gitu je třeba provést několik jednorázových kroků. Jedná se o systémová nastavení, takže je stačí na každém počítači provést pouze jednou:

```
$ git config --global user.name "Vaše Jméno"
$ git config --global user.email vas.email@priklad.cz
```

Zadané jméno a email budou k dispozici v jakémkoliv repozitáři, který se rozhodnete zveřejnit.


#### První nastavení repozitáře

Nyní nás čekají kroky, které jsou potřeba pokaždé, kdy vytváříte nový repozitář (úložiště, v angl. textu se občas zkracuje na "repo"). První krok je přejít do kořenového adresáře naší aplikace a inicializovat nový repozitář:

```
$ git init
```

Dalším krokem je přidání všech souborů projektu do repozitáře příkazem "git add -A":

```
$ git add -A
```

Příkaz přidá všechny soubory v daném adresáři krom těch, jejichž vzory jsou uvedeny ve speciálním souboru ".gitignore". Příkaz "rails new" automaticky vygeneruje ".gitignore" soubor vhodný pro projekty Rails, ale je možné přidat i vlastní vzory.

Přidané soubory jsou prvně umístěny do "shromaždiště" (staging area), které obsahuje změny k provedení v našem projektu. Jaké soubory tam konkrétně jsou lze zjistit přes příkaz "status":

```
$ git status
```
Gitu pak řekneme, že si provedené změny chceme uložit přes příkaz "commit":

```
$ git commit -m "Inicializace repozitare"
```

Parametr "-m" nám umožňuje přidat k ukládací akci komentář; pokud "-m" vynecháme, Git automaticky otevře základní systémový editor a nechá nás napsat komentář tam.

Je důležité vědět, že akce "commit" je pouze lokální, uložena na počítači na kterém je prováděna. Jak obdobnou akci provést na vzdáleném repozitáři (přes příkaz "git push") se dozvíme záhy.

Záznam provedených "commit" akcí lze pak vypsat přes příkaz "log":

```
$ git log
```



###K čemu je Git dobrý?

Pokud jste nikdy nepoužívali verzovací software, možná není úplně zřejmé, k čemu může být dobrý, takže si uvedeme příklad. Dejme tomu, že jste omylem provedli kritické změny, jako například smazání složky "app/controllers/".

```
$ ls app/controllers/
$ rm -rf app/controllers/
$ ls app/controllers/
```

První použijeme unixový příkaz "ls" pro výpis obsahu složky "app/controllers/" a pak ho příkazem "rm" smažeme. Parametr "-rf" znamená "recursive force" a rekurzivně smaže všechny obsažené soubory, složky, podsložky a tak dále, bez dotazování se u každého jednotlivé akce mazání.

Podívejme se na status, co se změnilo:

```
$ git status
```

Systém zjistil, že soubor byl smazán, ale změny jsou pouze v "pracovním stromě"; nebyly odeslány příkazem "commit". Změny tedy můžeme beztrestně vrátit příkazem "checkout" a parametrem "-f" pro vynucené přepsání současných změn:

```
$ git checkout -f
$ git status

$ ls app/controllers/
```

A chybějící soubory a složky jsou zpět na svém místě.



### Bitbucket

Když teď na náš projekt dohlíží Git, můžeme nahrát náš kód na [Bitbucket](http://www.bitbucket.com), což je stránka optimalizovaná k schraňování a sdílení repozitářů Gitu. Nahrání kopie Vašeho repozitáře na Bitbucket je užitečné především ze dvou důvodů: Slouží jako plnohodnotná záloha kódu (včetně úplné historie akce commit) a jakoukoliv případnou budoucí spolupráci výrazně ulehčí.

Práce s Bitbucketem je přímočará, i když vyžaduje určitou technickou dovednost, aby vše běželo, jak má:

1. [Založte si Bitbucket účet](https://bitbucket.org/account/signup/), pokud ho ještě nemáte.
2. Zkopírujte svůj [public key](https://en.wikipedia.org/wiki/Public-key_cryptography) ("veřejný klíč") do schránky.
3. Přidejte svůj veřejný klíč do Bitbucketu kliknutím na obrázek avatara v pravém horním rohu a vyberte "Bitbucket settings", a pak "SSH keys" (obr. 1.18).

// obrazek 1.18 s desc Přidání veřejného SSH klíče.

Jakmile máte přidaný veřejný klíč, klikněte na "Create" pro [vytvoření nového repozitáře](https://bitbucket.org/repo/create) (obr. 1.19).

// obrazek 1.19 s desc Vytváření repozitáře pro první aplikaci na Bitbucket.

Při vyplňování informací o projektu nezapomeňte zaškrtnout kolonku u “This is a private repository.” (aby repozitář zůstal soukromý). Po kliknutí na "Create repository" (vytvořit repozitář), postupujte podle instrukcí pod "Command line > I have an existing project", což by mělo vypadat nějak takto:
```
$ git remote add origin git@bitbucket.org:<uzivatelske_jmeno>/hello_app.git
$ git push -u origin --all
```
Příkaz první řekne Gitu, že chcete přidat Bitbucket jako zdroj/původ vašeho repozitáře, a pak repozitář odešle ke vzdálenému zdroji. Samozřejmě je třeba v příkazu nahradit jméno skutečným.

Výsledkem je stránka na Bitbucketu pro repozitář "hello_app", s možností procházet soubory, historií změn a spoustou dalších věcí (obr. 1.20).

// obrazek 1.20 s desc Stránka repozitáře na Bitbucketu.



### Branch, edit, commit, merge

Pokud jste následovali předchozí kroky, mohli jste si všimnout, že Bitbucket automaticky vytvořil README soubor repozitáře (obr. 1.21). Jak naznačuje jeho přípona ".md", je psaný v Markdown, což je člověkem čitelný jazyk navržen ke snadné konverzi do HTML — což je přesně to, co Bitbucket udělal.

Automatické generování README je praktické, ale samozřejmě by bylo lepší, kdybychom si ho uzpůsobili pro náš projekt. Naučíme se u toho i něco o metodikách "branch", "edit", "commit" a "merge", které je nezbytné při práci s Gitem znát.

// obrazek 1.21 s desc Základní zobrazení Railsového README v Bitbucketu.



#### Branch (větev)

Git je výborný pro tvorbu tzv. větví (branches), což jsou vlastně kopie repozitáře, kde můžeme dělat (i experimentální) změny aniž bychom změnili původní soubory. Obvykle se hlavní repozitář označuje jako "master branch" (hlavní větev), a můžeme vytvořit jeho odnož příkazem "checkout" a parametrem "-b".

```
$ git checkout -b modify-README

$ git branch
```

Druhý příkaz, "git branch", pouze vypíše seznam místních větví, a hvězdička určuje, kterou větev momentálně používáme. Příkaz "git checkout -b modify-README" tedy jednak vytvoří novou větev a zároveň nás do ní přepne.

Princip větví naplno doceníme hlavně tehdy, kdy pracujeme na projektu s větším množstvím vývojářů, ale i při samostatném vývoji najde své využití. Jelikož je hlavní větev izolována od jakýchkoliv změn, které provedeme ve vedlejších větvích, můžeme se kdykoliv vrátit zpět k hlavní a vedlejší větev smazat. Což je obzvlášť praktické, pokud se něco hodně nepovede.

Samozřejmě, u tak malých změn by v praxi nemělo smysl vytvářet novou větev, ale je dobré si osvojit reálné postupy (a dobré návyky) hned ze začátku.



#### Edit (úprava)

Po vytvoření vedlejší větve si upravíme soubor "README.md" trochou vlastního obsahu.

```
# Ruby on Rails Tutorial

## "hello world!"

Toto je naše první aplikace v Rails. Ahoj, světe!
```



#### Commit (uložení)

Po provedených změnách se podíváme na stav naší větve:

```
$ git status
```

a provedené změny uložíme s parametrem "-a", což bývá nejčastější případ, kdy chceme uložit všechny změny provedené ve stávajících souborech.

```
$ git commit -a -m "Zmena souboru README"
```

Pozor na nesprávné použití parametru "-a"; pokud jste přidali do projektu od posledně nové soubory, je třeba o nich Git informovat skrze příkaz "git add -A".



#### Merge (spojení)

Když teď máme všechny změny provedeny, můžeme spojit (merge) výsledky se základní větví:

```
$ git checkout master
$ git merge modify-README
```

Po spojení větví už můžeme vedlejší větev smazat:

```
$ git branch -d modify-README
```

Tento krok je volitelný, a je poměrně běžné vedlejší větev nemazat. Lze se pak libovolně přepínat mezi hlavní a vedlejší větví a ve vhodné chvíle je pouze spojovat.

Je také možné změny ve vedlejší větvi neuložit, v tomto případě příkazem "git branch -D":

```
# Pouze pro demonstrativní účely; netřeba provádět, pokud se ve větvi něco opravdu hodně nepokazí
$ git checkout -b topic-branch
<kroky ve kterých opravdu rozvrtáme větev>
$ git add -A
$ git commit -a -m "Opravdu velka chyba"
$ git checkout master
$ git branch -D topic-branch
```

Narozdíl od parametru "-d", parametr "-D" smaže větev i když jsme změny v ní nespojili.



####Push (nahrání)

Poslední krok, když už je náš README soubor upravený, je nahrání (push), změn na Bitbucket a kontrola výsledků. Jelikož jsme si zdroj definovali už dříve, můžeme opomenout "origin master" a provést pouze příkaz "git push":

```
$ git push
```

Bitbucket opět samostatně převede syntaxi Markdownu z našeho README souboru do čistého HTML.