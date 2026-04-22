# CLI_COMMANDS

Tento dokument shrnuje CLI příkazy ve firmware `MeshCore`, které jsou prakticky důležité pro provoz a ladění repeatru.

Nejde o popis shellu nebo build nástrojů. Jsou to příkazy z firmware, které zpracovává `CommonCLI` v [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:210).

Dokument je záměrně zaměřený na:

- zapnutí a vypnutí repeat funkce
- ladění rádiových parametrů
- řízení floodingu a forwarding politik
- práci s regiony a flood scope
- provozní diagnostiku a statistiky
- pomocné příkazy užitečné při správě repeater uzlu

## Kde se CLI používá

CLI je nadstavba nad firmwarem. Podle buildu může být dostupné:

- přes lokální serial konzoli
- přes klienta, který posílá textové příkazy do firmware
- přes admin / management vrstvu nad `CommonCLI`

Některé příkazy jsou v kódu omezené pouze na lokální CLI. To poznáte podle podmínky `sender_timestamp == 0` v implementaci.

Prakticky to znamená:

- lokální serial konzole umí vše
- vzdálené CLI může mít část příkazů blokovanou

## Základní poznámky k repeateru

V `MeshCore` není repeater řízen jedním jediným příkazem. Chování repeatru je výsledkem několika nastavení:

- `repeat on|off`
- `flood.max`
- `loop.detect`
- `path.hash.mode`
- `txdelay`, `direct.txdelay`, `rxdelay`
- `radio ...`
- `tx`
- region pravidla `allowf/denyf/default/home`
- případně `powersaving`

Pokud chcete uzel jako stabilní repeater, nejdůležitější jsou:

- `set repeat on`
- rozumné `flood.max`
- vhodný `loop.detect`
- správná konfigurace rádia
- vypnutý agresivní power saving, pokud by omezoval provoz

## Přehled nejdůležitějších příkazů

### `set repeat on|off`

Účel:
- zapíná nebo vypíná přeposílání paketů uzlem

Syntaxe:

```text
set repeat on
set repeat off
```

Význam:
- `on`: uzel přeposílá pakety
- `off`: uzel forwarding vypne

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:544)

Poznámka:
- interně se nastavuje `disable_fwd`
- odpověď `OK - repeat is now ON/OFF`

Související dotaz:

```text
get repeat
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:763)

### `set flood.max <0-64>`

Účel:
- nastavuje maximální flood scope / flood limit, který uzel používá

Syntaxe:

```text
set flood.max 6
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:599)

Rozsah:
- `0` až `64`

Význam:
- nižší hodnota omezuje dosah floodingu a snižuje provoz
- vyšší hodnota zvyšuje šanci doručení, ale i zatížení sítě

Zobrazení:

```text
get flood.max
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:782)

Praktické použití:
- pro repeater v menší síti držet spíš konzervativní hodnotu
- pro backbone nebo hraniční uzel může být potřeba vyšší flood scope

### `set loop.detect off|minimal|moderate|strict`

Účel:
- řídí ochranu proti smyčkám při přeposílání

Syntaxe:

```text
set loop.detect off
set loop.detect minimal
set loop.detect moderate
set loop.detect strict
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:637)

Význam:
- `off`: bez ochrany, vhodné jen pro testy
- `minimal`: nejmenší zásah do provozu
- `moderate`: rozumný kompromis
- `strict`: agresivní ochrana proti smyčkám

Zobrazení:

```text
get loop.detect
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:797)

Doporučení:
- pro běžný repeater nepoužívat `off`
- `moderate` nebo `strict` bývá bezpečnější volba

### `set path.hash.mode <0|1|2>`

Účel:
- nastavuje režim práce s path hashem pro routování / detekci tras

Syntaxe:

```text
set path.hash.mode 0
set path.hash.mode 1
set path.hash.mode 2
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:627)

Zobrazení:

```text
get path.hash.mode
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:795)

Poznámka:
- význam jednotlivých módů není v CLI detailně popsaný textem
- prakticky jde o tuning routingu a potlačení duplicit / kolizních cest

## Rádiové parametry důležité pro repeater

### `set radio <freq> <bw> <sf> <cr>`

Účel:
- nastavuje základní LoRa parametry

Syntaxe:

```text
set radio 869.525 125.0 11 8
```

Parametry:
- `freq`: MHz
- `bw`: bandwidth v kHz
- `sf`: spreading factor
- `cr`: coding rate

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:555)

Odpověď:
- `OK - reboot to apply`

Zobrazení:

```text
get radio
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:773)

Poznámka:
- změna je perzistentní
- po změně je potřeba restart

### `tempradio <freq> <bw> <sf> <cr> <mins>`

Účel:
- dočasně přepne rádio na jinou konfiguraci bez trvalého přepsání prefs

Syntaxe:

```text
tempradio 869.525 125.0 11 8 15
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:269)

Význam:
- užitečné pro testování coverage, interoperability nebo dočasnou servisní konfiguraci

Omezení:
- poslední parametr `mins` musí být větší než `0`

### `set tx <dbm>`

Účel:
- nastavuje vysílací výkon rádia

Syntaxe:

```text
set tx 22
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:657)

Zobrazení:

```text
get tx
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:807)

Poznámka:
- tohle je jeden z nejdůležitějších parametrů repeatru
- vyšší výkon zlepšuje reach, ale zvyšuje spotřebu a může zhoršit interferenci

### `set radio.rxgain on|off`

Účel:
- zapíná nebo vypíná zvýšený RX gain na podporovaných SX1262/SX1268 platformách

Syntaxe:

```text
set radio.rxgain on
set radio.rxgain off
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:549)

Zobrazení:

```text
get radio.rxgain
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:769)

Použití:
- vhodné při ladění citlivosti repeatru
- záleží na konkrétní desce a rádiu

## Časování a chování přeposílání

### `set rxdelay <float>`

Účel:
- nastavuje základní receive delay parametr

Syntaxe:

```text
set rxdelay 0.5
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:581)

Zobrazení:

```text
get rxdelay
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:778)

### `set txdelay <float>`

Účel:
- nastavuje faktor pro odklad vysílání

Syntaxe:

```text
set txdelay 0.8
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:590)

Zobrazení:

```text
get txdelay
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:780)

### `set direct.txdelay <float>`

Účel:
- nastavuje faktor odkladu pro direct přenosy

Syntaxe:

```text
set direct.txdelay 0.5
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:608)

Zobrazení:

```text
get direct.txdelay
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:784)

Praktický význam:
- důležité při jemném ladění kolizí a odezvy ve vytížené síti
- obvykle se upravuje až při hlubším diagnostickém testování

### `set multi.acks <0|1>`

Účel:
- zapnutí nebo vypnutí multi-ack chování

Syntaxe:

```text
set multi.acks 1
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:491)

Zobrazení:

```text
get multi.acks
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:746)

Praktický význam:
- má dopad na potvrzování a tím i na zatížení sítě
- pro repeater může měnit množství obslužného provozu

### `set allow.read.only on|off`

Účel:
- řídí, zda je povolen read-only přístup

Syntaxe:

```text
set allow.read.only on
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:495)

Zobrazení:

```text
get allow.read.only
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:748)

Poznámka:
- není to forwarding parametr v úzkém smyslu
- ale bývá relevantní při provozu administrativně sdíleného repeatru

## Advert a presence příkazy

### `advert`

Účel:
- ručně pošle vlastní flood advertisement

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:223)

Použití:
- rychlé ověření, že uzel vysílá
- test viditelnosti repeatru v síti

### `advert.zerohop`

Účel:
- pošle lokální zero-hop advertisement

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:219)

Použití:
- lokální servisní diagnostika bez plného floodingu

### `set advert.interval <minutes>`

Účel:
- nastavuje periodu lokálního advertismentu

Syntaxe:

```text
set advert.interval 60
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:509)

Zobrazení:

```text
get advert.interval
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:752)

Rozsah:
- minimálně `60` minut pro lokální advert
- maximum `240` minut

### `set flood.advert.interval <hours>`

Účel:
- nastavuje periodu flood advertu

Syntaxe:

```text
set flood.advert.interval 12
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:499)

Zobrazení:

```text
get flood.advert.interval
```

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:750)

Rozsah:
- `3` až `168` hodin

Praktický význam:
- důležité pro to, jak často se repeater znovu “oznámí” síti
- kratší interval zlepšuje discoverability, ale přidává provoz

## Region příkazy pro řízení flood scope

Regiony jsou pro repeater zásadní, protože určují, kde je flood povolen nebo zakázán.

Implementace všech region příkazů:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:894)

### `region`

Účel:
- vypíše export region mapy

### `region load`

Účel:
- zahájí načtení regionů

### `region save`

Účel:
- uloží region mapu

### `region get <name>`

Účel:
- vrátí detail regionu a případného parent regionu

### `region put <name> [parent]`

Účel:
- vytvoří region

Poznámka:
- nový region má defaultně flood povolený

### `region remove <name>`

Účel:
- smaže region, pokud není ne-prázdný

### `region allowf <name>`

Účel:
- explicitně povolí flood pro region

### `region denyf <name>`

Účel:
- explicitně zakáže flood pro region

### `region list allowed`

Účel:
- vypíše regiony, kde je flood povolený

### `region list denied`

Účel:
- vypíše regiony, kde je flood zakázaný

### `region home`

Účel:
- zobrazí home region

### `region home <name>`

Účel:
- nastaví home region

### `region default`

Účel:
- ukáže default region

### `region default <name>`

Účel:
- nastaví default region

### `region default <null>`

Účel:
- zruší default region

Praktický význam regionů pro repeater:

- dovolují omezit flood na definované části sítě
- pomáhají držet backbone čistší
- jsou důležité tam, kde nechcete přenášet flood napříč určitými oblastmi

## Diagnostika repeatru

### `neighbors`

Účel:
- vypíše známé sousedy

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:256)

Použití:
- první kontrola, jestli repeater skutečně někoho “vidí”

### `neighbor.remove <pubkeyhex>`

Účel:
- odstraní souseda ručně

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:258)

Použití:
- servisní zásah při testování neighbor tabulky

### `stats-core`

Účel:
- vypíše základní runtime statistiky uzlu

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:458)

### `stats-radio`

Účel:
- vypíše rádiové statistiky

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:456)

Použití:
- důležité při ladění RSSI, rušení a rádiové kvality

### `stats-packets`

Účel:
- vypíše packet statistiky

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:454)

Použití:
- klíčové pro ověření, jestli repeater skutečně přeposílá provoz

### `clear stats`

Účel:
- vynuluje statistiky

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:289)

Použití:
- vhodné před měřením nebo srovnávacím testem

## Power management a dostupnost repeatru

### `powersaving`

Účel:
- ukáže, jestli je power saving zapnutý

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:436)

### `powersaving on`

Účel:
- zapne power saving

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:428)

### `powersaving off`

Účel:
- vypne power saving

Implementace:
- [src/helpers/CommonCLI.cpp](/home/codex/MeshCore/src/helpers/CommonCLI.cpp:432)

Praktická poznámka:
- pro stacionární repeater obvykle dává smysl mít power saving vypnutý, pokud by omezoval kontinuitu provozu

## Pomocné příkazy pro správu repeater uzlu

### `clock`

Účel:
- zobrazí aktuální UTC čas

### `clock sync`

Účel:
- synchronizuje hodiny proti timestampu odesílatele

### `time <epoch>`

Účel:
- nastaví RTC čas

Proč je to důležité:
- timestamps a některé síťové mechanismy se chovají lépe při korektním čase

### `ver`

Účel:
- zobrazí verzi firmware a datum buildu

### `board`

Účel:
- vrátí jméno hardware platformy

### `get public.key`

Účel:
- vrátí veřejný klíč uzlu

Použití:
- užitečné při identifikaci konkrétního repeatru v síti

### `get role`

Účel:
- vrátí roli uzlu podle konkrétního buildu

### `reboot`

Účel:
- restart zařízení

### `poweroff` / `shutdown`

Účel:
- vypnutí zařízení

## Praktické scénáře

### Minimální aktivace repeatru

```text
set repeat on
set tx 22
set flood.max 6
set loop.detect moderate
```

### Kontrola stavu po změně

```text
get repeat
get tx
get flood.max
get loop.detect
stats-radio
stats-packets
neighbors
```

### Bezpečnější omezení floodingu podle regionů

```text
region put sklad
region denyf sklad
region list denied
region save
```

### Dočasný rádiový test bez trvalé změny

```text
tempradio 869.525 125.0 11 8 15
stats-radio
stats-packets
```

## Doporučení pro T114 repeater

Pro `Heltec T114` v roli repeatru má v praxi smysl sledovat hlavně:

- `set repeat on`
- `set tx <dbm>`
- `set flood.max <n>`
- `set loop.detect moderate|strict`
- `set radio ...`
- `stats-radio`
- `stats-packets`
- `neighbors`

Pokud je cílem stabilní stacionární uzel:

- zvážit `powersaving off`
- držet rozumný `flood.max`
- používat region pravidla tam, kde hrozí zbytečné šíření floodu

## Co v dokumentu záměrně není

Není zde:

- shell a git příkazy
- interní binární app protokol `companion_radio`
- příkazy ze `simple_secure_chat`, které nesouvisejí s repeater provozem

Tyto příkazy existují v repu také, ale nejsou primárním nástrojem pro správu repeatru.
