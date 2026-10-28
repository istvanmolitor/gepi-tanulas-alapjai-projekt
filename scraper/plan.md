# Scraper terv — saját, többforrású adatgyűjtő keretrendszer

Ez a dokumentum a `plan.md`-ben ("A) Saját scraping" opció) meghozott döntés
technikai kidolgozása: hogyan gyűjtjük össze a lakóingatlan-hirdetéseket, milyen
problémákkal kell számolni, és milyen megoldási lehetőségek vannak. Ebben a
mappában lesz majd a tényleges scraper kód is.

**Fontos irányelv:** nem egy konkrét honlaphoz (pl. ingatlan.com) kötött
scriptet írunk, hanem egy **saját, forrásfüggetlen scraper keretrendszert**,
amibe forrásonként (site-onként) egy-egy kis adapter modul köthető be. Így a
projekt nem válik egyetlen honlap HTML-struktúrájától függővé, és könnyen
bővíthető újabb hirdetési oldalakkal, ha az ingatlan.com önmagában nem
elegendő (mennyiség, lefedettség, vagy blokkolás miatt).

**Hatókör-elhatárolás (fontos):** a scraper **kizárólag** a következőkért
felel: hirdetés-linkek felderítése és **regisztrálása** (nyilvántartásba
vétele), a letöltések **ütemezése**, valamint a lista (kereső/találati) és
a részletes (egy hirdetéshez tartozó) oldalak nyers **HTML-jének letöltése
és nyilvántartása** (melyik link mikor és milyen állapotban lett
letöltve). A scraper **nem** parse-olja ki az ingatlan jellemzőit (ár,
méret, szobaszám stb.) a HTML-ből, és nem állít elő strukturált/CSV
adatot — ez a `preprocessing` fázis feladata, lásd `preprocessing/plan.md`.
Ez a szétválasztás azért fontos, mert ha a mezőkinyerő logika hibás vagy
változik, nem kell újra letölteni semmit: elég a már letöltött,
nyilvántartott HTML-eken újrafuttatni a preprocessing lépést.

**Lista vs. részletes HTML (fontos):** a két oldaltípus HTML-struktúrája és
tartalma eltérő (a lista oldal sok hirdetés rövid előnézetét listázza, a
részletes oldal egyetlen hirdetés teljes adatát tartalmazza), ezért a
scraper a kettőt **külön mappában** tárolja, és a nyilvántartásban is
külön kezeli (lásd 4. pont). Ennek oka, hogy a `preprocessing` a kétféle
HTML-ből **más-más mezőket** nyer ki (lásd `preprocessing/plan.md` 1. és
3. pont) — a lista oldal csak előnézeti/kiegészítő adatot ad, a részletes
oldal a modellezéshez szükséges teljes mezőkészletet.

## 1. Cél és hatókör

- **Architektúra:** egy közös "core" (letöltés, retry, mentés, dedup,
  pipeline-vezérlés) + ehhez csatlakozó, forrásonkénti **adapterek**, amelyek
  egy közös interfészt implementálnak (lista oldalak bejárása, egy hirdetés
  parse-olása egységes sémára). Lásd 3. és 6. pont.
- **Induló forrás:** ingatlan.com — ezen validáljuk a keretrendszert,
  kiindulópontként a főtervben linkelt referencia-projektek
  ([Mukedlii/ingatlancom-scraper](https://github.com/Mukedlii/ingatlancom-scraper),
  [wendig/ingatlan](https://github.com/wendig/ingatlan)) szerkezetének
  átnézésével.
- **További jelölt források (bővítésre, felmérés után):** más magyar
  ingatlanhirdetési oldalak, pl. otthontcentrum.hu, ingatlanbazar.hu,
  ingatlanok.hu, duna_house.hu stb. — mindegyiknél külön meg kell vizsgálni a
  robots.txt/ToS-t és a HTML-struktúrát, mielőtt adaptert írunk hozzá; a
  listát csak jelöltnek tekintjük, nem elköteleződésnek.
- **Szűkítés (javasolt induló scope):** eladó lakások, Budapest. Ha marad idő,
  bővíthető vidéki városokra, kiadó hirdetésekre, és/vagy további forrásokra.
- **Mennyiségi cél:** néhány ezer–kb. 10-15 ezer hirdetés forrásonként,
  hasonlóan a referenciaprojektekhez (~12 000 hirdetés volt összehasonlítható
  méret egyetlen forrásból).

## 2. Jogi / etikai kérdések — mielőtt kódolni kezdünk

- **robots.txt és ToS átnézése forrásonként**, mielőtt egy adott oldalra éles
  scrapingbe kezdünk — ez minden új forrás bevonásának előfeltétele, nem
  egyszeri lépés. Ha egy adott oldal robots.txt-je kifejezetten tiltja a
  releváns útvonalakat, azt a forrást nem vonjuk be.
- **Oktatói elfogadottság:** tisztázni az oktatóval/konzulenssel, hogy
  elfogadható-e scraped adat (akár több forrásból összesítve), és milyen
  hivatkozási/dokumentálási elvárás van (forrás, gyűjtés dátuma, módszertan
  rögzítése a leadásban) — forrásonként fel kell tüntetni, melyik rekord
  honnan jött (lásd 4. pont "forrás" mező).
- **Személyes adatok:** hirdető neve/telefonszáma kerülendő a datasetből —
  csak az ingatlan jellemzőire van szükség, a hirdető-szintű személyes adatot
  nem gyűjtjük/tároljuk, forrástól függetlenül.
- **Terhelés:** kérés-gyakoriság korlátozása (delay, egyidejű kapcsolatok
  száma) **forrásonként külön konfigurálva**, hogy semelyik cél szervert ne
  terheljük túl, és a saját IP-nk se kerüljön blokkolásra.

## 3. Technikai megközelítés

### 3.1 Adapter-alapú architektúra
A keretrendszer két rétegre bomlik:

- **Core (forrásfüggetlen):** HTTP lekérés + retry/backoff, rate limiting,
  nyers HTML mentése **típus szerint külön mappába** (lista vs. részletes,
  lásd 6. pont), link-nyilvántartás vezetése és deduplikáció (4. pont),
  letöltés-ütemezés (melyik linket/lista oldalt mikor töltjük le/töltjük le
  újra), futás-riport/logolás, pipeline-vezérlés (melyik forrást mikor
  futtatjuk), valamint a forrásonkénti adat **zip-be exportálása és
  zip-ből importálása** (lásd 4.2 pont). Ez a réteg semmit nem tud az
  ingatlan.com-ról vagy bármely konkrét honlapról, és nem nyúl a hirdetés
  tartalmi mezőihez (ár, méret stb.) — azokat nem ismeri, csak a HTML-t
  tölti le és tartja nyilván.
- **Site adapter (forrásspecifikus):** minden bevont honlaphoz egy kis modul,
  amely egy közös interfészt implementál, két függvénnyel/metódussal:
  1. `start_urls(...)` — a kereséshez/szűréshez tartozó induló lista oldal
     URL(ek) előállítása (pl. Budapest, eladó lakás szűrővel).
  2. `parse_list_page(html) -> (hirdetés_urlök, következő_oldal_url | None)`
     — egy már **letöltött** lista oldal HTML-jéből kiolvassa a rajta
     szereplő hirdetés-linkeket/ID-ket és a következő lapozási oldal URL-jét
     (vagy `None`-t, ha ez az utolsó oldal).

  A lista oldal HTML-jének **letöltése és mentése** forrásfüggetlen,
  core-szintű feladat (csak a `start_urls`/`következő_oldal_url` alapján);
  az adapter csak a már letöltött HTML **parse-olásáért** felel
  (`parse_list_page`). A hirdetés-linkek a `parse_list_page` eredménye
  alapján kerülnek regisztrálásra a nyilvántartásban (lásd 4. pont). A
  részletes (hirdetés-) oldal HTML-jének letöltése ugyanígy core-szintű
  feladat a regisztrált linkek alapján — nincs szükség adapter-szintű
  "parse_listing" lépésre, mert a scraper nem nyeri ki a hirdetés mezőit,
  csak letölti és elmenti a HTML-t (típus szerint külön mappába).

  Az adapter dönt arról is, hogy az adott oldalhoz `requests`/`httpx` +
  `BeautifulSoup` elég (statikus HTML), vagy `Playwright`/`Selenium` szükséges
  (JS-renderelt/SPA tartalom), illetve hogy van-e közvetlenül hívható belső
  API endpoint (ha van, az általában stabilabb és gyorsabb, mint a renderelt
  HTML parse-olása).

Ez a felbontás teszi lehetővé, hogy egy új forrás bevonása ne a core
logika módosítását, csak egy új adapter megírását jelentse.

### 3.2 Crawling-folyamat (minden forrásra ugyanaz a séma a core felől nézve)
1. **Lista oldal letöltése:** a core letölti az adapter `start_urls(...)`
   (vagy egy korábbi lapozásból kapott "következő oldal") által megadott
   lista/kereső oldal HTML-jét, és elmenti a `html/list/` mappába (lásd 6.
   pont) — ez még forrásfüggetlen lépés, mezőkinyerés nélkül.
2. **Lista oldal parse-olása:** az adapter `parse_list_page(html)`
   függvénye a letöltött HTML-ből kiolvassa a rajta szereplő
   hirdetés-linkeket/ID-ket, valamint a következő lapozási oldal URL-jét
   (ha van) — ez a lépés forrásspecifikus.
3. **Linkek regisztrálása:** az összegyűjtött hirdetés-linkek bekerülnek a
   nyilvántartásba (4. pont sémája szerint) `felfedezve` állapottal, ha még
   nem szerepelnek benne (dedup `(forrás, hirdetés ID)` alapján); a lista
   oldal letöltése is bekerül a saját, lista-szintű nyilvántartásába (4.
   pont).
4. **Lapozás:** ha a parse-olás adott egy "következő oldal" URL-t, az 1-3.
   lépés megismétlődik rá; ha nincs több oldal, a lista-bejárás ennél a
   forrásnál/szűrésnél befejeződik.
5. **Letöltés ütemezése (részletes oldalak):** a nyilvántartásból a
   letöltendő hirdetés-linkek kiválasztása (még nem letöltött, vagy
   frissítésre esedékes), rate limit/politeness-szabályok szerint sorba
   állítva — core-szintű, nem forrásspecifikus logika.
6. **Részletes HTML letöltése** az ütemezett linkekhez — a core
   HTTP/retry rétegét használja, forrásfüggetlen (nincs mezőkinyerés, csak
   nyers válasz).
7. **Mentés és nyilvántartás-frissítés:** a nyers HTML forrásonként és
   típusonként (lista/részletes) külön mappába kerül (visszakereshetőség/
   debug miatt, lásd 6. pont), a nyilvántartásban pedig frissül az állapot
   (`letöltve`/`hiba`), a letöltés dátuma, a mentett fájl elérési útja és a
   HTTP státuszkód.

### 3.3 Inkrementális gyűjtés
- Hirdetés-azonosító (ID) alapú deduplikáció **forráson belül**, hogy
  ismételt futásnál ne gyűjtsünk duplán; az ID-t a forrás nevével együtt
  tároljuk (pl. `ingatlan.com:12345678`), mert különböző forrásoknál az ID-k
  ütközhetnek.
- "Utoljára látott" időbélyeg mezőnkénti nyilvántartása, hogy követhető legyen
  mely hirdetések tűntek el (eladva/törölve) — ez később hasznos lehet az
  elemzésnél is (pl. mennyi idő alatt kelt el egy hirdetés, ha ez a cél).

### 3.4 Parancssori felület (CLI)

A scraper három fő paranccsal vezérelhető (`cli.py`, lásd 6. pont),
mindegyik forrás-paraméterrel (`--source`), hogy forrásonként külön
futtatható legyen:

1. **`run` — a scrape-elés futtatása:**
   `python -m scraper run --source ingatlan_com`
   A `core/pipeline.py`-t hívja: lefuttatja a 3.2 pontban leírt teljes
   crawling-folyamatot (lista oldalak bejárása + lapozás, hirdetés-linkek
   regisztrálása, majd a még le nem töltött/esedékes részletes oldalak
   letöltése) egy adott forrásra. Megszakítható és újraindítható
   (checkpoint-szerűen folytatja ott, ahol a nyilvántartás állapota
   szerint abbamaradt, lásd 5.6 pont). Hasznos opciók (kezdetben
   opcionálisak, bővíthetők): `--max-pages` (hány lista oldalig menjen),
   `--only-list` / `--only-detail` (csak a linkfelderítés, vagy csak a már
   regisztrált linkek letöltése fusson, pl. hibakereséshez vagy
   szakaszolt futtatáshoz).
2. **`export` — forrás adatának exportálása zip-be:**
   `python -m scraper export --source ingatlan_com --out exports/ingatlan_com_2026-10-27.zip`
   A `core/archive.py` export-funkcióját hívja (4.2 pont): az adott forrás
   teljes `data/<forrás>/` mappáját (nyilvántartás CSV-k + `html/list/` és
   `html/detail/` tartalma) egy zip fájlba tömöríti, a csapattagok közötti
   megosztáshoz.
3. **`import` — korábban exportált zip importálása:**
   `python -m scraper import --source ingatlan_com --in exports/ingatlan_com_2026-10-27.zip`
   A `core/archive.py` import-funkcióját hívja (4.2 pont): a zipben lévő
   nyilvántartást és HTML-eket a 4.2 pontban leírt összefésülési
   szabályok szerint beilleszti a cél forrás mappájába (nem felülírja,
   nem duplikálja); biztonságosan ismételhető művelet.

Mindhárom parancs csak egy vékony `cli.py` réteg — a tényleges logikát a
`core/pipeline.py` (run), illetve `core/archive.py` (export/import) adja,
hogy a parancsok maguk tesztelhetők/hívhatók maradjanak kódból is, ne csak
a parancssorból.

## 4. Tervezett nyilvántartási séma (link-szintű rekord, forrásfüggetlen)

Ez **nem** az ingatlan-adat sémája (azt lásd `preprocessing/plan.md`), hanem
a scraper saját link-nyilvántartása: minden felfedezett hirdetés-linkhez egy
sor, ami a letöltés állapotát követi.

**Tárolási forma:** a nyilvántartás **forrásonként külön CSV fájlban**
(nem közös, nem SQLite) kerül tárolásra, a forrás saját mappájában, a
letöltött HTML-ekkel együtt (lásd 6. pont mappastruktúrája) — így egy
forrás teljes adata (nyilvántartás + HTML-ek) egyetlen mappába zárva,
önállóan kezelhető (pl. zip-elhető, lásd 4.2 pont).

| Mező | Megjegyzés |
|---|---|
| forrás | melyik honlapról származik a link (pl. `ingatlan.com`) |
| hirdetés ID | forráson belül egyedi azonosító, deduplikációhoz |
| hirdetés URL | a letöltendő/letöltött oldal címe |
| állapot | `felfedezve` / `letöltve` / `hiba` / `eltávolítva` |
| felfedezés dátuma | mikor került be a nyilvántartásba (lista oldal bejárásakor) |
| letöltés dátuma | mikor történt a tényleges HTML letöltés (ha megtörtént) |
| nyers HTML fájl elérési útja | hova mentettük a letöltött HTML-t |
| HTTP státuszkód | a letöltési kísérlet eredménye |
| retry szám | hányszor próbáltuk újra sikertelen letöltés esetén |
| utoljára látva (lista oldalon) | később jelzi, ha egy hirdetés eltűnt (eladva/törölve) |

A pontos mezőkészlet a tényleges HTML-struktúra megismerése után finomodik.

### 4.1 Lista oldal nyilvántartás (külön séma)

A fenti séma a **részletes (hirdetés-) oldalakra** vonatkozik. A **lista
(kereső/találati) oldalak** letöltését egy külön, lista-szintű
nyilvántartás követi (forrásonként külön CSV, pl. `list_pages.csv`, lásd
6. pont), mert ezeknek nincs hirdetés-azonosítójuk, hanem a
keresési/lapozási paraméterek (pl. szűrés + oldalszám) azonosítják őket:

| Mező | Megjegyzés |
|---|---|
| forrás | melyik honlapról származik a lista oldal |
| lista URL | a letöltött kereső/találati oldal címe (szűréssel, lapozással) |
| oldalszám | hányadik lapozási oldal (1-től) |
| állapot | `letöltve` / `hiba` |
| letöltés dátuma | mikor történt a letöltés |
| nyers HTML fájl elérési útja | `html/list/` alatti fájl (lásd 6. pont) |
| HTTP státuszkód | a letöltési kísérlet eredménye |
| retry szám | hányszor próbáltuk újra sikertelen letöltés esetén |
| talált hirdetés-linkek száma | a `parse_list_page` hány linket adott vissza (diagnosztikához) |

### 4.2 Export/import zip-ben (csapatmunka, adatcsere)

A `data/<forrás>/` mappa (nyilvántartás CSV + letöltött HTML-ek, lásd 6.
pont) nem kerül verziókezelésbe (nagy méretű nyers adat, lásd 6. pont), így
a csapattagok közötti megosztáshoz és a géphatár-független mentéshez
szükséges egy explicit export/import mechanizmus:

- **Export:** egy adott forrás teljes mappáját (nyilvántartás CSV + az
  összes letöltött HTML fájl) egyetlen `.zip` fájlba tömöríti. Az export
  nem módosítja a forrás mappáját, csak egy pillanatkép-másolatot készít
  belőle.
- **Import:** egy korábban exportált `.zip` fájlt fogad, és a benne lévő
  nyilvántartást + HTML-eket **összefésüli** a cél forrás mappájának
  meglévő állapotával (nem felülírja, nem duplikálja):
  - **HTML fájlok:** amelyik hirdetés-ID-hoz még nincs letöltött HTML a
    cél mappában, azt egyszerűen átmásolja; amelyikhez már van, azt
    érintetlenül hagyja (a már meglévő HTML-t nem írja felül, mert a
    letöltés időpontjától függetlenül ugyanazt a tartalmat várjuk).
  - **Nyilvántartás CSV:** soronkénti összefésülés `(forrás, hirdetés ID)`
    kulcs alapján. Ha egy kulcs csak az egyik oldalon létezik, átvétel
    változatlanul. Ha mindkét oldalon létezik, a "teljesebb" állapotú sor
    nyer: `letöltve` > `hiba` > `felfedezve`; azonos állapot esetén a
    későbbi (nagyobb) dátummal rendelkező sor marad meg.
  - Az importálás így akkor is biztonságosan megismételhető, ha ugyanazt
    a zipet kétszer importáljuk, vagy ha két csapattag részben
    átfedő adatot gyűjtött.
- Ez a mechanizmus csak a scraper adatára (link-nyilvántartás + nyers
  HTML) vonatkozik — a `preprocessing` és a modellezési fázis kimenete
  nem ezen keresztül, hanem a normál git workflow-n (kis méretű CSV-k)
  keresztül kerül megosztásra.

## 5. Várható problémák és megoldási lehetőségek

### 5.1 Anti-bot védelem
- **Probléma:** IP-alapú rate limiting, esetleges captcha, user-agent/fejléc
  alapú szűrés.
- **Lehetőségek:** kérések közötti véletlenszerű delay, reális böngésző
  fejlécek, kis párhuzamosság (nem agresszív crawling), retry/backoff logika
  hibás válaszra. Ha captcha jelenik meg, az automatizálás helyett inkább a
  scraping ütemének csökkentése a megoldás, nem a captcha megkerülése.

### 5.2 A honlap struktúrájának változása
- **Probléma:** a HTML szerkezet (CSS selectorok, class nevek) idővel
  változhat, elromlik a parser.
- **Lehetőségek:** parse-oló függvények elkülönítése a letöltési logikától,
  hogy ha a struktúra változik, csak a parser-t kelljen javítani; nyers HTML
  mentése, hogy ne kelljen újra letölteni a hibás parse javításához.

### 5.3 Hiányos / inkonzisztens adatok
- Ez a scraper szintjén nem releváns probléma, mert a scraper nem nyeri ki a
  hirdetés mezőit (ár, méret stb.) — csak a nyers HTML-t tölti le és
  tárolja. A mezők formátumának eltérése, hiányzó értékek kezelése a
  `preprocessing` fázis feladata, lásd `preprocessing/plan.md` 4. pont.

### 5.4 Duplikált / frissülő hirdetések
- **Probléma:** ugyanazon ingatlan többször is meghirdethető, vagy egy
  hirdetés frissülhet (ár, szöveg változik) anélkül, hogy új ID-t kapna.
- **Lehetőségek:** ID alapú dedup az elsődleges védelem; opcionálisan
  cím+méret+szobaszám alapú közelítő egyezés-keresés későbbi tisztítási
  lépésként (EDA fázisban), nem feltétlenül a scraper részeként.

### 5.5 Törölt / lejárt hirdetések, hibás válaszok
- **Probléma:** 404-es vagy átirányított linkek, időközben eltávolított
  hirdetések.
- **Lehetőségek:** hibakezelés és logolás minden sikertelen lekérésről,
  összesítő futás-riport (hány sikeres/sikertelen kérés), hogy lássuk a
  scraper megbízhatóságát.

### 5.6 Teljesítmény és tárolás
- **Probléma:** több ezer hirdetés egyenkénti lekérése időigényes lehet.
- **Lehetőségek:** checkpoint/resume mechanizmus (ha megszakad a futás, nem
  kell elölről kezdeni), ütemezett, több kisebb futásra bontott gyűjtés
  (pl. naponta egy adag) a politeness és a megszakítás-tolerancia miatt.

### 5.7 Több forrás összehangolása
- **Probléma:** ugyanaz az ingatlan több oldalon is meghirdethető
  (cross-source duplikáció — ezt linkszinten nem lehet biztosan felismerni,
  csak a tartalom alapján); minden új adapter karbantartási terhet jelent
  (site-redesign esetén azt is javítani kell). Az eltérő mezőnevek,
  mértékegységek és kategóriaértékek kezelése **nem** a scraper, hanem a
  `preprocessing` fázis problémája (lásd `preprocessing/plan.md` 4.4 pont),
  mert a scraper nem nyeri ki ezeket a mezőket.
- **Lehetőségek:** szigorú adapter-interfész kikényszerítése (3.1 pont),
  hogy az eltérések az adapter rétegben maradjanak; a `forrás` mező
  megtartása minden nyilvántartott linknél, hogy forrásonként is lehessen
  vizsgálni az adatot; cross-source duplikáció kezelése külön, később az
  EDA/preprocessing fázisban, nem a scraper részeként; **induláskor egy
  forrás (ingatlan.com) is elég** — csak akkor érdemes második adaptert
  írni, ha a mennyiség/lefedettség ezt indokolja, hogy a karbantartási teher
  arányban álljon a haszonnal.

### 5.8 Időbeli torzítás
- **Probléma:** a gyűjtés időpontja befolyásolja az árakat (piaci mozgás).
- **Lehetőségek:** gyűjtés dátumának mentése minden rekordnál, és ha a
  gyűjtés több hétig tart, ennek dokumentálása a végső beadandóban mint
  ismert limitáció (ez már a főtervben is szerepel).

## 6. Tervezett mappa- és kódstruktúra (a kód elkészülésekor)

```
scraper/
  plan.md                  <- ez a dokumentum
  requirements.txt          <- requests/httpx, beautifulsoup4, (playwright, ha kell)
  __main__.py                <- `python -m scraper ...` belépési pont, a cli.py-t hívja
  cli.py                       <- parancssori felület: run / export / import (3.4 pont)
  core/
    config.py                <- globális + forrásonkénti beállítások (delay, rate limit)
    fetch.py                  <- HTTP lekérés, retry/backoff, nyers HTML mentése
    registry.py                 <- link-nyilvántartás séma, dedup, állapotkezelés (4. pont)
    scheduler.py                  <- letöltés-ütemezés: mely linkeket töltsük le/mikor
    archive.py                      <- zip export/import + összefésülés (4.2 pont)
    pipeline.py                       <- adapter-független vezérlés: adapter kiválasztása,
                                         futtatása, futás-riport
  sites/
    base.py                   <- adapter interfész (absztrakt osztály/protokoll):
                                  start_urls(), parse_list_page()
    ingatlan_com.py             <- első adapter: ingatlan.com-specifikus logika
    # további_forras.py         <- csak akkor, ha egy új forrás bevonása indokolt
  data/
    ingatlan_com/               <- forrásonként egy mappa (séparálva a többi forrástól)
      registry.csv                 <- részletes (hirdetés-) link-nyilvántartás (4. pont sémája)
      list_pages.csv                <- lista oldal nyilvántartás (4.1 pont sémája)
      html/
        list/                         <- letöltött lista/kereső oldalak nyers HTML-je
          <oldalszám_vagy_hash>.html
        detail/                       <- letöltött hirdetés-részletező oldalak nyers HTML-je
          <hirdetés_id>.html
    # további_forras/            <- csak akkor, ha egy új forrás bevonása indokolt
  exports/
    ingatlan_com_2026-10-27.zip   <- export (4.2 pont) kimenete; import ugyanilyen zipet vár
```

A `data/` (minden forrásmappa: nyilvántartás CSV + HTML-ek) és az
`exports/` is nagy/sokat változó, ezért `.gitignore`-ba kerülnek —
verziókezelésben csak a kód és a tervdokumentumok maradnak. A csapaton
belüli megosztás a 4.2 pontban leírt zip export/import mechanizmuson
keresztül történik, nem git commiton keresztül. A HTML-ből előálló
strukturált adat (CSV) a `preprocessing/data/` alatt jön létre, nem itt.

## 7. Következő lépések

1. **Core + adapter-interfész megtervezése és megírása elsőként** — mielőtt
   bármelyik konkrét honlaphoz kódolnánk, hogy a keretrendszer valóban
   forrásfüggetlen maradjon.
2. robots.txt / ToS átnézése az ingatlan.com-on, oktatói visszaigazolás
   beszerzése.
3. Kísérleti futás: 1-2 lista oldal + néhány hirdetés oldal manuális
   megvizsgálása (statikus vs. JS-renderelt tartalom eldöntése).
4. **Első adapter (ingatlan.com) megírása** — csak a `start_urls` és
   `parse_list_page` implementálása —, kis mintán (pl. 50-100 link)
   validálva.
5. Nyilvántartás-séma és ütemező finomítása a prototípus tapasztalatai
   alapján.
6. `archive.py` (export/import, 4.2 pont) megírása és tesztelése: export egy
   kis mintából, majd import egy másik (üres vagy részben feltöltött)
   mappába, ellenőrizve az összefésülés helyességét.
7. **`cli.py` megírása** (3.4 pont): `run`, `export`, `import` parancsok,
   amelyek a már meglévő `pipeline.py`/`archive.py` funkciókat hívják —
   ellenőrizve mindhárom parancsot a kis mintán (pl. `run --max-pages 2`,
   majd `export`, majd egy üres mappába `import`).
8. Teljes, ütemezett/korlátozott gyűjtés elindítása Budapestre, eladó
   lakásokra, egyetlen forrásból a `run` parancs segítségével: linkek
   regisztrálása és a hirdetés-oldalak HTML-jének letöltése.
9. **Csak ha indokolt** (mennyiség/lefedettség hiánya): további forrás(ok)
   felmérése és új adapter írása a meglévő interfészre, az 5.7 pontban
   leírt szempontok figyelembevételével.
