# Warenkorbanalyse – Plan & Entscheidungen

> **Dataset**: `amgdev_curated` (BigQuery, GCP)

## Übersicht

| Aspekt | Entscheidung |
|---|---|
| **Speicher** | BigQuery (GCP-nativ) – Dataset `amgdev_curated` |
| **Basket-Definition** | `bonNummer` + `tag` + `filialNummer` = eindeutiger Warenkorb |
| **Granularität** | SKU (`artikelNummer`) / Produkt/Marke (`produktNummer`) / Kategorie (`wg1Id`/`wg2Id`/`wg3Id`) – parallel |
| **Use Cases** | Cross-Selling, Sortimentsplanung, Prospekte, Filial-Logistik, Lieferanten-Konditionen, saisonale Muster |
| **Latenz** | Ad-hoc / One-Shot (initial) |
| **Algorithmus** | FP-Growth (via `mlxtend` in Vertex AI Workbench) |
| **Aktions-Kennzeichen** | ✅ `inAktion` (BOOL) direkt in `T_Scanner_curated` |
| **Warengruppen** | ✅ 3 Ebenen in `T_Warengruppen_curated` (selbst-referenzierend) |
| **Filial-Geografie** | ✅ `T_filiale_bundesland` (Filiale → Bundesland) |
| **Embeddings** | ✅ `T_Artikel_embeddings` + `T_Produkte_embeddings` (für Advanced) |

---

## Datenquellen (BigQuery-Tabellen)

| Tabelle | Inhalt | Schlüssel |
|---|---|---|
| `T_Scanner_curated` | **Bon-Positionen** – jede Zeile = ein gekaufter Artikel | `bonNummer` + `tag` + `filialNummer` = Korb-ID |
| `T_Artikel_curated` | Artikel-Stamm (SKU, Warengruppen, Produkt-Nummer) | `artikelNummer` |
| `T_Produkte_curated` | Produkt-Bezeichnungen (Marken-Ebene) | `produktNummer` |
| `T_Warengruppen_curated` | Warengruppen-Hierarchie (3 Ebenen) | `wgId`, `uebergeordneteWgId` |
| `T_filiale_bundesland` | Filiale → Bundesland | `filialNummer` |
| `artikel_warengruppe` | Denormalisiertes Mapping Artikel → Warengruppe | `artikelNummer` |

---

## Schritt 1: Daten-Profiling in BigQuery

Ziel: Datenvolumen, Zeitraum, Qualität und Struktur verstehen, bevor die Analyse-Tabelle gebaut wird.

### 1.1 Verfügbare Transaktionen & Zeitraum

```sql
SELECT
  COUNT(DISTINCT CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)))
                                                    AS n_baskets,
  COUNT(DISTINCT kundenNummer)                      AS n_customers,
  COUNT(DISTINCT artikelNummer)                     AS n_distinct_articles,
  COUNT(*)                                           AS n_line_items,
  MIN(tag)                                           AS first_date,
  MAX(tag)                                           AS last_date,
  DATE_DIFF(MAX(tag), MIN(tag), MONTH)               AS months_covered
FROM `amgdev_curated.T_Scanner_curated`;
```

### 1.2 Korbgrößen-Verteilung

```sql
SELECT
  basket_size,
  COUNT(*) AS n_baskets
FROM (
  SELECT
    CONCAT(CAST(filialNummer AS STRING), '_', CAST(bonNummer AS STRING), '_', CAST(tag AS STRING)) AS basket_id,
    COUNT(DISTINCT artikelNummer) AS basket_size
  FROM `amgdev_curated.T_Scanner_curated`
  GROUP BY basket_id
)
GROUP BY basket_size
ORDER BY basket_size;
```

### 1.3 Aktions- vs. Normalpreis-Anteil

```sql
SELECT
  inAktion,
  COUNT(*)                    AS n_line_items,
  ROUND(COUNT(*) * 100.0 / SUM(COUNT(*)) OVER(), 2) AS pct
FROM `amgdev_curated.T_Scanner_curated`
GROUP BY inAktion;
```

### 1.4 Filial-Verteilung (nach Bundesland)

```sql
SELECT
  FB.bundesLand,
  COUNT(DISTINCT CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING))) AS n_baskets,
  COUNT(*) AS n_line_items
FROM `amgdev_curated.T_Scanner_curated` AS SC
LEFT JOIN `amgdev_curated.T_filiale_bundesland` AS FB
  ON SC.filialNummer = FB.filialNummer
GROUP BY FB.bundesLand
ORDER BY n_baskets DESC;
```

### 1.5 Top-Artikel (Menge)

```sql
SELECT
  SC.artikelNummer,
  TA.full_artikel_beschreibung,
  SUM(SC.abverkaufsMenge) AS total_menge,
  COUNT(DISTINCT CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING))) AS n_baskets
FROM `amgdev_curated.T_Scanner_curated` AS SC
LEFT JOIN `amgdev_curated.T_Artikel_curated` AS TA
  ON SC.artikelNummer = TA.artikelNummer
GROUP BY SC.artikelNummer, TA.full_artikel_beschreibung
ORDER BY total_menge DESC
LIMIT 50;
```

### 1.6 Datenqualität: NULLs & Auffälligkeiten

```sql
SELECT
  COUNT(*) AS total_rows,
  COUNTIF(kundenNummer IS NULL) AS null_kunden,
  COUNTIF(artikelNummer IS NULL) AS null_artikel,
  COUNTIF(bonNummer IS NULL) AS null_bon,
  COUNTIF(abverkaufsMenge IS NULL OR abverkaufsMenge = 0) AS null_oder_null_menge,
  COUNTIF(vkNettoBereinigtEuro IS NULL) AS null_preis,
  COUNTIF(inAktion IS NULL) AS null_aktion
FROM `amgdev_curated.T_Scanner_curated`;
```

---

## Schritt 2: Basis-Tabelle bauen (BigQuery View)

Ziel: Eine flache, analysierbare Sicht, die alle relevanten Dimensionen für die Warenkorbanalyse enthält.

### 2.1 View-Definition

```sql
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_base` AS
SELECT
  -- Korb-Identifikation
  CONCAT(CAST(SC.filialNummer AS STRING), '_', CAST(SC.bonNummer AS STRING), '_', CAST(SC.tag AS STRING)) AS basket_id,
  SC.filialNummer,
  SC.bonNummer,
  SC.tag                          AS kaufdatum,
  SC.kundenNummer                 AS kunde_id,

  -- Artikel-Ebene (SKU)
  SC.artikelNummer                AS artikel_id,
  TA.full_artikel_beschreibung    AS artikel_beschreibung,
  TA.produktNummer                AS produkt_id,
  TP.produktBezeichnung           AS produkt_name,
  TA.eigenMarke                   AS ist_eigenmarke,

  -- Warengruppen (3 Ebenen)
  TA.wg1Id,
  TA.wg2Id,
  TA.wg3Id,
  AW.wgBezeichnung                AS warengruppe_name,

  -- Aktions-Kennzeichen
  SC.inAktion                     AS ist_aktion,

  -- Mengen & Preise
  SC.abverkaufsMenge              AS menge,
  SC.vkNettoBereinigtEuro         AS preis_netto,
  SC.vkBruttoBereinigtEuro        AS preis_brutto,
  SC.ekMixNettoEuro               AS ek_preis,

  -- Filial-Dimension
  FB.bundesLand                   AS bundesland,

  -- Zeit-Dimensionen
  EXTRACT(YEAR FROM SC.tag)      AS jahr,
  EXTRACT(MONTH FROM SC.tag)     AS monat,
  EXTRACT(QUARTER FROM SC.tag)   AS quartal,
  EXTRACT(DAYOFWEEK FROM SC.tag) AS wochentag,
  FORMAT_DATE('%A', SC.tag)      AS wochentag_name

FROM `amgdev_curated.T_Scanner_curated` AS SC

LEFT JOIN `amgdev_curated.T_Artikel_curated` AS TA
  ON SC.artikelNummer = TA.artikelNummer

LEFT JOIN `amgdev_curated.T_Produkte_curated` AS TP
  ON TA.produktNummer = TP.produktNummer

LEFT JOIN `amgdev_curated.artikel_warengruppe` AS AW
  ON SC.artikelNummer = AW.artikelNummer

LEFT JOIN `amgdev_curated.T_filiale_bundesland` AS FB
  ON SC.filialNummer = FB.filialNummer;
```

### 2.2 Aggregierte Sichten für die drei Granularitäten

#### a) SKU-Ebene (feinste Granularität – `artikelNummer`)

```sql
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_sku` AS
SELECT
  basket_id,
  kunde_id,
  kaufdatum,
  ist_aktion,
  bundesland,
  jahr, monat, quartal, wochentag,
  artikel_id AS item_id,
  artikel_beschreibung AS item_name
FROM `amgdev_curated.v_basket_base`
GROUP BY 1,2,3,4,5,6,7,8,9,10,11;
```

#### b) Produkt-Ebene (Marke/Produkt – `produktNummer`)

```sql
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_produkt` AS
SELECT
  basket_id,
  kunde_id,
  kaufdatum,
  ist_aktion,
  bundesland,
  jahr, monat, quartal, wochentag,
  produkt_id AS item_id,
  produkt_name AS item_name
FROM `amgdev_curated.v_basket_base`
WHERE produkt_id IS NOT NULL
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12;
```

#### c) Kategorie-Ebene (Warengruppe – `wg1Id`/`wg2Id`/`wg3Id`)

```sql
-- Ebene 3 (feinste Kategorie)
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_wg3` AS
SELECT
  basket_id,
  kunde_id,
  kaufdatum,
  ist_aktion,
  bundesland,
  jahr, monat, quartal, wochentag,
  wg3Id AS item_id,
  warengruppe_name AS item_name
FROM `amgdev_curated.v_basket_base`
WHERE wg3Id IS NOT NULL
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12;

-- Ebene 2 (mittlere Kategorie)
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_wg2` AS
SELECT
  basket_id,
  kunde_id,
  kaufdatum,
  ist_aktion,
  bundesland,
  jahr, monat, quartal, wochentag,
  wg2Id AS item_id,
  WG.wgBezeichnung AS item_name
FROM `amgdev_curated.v_basket_base`
LEFT JOIN `amgdev_curated.T_Warengruppen_curated` AS WG
  ON v_basket_base.wg2Id = WG.wgId
WHERE wg2Id IS NOT NULL
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12;

-- Ebene 1 (gröbste Kategorie)
CREATE OR REPLACE VIEW `amgdev_curated.v_basket_wg1` AS
SELECT
  basket_id,
  kunde_id,
  kaufdatum,
  ist_aktion,
  bundesland,
  jahr, monat, quartal, wochentag,
  wg1Id AS item_id,
  WG.wgBezeichnung AS item_name
FROM `amgdev_curated.v_basket_base`
LEFT JOIN `amgdev_curated.T_Warengruppen_curated` AS WG
  ON v_basket_base.wg1Id = WG.wgId
WHERE wg1Id IS NOT NULL
GROUP BY 1,2,3,4,5,6,7,8,9,10,11,12;
```

---

## Schritt 3: FP-Growth in Vertex AI Workbench

Nachdem die Views stehen, wird ein Python-Notebook in Vertex AI Workbench erstellt.

### 3.1 Ablauf

1. Daten aus BigQuery in Pandas DataFrame laden (pro Granularität)
2. Basket-Transformation: Pro `basket_id` eine Liste von Items
3. FP-Growth via `mlxtend.frequent_patterns`:
   - `min_support` = 0.01 (1% der Körbe) – anpassbar
   - `max_len` = 4 (max. 4 Items pro Regel)
4. Association Rules generieren:
   - `metric = "lift"`, `min_threshold = 1.5`
5. Ergebnisse zurück nach BigQuery schreiben

### 3.2 Ergebnis-Tabellen in BigQuery

```sql
-- Regeln auf SKU-Ebene
CREATE TABLE `amgdev_curated.regeln_sku` (
  granularitaet STRING,       -- 'sku'
  antecedents STRING,         -- ['artikel1', 'artikel2']
  consequents STRING,         -- ['artikel3']
  support FLOAT64,
  confidence FLOAT64,
  lift FLOAT64,
  n_baskets INT64,
  aktions_filter STRING       -- 'alle' | 'nur_aktion' | 'nur_normal'
);

-- Analog: regeln_produkt, regeln_wg3, regeln_wg2, regeln_wg1
```

---

## Schritt 4: Stakeholder-spezifische Auswertungen

### 4.1 Einkauf / Lieferanten-Konditionen
> "Wer A kauft, kauft auch B" → Primärartikel identifizieren, Beikauf-Wahrscheinlichkeit berechnen

```sql
-- Top-Regeln nach Lift (stärkste Verbindungen)
SELECT *
FROM `amgdev_curated.regeln_sku`
WHERE lift > 3.0
ORDER BY lift DESC
LIMIT 50;
```

### 4.2 Marketing / Prospekte
> Aktionsartikel + Beikauf: Welche Produkte werden zusammen mit Aktionsartikeln gekauft?

```sql
-- Nur Körbe mit Aktionsartikeln
SELECT *
FROM `amgdev_curated.regeln_sku`
WHERE aktions_filter = 'nur_aktion'
  AND lift > 2.0
ORDER BY support DESC
LIMIT 50;
```

### 4.3 Filial-Logistik / Kommissionierung
> Welche Artikel sollten räumlich zusammen platziert werden?

```sql
-- Hohe Confidence = starke Kopplung
SELECT *
FROM `amgdev_curated.regeln_sku`
WHERE confidence > 0.3
  AND lift > 2.0
ORDER BY confidence DESC
LIMIT 50;
```

### 4.4 Category Management
> Sortimentsblöcke auf Warengruppen-Ebene planen

```sql
SELECT *
FROM `amgdev_curated.regeln_wg1`
WHERE lift > 2.0
ORDER BY lift DESC;
```

### 4.5 Saisonale Muster
> Vergleich Schulanfangszeit vs. Rest des Jahres

```sql
-- Nutzung der T_Schulanfang_* Tabellen für zeitliche Filter
SELECT *
FROM `amgdev_curated.regeln_sku`
WHERE kaufdatum BETWEEN '2025-08-01' AND '2025-09-30'  -- Schulanfangszeit
  AND lift > 2.0
ORDER BY lift DESC;
```

---

## Schritt 5: Erweiterte Analysen (optional)

### 5.1 Embedding-basierte Ähnlichkeit
> Statt FP-Growth: Item2Vec-ähnliche Analyse über die bestehenden Embeddings

```sql
-- Ähnlichste Artikel zu einem gegebenen Artikel via Embedding
SELECT
  AE1.artikelNummer AS artikel,
  AE2.artikelNummer AS aehnlicher_artikel,
  (
    SELECT SUM(a * b)
    FROM UNNEST(AE1.artikel_embedding) a WITH OFFSET o1
    JOIN UNNEST(AE2.artikel_embedding) b WITH OFFSET o2 ON o1 = o2
  ) / (
    SQRT((SELECT SUM(a*a) FROM UNNEST(AE1.artikel_embedding) a))
    * SQRT((SELECT SUM(b*b) FROM UNNEST(AE2.artikel_embedding) b))
  ) AS cosine_similarity
FROM `amgdev_curated.T_Artikel_embeddings` AE1
CROSS JOIN `amgdev_curated.T_Artikel_embeddings` AE2
WHERE AE1.artikelNummer = 12345
  AND AE1.artikelNummer != AE2.artikelNummer
ORDER BY cosine_similarity DESC
LIMIT 10;
```

### 5.2 Kunden-basierte Analyse
> Wiederholungskäufe: Kauft ein Kunde immer wieder die gleichen Kombinationen?

```sql
-- Pro Kunde: Top-Kombinationen über mehrere Bestellungen
SELECT
  kunde_id,
  artikel_id,
  COUNT(DISTINCT basket_id) AS n_baskets
FROM `amgdev_curated.v_basket_sku`
GROUP BY kunde_id, artikel_id
HAVING n_baskets > 3
ORDER BY n_baskets DESC;
```

---

## Nächste Schritte

1. **Profiling-Queries ausführen** → Datenvolumen, Zeitraum, Qualität prüfen
2. **View `v_basket_base` erstellen** → Basissicht für alle Analysen
3. **Granularitäts-Views erstellen** → SKU, Produkt, WG1-3
4. **FP-Growth in Vertex AI Workbench** → Python-Notebook
5. **Regeln nach BigQuery schreiben** → `regeln_*` Tabellen
6. **Stakeholder-Dashboards** → Looker / Data Studio

---

## Offene Fragen (geklärt ✅)

- ✅ **Haupttabelle**: `T_Scanner_curated` (Bon-Positionen)
- ✅ **Warenkorb-ID**: `bonNummer` + `tag` + `filialNummer`
- ✅ **Aktions-Kennzeichen**: `inAktion` (BOOL) – direkt nutzbar
- ✅ **Warengruppen**: 3 Ebenen via `T_Warengruppen_curated`
- ✅ **Filial-Geografie**: `T_filiale_bundesland`
- ✅ **Embeddings**: Vorhanden für Artikel und Produkte
- ✅ **Keine Stornierungs-Problematik**: Scanner-Daten sind abgeschlossene Verkäufe
