# Projekt 4
Autori: Matej Rastocky a Ondrej Ambruš

---

## Názov: DUNGEON ATTACK

## Popis navrhu
Ide o hru kde sa hlavná postava musí prebojovať krajinou ktorá bola zakliata zlým čarodejníkom. Tá je teraz plná príšer a uveznených ľudí ktorý potrebujú hrdinovu pomoc.

Používateľ sa najprv musí zaregistrovať. Môže tak urobiť priamo mailom alebo pomocou Google/Facebook.
Po vytvorení účtu si hráč nastaví heslo `pass` a prezývku `nick` ktorý bude vidideľný pre ostatných hráčov v hre.
Následne si môže vytvoriť postavu. Vyberie si meno `name` pre svoju postavu. Potom si vyberie jednu z tried `class` napr. *Warrior*, *Assassin*, *Archer*, *Mage*. Každá trieda má iné počiatočné *stats*. *Warrior* má najviac života, *Assassin* najviac poškodenia, *Archer* môže útočiť na diaľku a *Mage* používa magické útoky ktoré aplikujú rôzne efekty. Každá postava začína na farme s `level`-om 1 s 0 `XP` a jediným predmetom `item` a to motykou. 

![Hero image](/img/DbS1.png)

Následne sa musí prebojovať cez lesy, katakomby, zamrznutú krajinu, močiare a púšť až ku hradu, v ktorom sa čarodejník ukrýva. Každá z týchto oblastí má vlastný typ nepriateľov. Čím ďalej je hráč v danom leveli tým viac druhov nepriateľov sa odomyká...

## Popis modelu
Základom celého modelu je `user`. Každý user má vlastnú `id` aby mohlo byť aj viacero hráčov s rovnakým menom.
Následne má svoj registračný `email` a `password`. Taktiež má svoj `nick` podľa ktorého v kombinácií s `id` ho budú poznať iný hráči.

``` postgres
CREATE TABLE IF NOT EXISTS "user" (
    player_id uuid PRIMARY KEY NOT NULL UNIQUE,
    nick varchar(64),
    email varchar(320),
    pass varchar(64),
    account_created date
);
```

### Google a Facebook login
V prípade, že používateľ chce použiť na prihlásenie **Facebook** alebo **Google**, môže tak urobiť. Databáza si uloží všetky potrebné informácie, ktoré *API* príslušnej stránky vráti.

``` postgres
CREATE TABLE IF NOT EXISTS facebook_login (
    player_id uuid REFERENCES "user" (player_id) NOT NULL UNIQUE,
    access_token varchar(256),
    expires_in bigint,
    signed_request varchar(256),
    user_id varchar(256)
);

CREATE TABLE IF NOT EXISTS google_login (
    player_id uuid REFERENCES "user" (player_id) NOT NULL UNIQUE,
    real_name varchar(128),
    icon_url text,
    jwt_issuer varchar(256),
    google_account_id bigint,
    azp varchar(256),
    client_id varchar(256),
    token_creation_date bigint,
    token_expiration_date bigint
);
```

`User` má svoj `contact` list. Na ňom sú všetci hráči s ktorými sa daný `user` kontaktoval alebo ktorý kontaktovali jeho. Takto kontaktovať sa môžu cez `chat`. Každý hráč má samostatný `chat` s ostatnými hráčmi. `Chat` obsahuje správy `message`. Tie obsahujú informáciu komu sú smerované (`to`), kto danú správu poslal (`from`), obsah danej správy (`message body`) a čas odoslania danej správy (`time`).

Hráči taktiež môžu vytvárať klany. Každý klan može mať neobmedzený počet členov ale každý hráč môže byť iba v jednom klane. Klan obsahuje identifikátor `clan id` ktorý spolu s `name` tvorí *unique* názov pre daný klan. Každý klan si taktiež môže vytvoriť *roles* pre svojich členov. V klane si môžu hráči písať v skupinovom chate kde každý hráč vidí každú správu.

Každý hráč má svoje herné postavy. S každou postavou si hráč vytvára aj novú hru. Postava (`character`), má svoje `character id` aby sa viacero hráčových postáv mohlo volať rovnako. Následne majú už spomínaný `name`. Postava má taktiež `class`, na základe ktorého sa jej nastaví počiatočný `health`, `attack` a `defense`. Podľa `class` sa postave budú zvyšovať *stats* a  odomykať `abilities`. Tieto majú vlastný `name`, level na ktorom sa odomknú `unlock level`, `damage` ako aj zvyšovanie poškodenia podľa levelov `damage scaling`. Niektoré môžu mať aj `effect`, ktorý sa aplikuje po ich použití. 

*Stats* sa však nezvyšujú postave iba podľa levelov ale aj podľa predmetov `item` ktoré počas hrania nájde. Tie sa najprv uložia do inventáru postavy `inventory` a následne ich može hráč použiť. Naraz može mať hráč aktivované iba 2 predmety. Každý predmet má svoj názov `name`, popis `description`, cenu `price`, za ktorú môže hráč predať daný predmet na trhu a efekt `effect`, ktorý má daný predmet na hráča.

Každá postava existuje v samostatnej hre `game` teda každá nová postava začína hru od začiatku. Hra da delí na levely `levels`. Každý level je samostatná mapa. Každá táto mapa má špeciálnych nepriateľov `enemy` a NPCčka `quest giver`.

Nepriatelia majú svoj názov `name`, pozíciu `location` a `level`, život `health`, poškodenie `damage`a obranu `defense` ktoré sa počítajú z levelu nepriatela. Hráč za ich zabitie dostane odmenu `death reward`. Nepriatelia sa môžu objaviť iba ak sú splnené ich podmienky `spawn conditions`. Tie môžu byť minimálny/maximálny level postavy, splnenie predošlej úlohy `prev. quest` alebo zabitie špecifickej príšery `parent monster`. Všetky tieto informácie sa nachádzajú v jednotlivých *log*-och.

NPCčka majú meno `name` a pozíciu na ktorej sa nachádzajú `location`. Pre hráča majú minimálne jeden *quest*. Tieto majú svoj `name`, `descritption`, cieľ, ktorý musí hráč splniť `goal`, odmenu `reward` a odomykaciu podmienku, ktorá sprístupní danú úlohu hráčovi. Táto podmienka môže byť zabitie istého počtu nepriatelov `monsters killed`, nájdenie predmetu `item found` alebo dosiahnuttie dostatočného levelu `user level achieved`.

Hra tiež obsahuje zápisy všetkého čo hráč urobí.

 ### Tabuľky

`Item log` obsahuje zápis o všetkých predmetoch ktoré hráč našiel. Obsahuje ich názov `name`, `level`, cenu `price`, čas nájdenia `time`, miesto nájdenia `location` a spôsob použitia `use`.

`Quest log` obsahuje zápisy všetkých úloch ktoré hráč plní/splnil. Obsahuje ich názov `name`, začiatok úlohy `quest started`, koniec úlohy, `quest ended`, čas, koľko hráč danú úlohu plnil `time taken`, odmenu za splnenie `reward` a miesto, kde hráč danú úlohu splnil `location`.

`Damage log` obsahuje zápisy o všetkom poškodení ktoré sa stalo v hre. Obsajuhe meno útočníka `attacker`, meno obrancu `defender`, základ poškodenia `damage base`, poškodenia zastavené obrancom `damage mitigated`, poškodenie ktoré obranca dostal `damage applied`, miesto kde sa boj odohráva `location` a čas kedy sa to stalo `time`.

`Kill log` obsahuje zápisy o všetkom, čo hráč zabil. Obsahuje názov nepriatela `enemy`, zdroj posledného poškodenia `last ability`, začiatok, dĺžku a koniec boja (`battle start`, `battle length` a `battle end`), odmenu `reward`, a na koniec miesto a čas (`location` a `time`).

`Effect log` obsahuje záznamy o všetkých efektoch. Tiež obsahuje záznam o útočníkovi a obrancovi (`attacker`, `defender`), efekte, ktorý bol aplikovaný `effect applied`, dĺžke toho efektu `effect length` a miesto a čas zápisu (`location`, `time`).
