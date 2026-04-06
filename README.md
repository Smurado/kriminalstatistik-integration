# Projekt: Integration historischer Kriminalitätsstatistiken (1984-2024)


## Phase 1: Data Collection & Extraction 

**Umsetzung:**
* Rohdaten aus drei Quellen heruntergeladen und in `data/01_raw/` abgelegt.
* Justiz- und Bevölkerungsdaten über die Genesis-API des Statistischen Bundesamtes (Destatis) als Flat-File-CSV bezogen.
* Die PKS wurde vom BKA als Excel-Datei bezogen. Es wurden bewusst getrennte Dateien für deutsche und nichtdeutsche Tatverdächtige genutzt.

**Begründung:**
* Die Datenquellen waren sehr unterschiedlich zugänglich (API vs. direkter Datei-Download), was unterschiedliche Extraktions-Skripte erforderte.
* Die Trennung nach Nationalität beim PKS-Download war notwendig, um das verbindende Merkmal "Nationalität" für die spätere Berechnung der Kriminalitätsraten sicherzustellen.


## Phase 2: Schema Mapping & Data Translation 

**Umsetzung:**
* Ziel-Schema definiert mit den Attributen: `jahr`, `straftat_bezeichnung`, `nationalitaet`, `tatverdaechtige`, `verurteilte`, `bevoelkerung_gesamt`.
* Schema-Integration nach dem **Global-as-View (GAV)**-Ansatz.
* **PKS:** Wide-Format in Long-Format überführt (Unpivoting/Melt). Die Nationalität wurde aus den Spaltenköpfen in eine eigene Datenspalte extrahiert.
* **Justiz:** Aggregation über die Dimensionen Alter und Geschlecht, um das gleiche Detaillierungslevel wie die PKS zu erreichen. Begriffe wie "Deutsche" und "Ausländer" wurden auf "Deutsch"/"Nichtdeutsch" normiert.
* **Bevölkerung:** Jahreszahlen aus dem Stichtags-Format extrahiert (z.B. "1984-12-31" wird zu 1984). Bundesländer-Zahlen auf nationale Gesamtsumme aufsummiert.

**Begründung:**
* Die Rohdaten wiesen hohe strukturelle Heterogenität auf (unterschiedliche Schemata, Datentypen, Granularitäten). 
* Der GAV-Ansatz eignet sich hier, da das analytische Ziel feststeht. 
* Für den späteren Join in der Data Fusion mussten alle Tabellen in Form, Terminologie und Datentypen auf das globale Ziel-Schema übersetzt werden.

## Phase 3: Identity Resolution (Hybrid Record Linkage)

**Umsetzung:**
* Die semantische Heterogenität zwischen PKS (905 Deliktbezeichnungen, inkl. historischer Umbenennungen) und Justizstatistik (21 aggregierte Kategorien) wurde über eine Mapping-Tabelle aufgelöst (Many-to-One).
* Dafür wurde ein **hybrider Identity Resolution Algorithmus** entwickelt:
  * **Heuristiken:** Regelbasierter Filter ordnet Delikte über juristische Schlüsselwörter zu (z.B. "btmg" wird zu Betäubungsmittelgesetz, "mord" wird zu Mord und Totschlag).
  * **Fuzzy String Matching:** Für nicht erfasste Delikte greift ein lexikalischer Ähnlichkeitsalgorithmus (`difflib`, Cutoff 0.5).
* Die verbleibenden Edge Cases wurden manuell geprüft und zugeordnet.

**Begründung:**
* Fuzzy Matching allein ist bei juristischem Vokabular unzureichend ("Betrug" und "Urkundenfälschung" sind textlich nicht ähnlich, gehören aber zusammen). 
* Die Kombination aus Domänenwissen (Regeln) und algorithmischer Ähnlichkeit minimiert den manuellen Aufwand bei hoher Trefferqualität.


## Phase 4: Data Quality Assessment

Die programmatische Qualitaetspruefung ist in `notebooks/06_data_quality_assessment.ipynb` dokumentiert. Das Notebook prueft:
* Vollstaendigkeit aller Quelldatensaetze (fehlende Werte, Duplikate, Wertebereiche)
* Coverage der Identity Resolution (wie viele PKS-Delikte wurden erfolgreich zugeordnet)
* Zeitreihen-Abdeckung als Heatmap (PKS vs. Justiz pro Kategorie und Jahr)
* Strukturbruch Wiedervereinigung (Bevoelkerungssprung um 1990)
* Deskriptive Statistik des Master-Datensatzes

## Phase 5: Final Data Fusion

**Datenqualitaet (Zusammenfassung):**
Nach der Fusion zeigt sich ein strukturierter Master-Datensatz (1.722 Zeilen, 21 Kategorien x 2 Nationalitaeten x bis zu 41 Jahre). Typische Luecken und Artefakte:
1. **Fehlende Werte:** PKS-Daten liegen erst ab 1987 vor. Die Jahre 1984-1986 enthalten `0` bei den Tatverdächtigen. Diese Nullen wurden bewusst beibehalten, um die Zeitreihe vollständig abzubilden.
2. **Datenartefakte:** Header-Reste (z.B. `Jahr=3`) in der PKS wurden per Filter (`jahr > 1900`) bereinigt.
3. **Wiedervereinigungs-Bruch:** Bevölkerungsdaten und Fallzahlen springen Anfang der 90er massiv (von ~61 Mio. auf >80 Mio. Einwohner). Vor 1991 nur Westdeutschland, danach Gesamtdeutschland.
4. **Granularitätsverlust:** 905 PKS-Delikte mussten auf 21 Justiz-Kategorien aggregiert werden. Detailtiefe einzelner Straftaten (z.B. einfacher vs. schwerer Diebstahl) geht dadurch verloren.
5. **Doppelzählung bereinigt:** Die Justiz-Rohdaten enthalten hierarchische Summenzeilen (z.B. „Straftaten insgesamt" = Summe aller Delikte) und die Nationalitäts-Dimension enthielt „Gesamt" (= Deutsch + Nichtdeutsch). Beide Artefakte wurden entfernt: Reine Summenzeilen (`Straftaten insgesamt`, `Straftaten ohne Straftaten im Straßenverkehr`, `Straftaten im Straßenverkehr`) sowie `Diebstahl` und `Schwerer Diebstahl` (in `Diebstahl und Unterschlagung` enthalten). Die Nationalität `Gesamt` wurde aus allen drei Quellen gefiltert.

**Verbesserungspotenzial:**
* **Imputation:** Fehlende PKS-Jahre (1984-1986) könnten aus Archiv-PDFs nacherfasst oder interpoliert werden.
* **Bevölkerungsdaten:** Die pro-Kopf-Raten werden bereits korrekt gegen die jeweilige Kohorte gerechnet (Nichtdeutsch-TV gegen Ausländer-Bevölkerung).
* **Alterskorrektur:** Bereinigung auf strafmündige Bevölkerung (ab 14 Jahren) würde die Aussagekraft der Raten verbessern.

**Die Data Fusion (Der Join):**
Die eigentliche Datenfusion wurde abschließend über einen `Outer Join` anhand der gemeinsamen Schlüssel (`jahr`, `nationalitaet`, `straftat_bezeichnung_mapped`) vollzogen. Im Anschluss wurde die Tabelle per `Left Join` um die Bevölkerungsdaten erweitert, um eine einheitliche Kriminalitätsrate (pro 100.000 Einwohner) als abgeleitetes Attribut (Feature Engineering) zu berechnen.