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
  nyers válasz mentése, deduplikáció, egységes séma validálása, feldolgozott
  adat mentése, futás-riport/logolás, pipeline-vezérlés (melyik forrást mikor
  futtatjuk). Ez a réteg semmit nem tud az ingatlan.com-ról vagy bármely
  konkrét honlapról.
- **Site adapter (forrásspecifikus):** minden bevont honlaphoz egy kis modul,
  amely egy közös interfészt implementál, tipikusan két függvényt/metódust:
  1. `list_listing_urls(...)` — lista/keresési oldalak bejárása, lapozás
     kezelése, hirdetés-linkek/ID-k visszaadása.
  2. `parse_listing(html_or_response) -> record` — egy hirdetés oldal
     feldolgozása az egységes (4. pontban leírt) sémára.

  Az adapter dönt arról is, hogy az adott oldalhoz `requests`/`httpx` +
  `BeautifulSoup` elég (statikus HTML), vagy `Playwright`/`Selenium` szükséges
  (JS-renderelt/SPA tartalom), illetve hogy van-e közvetlenül hívható belső
  API endpoint (ha van, az általában stabilabb és gyorsabb, mint a renderelt
  HTML parse-olása).

Ez a felbontás teszi lehetővé, hogy egy új forrás bevonása ne a core
logika módosítását, csak egy új adapter megírását jelentse.

### 3.2 Crawling-folyamat (minden forrásra ugyanaz a séma a core felől nézve)
1. **Lista oldalak bejárása** (keresési/szűrési URL-ek lapozással) →
   hirdetés-azonosítók/linkek kigyűjtése — az adapter végzi, forrásspecifikus
   logikával.
2. **Részletes hirdetés oldalak lekérése** az összegyűjtött linkekről →
   mezők kiparse-olása — ugyancsak adapter-szintű feladat, de a core
   HTTP/retry rétegét használja.
3. **Normalizálás:** az adapter kimenete a közös sémára illeszkedik, plusz
   egy `forrás` mezőre (pl. `"ingatlan.com"`), hogy a végső, összesített
   adatkészletben mindig visszakereshető legyen, melyik rekord honnan jött.
4. **Mentés** nyers (raw HTML, visszakereshetőség/debug miatt, forrásonként
   külön mappában) és feldolgozott (strukturált, pl. CSV/Parquet/SQLite,
   forrásonként és/vagy összesítve) formában is.

### 3.3 Inkrementális gyűjtés
- Hirdetés-azonosító (ID) alapú deduplikáció **forráson belül**, hogy
  ismételt futásnál ne gyűjtsünk duplán; az ID-t a forrás nevével együtt
  tároljuk (pl. `ingatlan.com:12345678`), mert különböző forrásoknál az ID-k
  ütközhetnek.
- "Utoljára látott" időbélyeg mezőnkénti nyilvántartása, hogy követhető legyen
  mely hirdetések tűntek el (eladva/törölve) — ez később hasznos lehet az
  elemzésnél is (pl. mennyi idő alatt kelt el egy hirdetés, ha ez a cél).

## 4. Tervezett adatséma (hirdetés-szintű rekord, forrásfüggetlen)

Ez a séma a **core/normalizált** réteg kimenete — minden adapternek ide kell
leképeznie a saját forrása mezőit, akkor is, ha az eredeti honlapon más
elnevezéssel/formátummal szerepelnek.

| Mező | Megjegyzés |
|---|---|
| forrás | melyik honlapról származik a rekord (pl. `ingatlan.com`) |
| hirdetés ID | forráson belül egyedi azonosító, deduplikációhoz |
| ár (Ft) | cél változó |
| alapterület (m²) | |
| szobaszám, félszobaszám | |
| emelet / összes emelet | |
| építés éve | |
| állapot | új, felújított, jó, közepes, felújítandó stb. |
| fűtéstípus, komfortfokozat | |
| erkély, lift, parkoló, kert | bool jellegű mezők |
| kerület / település | |
| cím / földrajzi koordináta | ha elérhető a hirdetésben |
| hirdetés típusa | új építésű / használt |
| hirdetés URL | visszakereshetőség miatt |
| gyűjtés dátuma | időbeli torzítás dokumentálásához |

A pontos mezőkészlet a tényleges HTML-struktúra megismerése után finomodik —
lehet, hogy egyes mezők nem minden hirdetésnél elérhetők.

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
- **Probléma:** nem minden hirdető tölt ki minden mezőt; mezők formátuma
  (pl. "65 m²" vs "65m2") eltérhet.
- **Lehetőségek:** toleráns parse-olás (regex, egységesített mértékegység-
  kezelés), hiányzó mezők explicit `NaN`-ként jelölése (ne 0 vagy üres string),
  séma-validáció minden rekordnál mentés előtt.

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
- **Probléma:** különböző honlapok eltérő mezőneveket, mértékegységeket,
  kategóriaértékeket (pl. "állapot" lehetséges értékei) használnak; ugyanaz az
  ingatlan több oldalon is meghirdethető (cross-source duplikáció); minden
  új adapter karbantartási terhet jelent (site-redesign esetén azt is
  javítani kell).
- **Lehetőségek:** szigorú, közös séma és adapter-interfész kikényszerítése
  (3.1 pont), hogy az eltérések az adapter rétegben maradjanak; a `forrás`
  mező megtartása minden rekordnál, hogy forrásonként is lehessen vizsgálni
  az adatot; cross-source duplikáció kezelése külön, később (EDA fázisban)
  cím+méret+szobaszám alapú közelítő egyezés-kereséssel, nem a scraper
  részeként; **induláskor egy forrás (ingatlan.com) is elég** — csak akkor
  érdemes második adaptert írni, ha a mennyiség/lefedettség ezt indokolja,
  hogy a karbantartási teher arányban álljon a haszonnal.

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
  core/
    config.py                <- globális + forrásonkénti beállítások (delay, rate limit)
    fetch.py                  <- HTTP lekérés, retry/backoff, nyers válasz mentése
    schema.py                  <- közös, forrásfüggetlen rekord-séma + validáció
    dedup.py                    <- (forrás, ID) alapú deduplikáció
    pipeline.py                  <- adapter-füttetlen vezérlés: adapter kiválasztása,
                                     futtatása, mentés, futás-riport
  sites/
    base.py                   <- adapter interfész (absztrakt osztály/protokoll):
                                  list_listing_urls(), parse_listing()
    ingatlan_com.py             <- első adapter: ingatlan.com-specifikus logika
    # további_forras.py         <- csak akkor, ha egy új forrás bevonása indokolt
  data/
    raw/
      ingatlan_com/              <- nyers HTML forrásonként külön mappában
    processed/
      listings.csv/parquet        <- végleges, összesített, normalizált adatkészlet
```

A `data/raw` és a nagy nyers fájlok várhatóan `.gitignore`-ba kerülnek majd,
csak a feldolgozott, végleges adatkészletet (vagy annak mintáját) tartjuk
verziókezelésben.

## 7. Következő lépések

1. **Core + adapter-interfész megtervezése és megírása elsőként** — mielőtt
   bármelyik konkrét honlaphoz kódolnánk, hogy a keretrendszer valóban
   forrásfüggetlen maradjon.
2. robots.txt / ToS átnézése az ingatlan.com-on, oktatói visszaigazolás
   beszerzése.
3. Kísérleti futás: 1-2 lista oldal + néhány hirdetés oldal manuális
   megvizsgálása (statikus vs. JS-renderelt tartalom eldöntése).
4. **Első adapter (ingatlan.com) megírása** az interfész implementálásával,
   kis mintán (pl. 50-100 hirdetés) validálva.
5. Séma és parser finomítása a prototípus tapasztalatai alapján.
6. Teljes, ütemezett/korlátozott gyűjtés elindítása Budapestre, eladó
   lakásokra, egyetlen forrásból.
7. **Csak ha indokolt** (mennyiség/lefedettség hiánya): további forrás(ok)
   felmérése és új adapter írása a meglévő interfészre, az 5.7 pontban
   leírt szempontok figyelembevételével.
