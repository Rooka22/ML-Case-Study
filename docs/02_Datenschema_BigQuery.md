# Datenschema – BigQuery (Dataset: `amgdev_curated`)

> Stand: Juni 2026

---

## Übersicht der Tabellen

| Tabelle | Zweck | Kern für Warenkorbanalyse |
|---|---|---|
| `T_Scanner_curated` | **Transaktionsdaten** (Bon-Positionen) | ✅ **Primäre Quelle** |
| `T_Artikel_curated` | Artikel-Stammdaten (Warengruppen, Maße, etc.) | ✅ Warengruppen-Hierarchie |
| `T_Produkte_curated` | Produkt-Bezeichnungen | ✅ Marken-/Produktname |
| `T_Warengruppen_curated` | Warengruppen-Hierarchie (3 Ebenen) | ✅ Kategorie-Granularität |
| `T_filiale_bundesland` | Filiale → Bundesland | ✅ Regionale Analyse |
| `T_Artikel_embeddings` | Artikel-Embeddings (Vektor) | 🔷 Erweiterte Analyse |
| `T_Produkte_embeddings` | Produkt-Embeddings (Vektor) | 🔷 Erweiterte Analyse |
| `T_Schulanfang_*` | Schulanfangs-Saison (3 Varianten) | 🔷 Saisonale Analyse |
| `artikel_warengruppe` | Mapping Artikel → Warengruppe (denormalisiert) | ✅ Vereinfachte Joins |
| `meta_inflow` | Metadaten zu Daten-Ladeprozessen | ❌ Nur Monitoring |

---

## 1. `T_Scanner_curated` – Die Haupttabelle

> **Jede Zeile = ein gekaufter Artikel auf einem Bon** (Bon-Position)

| Spalte | Typ | Bedeutung | Relevanz |
|---|---|---|---|
| `filialNummer` | `INT64` | Filial-ID | 🟡 Filial-Vergleich |
| `tag` | `DATE` | Kaufdatum | 🟠 Saisonalität, Zeitfenster |
| `kundenNummer` | `STRING` | Kunden-ID (Pseudonym) | 🟠 Kunden-basierte Analyse |
| `artikelNummer` | `INT64` | Artikel-ID (SKU) | 🔴 **Item im Warenkorb** |
| `abverkaufsMenge` | `INT64` | Gekaufte Menge | 🟠 Gewichtung der Regeln |
| `vkNettoBereinigtEuro` | `FLOAT64` | Verkaufspreis netto | 🟡 Warenkorb-Wert |
| `vkBruttoBereinigtEuro` | `FLOAT64` | Verkaufspreis brutto | 🟡 Warenkorb-Wert |
| `ekMixNettoEuro` | `FLOAT64` | Einkaufspreis (Mix) | 🟡 Marge / Deckungsbeitrag |
| `inAktion` | `BOOL` | **War der Artikel im Angebot?** | 🔴 **Aktions-Vergleich** |
| `bonNummer` | `STRING` | **Bon-Nummer = Warenkorb-ID** | 🔴 **Korb-Definition** |

### 🔑 Wichtigste Erkenntnis

> **`bonNummer` ist die Warenkorb-ID.**  
> Alle Zeilen mit gleicher `bonNummer` und gleichem `tag` bilden einen Einkaufskorb.
> `inAktion` (BOOL) erlaubt direkten Vergleich: Aktionskauf vs. Normalkauf.

---

## 2. `T_Artikel_curated` – Artikel-Stammdaten

| Spalte | Typ | Bedeutung | Relevanz |
|---|---|---|---|
| `artikelNummer` | `INT64` | **Primärschlüssel** | 🔴 Join zu Scanner |
| `eigenMarke` | `BOOL` | Eigenmarke (ja/nein) | 🟡 Marken-Analyse |
| `inhaltsMenge` | `FLOAT64` | Inhalt (z.B. 500g) | 🟡 |
| `inhaltsMengenId` | `INT64` | Einheit der Inhaltsmenge | 🟡 |
| `wg1Id` | `INT64` | **Warengruppe Ebene 1** | 🔴 **Kategorie-Granularität** |
| `wg2Id` | `INT64` | **Warengruppe Ebene 2** | 🔴 **Kategorie-Granularität** |
| `wg3Id` | `INT64` | **Warengruppe Ebene 3** | 🔴 **Kategorie-Granularität** |
| `produktNummer` | `INT64` | **Produkt-ID (Marken-Ebene)** | 🔴 **Marken-Granularität** |
| `full_artikel_beschreibung` | `STRING` | Volltext-Beschreibung | 🟡 |
| `artikelTypId` | `INT64` | Artikel-Typ | 🟡 |
| `artikelArtId` | `INT64` | Artikel-Art | 🟡 |
| `gefahrGutId` / `gefahrGutKlassenId` | `INT64` | Gefahrgut-Info | ❌ |
| `laenge/breite/hoehe/gewicht/volumenLogistik` | `INT64` | Logistik-Maße | 🟡 Filial-Logistik |
| `gefahrstoffNettoGewicht` | `INT64` | Gefahrstoff-Gewicht | ❌ |

### 🔑 Wichtigste Erkenntnis

> **Drei Warengruppen-Ebenen (`wg1Id`, `wg2Id`, `wg3Id`) und `produktNummer`**  
> → Erlauben die drei Granularitäten: **SKU** (`artikelNummer`), **Produkt/Marke** (`produktNummer`), **Kategorie** (`wg1Id`/`wg2Id`/`wg3Id`)

---

## 3. `T_Produkte_curated` – Produkt-Bezeichnungen

| Spalte | Typ | Bedeutung |
|---|---|---|
| `produktNummer` | `INT64` | **Primärschlüssel** (Join zu `T_Artikel_curated.produktNummer`) |
| `produktBezeichnung` | `STRING` | Produktname (z.B. "Milka Schokolade Alpenmilch 100g") |

---

## 4. `T_Warengruppen_curated` – Warengruppen-Hierarchie

| Spalte | Typ | Bedeutung |
|---|---|---|
| `wgId` | `INT64` | **Primärschlüssel** |
| `ebene` | `INT64` | Hierarchie-Ebene (1 = grob, 3 = fein) |
| `uebergeordneteWgId` | `INT64` | **Parent-Warengruppe** (für Hierarchie-Baum) |
| `wgNummer` | `INT64` | Warengruppen-Nummer |
| `wgBezeichnung` | `STRING` | Name (z.B. "Süßwaren", "Schokolade") |
| `wgSort` | `INT64` | Sortierreihenfolge |

### 🔑 Wichtigste Erkenntnis

> **Selbst-referenzierende Hierarchie** über `uebergeordneteWgId`.  
> Damit kann der gesamte Kategoriebaum aufgespannt werden.  
> `ebene` gibt die Tiefe an (1 = Hauptgruppe, 2 = Untergruppe, 3 = Fein-Gruppe).

---

## 5. `T_filiale_bundesland` – Filial-Geografie

| Spalte | Typ | Bedeutung |
|---|---|---|
| `filialNummer` | `INT64` | **Primärschlüssel** (Join zu `T_Scanner_curated.filialNummer`) |
| `bundesLand` | `STRING` | Bundesland (z.B. "Bayern", "NRW") |

---

## 6. `T_Artikel_embeddings` & `T_Produkte_embeddings` – Vektor-Repräsentationen

| Tabelle | Spalten | Zweck |
|---|---|---|
| `T_Artikel_embeddings` | `full_artikel_beschreibung`, `artikel_embedding` (ARRAY\<FLOAT64\>) | Semantische Ähnlichkeit von Artikeln |
| `T_Produkte_embeddings` | `produktBezeichnung`, `produkt_embedding` (ARRAY\<FLOAT64\>) | Semantische Ähnlichkeit von Produkten |

> **Einsatz**: Für erweiterte Analysen (z.B. ähnliche Produkte zu einem Regel-Antecedent finden, oder Embedding-basierte Item2Vec-Alternative).

---

## 7. `artikel_warengruppe` – Denormalisierte Sicht

| Spalte | Typ | Bedeutung |
|---|---|---|
| `artikelNummer` | `INT64` | Artikel-ID |
| `full_artikel_beschreibung` | `STRING` | Beschreibung |
| `wgBezeichnung` | `STRING` | Warengruppen-Name (direkt, ohne Join) |

> **Praktisch für schnelle Abfragen ohne Join über die Hierarchie.**

---

## 8. Schulanfangs-Tabellen (saisonal)

| Tabelle | Inhalt |
|---|---|
| `T_Schulanfang_ArtikelNummer_Basis` | Einzelartikel-Verkäufe in Schulanfangs-Perioden |
| `T_Schulanfang_Basis` | Aggregierte Schreibwaren-Verkäufe pro Periode |
| `T_schulanfang_vs_normal` | Vergleich Schulanfang vs. Normalzeit |
| `T_schulanfang_vs_vergleich` | Vergleich verschiedener Schulanfangs-Perioden |
| `schulanfang_bw` | Schulanfangs-Termine pro Bundesland |

---

## ERM vs. Realität – Mapping

| ERM-Entität | BigQuery-Tabelle | Bemerkung |
|---|---|---|
| `F_Bestellinformation` | `T_Scanner_curated` (teilweise) | Bon-Daten, aber ohne explizite Bestell-Tabelle |
| `T_Produkte` | `T_Artikel_curated` + `T_Produkte_curated` | Aufgeteilt in Artikel (SKU) und Produkt (Marke) |
| `T_Werbeträger` | – | Nicht explizit vorhanden; `inAktion` (BOOL) in Scanner |
| `T_Artikelzustand` | – | Nicht explizit vorhanden |
| `F_Filiale` / `T_Ort` / `T_Bundesland` | `T_filiale_bundesland` | Denormalisiert auf Filiale + Bundesland |
| `T_Kunde` | – | Nur `kundenNummer` (STRING) in Scanner |
| `T_Postleitzahl` / `T_Bewohnerzahl` | – | Nicht vorhanden |
| `T_Geschaeftsobjekt` | – | Nicht explizit vorhanden |

---

## Datenqualität & Besonderheiten

| Aspekt | Status |
|---|---|
| **Warenkorb-ID** | ✅ `bonNummer` + `tag` + `filialNummer` = eindeutiger Korb |
| **Aktions-Kennzeichen** | ✅ `inAktion` als BOOL (einfach auswertbar) |
| **Kunden-ID** | ✅ `kundenNummer` als STRING (pseudonymisiert) |
| **Warengruppen-Hierarchie** | ✅ 3 Ebenen + selbst-referenzierende Tabelle |
| **Embeddings** | ✅ Vorhanden für Artikel und Produkte (für Advanced Analytics) |
| **Filial-Geografie** | ✅ Bundesland-Ebene vorhanden |
| **Preise** | ✅ Netto + Brutto + EK-Mix |
| **Mengen** | ✅ `abverkaufsMenge` (auch >1 möglich) |
