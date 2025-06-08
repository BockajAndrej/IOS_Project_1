# IOS_Project_1 - Skript na analýzu transakčných logov

Tento skript v jazyku Bash slúži na spracovanie a analýzu transakčných logov. Umožňuje filtrovať záznamy podľa používateľa, časového rozsahu a meny, a následne vykonať rôzne typy výpisov alebo sumarizácií.

Podporuje spracovanie nekomprimovaných log súborov aj log súborov komprimovaných pomocou gzip (`.gz`).

## Použitie

```bash
xtf [-h|--help] [FILTR] [PŘÍKAZ] UŽIVATEL LOG [LOG2 [...]]
```

Kde:
*   `xtf`: Názov skriptu (predpokladá sa, že súbor je spustiteľný a pomenovaný `xtf`).
*   `[-h|--help]`: Zobrazenie nápovedy.
*   `[FILTR]`: Voliteľný filter záznamov (jedna alebo viacero voľieb `-a`, `-b`, `-c`).
*   `[PŘÍKAZ]`: Voliteľný príkaz určujúci typ výstupu/analýzy (predvolené je `list`). Môže sa zadať len jeden príkaz.
*   `UŽIVATEL`: Meno používateľa, pre ktorého sa záznamy analyzujú. Musí byť zadané pred log súbormi.
*   `LOG [LOG2 [...]]`: Jeden alebo viacero log súborov na spracovanie.

Voľby a príkazy sa môžu v príkazovom riadku vyskytovať v rôznom poradí s výnimkou toho, že filtre sú spracované pomocou `getopts` na začiatku, a meno používateľa musí byť identifikované pred názvami log súborov.

## Voľby (Filtre)

Voľby určujú, ktoré záznamy sa majú zahrnúť do spracovania. Môžu byť zadané v ľubovoľnom poradí pred používateľom, príkazom a log súbormi.

*   `-a DATETIME`: (after) Zahrnie len záznamy po tomto dátume a čase (výhradne).
    *   `DATETIME` musí byť vo formáte `YYYY-MM-DD HH:MM:SS`.
*   `-b DATETIME`: (before) Zahrnie len záznamy pred týmto dátumom a časom (výhradne).
    *   `DATETIME` musí byť vo formáte `YYYY-MM-DD HH:MM:SS`.
*   `-c CURRENCY`: Zahrnie len záznamy zodpovedajúce danej mene.
    *   `CURRENCY` musí byť trojpísmenový kód (napr. `EUR`, `USD`).
*   `-h` alebo `--help` (ako argument): Zobrazí túto nápovedu a ukončí skript. `-h` môže byť zadané aj ako voľba bez argumentu.

**Poznámka k dátumu/času:** Ak nie sú zadané `-a` ani `-b`, skript použije ako predvolený rozsah od "začiatku času" (`0000-00-00 00:00:00`) do "konca času" (`9999-99-99 99:99:99`).

## Príkazy

Príkazy určujú typ analýzy a formát výstupu. Môže byť zadaný maximálne jeden príkaz. Ak nie je zadaný žiadny príkaz, predvoleným je `list`.

*   `list`: Vypíše všetky záznamy pre daného používateľa, ktoré zodpovedajú zadaným filtrom (čas, mena).
*   `list-currency`: Vypíše zoradený zoznam unikátnych mien, ktoré sa vyskytujú v záznamoch pre daného používateľa, ktoré zodpovedajú časovým filtrom. Filter `-c` sa aplikuje aj na tento zoznam.
*   `status`: Vypíše aktuálny stav účtu používateľa. Záznamy sú zoskupené a zoradené podľa jednotlivých mien. Výstupom je zoznam `MENA : SUMA` pre každú relevantnú menu.
*   `profit`: Vypíše stav účtu používateľa s pripočítaným fiktívnym výnosom. Funguje rovnako ako `status`, ale k pozitívnym zostatkom v každej mene pripočíta percentuálny výnos definovaný premennou prostredia `XTF_PROFIT`.

## Premenné prostredia

*   `XTF_PROFIT`: Definuje percentuálny výnos, ktorý sa pripočíta k pozitívnym zostatkom pri príkaze `profit`. Ak táto premenná nie je nastavená, použije sa predvolená hodnota `20`.

## Štruktúra a fungovanie kódu

1.  **Nastavenie prostredia:** Na začiatku sú nastavené premenné `POSIXLY_CORRECT=yes` a `LC_ALL=C`, `LC_NUMERIC=utf8` pre konzistentné správanie príkazov a triedenie.
2.  **Funkcia `log_divider`:** Táto pomocná funkcia zistí, či log súbor končí na `.gz`. Ak áno, na čítanie použije `zcat` (na dekompresiu), inak použije `cat`. Zabezpečuje, že zvyšok skriptu spracuje log súbor ako štandardný textový stream bez ohľadu na kompresiu.
3.  **Funkcia `help_info`:** Vypíše formátovanú nápovedu k skriptu.
4.  **Predvolená hodnota `XTF_PROFIT`:** Ak premenná prostredia `XTF_PROFIT` nie je nastavená, inicializuje sa na hodnotu `20`.
5.  **Spracovanie volieb (`getopts`):** Pomocou `getopts` sú spracované voliteľné prepínače (`-h`, `-a`, `-b`, `-c`). Pri voľbách `-a` a `-b` je vykonaná kontrola formátu dátumu a času. Pri voľbe `-c` je kontrolovaný formát kódu meny. Nesprávny formát alebo opakované zadanie volieb vedie k chybovému hláseniu a ukončeniu skriptu. Po spracovaní volieb sa pomocou `shift` odstránia zo zoznamu pozičných parametrov.
6.  **Nastavenie predvoleného časového rozsahu:** Ak volby `-a` alebo `-b` neboli zadané, nastaví sa rozsah na "od počiatku do konca času" pomocou fiktívnych hodnôt `0000-00-00 00:00:00` a `9999-99-99 99:99:99`.
7.  **Spracovanie ostatných argumentov:** Zvyšné argumenty sú spracované v cykle. Skript sa snaží identifikovať:
    *   Argument `help` (vypíše nápovedu).
    *   Argument končiaci na `.` (predpokladá sa, že ide o log súbor). Pred spracovaním súboru skontroluje, či už bolo zadané meno používateľa a či súbor existuje. Následne, na základe zadaného príkazu (`com`), spracuje súbor pomocou `log_divider` a `awk` (prípadne s ďalšími nástrojmi ako `grep`, `sort`, `uniq`). Väčšina filtrovania (používateľ, dátum/čas) a agregácie (pre status a profit) sa vykonáva v `awk`.
    *   Iné argumenty (predpokladá sa, že ide o príkaz alebo meno používateľa). Skript najprv skúsi identifikovať argument ako známy príkaz (`list`, `list-currency`, `status`, `profit`). Ak áno, uloží ho do premennej `com`. Ak argument nie je príkaz a meno používateľa (`name`) ešte nebolo zadané, uloží ho ako meno používateľa. Nesprávne argumenty vedú k chybe.
8.  **Ukončenie:** Skript štandardne ukončí s návratovým kódom 0, ak nenastala chyba počas spracovania argumentov, volieb alebo súborov.

## Predpokladaný formát log súborov

Skript očakáva, že log súbory sú textové súbory (alebo gzip komprimované textové súbory), kde každý riadok predstavuje jeden záznam oddelený bodkočiarkou (`;`). Na základe použitia `awk -F ';'` a prístupu k poliam (`$1`, `$2`, `$3`, `$4`) sa predpokladá nasledujúca štruktúra záznamu:

`Používateľ;Dátum Čas;Mena;Suma`

Napríklad: `jozef;2023-10-27 10:30:00;EUR;150.50`

## Závislosti

Skript vyžaduje, aby boli na systéme dostupné nasledujúce štandardné nástroje Unixu:

*   `bash`
*   `awk`
*   `grep`
*   `sort`
*   `uniq`
*   `cat`
*   `zcat` (pre spracovanie .gz súborov)