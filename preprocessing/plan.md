# Preprocessing terv — nyers HTML-ből strukturált CSV adat

Ez a dokumentum a `plan.md` 4. pontjában ("Előfeldolgozás terv") vázolt lépések
technikai kidolgozása, és a `scraper/plan.md`-hez kapcsolódik: a scraper
feladata a nyers HTML letöltése és tárolása (`scraper/data/<forrás>/html/`), a
preprocessing feladata ebből a HTML-ből kinyerni, megtisztítani és egységes
sémára hozni az adatot, majd CSV-be menteni. A két lépés szándékosan külön
van választva: ha egy parser hibás vagy a séma változik, nem kell újra
scrape-elni, elég a meglévő nyers HTML-en újrafuttatni a feldolgozást.

A preprocessing maga is **két alfázisra** bomlik (lásd 3. pont):
1. **Forrásonkénti kinyerés:** a scraperhez hasonlóan itt is forrásonkénti
   **adapter modulok** felelnek a HTML-ből a mezők kinyeréséért — minden
   forrásnak megvan a saját parsere, mert minden honlap HTML-struktúrája
   más. Minden adapter a saját forrásának eredményét egy **önálló,
   forrásonkénti CSV-be** menti (pl. `ingatlan_com.csv`).
2. **Összevonás:** egy külön, forrásfüggetlen lépés a forrásonkénti CSV-ket
   egyesíti egyetlen, közös, modellezésre előkészített adatkészletbe.

## 1. Cél és hatókör

- **Bemenet:** nyers hirdetés-HTML fájlok, forrásonkénti mappákban
  (`scraper/data/ingatlan_com/html/<hirdetés_id>.html`, stb.).
- **Köztes kimenet:** forrásonkénti CSV-k, mindegyik az adott forrásból
  kinyert adatot tartalmazza, közös oszlopsémával
  (`preprocessing/data/processed/sources/ingatlan_com.csv`, stb.).
- **Végső kimenet:** a forrásonkénti CSV-k összevonásából létrejövő,
  egyesített, tisztított adatkészlet
  (`preprocessing/data/processed/listings.csv`), ami már modellezésre kész
  (vagy majdnem kész) formában tartalmazza az összes forrás adatát.
- **Nem feladata ennek a lépésnek:** letöltés/crawling (ez a `scraper`
  felelőssége), modellépítés (ez a modellezési fázis felelőssége — lásd
  `plan.md` 5-6. pont).

## 2. Milyen adatra (feature-ökre) van szükség?

Jó modellhez először azt kell eldönteni, mely változók valóban szükségesek és
elérhetők — ez határozza meg, mit kell a HTML-parserrel kinyerni. (A scraper
csak a linkeket és a nyers HTML-t tartja nyilván, a séma — lásd alább — már
itt, a preprocessing fázisban dől el és kerül kinyerésre.) Ez a közös séma
az, amire **minden forrásonkénti adapternek** le kell képeznie a saját
forrása mezőit, akkor is, ha az eredeti honlapon más elnevezéssel/
formátummal szerepelnek.

| Kategória | Mezők | Megjegyzés |
|---|---|---|
| Cél változó | ár (Ft) | kötelező, nélküle a rekord haszontalan |
| Méret, elrendezés | alapterület (m²), szobaszám, félszobaszám, emelet, összes emelet | alapterület és ár nélkül a rekord kidobandó |
| Épület | építés éve, állapot, fűtéstípus, komfortfokozat | gyakran hiányos, imputálás/kategória "ismeretlen" kezelés szükséges |
| Elhelyezkedés | kerület/település, cím, koordináta (ha van) | modellezéshez minimum a kerület/település kell |
| Egyéb | erkély, lift, parkoló, kert | bool mezők, hiány esetén alapértelmezett `False` helyett inkább `NaN` |
| Metaadat | forrás, hirdetés ID, hirdetés URL, gyűjtés dátuma | nem feature, de nyomonkövethetőséghez és deduplikációhoz kell |

**Minimálisan kötelező mezők** (ezek hiánya esetén a rekordot a feldolgozás
kidobja): ár, alapterület, forrás, hirdetés ID. Minden más mező hiányozhat,
és a hiány explicit `NaN`-ként jelenik meg a CSV-ben (lásd 4.3).

Ez a lista a HTML-struktúra tényleges megismerése után finomodik — lehet,
hogy egyes mezők nem parse-olhatók megbízhatóan, vagy újak válnak
elérhetővé.

## 3. Feldolgozási folyamat (HTML → forrásonkénti CSV → egyesített CSV)

### 3.1 Forrásonkénti kinyerés (adapterek — egy forrás, egy CSV)

Minden bevont forráshoz egy adapter modul tartozik (`parsers/<forrás>.py`),
ami a következő, közös interfészt implementálja:

1. **Beolvasás:** az adott forrás `scraper/data/<forrás>/html/` mappájának
   HTML fájljainak
   bejárása.
2. **Parse-olás:** forrásspecifikus HTML-parser (pl. BeautifulSoup
   selectorok) kinyeri a nyers mezőértékeket a HTML-ből (szöveg, pl.
   `"65 m²"`, `"3. emelet / 5"`, `"58 900 000 Ft"`), és a 2. pontban
   definiált közös sémára képezi le őket.
3. **Normalizálás:** szöveges nyers értékek egységes típusra/formátumra
   hozása (lásd 4. pont) — ez a lépés az adapteren belül, de a közös
   szabályokat (4. pont) felhasználva történik.
4. **Validáció:** kötelező mezők megléte, típus- és értéktartomány-ellenőrzés
   (pl. ár > 0, alapterület > 0) — érvénytelen rekord logolva és kihagyva.
5. **Mentés:** az adott forrás összes érvényes rekordja egy **önálló,
   forrásra jellemző CSV-be** kerül (pl. `data/processed/sources/
   ingatlan_com.csv`). Ez a fájl **csak** az adott forrásból kinyert
   adatokat tartalmazza.

Ez a szakasz forrásonként önállóan futtatható és ismételhető — ha egy
forrás parserét kell javítani, csak az ő CSV-je újragenerálódik, a többi
forrásé nem sérül.

### 3.2 Összevonás (forrásfüggetlen — a közös adatkészlet előállítása)

1. **Beolvasás:** az összes forrásonkénti CSV (`data/processed/sources/
   *.csv`) betöltése.
2. **Egyesítés:** a közös séma szerint egyetlen táblába fűzése (mivel a
   séma már az adapterek szintjén egységesített, ez technikailag egyszerű
   konkatenáció).
3. **Deduplikáció:** `(forrás, hirdetés ID)` pár alapján, ha egy hirdetés
   többször is feldolgozásra kerülne (pl. ismételt futás). A cross-source
   duplikáció (ugyanaz az ingatlan két különböző honlapon) külön, később,
   EDA-szintű, közelítő egyezés-kereséssel kezelhető — nem ennek a lépésnek
   a feladata.
4. **Kiugró érték szűrés** a teljes, egyesített adatkészleten (4.2 pont),
   mert az outlier-küszöbök forrásonként eltérő eloszlást takarhatnak, és
   összevontan értelmesebb eldönteni őket.
5. **Mentés:** a végleges, egyesített, modellezésre előkészített adatkészlet
   `data/processed/listings.csv`-be kerül.

Ez a szakasz forrásfüggetlen, közös kód — nem tud semmit az egyes honlapok
HTML-struktúrájáról, csak a forrásonkénti CSV-k közös sémáját használja.

## 4. Adattisztítási szabályok

### 4.1 Mértékegységek és formátum egységesítése (3.1 szakasz, adapterszinten)
- Ár: szöveges, elválasztóval/pénznemmel tagolt formából (`"58 900 000 Ft"`)
  egész szám Ft-ban.
- Alapterület: `"65 m²"` / `"65m2"` → `float`, m²-ben.
- Emelet: szöveges variánsok (pl. `"földszint"`, `"magasföldszint"`,
  `"3. emelet"`) → numerikus kódolás (pl. földszint = 0), eredeti szöveg
  megtartása egy külön mezőben, ha információvesztés nélkül nem kódolható.
- Ezek a szabályok közösek (egy helyen definiáltak, pl. `parsers/common.py`),
  de minden adapter maga hívja meg őket, mert a nyers szöveg forrásonként
  más formátumban érkezik.

### 4.2 Kiugró értékek (outlierek) (3.2 szakasz, összevonás után)
- Irreális ár/alapterület szűrése (pl. alapterület < 5 m² vagy > 1000 m²,
  négyzetméterár extrém kiugrása) — a pontos küszöbök a feltáró elemzés
  (EDA) során, a tényleges adat eloszlása alapján dőlnek el, itt csak a
  mechanizmus (konfigurálható küszöbök) kerül kialakításra.
- A kiszűrt rekordok nem törlődnek véglegesen a forrásonkénti CSV-kből,
  csak a végső, egyesített `listings.csv`-ből maradnak ki — így a döntés
  később felülvizsgálható.

### 4.3 Hiányzó értékek (3.1 szakasz, adapterszinten)
- Minden hiányzó mező explicit `NaN`-ként kerül a CSV-be, soha nem `0` vagy
  üres string — így a modellezési fázis egyértelműen meg tudja különböztetni
  a "nincs adat"-ot a "nulla értékű adat"-tól.
- Imputálás (átlag/medián/módusz, vagy "ismeretlen" kategória) **nem** ebben
  a lépésben történik, hanem a modellezési pipeline-ban — a preprocessing
  csak a tiszta, de hiányokat tartalmazó adatot adja ki, hogy a különböző
  modellek különböző imputálási stratégiát választhassanak.

### 4.4 Kategorikus értékek egységesítése (3.1 szakasz, adapterszinten)
- Pl. "állapot" mező lehetséges értékeinek szűkítése egy zárt listára
  (új, újszerű, jó, közepes, felújítandó, ismeretlen), mert a hirdetők
  gyakran szabad szöveget vagy szinonimákat használnak.
- A leképezési táblákat (szöveg → kanonikus kategória) forrásonként kell
  karbantartani, mert a hirdetési oldalak eltérő szóhasználatot használnak —
  ezért ez is adapterszintű feladat, nem az összevonásé.

## 5. CSV sémák

- **Forrásonkénti CSV** (`data/processed/sources/<forrás>.csv`): a 2.
  pontban felsorolt közös séma szerint, de csak az adott forrás rekordjaival.
  Minden adapternek ide kell leképeznie a saját mezőit.
- **Egyesített CSV** (`data/processed/listings.csv`): ugyanaz a séma, az
  összes forrásonkénti CSV összefűzve, deduplikálva és outlier-szűrve (3.2
  szakasz), kiegészítve a tisztítás során származtatott mezőkkel (pl.
  `négyzetméterár` = ár / alapterület, ha mindkettő elérhető). Ez a fájl a
  modellezési fázis bemenete.

Minden oszlop és várt típus a kód elkészülésekor egy `schema.py` (vagy
hasonló) fájlban kerül rögzítésre, amit mind az adapterek validációja
(3.1/4. lépés), mind az összevonás ellenőrzése használ.

## 6. Tervezett mappa- és kódstruktúra (a kód elkészülésekor)

```
preprocessing/
  plan.md                   <- ez a dokumentum
  requirements.txt           <- pandas, beautifulsoup4
  schema.py                   <- a közös séma + validációs szabályok
  parsers/
    base.py                    <- adapter interfész: extract(raw_dir) -> DataFrame/CSV
    common.py                    <- megosztott normalizáló függvények (4.1, 4.4)
    ingatlan_com.py                 <- ingatlan.com adapter: HTML -> sources/ingatlan_com.csv
    # további_forras.py             <- csak akkor, ha a scraper is bővül új forrással
  merge.py                    <- összevonás: sources/*.csv -> listings.csv (3.2 szakasz)
  data/
    processed/
      sources/
        ingatlan_com.csv            <- forrásonkénti, kinyert adat
      listings.csv                    <- végleges, egyesített, modellezésre kész adatkészlet
```

## 7. Következő lépések

1. A scraper első, kis mintás (50-100 hirdetés) futtatása, hogy legyen nyers
   HTML, amin a parser fejleszthető/tesztelhető.
2. Egy minta HTML alapján a tényleges mezőnevek/CSS-selectorok felmérése,
   a 2. pontban felsorolt mezők elérhetőségének ellenőrzése.
3. `schema.py`, `parsers/common.py` és az első adapter
   (`parsers/ingatlan_com.py`) megírása a kis mintán, ellenőrizve, hogy
   helyesen generálja a `sources/ingatlan_com.csv`-t.
4. `merge.py` megírása és validálása (kezdetben akár egyetlen forrással is
   működnie kell, hogy a végső `listings.csv` előálljon).
5. Teljes nyers adatkészleten lefuttatva ellenőrizni az eredmény CSV-k
   méretét, hiányzó érték arányát mezőnként, és ez alapján finomítani a
   4. pont szabályait.
6. Rövid EDA (eloszlások, hiányok, korrelációk) a kész `listings.csv`-n,
   mielőtt a modellezési fázis (lásd fő `plan.md` 5. pont) elkezdődik.
7. **Csak ha a scraper új forrással bővül:** az adott forráshoz is adapter
   írása (`parsers/<új_forrás>.py`), ami a saját `sources/<új_forrás>.csv`-jét
   állítja elő — a `merge.py`-t nem kell módosítani, mert az forrásfüggetlen.
