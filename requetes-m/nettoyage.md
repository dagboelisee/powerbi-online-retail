# Requêtes Power Query (extraits en M)

Les requêtes complètes sont dans le fichier Power BI. Voici les étapes qui portent la logique de nettoyage. Elles partent de `Ventes_Brut`, qui est l'ajout (Append) des deux feuilles du fichier Excel, lu à partir du paramètre `p_FichierSource`.

## Colonnes personnalisées de `Ventes_Brut`

`EstProduit` : vrai si le code article commence par cinq chiffres, ou par `DCGS`.

```
Text.Length(Text.Select(Text.Start([StockCode], 5), {"0".."9"})) = 5 or Text.StartsWith([StockCode], "DCGS")
```

`TypeLigne` :

```
if [EstProduit] <> true then "Hors produit"
else if Text.StartsWith([Invoice], "C") then "Annulation"
else if [Quantity] < 0 or [Price] <= 0 then "Ajustement"
else "Vente"
```

## `FaitVentes` : colonnes ajoutées

- `EstAnnulation` : `[TypeLigne] = "Annulation"`
- `Montant` : `[Quantity] * [Price]` (type nombre décimal fixe)
- `Date` : `DateTime.Date([InvoiceDate])`
- `StockCode` : `Text.Upper(Text.Trim(_))`, pour éviter les doublons de clé qui ne diffèrent que par la casse.

## `DimProduit`

Lignes de type *Vente* ou *Annulation* uniquement ; colonnes `StockCode` et `Description` ; `StockCode` normalisé (espaces supprimés, majuscules) ; regroupement par `StockCode` en gardant la plus grande description (`List.Max`) pour avoir exactement une description par code.

## `DimClient` et `DimPays`

`DimClient` : identifiants clients distincts et non vides, tirés de `FaitVentes`. `DimPays` : pays distincts, avec une colonne `Pays_Carte` qui normalise certains noms pour les cartes.

## Contrôles de réconciliation

- Total des lignes : 1 067 371 = ventes + annulations + ajustements + hors produit.
- CA net dans Power Query = CA net du rapport (£19 383 646,86).
- CA brut moins annulations = CA net, pour chaque année.
