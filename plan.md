# Terv: Magyar ingatlanárak előrejelzése

Ez a dokumentum a féléves "Gépi tanulás alapjai" beadandó előkészítő terve. Célja, hogy
felvázolja a lehetőségeket és a még nyitott döntéseket, mielőtt a csapat elköteleződne
egy konkrét irány mellett. Az értékelési szempontokat a `projekt_javaslatok.md` tartalmazza.

## 1. Bevezetés

**Probléma:** magyar lakóingatlanok árának (vagy négyzetméterárának) előrejelzése az
ingatlan jellemzői (méret, elhelyezkedés, szobaszám, állapot stb.) alapján.

**Motiváció:**
- Könnyen érthető, jól kommunikálható probléma, aminek van valós piaci relevanciája.
- Gazdag, vegyes típusú változókkal dolgozó adat (numerikus + kategorikus + földrajzi),
  ami jó terepet ad a klasszikus gépi tanulási előfeldolgozás és modellválasztás bemutatására.
- Regressziós feladatként több modell (lineáris, fa alapú, ensemble) természetes módon
  összehasonlítható.

**Döntés az előző körben:** a feladat **árpredikció (regresszió)** lesz, nem
kategória-osztályozás (pl. ár-sáv). Indok: informatívabb eredmény, gazdagabb
metrika- és modellpaletta.

## 2. Adatforrás — nyitott kérdés

Ez a legnagyobb még nyitott döntés, ettől sok más (ütemterv, előfeldolgozás, kockázatok)
függ. Gyors körülnézés alapján **nincs karbantartott, kész magyar ingatlan adatkészlet**
(pl. Kaggle-ön) — tehát valószínűleg saját adatgyűjtésre lesz szükség.

### A) Saját scraping (ingatlan.com)

- Kiindulópontok: [ingatlancom-scraper (Mukedlii)](https://github.com/Mukedlii/ingatlancom-scraper),
  [ML célú scraping projekt](https://github.com/wendig/ingatlan).
- Referencia módszertanhoz: [Budapesti lakásár regresszió — Krisztian Sandor blogja](https://kriszsandor.github.io/posts/20181213_apartment_price_regression/)
  (hasonló korábbi egyetemi/hobbi projekt, jó viszonyítási alap a feature engineeringhez
  és a modellekhez).
- Előny: friss, testreszabható adat, pontosan azokkal az oszlopokkal, amik kellenek.
- Kockázat: időigényes, és tisztázni kell a honlap ToS-ét / robots.txt-jét, illetve hogy
  az oktató/intézmény mit fogad el "sajátosan gyűjtött" adatként. Napi/heti scraping
  limitet, rate limitinget érdemes betartani.
- Becsült méret: korábbi hasonló projekt ~12 000 budapesti hirdetést gyűjtött pár hét alatt.

### B) Kész / más forrásból származó adatkészlet

- Egyelőre nem találtunk kifejezetten magyar, kész adatkészletet. Alternatíva lehet:
  - KSH (Központi Statisztikai Hivatal) nyílt adatai — de ezek jellemzően aggregált
    (település/régió szintű) statisztikák, nem hirdetés-szintű rekordok, kevésbé
    alkalmasak soronkénti regresszióra.
  - Más ország (pl. USA, India) Kaggle ingatlan adatkészlete — gyors indulás, de elveszik
    a "magyar ingatlan" relevancia, amit a csapat szeretne.

### Döntési szempontok (csapatülésen eldöntendő)

- [ ] Mennyi időnk van ténylegesen adatgyűjtésre?
- [ ] Van-e a csapatban, aki szívesen ír/karbantart egy scrapert?
- [ ] Elég-e Budapestre szűkíteni (kisebb, kezelhetőbb adat), vagy országos lefedettség a cél?
- [ ] Eladó vagy kiadó ingatlanok (vagy mindkettő, külön modellel)?

**Javaslat:** kezdjük A) opcióval, Budapestre szűkítve, eladó lakásokra — ez a
leginkább dokumentált, legkisebb kockázatú út, és szükség esetén utólag bővíthető
(vidéki városok, kiadó ingatlanok).

## 3. Adatkészlet terv

Várható / céloz ott változók (scraping esetén testreszabható):

| Kategória | Példa változók |
|---|---|
| Cél változó | ár (Ft), vagy négyzetméterár (Ft/m²) |
| Méret, elrendezés | alapterület, szobaszám, félszobaszám, emelet, összes emelet |
| Épület | építés éve, állapot (felújított/új/felújítandó), fűtéstípus, komfortfokozat |
| Elhelyezkedés | kerület/település, pontosabb földrajzi koordináta (ha elérhető) |
| Egyéb | erkély van-e, lift van-e, parkoló, kert, hirdetés típusa (új építésű/használt) |

A pontos séma csak a scraping/adatforrás döntés után válik véglegessé.

## 4. Előfeldolgozás terv

- Hiányzó értékek kezelése: oszloponként eldönteni (imputálás vs. sor/oszlop eldobás),
  különösen a scraped adatnál sok hiány várható (nem minden hirdető tölt ki mindent).
- Kiugró értékek (outlierek) kezelése: pl. extrém árak vagy irreális alapterületek szűrése.
- Kategorikus változók kódolása: one-hot / target encoding (pl. kerület, állapot).
- Numerikus változók skálázása (lineáris modellekhez, esetleg neurális hálóhoz).
- Földrajzi információ kezelése: kerület mint kategória, vagy koordinátákból származtatott
  jellemzők (pl. távolság a városközponttól), ha van rá idő.
- Train/test szétválasztás (pl. 80/20), illetve k-fold cross-validation a modellek
  hiperparaméter-hangolásához.

## 5. Módszertan — modellek

Regressziós feladathoz tervezett modell-összehasonlítás (legalább 3):

1. **Baseline:** lineáris regresszió (esetleg Ridge/Lasso regularizációval).
2. **Fa alapú modell:** Random Forest Regressor.
3. **Gradient boosting:** XGBoost / LightGBM / sklearn GradientBoostingRegressor.
4. *(opcionális, ha van idő/érdeklődés)* egyszerű neurális háló (pl. Keras/PyTorch MLP).

Minden modellhez hiperparaméter-hangolás (pl. `GridSearchCV` vagy `RandomizedSearchCV`)
cross-validationnel.

## 6. Kísérletek és kiértékelés terv

- **Metrikák:** MAE, RMSE, R² (esetleg MAPE, mivel az ár relatív eltérése is informatív).
- **Összehasonlítás:** táblázat / bar chart a modellek metrikáiról; tanulási görbék,
  feature importance ábrák (fa alapú modelleknél).
- **Hibaelemzés:** reziduálisok vizsgálata (van-e szisztematikus alul-/felülbecslés
  bizonyos árkategóriáknál vagy kerületeknél).

## 7. Elemzés és korlátok (előzetes gondolatmenet)

- A modell várhatóan nehezebben becsül extrém (nagyon olcsó/drága, egyedi) ingatlanokat.
- A hirdetési ár nem feltétlenül egyezik a tényleges eladási árral (ez a scraped adat
  egyik limitációja — érdemes ezt a dokumentációban is megemlíteni).
- Időbeli torzítás: ha a scraping egy adott időpontban történik, a modell csak arra az
  időszakra érvényes (piaci ár változhat).

## 8. Ütemterv (szakaszos, dátum nélkül — pótlandó, ha lesz határidő)

1. **Adatforrás döntés + első adatgyűjtés / kísérleti scraping** (kis minta, séma tesztelése)
2. **Teljes adatgyűjtés + tisztítás, feltáró elemzés (EDA)**
3. **Előfeldolgozási pipeline kialakítása**
4. **Baseline modell + első eredmények**
5. **Modell-összehasonlítás, hiperparaméter-hangolás**
6. **Elemzés, hibavizsgálat, ábrák**
7. **Dokumentáció megírása** (a `projekt_javaslatok.md` javasolt struktúrája szerint)
8. **Átnézés, finomítás, leadás**

Csapatlétszám és konkrét határidő még nincs megadva — ha ezek ismertté válnak, a
szakaszokhoz dátumok és felelősök rendelhetők.

## 9. Eszközök, környezet

- Python (pandas, numpy, scikit-learn, matplotlib/seaborn, esetleg xgboost/lightgbm)
- Jupyter notebook a feltáró elemzéshez és kísérletekhez
- Git a csapatmunkához (verziókezelés, ágak feladatonként)
- Scraping esetén: `requests`/`httpx` + `BeautifulSoup`, vagy a fent linkelt kész scraperek
  valamelyikének adaptálása

## 10. Nyitott döntések — gyors összefoglaló

- [ ] Adatforrás: saját scraping vs. más forrás (javaslat: saját scraping, Budapest, eladó)
- [ ] Földrajzi lefedettség: csak Budapest vagy országos
- [ ] Eladó vs. kiadó ingatlanok (vagy mindkettő külön)
- [ ] Határidő és csapatlétszám / feladatelosztás
- [ ] Opcionális neurális háló modell bevonása vagy sem
