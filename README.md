A CREVPipeline (Kognitív Relációs Kinyerési és Validálási Csővezeték) egy nagy teljesítményű rendszer, amelyet arra terveztek, hogy strukturálatlan és félig strukturált adatfolyamokat validált, indexelt Tudásgráffá alakítson át. Hidat képez a nyers bemenet (szöveg, strukturált adat, képmetaadat) és az nsir_core és chaos_core alrendszerek által használt formális gráfreprezentációk között.

A rendszer a RelationalTriplet fogalmára épül, amely egy adatstruktúra, ami két entitás közötti szemantikai kapcsolatot reprezentál. A csővezeték kezeli ezeknek a hármasoknak a teljes életciklusát: a kezdeti kinyeréstől és nyelvi tőképzéstől a statisztikai anomáliadetektáláson át a memóriamagban való végső tárolásig.

Főbb fogalmak

RelationalTriplet
A rendszer alapvető információegysége. Tartalmaz egy alanyt (subject), egy relációt (relation) és egy tárgyat (object), kiegészítve metaadatokkal, mint például a megbízhatósági pontszámok (confidence) és a kinyerési idő (extraction_time). A hármasok végül gráfcsomópontokká és élekké alakulnak.

Tudásgráf és Indexelés
Az integrált hármasok egy KnowledgeGraphIndex-ben tárolódnak, amely háromtengelyes indexelési stratégiát biztosít (alany, reláció és tárgy szerint). Ez O(1) vagy O(N) kereséseket tesz lehetővé, ahol N a legkisebb jelölthalmazt jelenti, optimalizálva a lekérdezési teljesítményt összetett gráfbejárásoknál.

Kinyerési Szakaszok
Az adatok az ExtractionStage enum által meghatározott öt fázison haladnak át szigorúan meghatározott sorrendben:
1. Tokenizálás: A nyers bemenet WordToken egységekre bontása.
2. Hármas kinyerés: Alany-reláció-tárgy minták azonosítása.
3. Validálás: Logikai konzisztencia és statisztikai anomáliák ellenőrzése.
4. Integráció: Új adatok összevonása a meglévő tudással.
5. Indexelés: A KnowledgeGraphIndex frissítése a lekérdezhetőség érdekében.

Főbb Alrendszerek

1. Adatfolyam-feldolgozás és Pufferelés
A csővezeték egy körkörös StreamBuffer-t használ a bejövő adatok kezelésére. Ha a puffer eléri a kapacitását, a rendszer automatikusan kizárja a legrégebbi hármasokat az új adatok befogadása érdekében, biztosítva a folyamatos működést korlátlan adatfolyamok esetén is.

2. NLP és Mintaillesztés
A rendszer egy morfémaalapú illesztőmotort alkalmaz. A stemWord algoritmust használja a toldalékok eltávolítására (pl. "-ing", "-ated", "-ness"), és a matchPatternMorphemeAware függvényt a relációk azonosítására akkor is, ha a szóalakok változnak.

3. Validálás és Anomáliadetektálás
Mielőtt egy hármas integrálódna, átmegy a validateTriplet folyamaton. Ez a folyamat kiszámít egy anomaly_score értéket a RelationStatistics-ban tárolt Welford online algoritmus segítségével. Ellentmondásokat észlel, és a megbízhatóságot a korábbi adatok alapján módosítja.

4. Integráció és Tárolás
A validált hármasok nsir_core gráfelemekké alakulnak. A toGraphElements metódus a megbízhatóságot egy quantum_state értékre (Complex f64) képezi le, a kinyerési időt pedig egy temporal phase értékre. Az adatok ezután a ChaosCoreKernel-en keresztül tárolódnak.

Külső Modulokkal való Interakció

A CREVPipeline hidat képez a természetes nyelvi bemenet és az alacsony szintű memória- és gráfstruktúrák között.

További Olvasnivaló
- Első lépések: Ismerje meg, hogyan inicializálja a csővezetéket és dolgozza fel az első adatfolyamot.
- Architektúra Áttekintés: Mélyebb betekintés a belső adatfolyamba és a szakaszátmenetekbe.
- Alapvető Adatmodell: Részletes dokumentáció a RelationalTriplet-ről és az indexelési stratégiáról.
---

1.1 ELSŐ LÉPÉSEK

A CREVPipeline egy adatfolyam-feldolgozó rendszer, amelyet arra terveztek, hogy relációs hármasokat nyerjen ki strukturálatlan és félig strukturált adatforrásokból. Öt különálló ExtractionStage fázist koordinál a nyers bemenet indexelt gráfelemekké alakításához a KnowledgeGraphIndex számára.

1. A Csővezeték Inicializálása

A csővezeték használatához meg kell adni egy std.mem.Allocator-t a belső állapotkezeléshez, valamint egy mutatót egy ChaosCoreKernel-re az integrált hármasok tartós memóriafoglalásához.

Alapbeállítás
```zig
// 1. A Kernel és az Allocator inicializálása
var kernel = try ChaosCoreKernel.init(allocator);
defer kernel.deinit();

// 2. A Csővezeték inicializálása
var pipeline = try CREVPipeline.init(allocator, &kernel);
defer pipeline.deinit();

// 3. (Opcionális) Tokenizáló konfigurálása
pipeline.config.confidence_threshold = 0.75;
pipeline.config.min_entity_length = 3;
```

Komponenskapcsolat
A CREVPipeline koordinálja belső komponenseit és külső függőségeit.

---

2. Adatfolyamok Feldolgozása

A csővezeték három speciális belépési pontot biztosít különböző adatmodalitásokhoz. Minden metódus feltölt egy PipelineResult-ot, amely nyomon követi a művelet sikerességét és metrikáit.

A. Szöveges adatfolyamok
A processTextStream metódus mondatszegmentálást végez, és morfémaalapú mintaillesztést alkalmaz az alanyok, relációk és tárgyak azonosítására.

```zig
const result = try pipeline.processTextStream("The reactor core exhibits high thermal flux.");
```

B. Strukturált adatfolyamok
A processStructuredDataStream metódus CSV-szerű vagy oszlopos adatokhoz van optimalizálva, ahol a mezőket egy elválasztó karakter választja el (alapértelmezés szerint `,`).

```zig
const result = try pipeline.processStructuredDataStream("Reactor-01,status,operational", ",");
```

C. Képmetaadat-folyamok
A processImageMetadataStream metódus kulcs-érték párokat elemez, amelyek jellemzően EXIF vagy technikai képfejlécekben találhatók.

```zig
const result = try pipeline.processImageMetadataStream("sensor_type=thermal_array;resolution=1024x768");
```

---

3. Adatfolyam és Végrehajtás
Amikor egy belépési pontot meghívnak, az adatok az ExtractionStage életcikluson haladnak át. A csővezeték megpróbálja minden hármast a tokenizálástól az indexelésig végigvezetni.

---

4. Az Eredmények Értelmezése

Minden feldolgozási hívás visszaad egy PipelineResult-ot. Ez a struktúra az elsődleges módja a csővezeték teljesítményének megfigyelésére és a hibák diagnosztizálására.

A PipelineResult főbb mezői

| Mező | Leírás |
| :--- | :--- |
| triplets_extracted | A bemenetben talált nyers hármasok száma. |
| triplets_validated | A validateTriplet ellenőrzéseken átment hármasok száma. |
| conflicts_resolved | A resolveConflicts által összevont vagy feloldott hármasok száma. |
| success | Logikai érték, amely jelzi, hogy a szakasz végzetes hiba nélkül fejeződött-e be. |
| processing_time_ns | Az adott hívás késleltetése nanoszekundumban. |

Minimális Használati Példa
```zig
const std = @import("std");
const crev = @import("crev_pipeline.zig");

pub fn main() !void {
    var gpa = std.heap.GeneralPurposeAllocator(.{}){};
    const allocator = gpa.allocator();

    var kernel = try crev.ChaosCoreKernel.init(allocator);
    var pipeline = try crev.CREVPipeline.init(allocator, &kernel);
    defer pipeline.deinit();

    const input = "System-A transmits signal-alpha. Signal-alpha carries telemetry.";
    const result = try pipeline.processTextStream(input);

    if (result.success) {
        std.debug.print("Extracted {d} triplets in {d}ns\n", .{
            result.triplets_extracted, 
            result.processing_time_ns
        });
    }
}
```
---

1.2 ARCHITEKTÚRA ÁTTEKINTÉS

A CREVPipeline a strukturálatlan és félig strukturált adatok nagy fidelitású Tudásgráffá alakításának központi koordinációs motorjaként szolgál. Kezeli az adatok életciklusát, ahogy azok öt különálló kinyerési fázison haladnak át, biztosítva, hogy minden információdarab validálva, kereszthivatkozva és a rendszer memóriamagjában tárolva legyen.

A csővezeték célja, hogy hidat képezzen a "Természetes Nyelvi Tér" (nyers karakterláncok és minták) és a "Kódentitás-tér" (gráfcsomópontok, élek és kvantumállapotok) között.

A Csővezeték Koordinációja és Tulajdonlása

A CREVPipeline osztály a kinyerési életciklushoz szükséges elsődleges adatstruktúrák tulajdonosa. Koordinálja a belső pufferek és a külső függőségek, például a ChaosCoreKernel közötti adatfolyamot.

Főbb Tulajdonlási Kapcsolatok:
- StreamBuffer: Egy körkörös sor, amely a legutóbbi RelationalTriplet objektumok ablakát tartja fenn.
- KnowledgeGraphIndex: Egy háromtengelyes index (alany, reláció, tárgy), amely O(1) keresést biztosít a meglévő hármasokhoz.
- ChaosCoreKernel: Egy külső függőség, amelyet az integrált adatok hosszú távú foglalásához és tárolásához használnak.
- Statisztikai motorok: RelationStatistics és EntityStatistics leképezések, amelyek nyomon követik a megbízhatósági eloszlásokat és az előfordulási gyakoriságokat.

---

Az Öt Kinyerési Szakasz

Az adatok a csővezetéken az ExtractionStage enum által meghatározott lineáris sorrendben haladnak át. Minden szakasznak sikeresen kell befejeződnie, mielőtt a következő a next() metóduson keresztül aktiválódna.

| Szakasz | Név | Felelősség |
| :--- | :--- | :--- |
| 0 | tokenization | A nyers bemenet WordToken struktúrákra bontása és a stemWord algoritmus alkalmazása. |
| 1 | triplet_extraction | Alany-Reláció-Tárgy minták azonosítása morfémaalapú illesztéssel. |
| 2 | validation | Konzisztenciaellenőrzések elvégzése és anomáliapontszámok kiszámítása a validateTriplet segítségével. |
| 3 | integration | Konfliktusok feloldása a meglévő hármasokkal és tárolás a ChaosCoreKernel-ben. |
| 4 | indexing | A KnowledgeGraphIndex és a globális statisztikák frissítése. |

Adatfolyam Logika
A csővezeték a processTextStream, processStructuredDataStream vagy processImageMetadataStream függvényeket használja belépési pontként. A forrástól függetlenül mindhárom egy RelationalTriplet létrehozásában konvergál.

---

Természetes Nyelv Áthidalása Kódentitásokhoz

A csővezeték az ember által olvasható karakterláncokat az nsir_core által használt matematikai reprezentációkká alakítja. Ezt elsősorban a toGraphElements függvény kezeli, amely a RelationalTriplet mezőit Node és Edge entitásokra képezi le.

1. Identitás-hashelés: Az alany és tárgy karakterláncok 16 bájtos egyedi azonosítókká alakulnak a hashTripletIdentity (SHA-256) segítségével.
2. Kvantumállapot-leképezés: A megbízhatósági pontszám (0,0-tól 1,0-ig) egy Complex(f64) kvantumállapotra képeződik le, ahol a valós komponens a megbízhatóság, a képzetes komponens pedig a bizonytalanságot reprezentálja: imag = sqrt(1 - confidence²).
3. Időbeli fázis: Az extraction_time egy 360 másodperces cikluson belüli fázisra képeződik le.

---

Memória- és Tárolási Architektúra

A CREVPipeline nem kezeli a tényleges adattartalom hosszú távú heap-tárolását; ezt a ChaosCoreKernel-re delegálja.

- StreamBuffer kezelés: Amikor a puffer eléri a kapacitását, a csővezeték automatikusan kizárja a legrégebbi hármast az új adatok befogadása érdekében.
- Kernel-foglalás: Az integration szakasz során a csővezeték meghívja a kernel.allocateMemory-t. Ha ez sikertelen, a csővezeték errdefer-t használ annak biztosítására, hogy a RelationalTriplet felszabaduljon, és a szakasz sikertelenként legyen megjelölve a PipelineResult-ban.
- Index-hivatkozások: A KnowledgeGraphIndex mutatókat tárol a csővezeték által jelenleg kezelt hármasokra. Ezek a mutatók addig érvényesek, amíg a hármas létezik a StreamBuffer-ben.
---

2. ALAPVETŐ ADATMODELL

A CREV Csővezeték adatstruktúrák hierarchiájára épül, amelyek célja a nyers bemeneti adatfolyamok strukturált, lekérdezhető tudásgráffá alakítása. Ez a modell hidat képez a strukturálatlan természetes nyelv és egy formális relációs index között.

RelationalTriplet
A RelationalTriplet a csővezeték alapvető információegysége. Egyetlen, egy bemeneti forrásból kinyert tényt reprezentál, amely tartalmaz egy alanyt, egy relációt és egy tárgyat. Az alaptriplet mellett kritikus metaadatokat is hordoz a validáláshoz és a gráfintegrációhoz, mint például a megbízhatósági pontszámok és az időbeli fázisok.

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| subject | []const u8 | A reláció forrás-entitása. |
| relation | []const u8 | A kapcsolat típusa vagy címkéje. |
| object | []const u8 | A reláció cél-entitása. |
| confidence | f64 | Valószínűségi pontszám (0,0-tól 1,0-ig) a kinyerés pontosságáról. |
| extraction_time | i128 | Unix időbélyeg nanoszekundumban. |

Az életciklus-metódusokról, a hashelési algoritmusokról és a kvantumállapot-konverzióról lásd a RelationalTriplet (2.1) oldalt.

---

KnowledgeGraphIndex
A KnowledgeGraphIndex nagy teljesítményű, háromtengelyes keresési mechanizmust biztosít. Három különálló StringHashMap struktúrát tart fenn, amelyek lehetővé teszik az O(1) vagy O(N_részhalmaz) lekérdezést a hármas bármely komponense alapján.

Az index egy "legkisebb jelölthalmaz" optimalizálást alkalmaz a többparaméteres lekérdezések során, hogy minimalizálja a hármas-összehasonlítások számát. Az indexelési API-ról és a morfémaalapú lekérdezésről lásd a KnowledgeGraphIndex (2.2) oldalt.

---

StreamBuffer
A StreamBuffer körkörös sorként működik, amely a legutóbbi érvényes hármasokat tárolja. Biztosítja, hogy a csővezeték folyamatos, korlátlan adatfolyamokat tudjon kezelni azáltal, hogy egy kizárási szabályzatot valósít meg, ahol a legrégebbi adatokat felülírja, amikor a puffer eléri a kapacitását.

- Rögzített memóriaigény: Előre lefoglalt kapacitással inicializálva, hogy megakadályozza a futásidejű foglalási csúcsokat.
- Metrikák: Nyomon követi az overflow_count és utilization értékeket a csővezeték visszanyomásának megfigyelhetőségéhez.
- Tulajdonlás: A hármasokat a végső integrációs szakasz során veszik ki a pufferből, vagy közvetlenül lekérdezik időbeli elemzéshez.

A körkörös sor mechanikájáról és a pufferműveletekről lásd a StreamBuffer (2.3) oldalt.

---

Adatmodell Integráció
A csővezeték koordinálja ezeket a struktúrákat annak biztosítása érdekében, hogy minden RelationalTriplet validálva legyen az indexelés előtt.

---

2.1 RELATIONALTRIPLET

A RelationalTriplet a CREVPipeline rendszer elsődleges adatstruktúrája, amely a különböző bemeneti adatfolyamokból kinyert tudás atomi egységét reprezentálja. Magában foglalja két entitás közötti kapcsolatot, a kapcsolódó megbízhatósági pontszámokat, az időbeli metaadatokat és egy rugalmas kulcs-érték tárolót a kiterjesztett kontextushoz.

Struktúra Definíció

A RelationalTriplet struktúra egy tárolóként van definiálva mind az alapvető szemantikai adatok, mind a gráfintegrációhoz szükséges adminisztratív metaadatok számára.

Mezők

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| subject | []const u8 | A kapcsolat forrás-entitása. |
| relation | []const u8 | A predikátum vagy a kapcsolat típusa, amely összeköti az alanyt és a tárgyat. |
| object | []const u8 | A kapcsolat cél-entitása. |
| confidence | f64 | 0,0 és 1,0 közötti érték, amely a kinyerés bizonyosságát reprezentálja. |
| source_hash | [32]u8 | SHA-256 hash, amely a hármas adott verzióját/példányát reprezentálja. |
| extraction_time | i128 | Unix nanoszekundumos időbélyeg a kinyerés időpontjáról. |
| metadata | StringHashMap([]const u8) | Leképezés a hármashoz kapcsolódó tetszőleges kulcs-érték párokhoz. |

---

Életciklus-kezelés

A RelationalTriplet életciklusa manuális memóriakezelést igényel egy megadott std.mem.Allocator segítségével.

Inicializálás
- init: Szabványos konstruktor, amely inicializálja a mezőket és automatikusan kiszámítja a source_hash-t a hashTripletFields segítségével.
- initWithHash: Lehetővé teszi a source_hash manuális megadását, amelyet jellemzően a hármasok tárolóból való visszatöltésekor használnak.

Segédmetódusok
- clone: Mély másolatot készít a hármasról, beleértve az összes karakterláncot és a metadata hash leképezést, biztosítva a független memóriatulajdonlást.
- deinit: Felszabadítja a subject, relation, object és a metadata leképezéshez kapcsolódó összes memóriát.

---

Hashelés és Identitás

A rendszer különbséget tesz egy tény identitása és egy kinyerés állapota között két különálló hashelési függvényen keresztül.

Identitás vs. Mező-hashelés
- hashTripletIdentity: Csak a subject, relation és object értékeket haseli. Ezt arra használják, hogy azonosítsák, ha két hármas ugyanarra a szemantikai tényre utal, függetlenül attól, hogy mikor vagy hogyan nyerték ki őket.
- hashTripletFields: Az entitásokat a confidence és az extraction_time értékekkel együtt haseli. Ez egyedi ujjlenyomatot hoz létre egy adott kinyerési eseményhez.

Egyenlőségi Logika
- equals: Közvetlen karakterlánc-összehasonlítást végez az alapmezőkön (subject, relation, object).
- hashEquals: Két hármast hasonlít össze a hashTripletIdentity eredményeinek kiszámításával és összehasonlításával.

---

Gráfkonverzió (nsir_core Integráció)

A toGraphElements metódus egy RelationalTriplet-et az nsir_core modul által megkövetelt formális gráfkomponensekké alakít.

Átalakítási Logika
1. Csomópont-létrehozás: Két nsir_core.Node objektum jön létre az alany és a tárgy számára. A csomópont-azonosítók a nevük SHA-256 hash-éből származnak.
2. Kvantumállapot: A confidence értéke egy komplex számra (quantum_state) képeződik le, ahol:
   - Valós rész = confidence
   - Képzetes rész = sqrt(1,0 - confidence²)
3. Időbeli fázis: Egy fázisérték kerül kiszámításra az extraction_time modulo 360 alapján, amely egy cikluson belüli időbeli orientációt reprezentál.
4. Él-létrehozás: Egy nsir_core.Edge jön létre, amely összeköti a csomópontokat EdgeQuality.coherent minőséggel és a confidence értékével megegyező súllyal.

---

Memória- és Tárolási Folyamat

Amikor egy hármas integrálódik a rendszerbe, az átmeneti csővezeték-memóriából tartós kernel-tárolóba kerül.

Szerializálás
A hármasok a ChaosCoreKernel számára a következő sorrendben kerülnek szerializálásra a memóriába:
1. source_hash (32 bájt)
2. extraction_time (16 bájt, little-endian)
3. confidence (8 bájt, bitcast f64)
4. Hossz-előtaggal ellátott subject, relation és object karakterláncok.
---

2.2 KNOWLEDGEGRAPHINDEX

A KnowledgeGraphIndex egy nagy teljesítményű, háromtengelyes indexelési rendszer, amelyet O(1) keresések biztosítására terveztek relációs adatokhoz. A CREVPipeline elsődleges memóriabeli lekérdező motorjaként szolgál, lehetővé téve a rendszer számára, hogy a hármasokat bármely három elsődleges komponensük alapján lekérje: alany, reláció vagy tárgy szerint.

Áttekintés és Cél

Az index RelationalTriplet mutatókat kezel, és három különálló hash-alapú leképezésbe rendezi őket. Ez az architektúra összetett gráflekérdezéseket tesz lehetővé, és támogatja a csővezeték validálási és konfliktusfeloldási szakaszait azáltal, hogy gyors hozzáférést biztosít a meglévő tudáshoz.

Főbb Felelősségek
- Háromtengelyes indexelés: Egyidejű indexeket tart fenn a subject_index, relation_index és object_index számára.
- Lekérdezés-optimalizálás: "Legkisebb jelölthalmaz" stratégiát valósít meg az iterációk minimalizálásához a többparaméteres lekérdezések során.
- Morfémaawareness: Nyelvi alapú keresést támogat a queryMorphemeAware API-n keresztül.
- Memóriakezelés: Szigorú tulajdonlási szabályokat érvényesít, ahol az index a ChaosCoreKernel-ben tárolt hármasokra mutató mutatókat tárol.

---

Adatstruktúrák és Memóriatulajdonlás

A KnowledgeGraphIndex std.StringHashMap-ot használ az entitás- vagy relációs karakterláncok ArrayList(*RelationalTriplet) értékekre való leképezéséhez.

Belső Séma

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| subject_index | StringHashMap(ArrayList(*RelationalTriplet)) | Mutatók indexelve a hármas alany-karakterlánca szerint. |
| relation_index | StringHashMap(ArrayList(*RelationalTriplet)) | Mutatók indexelve a hármas reláció-karakterlánca szerint. |
| object_index | StringHashMap(ArrayList(*RelationalTriplet)) | Mutatók indexelve a hármas tárgy-karakterlánca szerint. |
| allocator | std.mem.Allocator | A belső ArrayList struktúrák kezeléséhez használt. |

Memória Életciklus
Az index nem tulajdonosa a RelationalTriplet memóriának. Ehelyett:
1. A CREVPipeline szerializálja a hármast és a ChaosCoreKernel.allocateMemory segítségével tárolja.
2. Az eredményül kapott mutatót átadja a KnowledgeGraphIndex.index(triplet) függvénynek.
3. A remove hívásakor csak a mutató kerül eltávolításra az indexekből; az alapul szolgáló memóriát a hívónak vagy a kernelnek kell kezelnie.

---

API Referencia

Alapvető Műveletek

| Függvény | Leírás |
| :--- | :--- |
| index(triplet: *RelationalTriplet) | Hozzáad egy hármas-mutatót mindhárom belső leképezéshez. Ha egy kulcs nem létezik, egy új ArrayList inicializálódik. |
| remove(triplet: *RelationalTriplet) | Eltávolítja a mutatót mindhárom indexből. Lineáris keresést alkalmaz az adott vödör ArrayList-jén belül. |
| queryBySubject(s: []const u8) | Visszaadja az alanynak megfelelő hármas-mutatók szeletét. |
| queryByRelation(r: []const u8) | Visszaadja a relációnak megfelelő hármas-mutatók szeletét. |
| queryByObject(o: []const u8) | Visszaadja a tárgynak megfelelő hármas-mutatók szeletét. |

A query() Optimalizálás
Az általános query(s, r, o) függvény egy heurisztikát valósít meg a munka minimalizálásához, amikor több paramétert adnak meg. Azonosítja a legkisebb jelölthalmazú paramétert (a legrövidebb ArrayList-et a megfelelő indexben), és ezt használja szűrési alapként.

---

Morfémaalapú Lekérdezések

A queryMorphemeAware függvény kiterjeszti a szabványos lekérdezést a wordsMatchStem logika alkalmazásával. Ez lehetővé teszi az index számára, hogy olyan eredményeket adjon vissza, ahol a reláció vagy az entitások közös nyelvi gyököt osztanak (pl. a "running" illeszkedik a "run"-ra).

Megvalósítási Részletek
A szabványos query()-vel ellentétben, amely O(1) hash-kereséseket alkalmaz, a queryMorphemeAware-nek teljes vizsgálatot kell végeznie a releváns indexen, mivel egy szó és annak töve különböző hash-értékkel rendelkezik.

1. Iterálás: Végigmegy a célindex minden ArrayList-jén (pl. relation_index).
2. Tő-illesztés: Meghívja a wordsMatchStem-et a kulcsokon.
3. Gyűjtés: Összegyűjti az összes hármast az illeszkedő vödrökből.

---

Technikai Korlátok és Teljesítmény

| Metrika | Komplexitás | Megjegyzés |
| :--- | :--- | :--- |
| Beszúrás | O(1) | Amortizált hash-leképezés-beszúrás 3 leképezésen keresztül. |
| Pontos lekérdezés | O(K) | Ahol K a legkisebb illeszkedő vödörben lévő hármasok száma. |
| Eltávolítás | O(N_vödör) | Lineáris keresést igényel az adott vödör ArrayList-jén belül. |
| Morfémás lekérdezés | O(N_összes) | A hash-leképezés összes kulcsának vizsgálatát igényli. |

Párhuzamosság és Biztonság
A KnowledgeGraphIndex belsőleg nem szálbiztos. A CREVPipeline-tól elvárható, hogy kezelje az indexhez való hozzáférést, jellemzően az ExtractionStage sorrenden keresztül, ahol az indexelés az utolsó lépésként történik.
---

2.3 STREAMBUFFER

A StreamBuffer egy speciális körkörös sor, amelyet a RelationalTriplet objektumok folyamatos áramlásának kezelésére terveztek a CREVPipeline-on belül. Rögzített kapacitású ablakot biztosít a kinyert adatok folyamába, támogatva a csővezeték korlátlan folyamatos feldolgozási követelményét azáltal, hogy kizárási stratégiát valósít meg a legrégebbi adatokhoz, amikor a kapacitás eléri a határát.

Alapvető Mechanika

A StreamBuffer gyűrűpufferként működik egy fej/farok mutatórendszer segítségével a memóriahatékonyság kezelésére és a gyakori újrafoglalások elkerülésére.

- Kapacitás: Az inicializáláskor meghatározott, a memóriában tartott hármasok maximális számát jelöli.
- Fej (Head): A puffer legrégebbi elemének indexe.
- Farok (Tail): Az index, ahová a következő új elem kerül.
- Méret (Size): Az aktív elemek jelenlegi száma a pufferben.
- Túlcsordulás-kezelés: Amikor a push-t egy teli pufferen hívják meg, a legrégebbi hármas (a fejnél) automatikusan kizárásra kerül az új bejegyzés számára helyet csinálva, növelve az overflow_count értékét.

Főbb Függvények

| Függvény | Leírás | Megvalósítási Részlet |
| :--- | :--- | :--- |
| init | Lefoglalja a belső puffer tömböt. | Az allocator.alloc-ot használja a capacity számú hármashoz. |
| push | Hozzáad egy hármast a pufferhez. | Ha teli, meghívja a pop()-ot a legrégebbi kizárásához és növeli az overflow_count-ot. |
| pop | Eltávolítja és visszaadja a legrégebbi hármast. | Előrelépteti a fejet és csökkenti a méretet. |
| peek | Visszaadja a legrégebbi hármast eltávolítás nélkül. | Hozzáfér a buffer[head]-hez. |
| peekAt | Visszaad egy hármast a fejtől számított adott eltolásban. | A (head + index) % capacity képletet használja. |
| getUtilisation | Kiszámítja a pufferhasználat százalékát. | Visszaadja az f64(size) / f64(capacity) értéket. |

A Körkörös Logika Megvalósítása

A puffer a modulo operátort alkalmazza a kapacitással szemben az indexek becsomagolásához, biztosítva, hogy a fej és a farok az előre lefoglalt szelet határain belül maradjon.

Metrikák és Megfigyelhetőség

A StreamBuffer két elsődleges metrikát biztosít, amelyeket a PipelineStatistics használ az adatbeviteli csővezeték egészségének figyelésére:

1. Kihasználtság (Utilisation): 0,0 és 1,0 közötti érték, amely jelzi, hogy az előre lefoglalt kapacitás mekkora hányada van jelenleg használatban.
2. Túlcsordulás-számláló (Overflow Count): Kumulatív számláló, amely megmutatja, hány hármast zártak ki kapacitáskorlátok miatt. Egy gyorsan növekvő overflow_count azt jelzi, hogy a csővezeték gyorsabban dolgozza fel az adatokat, mint amennyit a downstream KnowledgeGraphIndex vagy ChaosCoreKernel integráció kezelni tud, vagy hogy a puffer kapacitása túl kicsi az adatfolyam lökésszerűségéhez.

Integráció a CREVPipeline-ban

A CREVPipeline.integrateTriplet metóduson belül a StreamBuffer a validált hármas végső célállomásaként szolgál. Mielőtt a hármas a pufferbe kerülne, szerializálódik és a ChaosCoreKernel-ben tárolódik. A StreamBuffer ezután a RelationalTriplet struktúrát memóriában tartja az azonnali hozzáférés vagy további ablakos elemzés céljából.

---

3. KINYERÉSI CSŐVEZETÉK

A kinyerési csővezeték a rendszer központi koordinációs rétege, amely felelős a strukturálatlan vagy félig strukturált bemeneti adatfolyamok validált, indexelt tudásgráf-elemekké alakításáért. A CREVPipeline osztály vezérli, amely az ExtractionStage enum által meghatározott szekvenciális ötszakaszos életciklust kezeli.

A Csővezeték Életciklusának Áttekintése

Az adatok szigorúan meghatározott sorrendben haladnak át a rendszeren. Az ExtractionStage enum meghatározza ezeket a fázisokat, a next() metódus pedig biztosítja az átmeneti logika megőrzését.

| Szakasz | Enum Tag | Leírás |
| :--- | :--- | :--- |
| 1 | tokenization | A nyers bemenet alkotó szavakra és tövekre bontása. |
| 2 | triplet_extraction | (Alany, Reláció, Tárgy) minták azonosítása. |
| 3 | validation | Konzisztencia ellenőrzése és statisztikai anomáliák detektálása. |
| 4 | integration | Új adatok összevonása a meglévő tudással és konfliktusok feloldása. |
| 5 | indexing | Hármasok véglegesítése a KnowledgeGraphIndex-ben lekérdezhetőség céljából. |

---

NLP: Tokenizálás és Mintaillesztés
A csővezeték azzal kezdi, hogy a nyers szöveget WordToken objektumokká alakítja a tokenizeIntoWords segítségével. A nyelvi változatok kezelésére a rendszer egy stemWord algoritmust alkalmaz, amely eltávolítja a közönséges toldalékokat (pl. "-ing", "-ated", "-ness") a gyök morfémájának megtalálásához.

A CREVPipeline a matchPatternMorphemeAware-t használja csúszóablakos összehasonlítások elvégzéséhez ezek a tövek és a RelationPattern definíciók egy halmaza között. Ez lehetővé teszi a rendszer számára, hogy felismerje a relációkat akkor is, ha az igeidő vagy a szám megváltozik.

---

Validálás és Konfliktusfeloldás
Miután egy RelationalTriplet kinyerésre kerül, belép a validálási fázisba. A csővezeték a validateTriplet-et használja egy ValidationResult előállítására. Ez a folyamat magában foglalja:
1. Konzisztenciaellenőrzések: A hármas ellenőrzése a contradicting_pairs táblával szemben.
2. Anomáliadetektálás: A Welford online algoritmus alkalmazása a RelationStatistics-on belül Z-pontszámok kiszámításához a megbízhatósághoz és a gyakorisághoz.
3. Konfliktusfeloldás: Ha egy új hármas ütközik a meglévő adatokkal, a csővezeték egy súlyozott megbízhatósági összevonási képletet alkalmaz: (a²+b²)/(a+b).

---

Integráció és Indexelés
A validált hármasok a ChaosCoreKernel-en keresztül tárolódnak. A csővezeték meghívja az allocateMemory-t a szerializált hármas számára szükséges hely biztosítása érdekében, mielőtt frissítené a belső KnowledgeGraphIndex-et.

Az indexelési szakasz során a hármas nsir_core.Node és nsir_core.Edge objektumokra bomlik. Ez magában foglalja a quantum_state kiszámítását (a megbízhatóság komplex számokra való leképezése) és a temporal phase értékét.

---

Csővezeték Belépési Pontok
A CREVPipeline három elsődleges bemeneti modalitást támogat, mindegyik ugyanabba az alapvető ötszakaszos életciklusba táplálkozik:
- processTextStream: Strukturálatlan természetes nyelvhez.
- processStructuredDataStream: CSV/táblázatos formátumokhoz.
- processImageMetadataStream: Vizuális eszközök kulcs-érték párjaihoz.

Minden belépési pont visszaad egy PipelineResult-ot, amely összesíti a teljesítménymetrikákat, mint például a throughput és a conflict_rate.

---

3.1 TOKENIZÁLÁS ÉS MINTAILLESZTÉS

A CREVPipeline természetes nyelvi feldolgozási (NLP) rétege felelős a nyers szöveg diszkrét nyelvi egységekké alakításáért és a relációs minták morfémaalapú elemzéssel való azonosításáért. Ez a folyamat hidat képez a strukturálatlan természetes nyelv és a Tudásgráf által használt strukturált RelationalTriplet formátum között.

Tokenizálási Folyamat

A tokenizálási szakasz a kinyerési csővezeték első fázisa. Határolójel-alapú felosztási stratégiát alkalmaz a bemeneti karakterláncok WordToken struktúrák sorozatává alakításához.

WordToken Struktúra

A WordToken struktúra az eredeti szeletet és annak pozicionális metaadatait tartja fenn a forrásszövegen belül:
- text: A szó karakterlánc-szelete.
- start: A szó kezdetének bájt-indexe.
- end: A szó végének bájt-indexe.

A tokenizeIntoWords Megvalósítása

A tokenizeIntoWords függvény végigpásztázza a bemeneti szöveget, és szóközök, illetve írásjelek alapján azonosítja a határokat. Kifejezetten a következő karaktereket kezeli határolójelként:
- Szóközök: ' ', \t, \n, \r
- Írásjelek: ',', ';', ':'

Morfémaalapú Tőképzés

A különböző nyelvtani alakok közötti illesztési pontosság javítása érdekében (pl. a "running" illesztése a "run"-ra) a csővezeték egy szabályalapú toldalékeltávolító algoritmust valósít meg a stemWord függvényben.

Tőképzési Szabályok

Az algoritmus feltételes ellenőrzések sorozatát alkalmazza a közönséges angol toldalékok eltávolítására. Tartalmaz logikát a kettős mássalhangzók és specifikus szóvégek kezelésére:

| Toldalék | Szabály | Megvalósítási Részlet |
| :--- | :--- | :--- |
| -ting | 4 karakter eltávolítása | word.len > 5 |
| -ing | 3 vagy 4 eltávolítása | Kettős mássalhangzók kezelése (pl. "running" → "run") |
| -ated | 4 karakter eltávolítása | word.len > 4 |
| -ed | 2 vagy 3 eltávolítása | Kettős mássalhangzók kezelése |
| -ies | 3 karakter eltávolítása | word.len > 4 |
| -ches/-shes/-sses | 2 karakter eltávolítása | Többes szám kezelése |
| -ly / -ally | 2 vagy 4 eltávolítása | Határozói kezelés |
| -ment / -ness | 4 karakter eltávolítása | Főnévképzés |
| -er / -est | 2 vagy 3 eltávolítása | Közép-/felsőfok; figyelmen kívül hagyja az "ever", "her" szavakat |

Tő-összehasonlítás

A wordsMatchStem függvény a következő lépésekkel határozza meg, hogy két szó szemantikailag egyenértékű-e:
1. Pontos egyenlőség ellenőrzése.
2. Kis- és nagybetűtől független verziók összehasonlítása.
3. A stemWord eredményeinek összehasonlítása mindkét bemeneten.

Mintaillesztési Architektúra

A rendszer csúszóablakos összehasonlítások elvégzésével azonosítja a relációkat a szövegben, a tokenizált mondatok és az előre definiált RelationPattern halmazok között.

Mintaillesztési Logika

A matchPatternMorphemeAware függvény a következő lépéseket hajtja végre:
1. Tokenizálja mind a forrásmondat, mind a célminta szövegét ArrayList(WordToken) segítségével.
2. Végigiterál a mondattokeneken egy, a mintahosszal egyenlő csúszóablak segítségével.
3. A wordsMatchStem-et alkalmazza az ablak minden tokenjére annak ellenőrzésére, hogy a sorozat illeszkedik-e a minta morfémáira.
4. Visszaad egy MorphemeMatch-et, amely tartalmazza az egyezés kezdő és záró bájt-eltolásait az eredeti karakterláncon belül.

Konfiguráció és Mintadefiníciók

Az NLP réteg viselkedését a TokenizerConfig és a RelationPattern struktúrák szabályozzák.

RelationPattern

Egy adott nyelvi triggert definiál egy kapcsolathoz:
- pattern: Az illesztendő karakterlánc (pl. "is located in").
- relation_type: A normalizált relációnév (pl. "location").
- weight: Egy megbízhatósági szorzó ehhez a specifikus mintához.

TokenizerConfig

A kinyerési folyamat korlátait szabályozza:
- min_entity_length / max_entity_length: Az érvényes alany/tárgy karakterláncok határai.
- confidence_threshold: A minimális pontszám, amely szükséges egy egyezés elfogadásához.
- language: Célnyelv (alapértelmezés szerint angol szabályok).
- enable_coreference: Jelző a jövőbeli koreferencia-feloldás támogatásához.

---

3.2 VALIDÁLÁS ÉS KONFLIKTUSFELOLDÁS

A CREVPipeline validálási alrendszere felelős a kinyert hármasok strukturális integritásának, logikai konzisztenciájának és statisztikai plauzibilitásának biztosításáért, mielőtt azok integrálódnának a Tudásgráfba. Háromfázisú csővezetékként működik, amely kiszűri a zajt, ellentmondásokat észlel a meglévő tudással szemben, és statisztikai anomáliákat jelöl meg online tanulási algoritmusok segítségével.

Validálási Folyamat Áttekintése

A validálási folyamat a validateTriplet függvénybe van beágyazva, amely egy ValidationResult-ot állít elő. Ez az eredmény határozza meg, hogy egy hármas továbblép-e az Integráció és Indexelés szakaszokba.

ValidationResult Struktúra

A ValidationResult tartalmazza a validálási folyamat kimenetelét, beleértve a módosított megbízhatósági pontszámot és az azonosított konfliktusok listáját.

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| is_valid | bool | Végső meghatározás arról, hogy a hármas integrálható-e. |
| confidence_adjusted | f64 | Az eredeti megbízhatósági pontszám, amelyet anomáliapontszámok és konfliktus-büntetések skáláznak. |
| anomaly_score | f64 | Z-pontszám, amely megmutatja, mennyire tér el a hármas megbízhatósága a reláció átlagától. |
| conflicts | ArrayList(*RelationalTriplet) | Az indexben lévő, a jelöltet megcáfoló meglévő hármasok listája. |

Háromfázisú Validálási Logika

A validateTriplet függvény három különálló fázist hajt végre egy RelationalTriplet kiértékeléséhez.

1. Alapvető Hossz- és Határellenőrzések

A rendszer először egy "józanság-ellenőrzést" végez a hármas karakterlánc-mezőin.
- Alany/Reláció/Tárgy hossza: Nem lehet nulla.
- Megbízhatósági tartomány: [0,0; 1,0] között kell lennie.

2. Konzisztenciaellenőrzés (checkConsistency)

Ez a fázis lekérdezi a KnowledgeGraphIndex-et, hogy megtalálja azokat a hármasokat, amelyek logikailag ellentmondanak az új jelöltnek. A contradicting_pairs táblát használja a kölcsönösen kizáró relációk azonosítására.

- Ellentmondó párok: Relációk statikus leképezése, amelyek nem lehetnek egyszerre igazak ugyanarra az Alany-Tárgy párra (pl. "is_friend_of" vs "is_enemy_of").
- Konfliktusfelderítés: A rendszer lekérdezi az indexet minden olyan hármasra, amely ugyanazt az Alanyt és Tárgyat osztja. Ha egy visszaadott hármas relációja megtalálható a jelölt relációjának contradicting_pairs listájában, hozzáadódik a ValidationResult.conflicts listához.

3. Anomáliadetektálás (computeAnomalyScore)

Az utolsó fázis összehasonlítja a hármas megbízhatóságát a RelationStatistics-ban tárolt korábbi adatokkal.

- Statisztikai alap: A rendszer lekéri a specifikus relációtípus RelationStatistics-át.
- Z-pontszám kiszámítása: Kiszámítja, hogy a jelenlegi megbízhatóság hány szórásnyira van az átlagtól.
- Pontszám hatása: Ha az anomaly_score magas (jellemzően > 3,0), a confidence_adjusted értéke jelentősen csökken.

Statisztikai Anomáliadetektálás

A rendszer a Welford Online Algoritmust alkalmazza minden relációtípus futó statisztikáinak fenntartásához anélkül, hogy minden egyes adatpontot tárolna. Ez lehetővé teszi az állandó idejű frissítéseket és a memóriahatékony anomáliadetektálást.

RelationStatistics és a Welford Algoritmus

Minden RelationStatistics objektum a következőket követi nyomon:
- count: Megfigyelt hármasok száma.
- mean: A megbízhatósági pontszámok futó átlaga.
- m2: Az átlagtól való eltérések négyzetösszege (a variancia kiszámításához használt).

A variancia és a szórás a következőképpen vezethető le:
- Variancia: m2 / count
- Szórás: sqrt(variancia)

Konfliktusfeloldás és Összevonás

Amikor egy hármas érvényes, de ütközik a meglévő adatokkal (vagy egy különböző megbízhatóságú duplikátumot képvisel), a resolveConflicts logika aktiválódik az integrációs fázis során.

Súlyozott Megbízhatósági Képlet

Ha egy hármas identitás (Alany-Reláció-Tárgy) már létezik, a rendszer összevonja az új megfigyelést a meglévővel egy súlyozott megbízhatósági képlet segítségével, amely a magasabb bizonyosságú bemeneteket részesíti előnyben, miközben elismeri az adatok mennyiségét.

A két a és b megbízhatósági pontszám összevonásához használt képlet:

Megbízhatóság_új = (a² + b²) / (a + b)

Ez a képlet biztosítja, hogy ha az egyik forrás lényegesen magabiztosabb a másiknál, az eredő megbízhatóság agresszívabban gravitál a magasabb érték felé, mint egy egyszerű lineáris átlag esetén.

Konfliktusfeloldási Logika Folyamata

A resolveConflicts függvény kezeli az ütköző hármasok életciklusát:
1. Jelöltek azonosítása: Összegyűjti az összes hármast a ValidationResult.conflicts listából.
2. Képlet alkalmazása: Iteratívan alkalmazza az (a²+b²)/(a+b) képletet a jelöltre és versenytársaira.
3. Frissítés vagy kizárás: Ha az új megbízhatóság meghalad egy küszöbértéket, a meglévő hármas frissül a KnowledgeGraphIndex-ben; ellenkező esetben az új hármas elvetésre kerülhet.

---

3.3 INTEGRÁCIÓ ÉS INDEXELÉS

Az Integráció és Indexelés szakasz a CREVPipeline életciklusának utolsó fázisát képviseli. Miután egy RelationalTriplet kinyerésre került és átment a validálási és konfliktusfeloldási logikán, be kell kerülnie a rendszer hosszú távú memóriastruktúráiba. Ez a folyamat magában foglalja az adatok szerializálását a ChaosCoreKernel számára, a KnowledgeGraphIndex frissítését a gyors visszakeresés érdekében, valamint a globális RelationStatistics és EntityStatistics módosítását a jövőbeli anomáliadetektálás tájékoztatásához.

Adatfolyam: Validálástól a Tartós Tárolásig

Miután egy hármas érvényesnek minősül, a csővezeték átmegy az ExtractionStage.validation-ről az ExtractionStage.integration-re, majd az ExtractionStage.indexing-re.

1. Memóriafoglalás és Szerializálás

A csővezeték egy ChaosCoreKernel-t használ a hármas tartós tárolásához. A hármas szerializálódik és az allocateMemory segítségével tárolódik, biztosítva, hogy a tudás megmaradjon a kernel memóriasubsztrátumán belül. Ha a kernel nem tud memóriát foglalni, a csővezeték errdefer mintákat alkalmaz az állapot konzisztenciájának fenntartásához.

2. Tudásgráf Indexelés

A hármas bekerül a KnowledgeGraphIndex-be. Ez a struktúra három különálló StringHashMap gyűjteményt tart fenn (subject_index, relation_index és object_index), hogy O(1) kereséseket tegyen lehetővé a hármas bármely tengelye mentén.

3. Statisztikai Frissítések

A csővezeték futó számlálókat és eloszlási adatokat tart fenn:
- RelationStatistics: Frissíti a számlálót, a teljes megbízhatóságot és a futó varianciát (a Welford online algoritmus segítségével) az adott relációtípushoz.
- EntityStatistics: Növeli az alany és tárgy számlálóit, nyomon követve azok gyakoriságát és szerepeit (alanyként vs. tárgyként).

4. Pufferkezelés

Az integrált hármasok bekerülnek a StreamBuffer-be. Ez a körkörös sor kezeli a feldolgozott adatok áramlását, kizárva a legrégebbi hármasokat, ha eléri a kapacitást, biztosítva, hogy a csővezeték folyamatos adatfolyamok esetén is blokkolásmentesen működjön.

Metrikák és Megfigyelhetőség

Minden integrációs ciklus frissíti a PipelineResult és PipelineStatistics objektumokat. Ezek valós idejű képet nyújtanak a rendszer egészségéről és teljesítményéről.

| Metrika | Forrás Mező | Leírás |
| :--- | :--- | :--- |
| Áteresztőképesség | PipelineStatistics.throughput | Másodpercenként integrált hármasok. |
| Konfliktusarány | PipelineStatistics.conflict_rate | A konfliktusfeloldást igénylő hármasok aránya az összes hármashoz képest. |
| Pufferkihasználtság | StreamBuffer.utilization | A körkörös sor jelenleg elfoglalt százaléka. |
| Üzemidő | PipelineStatistics.uptime_ms | A csővezeték-példány teljes működési ideje. |

Kulcsfontosságú Megvalósítási Részletek

Hash Identitás vs. Mező-hash

A rendszer különbséget tesz egy hármas identitása és annak specifikus példányadatai között:
- hashTripletIdentity: Csak a subject, relation és object karakterláncokat haseli. Ezt arra használják, hogy azonosítsák, ha két hármas ugyanarra a szemantikai tényre utal különböző kinyerések esetén.
- hashTripletFields: Tartalmazza a confidence és az extraction_time értékeket. Ezt egyedi szerializáláshoz használják a ChaosCoreKernel-ben.

Atomi Számlálók

Az integrációs fázis során a következő számlálók növekednek a PipelineResult struktúrán belül:
- triplets_integrated: Összes sikeres beszúrás.
- processing_time_ns: Az integrációs és indexelési logikában töltött kumulatív idő.

---

4. BEMENETI FORRÁSOK ÉS BELÉPÉSI PONTOK

A CREVPipeline heterogén forrásokból képes adatokat befogadni, nyers strukturálatlan vagy félig strukturált adatfolyamokat egységes relációs gráffá alakítva. A rendszer három elsődleges bemeneti modalitást támogat — strukturálatlan szöveg, strukturált táblázatos adat és képmetaadat — egy robusztus hook mechanizmus mellett, amely lehetővé teszi a csővezeték viselkedésének kiterjesztését a következtetés során.

Bemeneti Modalitások

A csővezeték három speciális belépési pontot tesz elérhetővé, mindegyik egy adott adatformátumhoz optimalizálva. Ezek a metódusok kezelik a kezdeti ExtractionStage.tokenization és ExtractionStage.triplet_extraction fázisokat, mielőtt az adatokat átadnák a megosztott validálási és indexelési alrendszereknek.

| Metódus | Bemeneti Formátum | Elsődleges Kinyerési Logika |
| :--- | :--- | :--- |
| processTextStream | Strukturálatlan karakterláncok | Mondatszegmentálás és morfémaalapú mintaillesztés. |
| processStructuredDataStream | CSV/Táblázatos karakterláncok | Oszlopalapú leképezés Alany-Reláció-Tárgy hármasokra. |
| processImageMetadataStream | Kulcs=Érték párok | Attribútumkinyerés képfejlécekből vagy sidecar fájlokból. |

InferenceHook Bővítési Mechanizmus

A csővezeték külső megfigyelhetőségének és futásidejű módosításának lehetővé tételéhez a CREVPipeline egy InferenceHook rendszert valósít meg. Ez a mechanizmus callback struktúrák ArrayList-jét használja, amelyek a következtetési életciklus kritikus pontjain aktiválódnak.

Az InferenceHook Interfész

Az InferenceHook struktúra négy függvénymutató-helyet biztosít:
- pre_process: A kinyerés megkezdése előtt hajtódik végre.
- post_process: Az integráció és indexelés befejezése után hajtódik végre.
- pre_query: A KnowledgeGraphIndex elleni keresés előtt aktiválódik.
- post_query: Az indexből való eredménylekérés után aktiválódik.

Minden hook kap egy opaque context mutatót, lehetővé téve az állapotteljes bővítmények integrálását a crev_pipeline.zig alaplogikájának módosítása nélkül.

---

4.1 SZÖVEG-, STRUKTURÁLT ADAT- ÉS KÉPMETAADAT-FOLYAMOK

Ez az oldal részletezi a CREVPipeline-ba való adatbevitel három elsődleges belépési pontját. Minden metódus egy adott bemeneti modalitáshoz van igazítva, meghatározva, hogyan kerülnek elemzésre, szegmentálásra és leképezésre a nyers pufferek a RelationalTriplet struktúrára. Bár mindhárom végül ugyanazon validálási és indexelési logikán konvergál, jelentősen különböznek kinyerési heurisztikáikban és kezdeti megbízhatósági hozzárendeléseikben.

Bemeneti Modalitások Áttekintése

A CREVPipeline speciális metódusokat biztosít a természetes nyelv, a táblázatos adatok és a kulcs-érték metaadatok kezelésére. Minden metódus feltölti egy RelationalTriplet alapvető mezőit: subject, relation és object.

| Metódus | Bemeneti Formátum | Szegmentálási Logika | Kinyerési Stratégia | Alapértelmezett Megbízhatóság |
| :--- | :--- | :--- | :--- | :--- |
| processTextStream | Strukturálatlan UTF-8 szöveg | Mondatalapú (írásjelek) | Morfémaalapú mintaillesztés | Változó (Minta súlya) |
| processStructuredDataStream | CSV-szerű / Táblázatos | Soralapú (újsorok) | Oszlopindex-leképezés | 0,85 (Magas) |
| processImageMetadataStream | Kulcs=Érték párok | Soralapú | Karakterlánc-felosztás '='-nél | 0,70 (Közepes) |

1. Szöveges Adatfolyam Feldolgozása

A processTextStream metódus a legösszetettebb belépési pont, amelyet a természetes nyelv kétértelműségének kezelésére terveztek. Csúszóablakos megközelítést alkalmaz egy toldalékeltávolító tőképző algoritmussal kombinálva, hogy azonosítsa a kapcsolatokat még akkor is, ha a szavak ragozottak (pl. a "running" illesztése a "run"-ra).

Kinyerési Logika
1. Mondatszegmentálás: A bemeneti adatfolyam mondatokra bomlik standard írásjelhatárolók (., !, ?) segítségével.
2. Morfémaalapú Illesztés: Minden mondatnál a csővezeték megpróbálja illeszteni a regisztrált RelationPattern objektumokat. Ez a matchPatternMorphemeAware-t használja, amely tokenizálja a mondatot és összehasonlítja a szótöveket.
3. Tőképzés: A stemWord függvény szabályokat alkalmaz az olyan toldalékok eltávolítására, mint az -ing, -ated, -ies és -ness, hogy normalizálja a tokeneket az összehasonlítás előtt.
4. Hármas Képzés: Ha egy minta (a reláció) megtalálható két azonosított entitás (alany és tárgy) között, egy hármas jön létre. A megbízhatóság a RelationPattern.weight-ből származik.

2. Strukturált Adatfolyam Feldolgozása

A processStructuredDataStream metódus kiszámítható, táblázatos formátumokat kezel, ahol az oszlopok közötti kapcsolat előre meghatározott.

Leképezési Mechanika

A hívó megadja az alany, reláció és tárgy oszlopindexeit. Ez a metódus rendkívül hatékony, mivel megkerüli az NLP tőképzési és mintaillesztési szakaszokat.
- Sorfelosztás: Az adatfolyam újsor karakterek alapján bomlik fel.
- Oszlopelemzés: Minden sor egy határolójel alapján bomlik fel (alapértelmezés szerint vessző).
- Megbízhatóság: Alapértelmezett 0,85 megbízhatóságot rendel hozzá, tükrözve a strukturált séma-leképezések magas megbízhatóságát.

Leképezési Táblázat

| Bemeneti Oszlop | Hármas Mező |
| :--- | :--- |
| row[subject_idx] | subject |
| row[relation_idx] | relation |
| row[object_idx] | object |

3. Képmetaadat-folyam Feldolgozása

A processImageMetadataStream metódus technikai metaadatokhoz (pl. EXIF adatok vagy sidecar fájlok) készült, amelyek Kulcs=Érték párokként vannak formázva.

Kinyerési Stratégia
- Kontextuális Alany: Mivel a metaadatok gyakran egyetlen entitást (a képet) írnak le, a subject jellemzően konstansként kerül átadásra (pl. a fájlnév vagy UUID).
- Kulcs-Érték Felosztás: A sor az első '=' karakternél bomlik fel. Az '=' előtti karakterlánc lesz a relation, az utána lévő az object.
- Megbízhatóság: Alapértelmezett 0,70 megbízhatóságot rendel hozzá, figyelembe véve az automatizált metaadat-kinyerő eszközök lehetséges zajait.

Belépési Pontok Összehasonlítása

| Jellemző | processTextStream | processStructuredDataStream | processImageMetadataStream |
| :--- | :--- | :--- | :--- |
| Tokenizálás | tokenizeIntoWords | Határolójel-alapú | Kulcs=Érték felosztás |
| Tőképzés | Engedélyezett (stemWord) | Letiltott | Letiltott |
| Mintaillesztés | Szükséges | N/A (Index-alapú) | N/A (Kulcs-alapú) |
| Validálási Szakasz | Teljes Validálás | Teljes Validálás | Teljes Validálás |
| Alapértelmezett Megbízhatóság | Dinamikus (0,0 - 1,0) | 0,85 | 0,70 |

Megvalósítási Megjegyzés: Validálási Konvergencia

A belépési ponttól függetlenül az összes kinyert hármas átkerül a validateTriplet-hez. Ez biztosítja, hogy még a "megbízható" strukturált adatok is ellenőrzésre kerüljenek a meglévő KnowledgeGraphIndex konzisztenciájával szemben, és statisztikai anomáliák szempontjából elemzésre kerüljenek a Welford-alapú RelationStatistics segítségével.

---

4.2 INFERENCEHOOK BŐVÍTÉSI PONTOK

A CREVPipeline rendszer rugalmas bővítési mechanizmust biztosít az InferenceHook interfészen keresztül. Ez a rendszer lehetővé teszi a fejlesztők számára, hogy egyéni logikát injektáljanak a szövegfeldolgozási életciklusba az alapvető csővezeték-megvalósítás módosítása nélkül. A hook-ok naplózásra, auditálásra, valós idejű adattranszformációra vagy külső validálási szolgáltatások integrálására használhatók.

Az InferenceHook Interfész

Az InferenceHook négy opcionális függvénymutatót és egy opaque context mutatót tartalmazó struktúraként van definiálva. Ez a tervezés a közönséges "figyelő" mintát követi, lehetővé téve az állapotteljes hook-okat a context mező segítségével.

Callback Helyek

| Függvénymutató | Aktiválási Pont | Cél |
| :--- | :--- | :--- |
| pre_process | A processInferenceText kezdete | Nyers bemeneti szöveg módosítása vagy vizsgálata a tokenizálás előtt. |
| post_process | A processInferenceText vége | Eredmények összesítése vagy mellékhatások kiváltása egy köteg befejezése után. |
| pre_query | A KnowledgeGraphIndex.query előtt | Keresési paraméterek naplózása vagy lekérdezési karakterláncok módosítása bővítéshez. |
| post_query | A KnowledgeGraphIndex.query után | Az eredményül kapott hármasok szűrése vagy rangsorolása, mielőtt visszakerülnek a hívóhoz. |

Adatstruktúra Definíció

```zig
pub const InferenceHook = struct {
    context: ?*anyopaque = null,
    pre_process: ?*const fn (ctx: ?*anyopaque, text: []const u8) void = null,
    post_process: ?*const fn (ctx: ?*anyopaque, result: *const PipelineResult) void = null,
    pre_query: ?*const fn (ctx: ?*anyopaque, subject: ?[]const u8, relation: ?[]const u8, object: ?[]const u8) void = null,
    post_query: ?*const fn (ctx: ?*anyopaque, results: []const *RelationalTriplet) void = null,
};
```

Regisztráció és Tárolás

A hook-ok a CREVPipeline inicializálása során vagy dinamikusan, később kerülnek regisztrálásra. Egy ArrayList(InferenceHook) nevű inference_hooks listában tárolódnak a CREVPipeline struktúrán belül.

- Inicializálás: Az init függvény inicializálja az inference_hooks listát a megadott allocator segítségével.
- Regisztráció: A felhasználók hozzáfűzik a hook-okat az inference_hooks listához.
- Tisztítás: A deinit függvény törli a listát.

Végrehajtási Folyamat: processInferenceText

A hook-olt végrehajtás elsődleges belépési pontja a processInferenceText. Ez a függvény burkolóként működik a szabványos processTextStream logika körül, biztosítva, hogy a pre_process és post_process callback-ek megfelelően aktiválódjanak.

Műveletek Sorrendje
1. Pre-Process: A csővezeték végigiterál az összes regisztrált hook-on és végrehajtja a pre_process-t, ha definiált.
2. Alapvető Feldolgozás: A csővezeték meghívja a processTextStream-et a tokenizálás, kinyerés és indexelés elvégzéséhez.
3. Post-Process: A csővezeték ismét végigiterál a hook-okon, átadva az eredményül kapott PipelineResult-ot minden definiált post_process callback-nek.

Lekérdezési Hook-ok

A pre_query és post_query hook-ok integrálva vannak a KnowledgeGraphIndex.query metódusba (vagy a csővezeték burkolójába). Ezek lehetővé teszik a megfigyelhetőséget abban, hogyan kerül keresésre a gráf.

Megvalósítás a query()-ben

Amikor a query meghívódik:
1. pre_query: Aktiválódik a subject, relation és object szűrőkkel.
2. Végrehajtás: A belső index-keresési logika fut.
3. post_query: Aktiválódik a talált RelationalTriplet mutatók szeletével.

Egyéni Hook Megvalósítása

Egyéni hook megvalósításához definiáljon egy tárolót (struktúrát) az állapot tárolásához, és adjon meg statikus függvényeket, amelyek megfelelnek az elvárt aláírásoknak.

Példa: Audit Naplózó

Ez a fogalmi megvalósítás bemutatja, hogyan használható a context mutató az állapot (egy számláló) fenntartásához a hook hívások között.

```zig
const AuditLogger = struct {
    processed_count: usize = 0,

    pub fn preProcess(ctx: ?*anyopaque, text: []const u8) void {
        const self: *AuditLogger = @ptrCast(@alignCast(ctx));
        std.debug.print("Processing text: {s}\n", .{text});
        self.processed_count += 1;
    }

    pub fn postProcess(ctx: ?*anyopaque, result: *const PipelineResult) void {
        _ = ctx;
        std.debug.print("Extracted {d} triplets\n", .{result.triplets_extracted});
    }
};

// Használat:
// var logger = AuditLogger{};
// try pipeline.inference_hooks.append(.{
//     .context = &logger,
//     .pre_process = AuditLogger.preProcess,
//     .post_process = AuditLogger.postProcess,
// });
```

Útmutatás
1. Memóriabiztonság: A context mutatónak érvényesnek kell maradnia a csővezeték élettartama alatt, vagy amíg a hook el nem távolítódik.
2. Teljesítmény: A hook-ok szinkron módon hajtódnak végre. A pre_process vagy post_query-ben végzett nehéz számítások közvetlenül növelik a csővezeték belépési pontjainak késleltetését.
3. Hibakezelés: A callback függvények void-ot adnak vissza. Ha egy hook kritikus hibával találkozik, azt belsőleg kell kezelnie (pl. naplózással vagy egy jelző beállításával a context-ben), mivel nem tudja megszakítani a csővezeték végrehajtását.

---

5. STATISZTIKÁK ÉS MEGFIGYELHETŐSÉG

A CREVPipeline átfogó megfigyelhetőségi csomagot tartalmaz, amelyet az adatminőség és a rendszerteljesítmény valós idejű figyelésére terveztek. Ez az alrendszer nyomon követi a Tudásgráf egészségét entitás- és relációspecifikus metrikákon keresztül, miközben magas szintű telemetriát biztosít a csővezeték áteresztőképességéről és erőforrás-kihasználtságáról.

A Megfigyelhetőség Architektúrája

A megfigyelhetőség két elsődleges tartományra oszlik: a Tudásstatisztikák, amelyek nyomon követik a kinyert adatok matematikai tulajdonságait az anomáliadetektáláshoz, és a Csővezeték-metrikák, amelyek a folyamfeldolgozó motor működési teljesítményét követik nyomon.

Reláció- és Entitásstatisztikák

A csővezeték részletes statisztikákat tart fenn minden feldolgozott egyedi relációtípushoz és entitáshoz. Ezek StringHashMap struktúrákban tárolódnak a CREVPipeline példányon belül.

- RelationStatistics: Nyomon követi a relációtípusok gyakoriságát és megbízhatósági eloszlását. A Welford online algoritmust használja a futó átlag és variancia fenntartásához anélkül, hogy minden egyes adatpontot tárolna, lehetővé téve a hatékony Z-pontszám kiszámítást az anomáliadetektáláshoz.
- EntityStatistics: Figyeli, hogy az egyes entitások milyen gyakran jelennek meg, és milyen szerepeket töltenek be (alany vs. tárgy) a gráfon belül.

Ezek a statisztikák az ExtractionStage.indexing fázis során frissülnek, miután egy hármas sikeresen integrálódott.

Csővezeték-szintű Metrikák

A működési megfigyelhetőséget egy kétszintű jelentési rendszer kezeli, amely mind az átmeneti hívásszintű adatokat, mind a hosszú távú összesített egészséget rögzíti.

- PipelineResult: Minden belépési pont (pl. processTextStream) visszaad egy PipelineResult-ot, amely számlálókat tartalmaz az adott híváshoz, mint például a triplets_extracted, conflicts_resolved és a pontos processing_time_ns.
- PipelineStatistics: A CREVPipeline egy globális PipelineStatistics objektumot tart fenn. A mergeResult függvény az egyedi eredmények összesítésére szolgál ebbe a globális nézetbe, összesített metrikákat számítva, mint az average_confidence és a throughput (hármas/mp).

---

5.1 RELÁCIÓ- ÉS ENTITÁSSTATISZTIKÁK

Ez a szakasz technikai mélymerülést nyújt a CREVPipeline statisztikai nyomkövetési mechanizmusaiba. A csővezeték valós idejű metrikákat tart fenn minden egyedi relációtípushoz és entitáshoz, amelyet a kinyerési folyamat során fedez fel. Ezek a statisztikák elengedhetetlenek az anomáliapontszámok kiszámításához a hármas validálása során, és megfigyelhetőséget biztosítanak a Tudásgráf növekedéséhez.

RelationStatistics

A RelationStatistics struktúra nyomon követi az adott relációtípusok (pl. "located_in", "author_of") gyakoriságát, megbízhatóságát és varianciáját. A Welford online algoritmust használja a pontos futó variancia fenntartásához anélkül, hogy a hármasok teljes előzményét tárolná, ami kritikus a memóriahatékony folyamfeldolgozáshoz.

Adatstruktúra és Megvalósítás

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| count | u64 | Az adott relációtípus integrálásának összes száma. |
| total_confidence | f64 | A reláció összes példányának megbízhatósági pontszámainak összege. |
| avg_confidence | f64 | A megbízhatósági pontszámok futó átlaga. |
| m2 | f64 | Az átlagtól való eltérések négyzetösszege (a Welford-algoritmushoz használt). |

Kulcsfontosságú Függvények
- getVariance(self: RelationStatistics) f64: Visszaadja a megbízhatósági pontszámok varianciáját. Ha count < 2, 0,0-t ad vissza.
- getStdDev(self: RelationStatistics) f64: Visszaadja a szórást a variancia négyzetgyökének kiszámításával.

A Welford Algoritmus Integrációja

Amikor egy új hármas integrálódik, a csővezeték a következő logikával frissíti ezeket a statisztikákat:
1. A count növekszik.
2. Kiszámítódik az új megbízhatóság és a régi átlag közötti különbség (delta).
3. Az avg_confidence frissül.
4. Kiszámítódik az új megbízhatóság és az új átlag közötti különbség (delta2).
5. Az m2 frissül a delta * delta2 hozzáadásával.

Ez a logika a CREVPipeline.updateStats metódusban van megvalósítva.

EntityStatistics

Az EntityStatistics struktúra nyomon követi az adott karakterláncok előfordulását, ahogy azok egy RelationalTriplet subject vagy object pozíciójában megjelennek.

Adatstruktúra és Megvalósítás

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| count | u64 | Az entitás összes megjelenésének száma. |
| as_subject | u64 | Az entitás hármas alanyaként való megjelenésének gyakorisága. |
| as_object | u64 | Az entitás hármas tárgyaként való megjelenésének gyakorisága. |
| total_confidence | f64 | Kumulatív megbízhatósági pontszám az összes megjelenés alapján. |

Ezek a statisztikák lehetővé teszik a rendszer számára, hogy azonosítsa a gráf "csomópontjait" (magas count értékű entitások) és meghatározza az egyes entitások irányvonalát (alany-domináns vs. tárgy-domináns).

Tárolás és Kulcsolás

Mindkét statisztikai struktúrát a CREVPipeline kezeli std.StringHashMap segítségével.

HashMapek a CREVPipeline-ban

A csővezeték inicializálja ezeket a leképezéseket a metrikák tárolásához, a reláció vagy entitás nyers karakterlánc-reprezentációja szerint kulcsolva.
- relation_stats: StringHashMap(RelationStatistics)
- entity_stats: StringHashMap(EntityStatistics)

Memória Életciklus

A statisztikák az ExtractionStage.indexing fázis során frissülnek. Ha egy relációval vagy entitással először találkoznak, a csővezeték a csővezeték allocator-jával lefoglalja a karakterlánc-kulcs másolatát, biztosítva, hogy a kulcs megmaradjon a leképezésben.

Anomáliadetektálás Z-Pontszám Segítségével

A RelationStatistics elsődleges fogyasztója a validálási alrendszer. A validateTriplet során a rendszer kiszámít egy anomaly_score értéket azon alapulva, hogy a jelenlegi hármas megbízhatósága hány szórásnyira van az adott relációtípus korábbi átlagától.

A Képlet

Az anomaly_score a következőképpen vezethető le:
1. A RelationStatistics lekérése a hármas relációjához.
2. A szórás kiszámítása a getStdDev() segítségével.
3. Ha std_dev > 0,001, a pontszám: |confidence - avg_confidence| / std_dev.
4. Ha a reláció új vagy nincs varianciája, a pontszám alapértelmezés szerint 0,0.

Integráció a Validálásban

A computeAnomalyScore függvény végzi ezt a számítást. Egy magas anomáliapontszám (jellemzően > 3,0) azt jelzi, hogy a hármas megbízhatósága jelentősen eltér az adott reláció kialakult mintáitól, potenciálisan megjelölve azt elutasításra vagy manuális felülvizsgálatra.

Teljesítményszempontok
- O(1) Frissítések: A statisztikák frissítése O(1) művelet (HashMap keresés + állandó idejű aritmetika).
- Memóriaigény: A memória az egyedi relációtípusok és entitások számával skálázódik, nem a hármasok teljes számával.
- Pontosság: Az f64 használata az m2 és total_confidence értékekhez megakadályozza a jelentős pontosságveszteséget hosszan futó adatfolyamok esetén.

---

5.2 CSŐVEZETÉK-SZINTŰ METRIKÁK

A CREVPipeline rendszer részletes megfigyelhetőséget biztosít működésébe két elsődleges struktúrán keresztül: a PipelineResult, amely egyetlen feldolgozási hívás életciklusát követi nyomon, és a PipelineStatistics, amely a rendszerteljesítmény és adatintegritás globális, összesített nézetét nyújtja.

PipelineResult

Egy PipelineResult-ot ad vissza minden adatfolyam-feldolgozási belépési pont (pl. processTextStream, processStructuredDataStream). Rögzíti a kinyerési életciklus hatékonyságát és kimenetelét az öt ExtractionStage fázison keresztül.

Struktúra és Mezők

A PipelineResult struktúra számlálókat követ nyomon a csővezeték minden fázisához, lehetővé téve a fejlesztők számára, hogy azonosítsák, hol kerülnek szűrésre vagy buknak meg az adatok a validálás során.

| Mező | Típus | Leírás |
| :--- | :--- | :--- |
| triplets_extracted | u32 | A triplet_extraction szakasz során azonosított hármasok száma. |
| triplets_validated | u32 | A validation szakasz ellenőrzésein átment hármasok száma. |
| triplets_integrated | u32 | A ChaosCoreKernel-be sikeresen tartósan tárolt hármasok száma. |
| conflicts_resolved | u32 | A súlyozott megbízhatósági összevonási képletet kiváltó hármasok száma. |
| processing_time_ns | u64 | A hívásban töltött összes nanoszekundum. |
| stage | ExtractionStage | Az elért végső szakasz (hibajentéshez használt). |
| success | bool | Jelzi, hogy a hívás végzetes hibák nélkül fejeződött-e be. |
| error_message | ?[]const u8 | Részletes hibaleírás, ha a success hamis. |

Eredmény Összevonása

Nagy adatfolyamok feldolgozásakor vagy eredmények kötegelt kezelésekor a PipelineResult példányok kombinálhatók a merge metódus segítségével. Ez a metódus additív felhalmozást végez a számlálókon és megőrzi a legutóbbi hibaállapotot.

- Számlálók: Összeadódnak (pl. total.triplets_extracted += new.triplets_extracted).
- Siker: Logikai ÉS (ha bármely rész meghibásodik, az összevont eredmény tükrözi a hibát).
- Idő: Összesített feldolgozási idő.

PipelineStatistics

Míg a PipelineResult efemer, a PipelineStatistics a CREVPipeline kumulatív állapotát képviseli az inicializálás óta. Az adatok összesítésével kerül kiszámításra a KnowledgeGraphIndex, StreamBuffer és belső statisztikai leképezések alapján.

Kulcsfontosságú Teljesítménymutatók (KPI-k)

| Metrika | Forrás / Számítás |
| :--- | :--- |
| Áteresztőképesség | Számítása: total_triplets_integrated / (uptime_ms / 1000,0). |
| Konfliktusarány | A conflicts_resolved és a triplets_extracted aránya. |
| Átlagos Megbízhatóság | A KnowledgeGraphIndex-ben lévő összes integrált hármas átlagos megbízhatósági pontszáma. |
| Pufferkihasználtság | Jelenlegi StreamBuffer.size osztva StreamBuffer.capacity-vel. |
| Egyedi Számok | A subject_index, relation_index és object_index mérete. |

Megvalósítási Részletek

Áteresztőképesség és Üzemidő

A csővezeték az init során rögzíti az indítási időt a std.time.milliTimestamp() segítségével. Ez az uptime_ms mező kiszámítására szolgál a PipelineStatistics-ban, amely az áteresztőképesség-számítások nevezőjeként szolgál.

Integráció az Entitás/Reláció Statisztikákkal

A metrikarendszer a RelationStatistics és EntityStatistics struktúrákat használja az average_confidence metrika biztosításához. Amikor egy hármas integrálódik, a csővezeték frissíti az adott relációtípus RelationStatistics-át a Welford online algoritmus segítségével, amelyet aztán összesítenek a globális átlagba.

Értelmezés és Figyelés

A fejlesztőknek a következő mintákat kell figyelniük ezekben a metrikákban:

1. Magas Konfliktusarány: Ha a conflict_rate meghaladja a 0,20-at, ez azt jelzi, hogy a bemeneti adatfolyam gyakran ellentmond a meglévő tudásnak, vagy a ValidationStage túl agresszívan jelöli meg az átfedéseket.
2. Puffertelítettség: Ha a buffer_utilization megközelíti az 1,0-t, a StreamBuffer tele van. A csővezeték elkezdi kizárni a legrégebbi hármasokat az új adatok befogadásához, ami potenciálisan adatvesztéshez vezethet, ha a KnowledgeGraphIndex nem kerül elég gyakran lekérdezésre.
3. Megbízhatóság Csökkenése: Egy csökkenő average_confidence azt sugallja, hogy a kinyerési minták vagy a forrásadatok minősége idővel romlik.
4. Kinyerési Hatékonyság: A triplets_extracted és triplets_validated összehasonlítása a PipelineResult-ban segít a TokenizerConfig és a validálási küszöbértékek finomhangolásában.

---

6. KÜLSŐ INTEGRÁCIÓK

A CREVPipeline rendszer magas szintű vezénylési rétegként van tervezve, amely hidat képez a nyers adatfolyamok és a speciális alacsony szintű szubsztrátumok között. Ennek eléréséhez két elsődleges külső modulra támaszkodik: az nsir_core-ra a gráfelméleti reprezentációkhoz és a chaos_core-ra a tartós memóriakezeléshez.

Ez az oldal magas szintű áttekintést nyújt arról, hogyan integrálja a CREVPipeline ezeket a modulokat a munkafolyamatába.

Integráció Áttekintése

A csővezeték fordítóként működik, a validált RelationalTriplet adatokat ezeknek a külső függőségeknek megfelelő formátumokká alakítva.

- nsir_core: A Tudásgráf strukturális definícióját biztosítja. A csővezeték ezt arra használja, hogy a hármasokat komplex értékű, időbeli gráftérbe vetítse.
- chaos_core: A fizikai tárolási réteget biztosítja. A csővezeték ezt arra használja, hogy biztosítsa az integrált tudás megmaradását az alkalmazás memória-életciklusán túl.

nsir_core: Gráfreprezentáció

A CREVPipeline újraexportálja és felhasználja az nsir_core.zig típusait a Tudásgráf belső reprezentációjának felépítéséhez. Amikor egy hármas feldolgozásra kerül, Node és Edge struktúrákra bomlik.

Az integráció elsősorban a RelationalTriplet.toGraphElements metóduson keresztül történik, amely kiszámítja:
- Kvantumállapot: Egy Complex(f64), ahol a valós rész a megbízhatóságot, a képzetes rész a bizonytalanságot reprezentálja.
- Időbeli Fázis: Egy 360 másodperces cikluson alapuló számítás a kinyerés "frissességének" vagy időzítésének reprezentálásához.

Kódentitás Leképezés

| Fogalom | Kódentitás |
| :--- | :--- |
| Gráf Tároló | SelfSimilarRelationalGraph |
| Entitás Reprezentáció | Node |
| Reláció Reprezentáció | Edge |
| Él Osztályozás | EdgeQuality.coherent |

chaos_core: Memóriamag

Míg az nsir_core az adatok alakját kezeli, a chaos_core a tartós tárolást. A CREVPipeline mutatót tart egy ChaosCoreKernel példányra, amely az integrált hármasok foglalójaként szolgál.

Amikor egy hármas átmegy a validálási szakaszon, a csővezeték meghívja az allocateMemory-t a kernelen. Ez a lépés kritikus annak biztosításához, hogy az adatok a mögöttes chaos-kezelt memóriaterületen tárolódjanak. A csővezeték szigorú errdefer mintákat alkalmaz annak biztosítására, hogy ha egy kernel-foglalás meghiúsul, a csővezeték állapota konzisztens maradjon és ne szivárogjon memória.

Összefoglaló Táblázat: Külső Függőségek

| Függőség | Fájl Hivatkozás | Elsődleges Felhasználás a Csővezetékben |
| :--- | :--- | :--- |
| nsir_core | nsir_core.zig | Node és Edge geometria definiálása a Tudásgráfhoz. |
| chaos_core | chaos_core.zig | Alacsonyabb szintű memória szubsztrátum a ChaosCoreKernel-en keresztül. |

---

6.1 NSIR_CORE: GRÁFREPREZENTÁCIÓ

Ez az oldal leírja a csővezeték által a tudás reprezentálásához használt gráfalapú adatstruktúrákat. A rendszer újraexportálja és felhasználja az nsir_core modul típusait a kinyert relációs hármasok komplex értékű, önhasonló gráfreprezentációvá alakításához.

Gráfentitások Áttekintése

A csővezeték minden validált RelationalTriplet-et csomópontok és élek halmazára képez le egy SelfSimilarRelationalGraph-on belül. Ez a reprezentáció kvantum-inspirált állapotvektorokat használ a megbízhatóság és az időbeli fázisok nyomon követéséhez minden entitásnál és kapcsolatnál.

| Kódentitás | Leírás |
| :--- | :--- |
| SelfSimilarRelationalGraph | A relációs hálózat legfelső szintű tárolója. |
| Node | Egyedi entitást (Alanyt vagy Tárgyat) reprezentál 16 bájtos azonosítóval. |
| Edge | Irányított kapcsolatot reprezentál két csomópont között. |
| EdgeQuality | Felsorolás, amely meghatározza egy kapcsolat koherenciáját (jellemzően coherent). |

Relációs Tér Leképezése Gráftérre

A természetes nyelvi kinyerésekből a formális gráfstruktúrákba való átmenet a RelationalTriplet.toGraphElements() metóduson belül történik. Ez a függvény egyetlen hármast két csomópontra (Alany és Tárgy) és egy összekötő élre bont.

Identitás és Azonosítógenerálás

A csomópont-identitások az entitás karakterlánc-reprezentációjának SHA-256 hashelésével jönnek létre. A rendszer 32 bájtos hasht generál, de 16 bájtos azonosítóra csonkítja a Node struktúrához.
- Alany azonosítója: A hashTripletIdentity(subject, "", "") alapján vezethető le.
- Tárgy azonosítója: A hashTripletIdentity(object, "", "") alapján vezethető le.

Kvantumállapot és Időbeli Fázis

Az nsir_core reprezentáció Complex(f64)-et használ egy csomópont "Kvantumállapotának" reprezentálásához. Ez lehetővé teszi a rendszer számára, hogy kódolja mind egy entitás létezésének bizonyosságát, mind annak matematikai komplementerét.

Kvantumállapot Képlet

Az állapot a hármas megbízhatósági pontszámából (c) kerül kiszámításra:
- Valós Rész (Megbízhatóság): A megfigyelt entitás valószínűségét reprezentálja.
- Képzetes Rész (Bizonytalanság): A szuperpozíciós vagy "potenciális" állapotot reprezentálja, kiszámítva: sqrt(1 - c²).

A megvalósítás biztosítja, hogy c 0,0 és 1,0 közé legyen szorítva a számítás előtt.

Időbeli Fázis

Minden csomóponthoz időbeli fázis kerül hozzárendelésre az extraction_time alapján. Ez a következőképpen kerül kiszámításra:

Fázis = extraction_time mod 360

Ez egy ciklikus 360 másodperces időbeli koordináta-rendszert hoz létre, amelyet a gráfban lévő kapcsolódó csomópontok szinkronizálásához használnak.

Megvalósítási Részletek

A toGraphElements függvény az elsődleges belépési pont a gráfgeneráláshoz. Egy GraphElements struktúrát ad vissza, amely tartalmazza az inicializált csomópontokat és éleket.

Csomópont Inicializálás
A csomópontok a Node.initWithComplex(id, state, phase) segítségével inicializálódnak.
- Az azonosító az identitás-hash első 16 bájtja.
- Az állapot a fent leírt Complex(f64) érték.

Él Inicializálás
Az élek az Edge.initWithComplex(sub_id, obj_id, .coherent, weight) segítségével inicializálódnak.
- Az él súlya közvetlenül a hármas megbízhatóságából kerül leképezésre.
- A minőség keményen kódolt EdgeQuality.coherent értékre van állítva.

---

6.2 CHAOS_CORE: MEMÓRIAMAG

A chaos_core modul alacsony szintű memóriakezelési és foglalási szubsztrátumot biztosít a CREV csővezeték számára. A CREVPipeline-on belül a ChaosCoreKernel felelős az integrált hármasok hosszú távú megőrzéséért. Amikor egy RelationalTriplet átmegy a validáláson és belép az integrációs szakaszba, szerializált formája a kernel memóriaterületére kerül, biztosítva, hogy a Tudásgráf egy stabil, alacsonyabb szintű foglalási rétegen alapuljon.

Kapcsolat a CREVPipeline-nal

A CREVPipeline közvetlen mutatót tart fenn egy ChaosCoreKernel példányra. Ez a függőség a csővezeték inicializálásakor kerül befecskendezésre, és elsősorban az ExtractionStage.integration fázisban kerül felhasználásra.

Az allocateMemory Interfész

A csővezeték és a kernel közötti elsődleges interakciós pont az allocateMemory függvény. Ez a hívás egy stabil memóriacím megszerzésére szolgál, ahol a hármas adatai megőrizhetők.

Megvalósítás az Integrációs Szakaszban

Az integrációs szakasz során a csővezeték megpróbál memóriát biztosítani a kerneltől. Ez a folyamat biztonsági mintákba van csomagolva a kernel memóriaerőforrásainak kimerülése esetén.

| Jellemző | Leírás | Kódentitás |
| :--- | :--- | :--- |
| Függőség | Az összes tartós foglaláshoz használt kernel példány. | CREVPipeline.kernel |
| Foglalási Hívás | Memóriablokkot kér a kernel szubsztrátumtól. | ChaosCoreKernel.allocateMemory() |
| Biztonsági Minta | errdefer-t használ annak biztosítására, hogy a csővezeték állapota konzisztens maradjon, ha a kernel nem tud foglalni. | integrateTriplet logika |

Hibakezelés és Biztonság

A ChaosCoreKernel-lel való integráció kritikus útvonal. Ha a kernel nem tud memóriát biztosítani (pl. memóriahiány vagy töredezettség esetén), a csővezetéknek meg kell akadályoznia a KnowledgeGraphIndex részleges állapotfrissítéseit.

1. Validálási Ellenőrzés: A hármas először a validateTriplet-tel kerül ellenőrzésre, biztosítva, hogy csak magas megbízhatóságú adatok jussanak el a kernelhez.
2. Kernel Foglalás: A csővezeték meghívja a kernel.allocateMemory-t.
3. Errdefer Visszaállítás: Ha a foglalás meghiúsul, a csővezeték errdefer mintákat alkalmaz az előzetes számlálónövelések (pl. triplets_extracted) visszagörgetéséhez, mielőtt a hiba a hívóhoz propagálódna.
4. Indexelés: Csak azután, hogy a kernel sikeresen megőrizte az adatokat, kerül a hármas mutatója a KnowledgeGraphIndex-be (alany, reláció és tárgy leképezések).

---

7. SZÓJEGYZÉK

Ez a szójegyzék meghatározza a CREVPipeline-ban használt szakterület-specifikus terminológiát, technikai rövidítéseket és alapvető adatstruktúrákat. Referenciát biztosít az újonnan csatlakozó mérnökök számára a fogalmi rendszerviselkedések és azok konkrét megvalósításainak összekapcsolásához a crev_pipeline.zig fájlban.

Alapvető Rendszerfogalmak

Anomáliapontszám (Anomaly Score)
Statisztikai mérőszám, amelyet a Validálási szakasz során használnak annak meghatározására, hogy egy új hármas mennyire tér el a korábbi adatoktól. Kiszámítása a hármas megbízhatóságának Z-pontszámaként történik az adott relációtípushoz korábban integrált hármasok átlagához és varianciájához képest.
- Megvalósítás: A computeAnomalyScore függvényben kerül kiszámításra.
- Adatforrás: A relation_stats leképezésben tárolt RelationStatistics-t használja.

Kinyerési Szakasz (Extraction Stage)
A csővezeték diszkrét fázisai, amelyeken az adatoknak át kell haladniuk a sikeres indexeléshez. A csővezeték öt szakaszt definiál: tokenization, triplet_extraction, validation, integration és indexing.
- Megvalósítás: Az ExtractionStage enum definiálja.
- Folyamat: A next() függvény kezeli, amely meghatározza a lineáris előrehaladást.

Morfémaalapú Illesztés (Morpheme-Aware Matching)
Egy nyelvi illesztési stratégia, amely a szavakat alapvető "töveik" alapján hasonlítja össze, nem pedig pontos karakterlánc-egyenlőség alapján. Ez lehetővé teszi a csővezeték számára, hogy felismerje a relációkat akkor is, ha az igék vagy főnevek ragozottak vagy többes számban vannak (pl. a "created" illesztése a "create"-re).
- Megvalósítás: A wordsMatchStem és a stemWord algoritmus kezeli.
- Felhasználás: A matchPatternMorphemeAware-ben kerül alkalmazásra mintafelismeréshez.

---

Adatstruktúrák

RelationalTriplet
A rendszer alapvető információegysége, amely két entitás közötti irányított kapcsolatot reprezentál.
- Mezők: Tartalmaz subject, relation, object, confidence és metadata mezőket.
- Identitás: Az alany, reláció és tárgy karakterláncok hash-e határozza meg a hashTripletIdentity segítségével.
- Gráfkonverzió: A toGraphElements metódus nsir_core csomópontokká és élekké alakítja.

KnowledgeGraphIndex
Memóriabeli, háromtengelyes index, amely O(1) vagy O(N_jelölt) kereséseket tesz lehetővé a hármasokhoz azok három elsődleges komponense bármelyike alapján.
- Megvalósítás: Három StringHashMap struktúrát használ: subject_index, relation_index és object_index.
- Optimalizálás: A query metódus a három index közül a legkisebb jelölthalmazt választja ki az iteráció minimalizálásához.

StreamBuffer
Körkörös puffer (FIFO sor), amely RelationalTriplet mutatókat tárol. Lehetővé teszi az aszinkron stílusú feldolgozást, ahol a csővezeték folyamatosan tud adatokat befogadni.
- Megvalósítás: StreamBuffer-ként definiálva.
- Kizárási Szabályzat: Ha a puffer tele van, a legrégebbi hármas kerül kizárásra az új adatok számára helyet csinálva.

---

Technikai Leképezés

Természetes Nyelvtől a Kódentitás-térig

A következő táblázat a magas szintű nyelvi fogalmakat a konkrét Zig struktúrákra és függvényekre képezi le, amelyek feldolgozzák azokat.

| Természetes Nyelvi Fogalom | Kódentitás |
| :--- | :--- |
| Nyers szövegfolyam | processTextStream() |
| Mondat | tokenizeIntoWords() |
| Szóváltozatok (pl. "running") | stemWord() |
| Morfémaillesztés | wordsMatchStem() / matchPatternMorphemeAware() |
| Relációs minta | RelationPattern |
| Kinyert tény | RelationalTriplet |

---

Kulcsfontosságú Rövidítések

| Rövidítés | Teljes Kifejezés | Kontextus |
| :--- | :--- | :--- |
| Welford | Welford Online Algoritmus | A RelationStatistics-ban használják a futó variancia és átlag kiszámításához az összes minta tárolása nélkül. |
| NSIR | Non-Spatial Information Representation (Nem-térbeli Információreprezentáció) | Az nsir_core modulra utal, amely kezeli az alapul szolgáló gráfmatematikát és a kvantumállapotot. |
| CCK | Chaos Core Kernel | A ChaosCoreKernel interfész, amelyet tartós memóriafoglaláshoz használnak. |
| M2 | Eltérések négyzetösszege | A RelationStatistics egyik mezője, amelyet a Welford algoritmushoz használnak. |

---

Statisztikai Fogalmak

RelationStatistics
Nyomkövető minden egyedi relációtípushoz (pl. "is_a", "located_in"). Fenntartja az összesített megbízhatóságot és varianciát, amelyek az anomáliadetektáláshoz szükségesek.
- Megvalósítás: RelationStatistics struktúra.
- Metódus: Az update(confidence) módosítja az átlagot és az m2 értékeket.

EntityStatistics
Nyomkövető minden egyedi entitáshoz (alany vagy tárgy). Figyeli, hogy egy entitás milyen gyakran jelenik meg, és milyen az átlagos megbízhatósága az összes kapcsolatán keresztül.
- Megvalósítás: EntityStatistics struktúra.
- Mezők: Tartalmaz as_subject és as_object számlálókat.
