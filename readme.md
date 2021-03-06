# Projekt 4
Autori: Matej Rástocký a Ondrej Ambruš

---

## Názov: DUNGEON ATTACK

## Popis návrhu
Ide o hru kde sa hlavná postava musí prebojovať krajinou ktorá bola zakliata zlým čarodejníkom. Tá je teraz plná príšer a uveznených ľudí ktorý potrebujú hrdinovu pomoc.

Používateľ (`user`) sa najprv musí zaregistrovať. Môže tak urobiť priamo mailom alebo pomocou Google/Facebook.
Po vytvorení účtu si hráč nastaví heslo (`pass`) a prezývku (`nick`) ktorý bude vidideľný pre ostatných hráčov v hre.
Následne si môže vytvoriť postavu. Vyberie si meno (`name`) pre svoju postavu. Potom si vyberie jednu z tried (`class`) napr. *Warrior*, *Assassin*, *Archer*, *Mage*. Každá trieda má iné počiatočné *stats*. *Warrior* má najviac života, *Assassin* najviac poškodenia, *Archer* môže útočiť na diaľku a *Mage* používa magické útoky ktoré aplikujú rôzne efekty. Každá postava začína na farme s `level`-om 1 s 0 bodmi skúseností (`XP`) a jediným predmetom (`item`) a to motykou. 

![Hero image](/img/DbS1.png)

Následne sa musí prebojovať cez lesy, katakomby, zamrznutú krajinu, močiare a púšť až ku hradu, v ktorom sa čarodejník ukrýva. Každá z týchto oblastí má vlastný typ nepriateľov. Čím ďalej je hráč v danom leveli tým viac druhov nepriateľov sa odomyká. Náš hrdina avšak tiež zosilnieva. Zbiera nové predmety a naberá levely. Otázkou ostáva, či sa na konci svojou silou vyrovná mocnému čarodejníkovi alebo podľahne čiernej mágií.

---

## Popis modelu
### Používateľ

Základom celého modelu je používateľ (`user`). Každý použivateľ má vlastnú `id` aby mohlo byť aj viacero hráčov s rovnakým menom.
Následne má svoj registračný `email` a heslo (`password`). Taktiež má svoju prezývku (`nick`) podľa ktorého v kombinácií s `id` ho budú poznať iný hráči napr. *database_master #420*

``` sql
CREATE TABLE IF NOT EXISTS "user" (
    player_id uuid PRIMARY KEY NOT NULL UNIQUE,
    nick varchar(64),
    email varchar(320),
    pass varchar(64),
    account_created date
);
```

---

### Google a Facebook prihlásenie
V prípade, že používateľ chce použiť na prihlásenie **Facebook** alebo **Google**, môže tak urobiť. Databáza si uloží všetky potrebné informácie, ktoré vráti *API* príslušnej stránky.

``` sql
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

---

### Chat
Používateľ má svoj zoznam kontaktov (`contact`). Na ňom sú všetci hráči s ktorými sa daný používateľ kontaktoval alebo ktorý kontaktovali jeho. Tento zoznam taktiež obsahuje hráčov ktorých hráč zablokoval, toto sa označuje cez `status` napr. *friends*, *blocked* atď. Takto kontaktovať sa môžu cez `chat`. Každý dvaja hráči majú samostatný `chat`. `Chat` obsahuje `id` druhého hráča (`other_side`). Samotné správy sú na tento `chat` naviazané pomocou `id`. Tie obsahujú informáciu kedy boli odoslané (`time_sent`), a obsah danej správy (`message body`).

``` sql
CREATE TABLE IF NOT EXISTS chat (
    chat_id uuid PRIMARY KEY NOT NULL UNIQUE,
    player_id uuid REFERENCES "user" (player_id) NOT NULL,
    other_side uuid REFERENCES "user" (player_id) NOT NULL
);

CREATE TABLE IF NOT EXISTS contact (
    player_id uuid REFERENCES "user" (player_id) NOT NULL,
    contact_id uuid REFERENCES "user" (player_id) NOT NULL,
    status varchar(16)
);

CREATE TABLE IF NOT EXISTS message (
    chat_id uuid REFERENCES chat (chat_id) NOT NULL,
    time_sent timestamp NOT NULL,
    message_body varchar(256)
);
```

---

### Klan

Hráči taktiež môžu vytvárať klany (`clan`). Každý klan može mať neobmedzený počet členov ale každý hráč môže byť iba v jednom klane. Klan obsahuje identifikátor (`clan id`) ktorý spolu s menom klanu (`name`) tvorí unikátny názov pre daný klan napr. *postgres_slayers #5432*.
Každý klan si taktiež môže vytvoriť role (`roles`) pre svojich členov. V klane si môžu hráči písať v skupinovom chate kde každý hráč vidí každú správu.

``` sql
CREATE TABLE IF NOT EXISTS clan (
    clan_id uuid PRIMARY KEY NOT NULL UNIQUE,
    creator_id uuid REFERENCES "user" (player_id) NOT NULL UNIQUE,
    name varchar(64),
    roles text
);

CREATE TABLE IF NOT EXISTS clan_member (
    clan_id uuid REFERENCES clan (clan_id) NOT NULL,
    player_id uuid REFERENCES "user" (player_id) NOT NULL UNIQUE,
    role varchar(32)
);

CREATE TABLE IF NOT EXISTS clan_chat (
    chat_id uuid PRIMARY KEY NOT NULL UNIQUE,
    player_id uuid REFERENCES "user" (player_id) NOT NULL,
    clan_id uuid REFERENCES clan (clan_id) NOT NULL
);

CREATE TABLE IF NOT EXISTS clan_message (
    chat_id uuid REFERENCES clan_chat (chat_id) NOT NULL,
    time_sent timestamp NOT NULL,
    message_body varchar(256)
);
```

---

### Postava
Každý hráč má svoje herné postavy. S každou postavou si hráč vytvára aj novú hru. Postava (`character`), má svoje `character id` aby sa viacero hráčových postáv mohlo volať rovnako. Následne majú už spomínané meno (`name`). Postava má taktiež triedu (`class`), na základe ktorého sa jej nastaví počiatočný živort (`health`), útok (`attack`) a obrana (`defense`). Podľa triedy sa postave budú zvyšovať *stats* a  odomykať schopnosti (`ability`). Tieto majú vlastné meno (`name`), level na ktorom sa odomknú (`unlock level`), poškodenie (`damage`), zvyšovanie poškodenia podľa levelov (`damage scaling`), manu potrebnú na ich použitie *v prípade čarodejníka* (`mana_cost`) a jej zvyšovanie s levelmi (`mana_cost_scaling`). Niektoré schopnosti môžu mať aj efekt (`effect`) a dĺžku daného efektu (`effect_duration`), ktorý sa aplikuje po ich použití.

``` sql
CREATE TABLE IF NOT EXISTS character (
    character_id uuid PRIMARY KEY UNIQUE NOT NULL,
    player_id uuid REFERENCES "user" (player_id) NOT NULL,
    name varchar(64),
    class_id uuid REFERENCES class (class_id) NOT NULL,
    level int,
    xp bigint,
    xp_next_lvl bigint,
    health int,
    attack int,
    defense int,
    location point
);

CREATE TABLE IF NOT EXISTS class (
    class_id uuid PRIMARY KEY UNIQUE NOT NULL,
    name varchar(32),
    base_health int,
    health_scaling int,
    base_health_regen int,
    health_regen_scaling int,
    base_attack int,
    attack_scaling int,
    base_defense int,
    defense_scaling int,
    base_mana int,
    mana_scaling int,
    mana_regen int,
    mana_regen_scaling int
);

CREATE TABLE IF NOT EXISTS ability (
    ability_id uuid PRIMARY KEY UNIQUE NOT NULL,
    class_id uuid REFERENCES class (class_id) NOT NULL,
    name varchar(32),
    unlock_level int,
    damage int,
    damage_scaling int,
    mana_cost int,
    mana_cost_scaling int,
    effect text,
    effect_duration interval
);
```

---

### Predmety
*Stats* sa však nezvyšujú postave iba podľa levelov ale aj podľa predmetov (`item`) ktoré počas hrania nájde. Tie sa najprv uložia do inventáru postavy (`inventory`) a následne ich može hráč použiť. Naraz može mať hráč aktivované iba 2 predmety. Každý predmet má svoj názov (`name`), popis (`description`), cenu (`price`), za ktorú môže hráč predať daný predmet na trhu a efekt (`effect`), ktorý má daný predmet na hráča. Taktiež obsahuje informácie o veľkosti (`size`) a váhe (`weight`) predmetu.

``` sql
CREATE TABLE IF NOT EXISTS item(
    item_id uuid PRIMARY KEY NOT NULL UNIQUE,
    name varchar(64),
    description varchar(256),
    price money,
    effect varchar(16),
    size int,
    weight bigint
);
```

---

### Hra / levely
Každá postava existuje v samostatnej hre (`game`) teda každá nová postava začína hru od začiatku. Hra da delí na levely (`levels`). Každý level je samostatná mapa. Každá táto mapa má špeciálnych nepriateľov (`enemy`) a NPCčka (`quest_giver`).

![Logical diagram for level building](/img/diag1.png)

``` sql

CREATE TABLE IF NOT EXISTS game(
    game_id uuid PRIMARY KEY UNIQUE NOT NULL,
    character_id uuid REFERENCES character (character_id),
    level_id uuid REFERENCES level (level_id)
);

CREATE TABLE IF NOT EXISTS level(
    level_id uuid PRIMARY KEY NOT NULL UNIQUE,
    map text,
    starting_point point,
    end_point point,
    enemy_spawn_points path
);

CREATE TABLE IF NOT EXISTS area(
    area_id uuid PRIMARY KEY UNIQUE NOT NULL,
    level_id uuid REFERENCES level (level_id),
    name varchar(64) NOT NULL,
    description varchar(256),
    area_area box,
    location point
);
```

---

### Nepriatelia

Nepriatelia majú svoj názov (`name`) a základný život (`base_health`), poškodenie (`base_damage`),
obranu(`base_defense`) a odmenu (`base_death_reward`). Zároveň obsahujú podmienky svojho vyvolania (`spawn_conditions`). Týmito podmienkami môžu byť minimálny/maximálny level hráča(`min_player_level`)(`max_palayer_level`), mapa na ktorej sa práve hráč nachádza (`level`), predošlá úloha ktorú hráč dokončil (`previous_quest`) alebo príšera ktorú hráč zabil (`previous_monster`). 

``` sql
CREATE TABLE IF NOT EXISTS spawn_conditions(
    sc_id uuid PRIMARY KEY UNIQUE NOT NULL,
    min_player_level int,
    max_player_level int,
    level uuid REFERENCES level (level_id),
    previous_quest uuid REFERENCES quest (quest_id)
);

CREATE TABLE IF NOT EXISTS enemy(
    enemy_id uuid PRIMARY KEY UNIQUE NOT NULL,
    name varchar(64),
    base_health int,
    base_damage int,
    base_defense int,
    base_death_reward json,
    spawn_condition uuid REFERENCES spawn_conditions (sc_id)
);
```

Tabuľka `enemy` funguje iba ako akýsi zoznam všetkých možných príšer s ich základnými informáciacmi. Priamo v hre sa vytvárajú inštancie nepriateľov, tieto obsahujú všetky informácie samostatne kedže ich život sa môže meniť na základe napr. toho, či ich hráč už napadol alebo nie.

![Bonk](https://oldschool.runescape.wiki/images/thumb/c/c5/Stranger.gif/220px-Stranger.gif?aca4b)

``` sql
CREATE TABLE IF NOT EXISTS enemy_instance(
    enemy_instance_id uuid PRIMARY KEY NOT NULL UNIQUE,
    enemy_id uuid REFERENCES enemy (enemy_id),
    location point,
    health int,
    damage int,
    defense int,
    death_reward json,
    level int
);
```

Všetky informácie o interakciách takýchto interakciách sa ukladajú do [tabuliek](###tabuľky).

---

### NPC
NPCčka majú meno (`name`) a pozíciu na ktorej sa nachádzajú (`location`). Pre hráča majú minimálne jednu úlohu. Tieto majú svoj názov (`name`), popis (`descritption`), cieľ ktorým je kombinácia počtu zabitých príšer (`monsters_killed`), nájdenia špeciálneho predmetu (`item_found`), odmenu (`reward`) a predošlú úlohu, ktorá sprístupní danú úlohu hráčovi.

``` sql
CREATE TABLE IF NOT EXISTS quest_givers(
    quest_giver_id uuid PRIMARY KEY UNIQUE NOT NULL,
    name varchar(64),
    level uuid REFERENCES level (level_id),
    location point
);

CREATE TABLE IF NOT EXISTS quest(
    quest_id uuid PRIMARY KEY UNIQUE NOT NULL,
    quest_giver_id uuid REFERENCES quest_givers (quest_giver_id) NOT NULL,
    name varchar(64) NOT NULL,
    description varchar(256) NOT NULL,
    monsters_killed bigint,
    item_found uuid REFERENCES item (item_id),
    character_level int,
    reward json,
    previous_quest uuid REFERENCES quest (quest_id)
);
```

---

### Inventár
Inventár je v tabuľkách definovaný ako `storage`. Teda nemusí ísť iba o batoh postavy ale aj o napr. vrecká, bedne a iné predmety ktoré si náš hrdina čarovne uviaže na chrbát. Samotná tabuľka obsahuje id úložného priestoru (`storage id`), postavy ktorej daný priestor patrí(`character_id`), názov úložného predmetu(`name`), jeho internú veľkosť (`size`) a váhový limit (`weight_limit`).

``` sql
CREATE TABLE IF NOT EXISTS storage(
    storage_id uuid PRIMARY KEY NOT NULL UNIQUE,
    character_id uuid REFERENCES character (character_id) NOT NULL,
    name varchar(64),
    size point,
    weight_limit bigint
);

CREATE TABLE IF NOT EXISTS item(
    item_id uuid PRIMARY KEY NOT NULL UNIQUE,
    name varchar(64),
    description varchar(256),
    price money,
    effect varchar(16),
    size int,
    weight bigint
);
```
---

 ### Tabuľky

`Item log` obsahuje zápis o všetkých predmetoch ktoré hráč našiel. Obsahuje ich názov (`name`), `level`, cenu (`price`), čas nájdenia (`time`), miesto nájdenia (`location`) a spôsob použitia (`use`). *Pod spôdobom použitia sa myslí, čo s daným predmetom hráč spravil napr. predal, zahodil, rozobral atď.*

``` sql
CREATE TABLE IF NOT EXISTS item_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    item_id uuid REFERENCES item (item_id),
    level_id uuid REFERENCES level (level_id),
    time timestamp DEFAULT now(),
    location point,
    use varchar(16)
);
```

`Quest log` obsahuje zápisy všetkých úloch ktoré hráč plní/splnil. Obsahuje ich názov (`name`), začiatok úlohy (`quest started`), koniec úlohy (`quest ended`), čas, koľko hráč danú úlohu plnil (`time taken`), odmenu za splnenie (`reward`) a miesto, kde hráč danú úlohu splnil (`location`).

``` sql
CREATE TABLE IF NOT EXISTS quest_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    quest_id uuid REFERENCES quest (quest_id),
    quest_started timestamp DEFAULT now(),
    quest_ended timestamp,
    time_taken interval,
    reward json,
    location point
);
```

`Damage dealt log` a `damage taken log` obsahuje zápisy o všetkom poškodení ktoré sa stalo v hre. Obsahuju id postavy a príšery (`character id`) (`enemy id`), základ poškodenia (`damage base`), poškodenia zastavené obrancom (`damage mitigated`), poškodenie ktoré obranca dostal (`damage applied`), miesto kde sa boj odohráva (`location`) a čas kedy sa to stalo (`time`).

``` sql
CREATE TABLE IF NOT EXISTS damage_dealt_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    character_id uuid REFERENCES character (character_id),
    enemy_id uuid REFERENCES enemy (enemy_id),
    damage_base int,
    damage_mitigated int,
    damage_taken int,
    location point,
    time timestamp DEFAULT now()
);

CREATE TABLE IF NOT EXISTS damage_taken_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    character_id uuid REFERENCES character (character_id),
    enemy_id uuid REFERENCES enemy (enemy_id),
    damage_base int,
    damage_mitigated int,
    damage_taken int,
    location point,
    time timestamp DEFAULT now()
);
```

`Kill log` obsahuje zápisy o všetkom, čo hráč zabil. Obsahuje id nepriatela `enemy id`, zdroj posledného poškodenia `last ability`, začiatok, dĺžku a koniec boja (`battle start`, `battle length` a `battle end`), informácie o poškodení v danom boji (`damage_taken_by_player`) (`damage_dealt_by_player`) (`damage_mitigated_by_player`) odmenu (`reward`), a na koniec miesto a čas (`location`) (`time`).

``` sql
CREATE TABLE IF NOT EXISTS kill_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    character_id uuid REFERENCES character (character_id),
    enemy_id uuid REFERENCES enemy (enemy_id),
    last_ability uuid REFERENCES ability (ability_id),
    battle_start timestamp,
    battle_end timestamp DEFAULT now(),
    battle_length interval,
    damage_taken_by_player int,
    damage_dealt_by_player int,
    damage_mitigated_by_player int,
    reward json,
    location point,
    time timestamp DEFAULT now()
);
```

`Effect log` obsahuje záznamy o všetkých efektoch. Tiež obsahuje id postavy a príšery (`character_id`), (`enemy_id`), efekte, ktorý bol aplikovaný (`effect`), dĺžke toho efektu (`effect length`) a miesto a čas zápisu (`location`) (`time`).

``` sql
CREATE TABLE IF NOT EXISTS effect_log(
    no uuid PRIMARY KEY UNIQUE NOT NULL,
    character_id uuid REFERENCES character (character_id),
    enemy_id uuid REFERENCES enemy (enemy_id),
    effect varchar(16),
    effect_duration interval,
    location point,
    time timestamp DEFAULT now()
);
```

---

## Príkladné scenáre

### Vytvorenie účtu

Hráč si chce založiť účet v hre. Na začiatok má na výber z prihlásenia pomocou emailu, **Google** účtu alebo **Facebook** účtu. 
+ V prípade použitia emailu si hráč zvolí heslo a následne overí email pomocou potvrdzovacieho emailu. Až následne môže pokračovať do poslednej časti registrácie.
+ V prípade použitia **Google** alebo **Facebook** prihlásenia sa hráč prihlási do danej služby ktorá pomocou *API* v vráti prihlasovací token. Tento sa uloží do príslušnej databázy.

Následne si hráč zvolí svoju prezývku, ktorú bude používať v hre. Týmto úspešne dokončí registráciu, všetky príslušné údaje sa v tomto bode uložia do databázy a hráč môže začať hrať.

### Vytvorenie postavy

Pri vytváraní postavy si hráč na obrazovke zvolí triedu svojej postavy a jej meno. Po potvrdení výberu sa z tabuľky `class` vytiahnú informácie o danej triede a zapíšu sa do tabuľky `character`. Táto tabuľka sa avšak používa iba na uloženie aktuálneho stavu postavy a postupu hráča. Všetky ostatné informácie ako napr. maximálny život sa vždy počítajú z tabuľky `class`. Z takto vytvorenou postavou môže hráč pokračovať do hry.

### Pohyb v hre

Hráč sa v hre pohybuje pomocou myšky. Pohybuje sa v štvorcovej mriežke podobne ako pri šachu. Preto sa pozícia ukladá ako `point` v tejto mriežke. Všetci nepriatelia a predmety zaberajú jedno takéto políčko. Pri stúpení na nepriateľa alebo predmet s nimi hráč automaticky interaguje. 

+ [V prípade nepriateľa sa otvorí nová *šachovnica* na ktorej sú umiestnený iba hráč a nepriateľ.](###boj)
+ V prípade predmetu si ho postava vloží do inventáru pokiaľ má postačujúce miesto. V takomto prípade sa tiež spraví zápis do tabuľky `item log`

### Boj

Boj prebieha na samostatnej ploche. Pohyb funguje rovnako ako v hre avšak tu môže hráč používať svoje schopnosti na boj. Samotný boj je rozdelený na kolá (*Turn based*). Po každom kole sa spraví zápis do tabuliek poškodenia a efektov. V prípade konca boja sa spraví zápis aj do tabuľky `kill log`.

#### Útoky

Pri použití útoku sa všetky informácie načítajú z tabuľky `ability`. Následne sa táto schopnosť vyvolá na hernej ploche. Jej tvar je určený pomocou dvojrozmerného poľa `cast shape` a aktuálneho otočenia postavy. Pokiaľ je nepriateľ zachytený v tomto území, dostane poškodenie. 

V prípade, že útok má byť vyslaný 3 políčka rovno pred postavu, pole `cast shape` bude vyzerať takto.

|  - | -  | O | -  | -  |
|---|---|---|---|---|
| -  | -  | x | -  | -  |
|  - | -  | x | -  | -  |
| -  |  - | x | -  | -  |

![Full logical diagram](/img/catdead1.png)
![Full logical diagram](/img/catdead2.png)

### Štatistiky

Do štatistických tabuľiek sa robia zápisy vždy, keď hráč vykoná súvisiacu akciu. 
+ Pri bojových tabuľkách sa spraví zápis po každom kole boja.
+ Do tabuľky predmetov sa spraví zápis vždy, keď hráč interaguje s predmetom (zdvihnutie, predaj atď.)
+ ...

Zo zozbieraných informácií v štatistických tabuľkách sa hráčovi môžu udelovať ocenenia za dosiahnuté úspechy poprípade neúspechy. Niektoré z nich môžu byť: 

+ Tu to začína!
    + Zabi prvú príšeru.

+ Oh no!
    + Predaj najlepší meč v hre.

+ Veterán
    + Dostaň 100 000 poškodenia.

#### Plnenie úloh

Splnenie úloh pridelených hráčovi počas prechádzania jednotlivých levelov sa tiež dá vyhodnotiť pomocou štatistických tabuľiek. Na príklad v prípade, že hráč dostane za úlohu zabiť 4 prekliate žaby ale on bol aktívny a tieto žaby už zabil nemusí tak urobiť znova. Úloha sa automaticky berie ako splnená.

---

## Logický diagram

![Full logical diagram](/logical&#32;diagram.png)

---

## Fyzický diagram

![Full logical diagram](/physical&#32;diagram.png)

---

Žiadny z obrázkov nesúvisí priamo s projektom. Ide iba o inšpiráciu zozbieranú počas mnohých rokov hrania RPG hier.