# Backup Kontext

Nize je souhrnny textovy kontext k zaloze aktualni prace na `MeshCore`.

## Repo a vetve

Repo: `git@github.com:norman67cz/MeshCore.git`  
Lokalni cesta: [MeshCore](/home/codex/MeshCore)  
Puvodni pracovni vetev pro T114/QSPI upravy: `t114_big_ram`  
Aktualni pracovni vetev: `t114_pwm_off`

## Hlavni cil dosavadnich uprav

Byla resena podpora `Heltec T114` pro vetsi vyuziti externi `QSPI flash`, navyseni poctu kontaktu v `companion_radio`, analyza CLI prikazu a nove i vypnuti `NRF52_POWER_MANAGEMENT` pro T114.

## Dulezite technicke zavery

`Heltec T114` pouziva `nRF52840`, fyzicka SRAM je `256 KB`, v tomto buildu je pro aplikaci dostupnych `235520 B`.  
Externi rozsirená pamet na T114 je `QSPI flash`, ne externi RAM.  
`OFFLINE_QUEUE_SIZE` pouziva interni SRAM, ne QSPI flash.  
Puvodni problem s nebootujicim T114 po zapnuti `QSPIFLASH=1` nebyl v samotne QSPI flash funkci ani v `MAX_CONTACTS`, ale v chybejici QSPI pin mape v T114 variante.

## Root cause a oprava QSPI boot problemu

V [variants/heltec_t114/variant.h](/home/codex/MeshCore/variants/heltec_t114/variant.h) byly doplneny chybejici `PIN_QSPI_*` definice.  
Doplnene piny:

- `PIN_QSPI_SCK=46`
- `PIN_QSPI_CS=47`
- `PIN_QSPI_IO0=44`
- `PIN_QSPI_IO1=45`
- `PIN_QSPI_IO2=7`
- `PIN_QSPI_IO3=5`

Po teto oprave T114 s `QSPIFLASH=1` podle testu uz normalne bootuje.

## QSPI a companion buildy

`-D QSPIFLASH=1` bylo zapnuto pro vsechny ctyri T114 `companion_radio` envy v [variants/heltec_t114/platformio.ini](/home/codex/MeshCore/variants/heltec_t114/platformio.ini):

- `Heltec_t114_without_display_companion_radio_ble`
- `Heltec_t114_without_display_companion_radio_usb`
- `Heltec_t114_companion_radio_ble`
- `Heltec_t114_companion_radio_usb`

## Pocet kontaktu a kanalu

Pro T114 `companion_radio` buildy bylo nastaveno:

- `MAX_CONTACTS=600`
- `MAX_GROUP_CHANNELS=40`

Dulezita poznamka:

- closed-source app neumi stary `DEVICE_INFO` protokol spravne reprezentovat `600`
- firmware interne opravdu pouziva `600`
- stara appka typicky ukazuje `510`

## Uprava reportovani MAX_CONTACTS

V `companion_radio` bylo upraveno `RESP_CODE_DEVICE_INFO`, aby pro novejsi klienty umelo posilat `MAX_CONTACTS` jako `uint16_t`.  
Byl zvysen `FIRMWARE_VER_CODE` na `12`.  
Starsi klienti zustavaji na legacy formatu a mohou zobrazovat jen `510`.

Relevantni soubory:

- [examples/companion_radio/MyMesh.h](/home/codex/MeshCore/examples/companion_radio/MyMesh.h)
- [examples/companion_radio/MyMesh.cpp](/home/codex/MeshCore/examples/companion_radio/MyMesh.cpp)

## CLI dokumentace

Do repa byl pridan dokument [CLI_COMMANDS.md](/home/codex/MeshCore/CLI_COMMANDS.md) s ceskym popisem firmware CLI, hlavne prikazu relevantnich pro repeater a provozni spravu.

## Telemetrie

Textove CLI nema samostatny `telemetry` command.  
Pres textove CLI lze ovlivnovat hlavne GPS/sensor cast:

- `sensor get <key>`
- `sensor set <key> <value>`
- `sensor list`
- `gps on`
- `gps off`
- `gps sync`
- `gps setloc`
- `gps advert`
- `gps`

Napriklad `sensor set gps_interval <sekundy>` upravuje periodu GPS cteni.  
Telemetry mode bity pro `companion_radio` (`base/loc/env`) se nenastavuji textovym CLI, ale jen pres binarni app protokol.

## Nova vetev t114_pwm_off

Z vetve `t114_big_ram` byla vytvorena nova vetev `t114_pwm_off`.

Na teto vetvi byla provedena jedina funkcni zmena:

- odstraneno `-D NRF52_POWER_MANAGEMENT` ze spolecne T114 konfigurace v [variants/heltec_t114/platformio.ini](/home/codex/MeshCore/variants/heltec_t114/platformio.ini)

Tim je `NRF52_POWER_MANAGEMENT` vypnute pro vsechny `Heltec_t114*` buildy.

Commit na teto vetvi:

- `b76351d2` Disable NRF52 power management for Heltec T114

Vetev `t114_pwm_off` je pushnuta na `origin`.

## Overeny build na vetvi t114_pwm_off

Byl uspesne postaven:

- `Heltec_t114_without_display_room_server`

Vysledek:

- RAM: `33932 / 235520` B (`14.4 %`)
- Flash: `423364 / 815104` B (`51.9 %`)

Artefakty na build serveru:

- `/home/codex/meshtastic-builds/MeshCore/.pio/build/Heltec_t114_without_display_room_server/firmware.uf2`
- `/home/codex/meshtastic-builds/MeshCore/.pio/build/Heltec_t114_without_display_room_server/firmware.zip`

## Build server

Buildy se delaly na serveru `10.5.0.146` pres Docker.  
Repo na serveru je v:

- `/home/codex/meshtastic-builds/MeshCore`

## Doporuceny stav pro pokracovani

Pokud se bude pokracovat v PWM/power management diagnostice, vychozi vetev je:

- `t114_pwm_off`

Pokud se bude pokracovat v QSPI/contact kapacite a `companion_radio`, vychozi vetev je:

- `t114_big_ram`
