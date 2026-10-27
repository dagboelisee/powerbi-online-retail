# Mesures et colonnes DAX

Les noms de colonnes sont ceux du modèle : `FaitVentes` (Invoice, StockCode, Customer ID, Date, Quantity, Price, Montant, EstAnnulation), `DimDate`, `DimProduit`, `DimClient`, `DimPays`, `RFM`.
Dans une version française de Power BI, remplace les virgules entre arguments par des points-virgules si l'éditeur le demande.

## Table de dates

```
DimDate =
ADDCOLUMNS(
    CALENDAR(DATE(YEAR(MIN(FaitVentes[Date])), 1, 1), DATE(YEAR(MAX(FaitVentes[Date])), 12, 31)),
    "Année", YEAR([Date]),
    "Trimestre", "T" & QUARTER([Date]),
    "AnnéeMois", YEAR([Date]) * 100 + MONTH([Date]),
    "Mois", FORMAT([Date], "mmm yyyy")
)
```

Colonne ajoutée pour l'axe continu des courbes :

```
DébutMois = DATE(YEAR(DimDate[Date]), MONTH(DimDate[Date]), 1)
```

## Mesures de base

| Mesure | Formule | Explication |
|---|---|---|
| CA net | `SUM(FaitVentes[Montant])` | Ventes moins annulations (les annulations ont un montant négatif) |
| CA brut | `CALCULATE(SUM(FaitVentes[Montant]), FaitVentes[EstAnnulation] = FALSE())` | Ventes avant annulations |
| Annulations | `-CALCULATE(SUM(FaitVentes[Montant]), FaitVentes[EstAnnulation] = TRUE())` | Montant annulé, en positif |
| Taux d'annulation | `DIVIDE([Annulations], [CA brut])` | Part du CA brut annulée |
| Nb factures | `CALCULATE(DISTINCTCOUNT(FaitVentes[Invoice]), FaitVentes[EstAnnulation] = FALSE())` | Factures de vente |
| Nb clients | `DISTINCTCOUNTNOBLANK(FaitVentes[Customer ID])` | Clients identifiés |
| Panier moyen | `DIVIDE([CA brut], [Nb factures])` | CA brut par facture |
| Quantité vendue | `CALCULATE(SUM(FaitVentes[Quantity]), FaitVentes[EstAnnulation] = FALSE())` | Unités vendues |
| Nb produits vendus | `CALCULATE(DISTINCTCOUNT(FaitVentes[StockCode]), FaitVentes[EstAnnulation] = FALSE())` | Produits différents vendus |

## Dans le temps

```
CA net N-1 = CALCULATE([CA net], SAMEPERIODLASTYEAR(DimDate[Date]))
Variation N-1 = DIVIDE([CA net] - [CA net N-1], [CA net N-1])
CA net cumul annuel = TOTALYTD([CA net], DimDate[Date])
```

À lire sur des mois complets : 2009 ne compte que décembre et 2011 s'arrête au 9 décembre.

## Pays

```
Part du CA pays = DIVIDE([CA net], CALCULATE([CA net], ALL(DimPays[Country])))

Part du CA Royaume-Uni =
DIVIDE(
    CALCULATE([CA net], DimPays[Country] = "United Kingdom"),
    CALCULATE([CA net], ALL(DimPays))
)
```

## Produits : Pareto et classes de prix

```
Pareto cumul % =
VAR _ca = [CA net]
VAR _tous = ADDCOLUMNS(ALL(DimProduit[StockCode]), "@ca", [CA net])
RETURN
DIVIDE(
    SUMX(FILTER(_tous, [@ca] >= _ca), [@ca]),
    SUMX(_tous, [@ca])
)

Nb produits pour 80 % du CA =
VAR _t = ADDCOLUMNS(ALL(DimProduit[StockCode]), "@ca", [CA net])
VAR _tot = SUMX(_t, [@ca])
VAR _cum =
    ADDCOLUMNS(
        _t,
        "@cum",
        VAR _v = [@ca]
        RETURN SUMX(FILTER(_t, [@ca] >= _v), [@ca])
    )
RETURN
COUNTROWS(FILTER(_cum, [@cum] <= 0.8 * _tot))

Part du catalogue pour 80 % du CA =
DIVIDE([Nb produits pour 80 % du CA], [Nb produits vendus])
```

Colonnes calculées de `DimProduit` :

```
Prix moyen = CALCULATE(AVERAGE(FaitVentes[Price]), FaitVentes[EstAnnulation] = FALSE())

Classe de prix =
SWITCH(TRUE(),
    ISBLANK(DimProduit[Prix moyen]), "Non vendu",
    DimProduit[Prix moyen] < 1, "Moins de £1",
    DimProduit[Prix moyen] < 2, "£1 à 2",
    DimProduit[Prix moyen] < 5, "£2 à 5",
    DimProduit[Prix moyen] < 10, "£5 à 10",
    DimProduit[Prix moyen] < 20, "£10 à 20",
    "£20 et plus")
```

`Classe de prix` est triée par une colonne `Ordre classe de prix` (valeurs de 0 à 6, même `SWITCH`).

## Segmentation RFM

Table `RFM` : une ligne par client (récence, fréquence, montant), puis les colonnes calculées suivantes.

```
Score R = 6 - ROUNDUP(5 * DIVIDE(RANKX(ALL(RFM), RFM[Recence], , ASC, Skip), COUNTROWS(RFM)), 0)
Score M = 6 - ROUNDUP(5 * DIVIDE(RANKX(ALL(RFM), RFM[Montant], , DESC, Skip), COUNTROWS(RFM)), 0)

Score F =
SWITCH(TRUE(),
    RFM[Frequence] >= 10, 5,
    RFM[Frequence] >= 5, 4,
    RFM[Frequence] >= 3, 3,
    RFM[Frequence] >= 2, 2,
    1)

Segment =
IF(RFM[Montant] <= 0, "Net nul",
SWITCH(TRUE(),
    RFM[Score R] >= 4 && RFM[Score F] >= 4, "Champions",
    RFM[Score R] >= 3 && RFM[Score F] >= 3, "Clients fidèles",
    RFM[Score R] <= 2 && RFM[Score F] >= 3, "À risque",
    RFM[Score R] <= 2 && RFM[Score F] <= 2, "Perdus",
    "Occasionnels"))

Log Montant = IF(RFM[Montant] > 0, LOG10(RFM[Montant]))
```

`Log Montant` sert à afficher les montants en échelle logarithmique dans le nuage de points.

```
Part du CA des Champions =
DIVIDE(
    CALCULATE([CA net], RFM[Segment] = "Champions"),
    CALCULATE([CA net], ALL(RFM))
)

CA des clients À risque = CALCULATE([CA net], RFM[Segment] = "À risque")
```

## Mesures d'affichage

Mesures texte utilisées dans les cartes pour garder les nombres exacts :

```
Nb factures (affichage) = FORMAT([Nb factures], "#,##0")
Nb clients (affichage) = FORMAT([Nb clients], "#,##0")
CA clients À risque (affichage) = "£" & FORMAT([CA des clients À risque] / 1000000, "0.0") & "M"
```
