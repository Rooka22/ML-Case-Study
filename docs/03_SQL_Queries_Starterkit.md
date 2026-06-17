# SQL Starterkit – Warenkorbanalyse in BigQuery

> **Dataset**: `itse-a.ml_itse_cleaned`  
> **Ausführung**: Alle Queries laufen direkt in der BigQuery Console (Web UI)  
> **Ziel**: Daten verstehen, erste Muster erkennen, Co-Occurrence-Matrix bauen

---

## 1. Daten-Profiling – Basis-Verständnis

### 1.1 Gesamtüberblick

```sql
-- Einmaliger Überblick: Volumen, Zeitraum, Kunden, Artikel
SELECT
  COUNT(*)                                                    AS anzahl_bon_positionen,
  COUNT(DISTINCT CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)))
                                                                AS anzahl_warenkoerbe,
  COUNT(DISTINCT kundenNummer)                                AS anzahl_kunden,
  COUNT(DISTINCT artikelNummer)                               AS anzahl_artikel,
  COUNT(DISTINCT filialNummer)                                AS anzahl_filialen,
  MIN(tag)                                                    AS erster_tag,
  MAX(tag)                                                    AS letzter_tag,
  DATE_DIFF(MAX(tag), MIN(tag), MONTH)                        AS monate_abgedeckt
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`;
```

### 1.2 Korbgrößen-Verteilung

```sql
-- Wie viele Artikel sind durchschnittlich in einem Korb?
-- Gibt Auskunft über die Komplexität der Analyse
SELECT
  basket_size,
  COUNT(*) AS anzahl_koerbe,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS prozent_anteil
FROM (
  SELECT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    COUNT(DISTINCT artikelNummer) AS basket_size
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
  GROUP BY basket_id
)
GROUP BY basket_size
ORDER BY basket_size;
```

### 1.3 Aktions-Anteil

```sql
-- Wie viel Prozent der Käufe sind Aktionsware?
SELECT
  inAktion,
  COUNT(*) AS anzahl_positionen,
  COUNT(DISTINCT CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING))) AS anzahl_koerbe,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS prozent_positionen
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
GROUP BY inAktion;
```

### 1.4 Aktions-Körbe vs. Normal-Körbe

```sql
-- Wie viele Körbe enthalten mindestens einen Aktionsartikel?
-- Körbe NUR mit Aktionsartikeln vs. NUR mit Normalpreis vs. gemischt
SELECT
  CASE
    WHEN SUM(CASE WHEN inAktion = TRUE THEN 1 ELSE 0 END) > 0
     AND SUM(CASE WHEN inAktion = FALSE THEN 1 ELSE 0 END) = 0 THEN 'nur_aktion'
    WHEN SUM(CASE WHEN inAktion = FALSE THEN 1 ELSE 0 END) > 0
     AND SUM(CASE WHEN inAktion = TRUE THEN 1 ELSE 0 END) = 0 THEN 'nur_normal'
    ELSE 'gemischt'
  END AS korb_typ,
  COUNT(*) AS anzahl_koerbe,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS prozent
FROM (
  SELECT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    MAX(inAktion) AS inAktion
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
  GROUP BY basket_id
)
GROUP BY korb_typ
ORDER BY anzahl_koerbe DESC;
```

### 1.5 Top-Artikel (nach Häufigkeit in Körben)

```sql
-- Welche Artikel kommen in den meisten Körben vor?
SELECT
  SC.artikelNummer,
  TA.full_artikel_beschreibung,
  COUNT(DISTINCT CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING))) AS anzahl_koerbe,
  SUM(SC.abverkaufsMenge) AS gesamt_menge,
  ROUND(SUM(SC.vkNettoBereinigtEuro), 2) AS umsatz_netto
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging` AS SC
LEFT JOIN `itse-a.ml_itse_cleaned.T_Artikel_curated` AS TA
  ON SC.artikelNummer = TA.artikelNummer
GROUP BY SC.artikelNummer, TA.full_artikel_beschreibung
ORDER BY anzahl_koerbe DESC
LIMIT 100;
```

### 1.6 Filialen & Bundesländer

```sql
-- Verteilung der Körbe über Filialen und Bundesländer
SELECT
  FB.bundesLand,
  SC.filialNummer,
  COUNT(DISTINCT CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING))) AS anzahl_koerbe,
  COUNT(*) AS anzahl_positionen,
  ROUND(SUM(SC.vkNettoBereinigtEuro), 2) AS umsatz_netto
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging` AS SC
LEFT JOIN `itse-a.ml_itse_cleaned.T_filiale_bundesland` AS FB
  ON SC.filialNummer = FB.filialNummer
GROUP BY FB.bundesLand, SC.filialNummer
ORDER BY anzahl_koerbe DESC;
```

### 1.7 Zeitliche Verteilung

```sql
-- Körbe pro Monat – um Saisonalität zu erkennen
SELECT
  EXTRACT(YEAR FROM tag) AS jahr,
  EXTRACT(MONTH FROM tag) AS monat,
  COUNT(DISTINCT CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING))) AS anzahl_koerbe,
  COUNT(*) AS anzahl_positionen,
  ROUND(SUM(vkNettoBereinigtEuro), 2) AS umsatz_netto
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
GROUP BY jahr, monat
ORDER BY jahr, monat;
```

### 1.8 Wochentag-Verteilung

```sql
-- An welchen Tagen wird wie viel gekauft?
SELECT
  EXTRACT(DAYOFWEEK FROM tag) AS wochentag_nr,
  FORMAT_DATE('%A', tag) AS wochentag_name,
  COUNT(DISTINCT CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING))) AS anzahl_koerbe,
  ROUND(AVG(basket_groesse), 1) AS avg_korbgroesse
FROM (
  SELECT
    tag,
    filialNummer,
    bonNummer,
    COUNT(DISTINCT artikelNummer) AS basket_groesse
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
  GROUP BY tag, filialNummer, bonNummer
)
GROUP BY wochentag_nr, wochentag_name
ORDER BY wochentag_nr;
```

### 1.9 Datenqualitäts-Check

```sql
-- Fehlende Werte und Auffälligkeiten prüfen
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(kundenNummer IS NULL) AS null_kunden,
  COUNTIF(kundenNummer = '') AS leere_kunden,
  COUNTIF(artikelNummer IS NULL) AS null_artikel,
  COUNTIF(bonNummer IS NULL) AS null_bon,
  COUNTIF(bonNummer = '') AS leere_bon,
  COUNTIF(abverkaufsMenge IS NULL OR abverkaufsMenge <= 0) AS ungueltige_menge,
  COUNTIF(vkNettoBereinigtEuro IS NULL) AS null_preis,
  COUNTIF(vkNettoBereinigtEuro < 0) AS negativer_preis,
  COUNTIF(inAktion IS NULL) AS null_aktion
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`;
```

---

## 2. Co-Occurrence-Analyse (2er-Kombinationen)

### 2.1 Alle Artikel-Paare mit Support

```sql
-- Findet alle Paare (A, B), die zusammen in einem Korb vorkommen
-- Berechnet Support = Häufigkeit(A+B) / Alle Körbe
WITH basket_base AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
),
total_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM basket_base
)
SELECT
  A.artikelNummer AS artikel_a,
  B.artikelNummer AS artikel_b,
  COUNT(*) AS gemeinsam,
  ROUND(COUNT(*) * 100.0 / (SELECT n_total FROM total_baskets), 4) AS support_prozent
FROM basket_base A
JOIN basket_base B
  ON A.basket_id = B.basket_id
  AND A.artikelNummer < B.artikelNummer   -- verhindert Doppelungen (A,B) und (B,A)
GROUP BY A.artikelNummer, B.artikelNummer
HAVING gemeinsam >= 5                      -- Mindesthürde: mind. 5 gemeinsame Körbe
ORDER BY gemeinsam DESC
LIMIT 200;
```

### 2.2 Top-Paare mit Support, Confidence und Lift

```sql
-- Die vollständige Metrik: Support, Confidence, Lift
-- Das ist die Kern-Query für die Warenkorbanalyse!
WITH basket_base AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
),
total_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM basket_base
),
item_frequency AS (
  SELECT
    artikelNummer,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM basket_base
  GROUP BY artikelNummer
),
pairs AS (
  SELECT
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM basket_base A
  JOIN basket_base B
    ON A.basket_id = B.basket_id
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 5
)
SELECT
  P.artikel_a,
  P.artikel_b,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 4) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs P
JOIN item_frequency FA ON P.artikel_a = FA.artikelNummer
JOIN item_frequency FB ON P.artikel_b = FB.artikelNummer
CROSS JOIN total_baskets T
ORDER BY lift DESC
LIMIT 200;
```

### 2.3 Top-Paare nach Confidence (stärkste Kopplung)

```sql
-- Welche Paare haben die höchste Confidence?
-- D.h.: Wenn A gekauft wird, wie oft ist auch B dabei?
-- Gleiche Query wie 2.2, nur anders sortiert
WITH basket_base AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
),
total_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM basket_base
),
item_frequency AS (
  SELECT
    artikelNummer,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM basket_base
  GROUP BY artikelNummer
),
pairs AS (
  SELECT
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM basket_base A
  JOIN basket_base B
    ON A.basket_id = B.basket_id
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 5
)
SELECT
  P.artikel_a,
  P.artikel_b,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 4) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs P
JOIN item_frequency FA ON P.artikel_a = FA.artikelNummer
JOIN item_frequency FB ON P.artikel_b = FB.artikelNummer
CROSS JOIN total_baskets T
ORDER BY confidence_prozent DESC
LIMIT 200;
```

### 2.4 Top-Paare nach Support (häufigste Kombinationen)

```sql
-- Welche Paare kommen in den meisten Körben vor?
-- Gleiche Query, sortiert nach Support
WITH basket_base AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
),
total_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM basket_base
),
item_frequency AS (
  SELECT
    artikelNummer,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM basket_base
  GROUP BY artikelNummer
),
pairs AS (
  SELECT
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM basket_base A
  JOIN basket_base B
    ON A.basket_id = B.basket_id
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 5
)
SELECT
  P.artikel_a,
  P.artikel_b,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 4) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs P
JOIN item_frequency FA ON P.artikel_a = FA.artikelNummer
JOIN item_frequency FB ON P.artikel_b = FB.artikelNummer
CROSS JOIN total_baskets T
ORDER BY support_prozent DESC
LIMIT 200;
```

---

## 3. Aktions-Vergleich (inAktion)

### 3.1 Paare NUR in Aktions-Körben

```sql
-- Welche Paare werden zusammen gekauft, wenn mind. einer der Artikel im Angebot ist?
WITH aktion_baskets AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
  WHERE inAktion = TRUE
),
total_aktion_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM aktion_baskets
),
item_freq_aktion AS (
  SELECT
    artikelNummer,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM aktion_baskets
  GROUP BY artikelNummer
),
pairs_aktion AS (
  SELECT
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM aktion_baskets A
  JOIN aktion_baskets B
    ON A.basket_id = B.basket_id
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 3
)
SELECT
  P.artikel_a,
  P.artikel_b,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 4) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs_aktion P
JOIN item_freq_aktion FA ON P.artikel_a = FA.artikelNummer
JOIN item_freq_aktion FB ON P.artikel_b = FB.artikelNummer
CROSS JOIN total_aktion_baskets T
ORDER BY lift DESC
LIMIT 100;
```

### 3.2 Paare NUR in Normalpreis-Körben

```sql
-- Gleiche Analyse, aber nur für Körbe ohne Aktionsartikel
WITH normal_baskets AS (
  SELECT DISTINCT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    artikelNummer
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging`
  WHERE inAktion = FALSE
),
total_normal_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM normal_baskets
),
item_freq_normal AS (
  SELECT
    artikelNummer,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM normal_baskets
  GROUP BY artikelNummer
),
pairs_normal AS (
  SELECT
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM normal_baskets A
  JOIN normal_baskets B
    ON A.basket_id = B.basket_id
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 3
)
SELECT
  P.artikel_a,
  P.artikel_b,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 4) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs_normal P
JOIN item_freq_normal FA ON P.artikel_a = FA.artikelNummer
JOIN item_freq_normal FB ON P.artikel_b = FB.artikelNummer
CROSS JOIN total_normal_baskets T
ORDER BY lift DESC
LIMIT 100;
```

### 3.3 Direkter Vergleich: Lift(Aktion) vs. Lift(Normal)

```sql
-- Vergleich: Steigt der Lift in Aktions-Körben im Vergleich zu Normal-Körben?
-- Findet Paare, die bei Aktion besonders stark koppeln
WITH aktion AS (
  -- Query 3.1 als CTE (hier vereinfacht)
  SELECT artikel_a, artikel_b, lift AS lift_aktion
  FROM (...)  -- Platzhalter: Ergebnis von 3.1 hier einfügen
),
normal AS (
  -- Query 3.2 als CTE
  SELECT artikel_a, artikel_b, lift AS lift_normal
  FROM (...)  -- Platzhalter: Ergebnis von 3.2 hier einfügen
)
SELECT
  COALESCE(A.artikel_a, N.artikel_a) AS artikel_a,
  COALESCE(A.artikel_b, N.artikel_b) AS artikel_b,
  A.lift_aktion,
  N.lift_normal,
  ROUND(A.lift_aktion / NULLIF(N.lift_normal, 0), 2) AS lift_veraenderung
FROM aktion A
FULL OUTER JOIN normal N
  ON A.artikel_a = N.artikel_a AND A.artikel_b = N.artikel_b
WHERE A.lift_aktion > N.lift_normal
  OR N.lift_normal IS NULL
ORDER BY lift_veraenderung DESC
LIMIT 50;
```

---

## 4. Regionale Analyse (nach Bundesland)

### 4.1 Top-Paare pro Bundesland

```sql
-- Welche Paare koppeln in welchem Bundesland besonders stark?
WITH basket_base AS (
  SELECT DISTINCT
    CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING)) AS basket_id,
    SC.artikelNummer,
    FB.bundesLand
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging` AS SC
  LEFT JOIN `itse-a.ml_itse_cleaned.T_filiale_bundesland` AS FB
    ON SC.filialNummer = FB.filialNummer
),
bundesland_stats AS (
  SELECT
    bundesLand,
    COUNT(DISTINCT basket_id) AS n_baskets_bl
  FROM basket_base
  GROUP BY bundesLand
),
pairs_bl AS (
  SELECT
    A.bundesLand,
    A.artikelNummer AS artikel_a,
    B.artikelNummer AS artikel_b,
    COUNT(*) AS n_ab
  FROM basket_base A
  JOIN basket_base B
    ON A.basket_id = B.basket_id
    AND A.bundesLand = B.bundesLand
    AND A.artikelNummer < B.artikelNummer
  GROUP BY A.bundesLand, A.artikelNummer, B.artikelNummer
  HAVING n_ab >= 3
)
SELECT
  P.bundesLand,
  P.artikel_a,
  P.artikel_b,
  P.n_ab,
  BS.n_baskets_bl,
  ROUND(P.n_ab * 100.0 / BS.n_baskets_bl, 4) AS support_prozent
FROM pairs_bl P
JOIN bundesland_stats BS ON P.bundesLand = BS.bundesLand
ORDER BY P.bundesLand, P.n_ab DESC;
```

---

## 5. Warengruppen-Analyse (Kategorie-Ebene)

### 5.1 Warengruppen-Paare (wg3 – feinste Kategorie)

```sql
-- Welche Warengruppen werden zusammen gekauft?
-- Statt einzelner Artikel werden ganze Kategorien verglichen
WITH basket_wg AS (
  SELECT DISTINCT
    CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING)) AS basket_id,
    TA.wg3Id
  FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging` AS SC
  LEFT JOIN `itse-a.ml_itse_cleaned.T_Artikel_curated` AS TA
    ON SC.artikelNummer = TA.artikelNummer
  WHERE TA.wg3Id IS NOT NULL
),
total_baskets AS (
  SELECT COUNT(DISTINCT basket_id) AS n_total FROM basket_wg
),
wg_frequency AS (
  SELECT
    wg3Id,
    COUNT(DISTINCT basket_id) AS n_baskets
  FROM basket_wg
  GROUP BY wg3Id
),
pairs_wg AS (
  SELECT
    A.wg3Id AS wg_a,
    B.wg3Id AS wg_b,
    COUNT(*) AS n_ab
  FROM basket_wg A
  JOIN basket_wg B
    ON A.basket_id = B.basket_id
    AND A.wg3Id < B.wg3Id
  GROUP BY A.wg3Id, B.wg3Id
  HAVING n_ab >= 10
)
SELECT
  P.wg_a,
  P.wg_b,
  WA.wgBezeichnung AS wg_a_name,
  WB.wgBezeichnung AS wg_b_name,
  FA.n_baskets AS n_a,
  FB.n_baskets AS n_b,
  P.n_ab,
  ROUND(P.n_ab * 100.0 / T.n_total, 2) AS support_prozent,
  ROUND(P.n_ab * 100.0 / FA.n_baskets, 2) AS confidence_prozent,
  ROUND((P.n_ab * 1.0 / FA.n_baskets) / (FB.n_baskets * 1.0 / T.n_total), 2) AS lift
FROM pairs_wg P
JOIN wg_frequency FA ON P.wg_a = FA.wg3Id
JOIN wg_frequency FB ON P.wg_b = FB.wg3Id
JOIN `itse-a.ml_itse_cleaned.T_Warengruppen_curated` WA ON P.wg_a = WA.wgId
JOIN `itse-a.ml_itse_cleaned.T_Warengruppen_curated` WB ON P.wg_b = WB.wgId
CROSS JOIN total_baskets T
ORDER BY lift DESC
LIMIT 100;
```

### 5.2 Eigenmarken-Analyse

```sql
-- Werden Eigenmarken zusammen mit bestimmten Artikeln gekauft?
SELECT
  SC.artikelNummer,
  TA.full_artikel_beschreibung,
  TA.eigenMarke,
  COUNT(DISTINCT CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING))) AS anzahl_koerbe,
  SUM(SC.abverkaufsMenge) AS menge
FROM `itse-a.ml_itse_cleaned.T_Scannerzahlen_staging` AS SC
LEFT JOIN `itse-a.ml_itse_cleaned.T_Artikel_curated` AS TA
  ON SC.artikelNummer = TA.artikelNummer
WHERE TA.eigenMarke = TRUE
GROUP BY SC.artikelNummer, TA.full_artikel_beschreibung, TA.eigenMarke
ORDER BY anzahl_koerbe DESC
LIMIT 50;
```

---

## 6. Saisonale Analyse (Schulanfang)

### 6.1 Schulanfangs-Perioden anzeigen

```sql
-- Welche Zeiträume sind als Schulanfang markiert?
SELECT * FROM `itse-a.ml_itse_cleaned.T_Schulanfang_Basis`
ORDER BY referenz_tag;
```

### 6.2 Top-Artikel in Schulanfangs-Zeit

```sql
-- Welche Artikel werden in Schulanfangs-Perioden besonders viel gekauft?
SELECT
  SA.artikel_nummer,
  TA.full_artikel_beschreibung,
  SUM(SA.abverkaufsmenge) AS menge_schulanfang,
  COUNT(DISTINCT SA.artikel_nummer) AS anzahl_koerbe
FROM `itse-a.ml_itse_cleaned.T_Schulanfang_ArtikelNummer_Basis` AS SA
LEFT JOIN `itse-a.ml_itse_cleaned.T_Artikel_curated` AS TA
  ON SA.artikel_nummer = TA.artikelNummer
GROUP BY SA.artikel_nummer, TA.full_artikel_beschreibung
ORDER BY menge_schulanfang DESC
LIMIT 50;
```

---

## 7. Ergebnis-Tabellen anlegen (für spätere Nutzung)

```sql
-- Sobald die Analysen laufen, Ergebnisse in Tabellen speichern
-- Dann können Looker/Data Studio darauf zugreifen

CREATE OR REPLACE TABLE `itse-a.ml_itse_cleaned.ergebnis_paare_alle` AS
-- Hier Query 2.2 einfügen
;

CREATE OR REPLACE TABLE `itse-a.ml_itse_cleaned.ergebnis_paare_aktion` AS
-- Hier Query 3.1 einfügen
;

CREATE OR REPLACE TABLE `itse-a.ml_itse_cleaned.ergebnis_paare_normal` AS
-- Hier Query 3.2 einfügen
;

CREATE OR REPLACE TABLE `itse-a.ml_itse_cleaned.ergebnis_warengruppen_paare` AS
-- Hier Query 5.1 einfügen
;
```

---

## Empfohlene Reihenfolge

| Schritt | Query | Erwartung |
|---|---|---|
| 1 | **1.1 Gesamtüberblick** | Verstehen, wie viele Daten da sind |
| 2 | **1.9 Datenqualität** | Prüfen, ob die Daten sauber sind |
| 3 | **1.2 Korbgrößen** | Wie komplex sind die Körbe? |
| 4 | **1.3 + 1.4 Aktions-Anteil** | Wie viel Aktion? |
| 5 | **1.5 Top-Artikel** | Welche Artikel sind relevant? |
| 6 | **2.2 Top-Paare (Lift)** | Erste Regeln finden |
| 7 | **3.1 + 3.2 Aktion vs. Normal** | Aktions-Effekt messen |
| 8 | **5.1 Warengruppen** | Kategorie-Ebene |
| 9 | **4.1 Regional** | Filial-Unterschiede |
| 10 | **6.2 Schulanfang** | Saisonales Muster |
