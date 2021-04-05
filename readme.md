# Projekt 4
Autor: Ondrej Ambruš

---

## Názov:

## Popis navrhu
Ide o hru kde sa hlavná postava musí prebojovať krajinou ktorá bola zakliata zlým čarodejníkom. Tá je teraz plná príšer a uveznených ľudí ktorý potrebujú hrdinovu pomoc.

Používateľ sa najprv musí zaregistrovať. Môže tak urobiť priamo mailom alebo pomocou Google/Facebook.
Po vytvorení účtu si hráč nastaví `heslo` a `nick` ktorý bude vidideľný pre ostatných hráčov v hre.
Následne si môže vytvoriť postavu. Vyberie si `meno` pre svoju postavu. Potom si vyberie jednu z `class` *Warrior*, *Assassin*, *Archer*, *Mage*. Každá `class` má iné počiatočné *stats*. *Warrior* má najviac života, *Assassin* najviac poškodenia, *Archer* môže útočiť na diaľku a *Mage* používa magické útoky ktoré aplikujú rôzne efekty. Každá postava začína na svojej farme `level`-i 1 s 0 `XP` a jediným `item` a to motykou. Následne sa musí prebojovať cez lesy, katakomby, zamrznutú krajinu, močiare a púšť až ku hradu, v ktorom sa čarodejník ukrýva. Každá z týchto oblastí má vlastný typ nepriateľov. Čím ďalej je hráč v danom leveli tým viac druhov nepriateľov sa odomyká...

## Popis modelu
Základom celého modelu je `user`. Každý user má vlastnú `id` by mohlo byť aj viacero hráčov s rovnakým menom.
Následne má svoj registračný `email` a `password`. Taktiež má svoj `nick` podľa ktorého v kombinácií s `id` ho budú poznať iný hráči.

`User` má svoj `contact` list. Na ňom sú všetci hráči s ktorými sa daný `user` kontaktoval alebo ktorý kontaktovali jeho. Takto kontaktovať sa môžu cez `chat`. Každý hráč má samostatný `chat` s ostatnými hráčmi. `Chat` obsahuje správy `message`. Tie obsahujú informáciu komu sú smerované (`to`), kto danú správu poslal (`from`), obsah danej správy (`message body`) a čas odoslania danej správy (`time`).

Hráči taktiež môžu vytvárať klany. Každý klan može mať neobmedzený počet členov ale každý hráč môže byť iba v jednom klane. Klan obsahuje identifikátor `clan id` ktorý spolu s `name` tvorí *unique* názov pre daný klan. Každý klan si taktiež môže vytvoriť *roles* pre svojich členov. V klane si môžu hráči písať v skupinovom chate kde každý hráč vidí každú správu.

Každý hráč má svoje herné postavy. S každou postavou si hráč vytvára aj novú hru. Postava (`character`), má svoje `character id` aby sa viacero hráčových postáv mohlo volať rovnako. Následne majú už spomínaný `name`. Postava má taktiež `class`, na základe ktorého sa jej nastaví počiatočný `health`, `attack` a `defense`. Podľa `class` sa postave budú zvyšovať *stats* a  odomykať `abilities`. Tieto majú vlastný `name`, level na ktorom sa odomknú `unlock level`, `damage` ako aj zvyšovanie poškodenia podľa levelov `damage scaling`. Niektoré môžu mať aj `effect`, ktorý sa aplikuje po ich použití. *Stats* sa však nezvyšujú postave iba podľa levelov ale aj podľa predmetov `item` ktoré počas hrania nájde. Tie sa najprv uložia do inventáru postavy `inventory` a následne ich može hráč použiť. Naraz može mať hráč aktivované iba 2 predmety. Každý predmet má svoj názov `name`, popis `description`, cenu `price`, za ktorú môže hráč predať daný predmet na trhu a efekt `effect`, ktorý má daný predmet na hráča.
