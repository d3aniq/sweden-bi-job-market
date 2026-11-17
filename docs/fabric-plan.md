# Plan: samma pipeline i Microsoft Fabric

**Status: plan. Inget av detta är byggt än.** Planen beskriver hur samma
data, samma semantiska modell och samma rapport kan köras i Microsoft
Fabric, utan att analysen byggs om från början. Det som faktiskt blir gjort
redovisas i README:n när det finns på GitHub.

---

## 1. Varför

Pipelinen i dag: inläsning i Python till bronze, dbt med DuckDB för silver
och gold (med kvalitetsgrindar som stoppar körningen), export av gold till
Parquet, och en Power BI-modell och rapport skrivna som kod (TMDL och PBIR).

Frågan för Fabric-versionen är densamma som för hela projektet: **vad
händer när samma sak beskrivs på två ställen?** Här är det två plattformar
i stället för två källor. Det intressanta är vad som går att flytta som det
är, vad som måste skrivas om, och varför.

Allt läggs i mappen `fabric/` i det här repot, så att historien hänger ihop
och befintliga länkar fortsätter att fungera.

---

## 2. Steg 0: förutsättningar

- **Fabric-miljö.** Kontrollera vilken provperiod eller kapacitet som går
  att få. Fabric har tidigare krävt ett organisationskonto för
  provperioden. Det avgör hela upplägget och kontrolleras först.
- **Datavolym.** Gold-filerna i `powerbi/data` är cirka 11 MB. Bronze med
  alla 7,7 miljoner annonser är betydligt större. Mät innan något laddas,
  så att det ryms i den kapacitet som finns.

---

## 3. Fas 1: gold till Lakehouse och rapport med Direct Lake

Snabbaste vägen till ett visbart resultat.

1. **Lakehouse.** Läs in de tolv gold-tabellerna från Parquet som
   Delta-tabeller, med en notebook eller en Data Pipeline.
2. **Semantisk modell med Direct Lake.** Återanvänd TMDL-modellen. Mått,
   relationer och tabellnamn behålls; bara partitionerna byts från
   Parquet-import till Direct Lake. Att modellen som kod flyttar med är
   huvudpoängen.
3. **Rapporten (PBIR)** kopplas mot den nya modellen.
4. **Git-integration.** Arbetsytan kopplas till repot, så att modell och
   rapport synkas som kod.

**Resultat:** samma rapport, men datan ligger i OneLake och läses med
Direct Lake.

---

## 4. Fas 2: transformeringen i Fabric

Två vägar. Valet görs efter en kort förstudie.

| Väg | Fördel | Nackdel |
|---|---|---|
| **A. dbt mot Fabric Warehouse** (adaptern dbt-fabric) | Samma dbt-projekt med två adaptrar, DuckDB och Fabric | DuckDB-specifik SQL måste skrivas om till T-SQL |
| **B. PySpark-notebooks i Lakehouse** | Nära Spark-arbetet i `sweden-power-lakehouse` | dbt-testerna måste göras om |

**Förstudien:** gå igenom modellerna i `dbt/models`, räkna DuckDB-specifika
funktioner och uppskatta omskrivningen. Väg A väljs om den är rimlig.

**Kvalitetsgrindarna följer med oavsett väg**, bland annat
`bronze_rows_match_load_log`, `volume_balances_with_silver`,
`no_personal_data_in_gold` och `dbt_not_in_care_occupations`. En körning
som inte klarar dem ska stoppas, precis som i dag.

---

## 5. Fas 3: schemaläggning och redovisning

- **Data Pipeline** som kör kedjan bronze → silver → gold → modell, med
  testerna som grind.
- **README:** ett avsnitt "Samma modell i Fabric" med arkitekturbild, vad
  som återanvändes, vad som skrevs om och vilka skillnader som finns mellan
  DuckDB och Fabric. Ärliga skillnader är mer intressanta än att allt gick
  bra.

---

## 6. Risker

- **Konto eller kapacitet** kan stoppa projektet. Därför steg 0 först.
- **Omskrivningen till T-SQL** kan bli större än väntat. Därför förstudien.
- **Omfattning.** Fas 1 ensam är ett komplett, visbart resultat. Fas 2 och
  3 är tillägg.

## 7. Uppskattad tid

| Del | Tid |
|---|---|
| Steg 0 | 1 kväll |
| Fas 1 | 3–4 kvällar |
| Fas 2 | 4–6 kvällar, inklusive förstudie |
| Fas 3 | 2 kvällar |
