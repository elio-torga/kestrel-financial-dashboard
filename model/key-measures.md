# Key DAX measures · Mesures DAX clés

A selection of 8 measures out of the 235 in the model, chosen because they carry the main business logic of the report.
*Une sélection de 8 mesures sur les 235 du modèle, choisies parce qu'elles portent la logique métier principale du rapport.*

Conventions used in the model · *Conventions du modèle* :

- `FactFinance[Amount]` is **signed**: revenue is positive and costs are negative, so subtotals are simple sums.
  *Les montants sont signés (produits positifs, charges négatives), donc les sous-totaux sont de simples sommes.*
- `[Last Actual Date]` returns the last closed month (Oct 2026). Every comparison is cut at that date.
  *`[Last Actual Date]` renvoie le dernier mois clôturé (oct. 2026). Toutes les comparaisons s'arrêtent à cette date.*
- `[Total Amount]` is the base sum of `FactFinance[Amount]`. *`[Total Amount]` est la somme de base.*

---

## 1. Like-for-like budget YTD · Budget à date comparable

```dax
Amount BUD YTD =
VAR _LastActual = [Last Actual Date]
RETURN
    CALCULATE (
        [Total Amount],
        DimScenario[ScenarioKey] = "BUD",
        DATESYTD ( DimDate[Date] ),
        DimDate[Date] <= _LastActual
    )
```

**EN:** The budget covers the full year but actuals stop at October. Without the cut-off, actual YTD would be compared with a 12-month budget and every variance would look like a huge miss. This is the most common mistake in budget-vs-actual reports.
**FR :** Le budget couvre toute l'année, mais le réel s'arrête à octobre. Sans cette limite, on comparerait 10 mois de réel à 12 mois de budget et tous les écarts seraient faussés. C'est l'erreur la plus fréquente dans les rapports réel vs budget.

## 2. Prior year YTD, same cut-off · Année précédente à date

```dax
Amount PY YTD =
VAR _LastActual = [Last Actual Date]
RETURN
    CALCULATE (
        [Amount ACT YTD],
        SAMEPERIODLASTYEAR ( DimDate[Date] ),
        DimDate[Date] <= EDATE ( _LastActual, -12 )
    )
```

**EN:** Same principle for year-on-year growth: Jan–Oct 2026 is compared with Jan–Oct 2025, not with the full year 2025.
**FR :** Même principe pour la croissance annuelle : janvier–octobre 2026 est comparé à janvier–octobre 2025, pas à toute l'année 2025.

## 3. Full-year outlook = actuals + latest forecast · Atterrissage annuel

```dax
Amount Outlook FY =
VAR _LastActual = [Last Actual Date]
VAR _Year = MAX ( DimDate[Year] )
RETURN
    CALCULATE (
        [Total Amount],
        DimScenario[ScenarioKey] = "ACT",
        REMOVEFILTERS ( DimDate ),
        DimDate[Year] = _Year,
        DimDate[Date] <= _LastActual
    )
    + CALCULATE (
        [Total Amount],
        DimScenario[IsLatestForecast] = TRUE (),
        REMOVEFILTERS ( DimDate ),
        DimDate[Year] = _Year,
        DimDate[Date] > _LastActual
    )
```

**EN:** The question a CFO actually asks is "where will we land this year?". The measure stitches closed months (actuals) to the remaining months of the latest rolling forecast. The `IsLatestForecast` flag means a new forecast version only needs a new row in `DimScenario`, with no change to the DAX.
**FR :** La vraie question d'un DAF est « où va-t-on atterrir cette année ? ». La mesure assemble les mois clôturés (réel) et les mois restants du dernier forecast. Grâce à l'indicateur `IsLatestForecast`, une nouvelle version de forecast demande seulement une ligne dans `DimScenario`, sans toucher au DAX.

## 4. P&L layout with a mapping table · Compte de résultat avec table de correspondance

```dax
-- 'PL Line' is a small calculated table: one row per (P&L line, account group).
-- Subtotals (Gross profit, EBITDA, EBIT, Net income) simply list several groups.
PL ACT YTD =
CALCULATE (
    [Amount ACT YTD],
    TREATAS ( VALUES ( 'PL Line'[AccountGroup] ), DimAccount[AccountGroup] )
) / 1000000
```

**EN:** A P&L mixes detail lines and subtotals in one table. Instead of hard-coding each line in a `SWITCH`, a disconnected mapping table says which account groups feed each line, and `TREATAS` applies that list as a filter on the account dimension. Adding a new subtotal means adding rows, not rewriting measures.
**FR :** Un compte de résultat mélange lignes de détail et sous-totaux. Au lieu de coder chaque ligne dans un `SWITCH`, une table de correspondance indique quels groupes de comptes alimentent chaque ligne, et `TREATAS` applique cette liste comme filtre. Ajouter un sous-total revient à ajouter des lignes, pas à réécrire des mesures.

## 5. Forecast accuracy · Fiabilité du forecast

```dax
Revenue Forecast Error % =
VAR _LastActual = [Last Actual Date]
VAR _Start =
    CALCULATE ( MIN ( FactFinance[Date] ), REMOVEFILTERS (), DimScenario[ScenarioKey] = "RF_JUL26" )
VAR _Fc =
    CALCULATE ( [Revenue], DimScenario[ScenarioKey] = "RF_JUL26",
        DimDate[Date] >= _Start, DimDate[Date] <= _LastActual )
VAR _Act =
    CALCULATE ( [Revenue], DimScenario[ScenarioKey] = "ACT",
        DimDate[Date] >= _Start, DimDate[Date] <= _LastActual )
RETURN
    DIVIDE ( _Fc - _Act, _Act )

Revenue Forecast Accuracy % = 1 - ABS ( [Revenue Forecast Error %] )
```

**EN:** The July forecast predicted August to October. Those months are now closed, so we can measure how good it was. The signed error shows the **bias** (+2.4% means the forecast was too optimistic) and the accuracy (97.6%) gives the headline figure.
**FR :** Le forecast de juillet prévoyait août à octobre. Ces mois sont maintenant clôturés, on peut donc mesurer sa qualité. L'erreur signée montre le **biais** (+2,4 % = forecast trop optimiste) et la fiabilité (97,6 %) donne le chiffre clé.

## 6. What-if simulation · Simulation what-if

```dax
Sim Revenue = [RF Revenue] * ( 1 + [WI Volume %] ) * ( 1 + [WI Price %] )

Sim EBITDA =
VAR _v = [WI Volume %]
VAR _m = [WI Material Cost %]
VAR _dRev = [Sim Revenue] - [RF Revenue]
VAR _dMat = [RF Materials] * ( ( 1 + _v ) * ( 1 + _m ) - 1 )   -- materials move with volume AND price
VAR _dOther = [RF Volume Driven Costs] * _v                     -- other variable costs move with volume only
RETURN
    [RF EBITDA] + _dRev + _dMat + _dOther
```

**EN:** Three drivers (volume, selling price, raw-material cost) come from disconnected `GENERATESERIES` tables used as slicers. The logic follows how costs really behave: materials depend on both volume and purchase price, other variable costs only on volume, and fixed costs do not move. Costs are negative in the model, so the deltas add up without sign tricks.
**FR :** Trois leviers (volume, prix de vente, coût matière) viennent de tables `GENERATESERIES` déconnectées utilisées comme segments. La logique suit le comportement réel des coûts : la matière dépend du volume et du prix d'achat, les autres coûts variables du volume seulement, et les coûts fixes ne bougent pas. Les coûts étant négatifs dans le modèle, les écarts s'additionnent sans jeu de signes.

## 7. KPI scorecard with one set of visuals · Tableau de bord KPI générique

```dax
-- KPI is a disconnected table: Revenue, EBITDA, Net Income, Free Cash Flow
KPI Var BUD YTD % =
SWITCH (
    SELECTEDVALUE ( KPI[KPI] ),
    "Revenue",        [Revenue Var BUD YTD %],
    "EBITDA",         [EBITDA Var BUD YTD %],
    "Net Income",     [Net Income Var BUD YTD %],
    "Free Cash Flow", [Free Cash Flow Var BUD YTD %]
)

-- Conditional formatting returns a colour, applied with "Format style: Field value"
CF KPI Var BUD YTD = IF ( [KPI Var BUD YTD %] < 0, "#E02424", "#0E9F6E" )
```

**EN:** One table visual shows the four KPIs side by side with the same columns (YTD, budget, variance, PY, outlook). Colours come from measures, so the red/green rules live in one place and stay consistent across all pages.
**FR :** Un seul tableau affiche les quatre KPI avec les mêmes colonnes (YTD, budget, écart, N-1, atterrissage). Les couleurs viennent de mesures : les règles rouge/vert sont définies une seule fois et restent cohérentes sur toutes les pages.

## 8. Auto-generated commentary · Commentaire généré automatiquement

```dax
Exec Commentary =
VAR _r = [Revenue Var BUD YTD %]
VAR _e = [EBITDA Var BUD YTD %]
VAR _o = [EBITDA Outlook vs BUD FY %]
VAR _f = [Free Cash Flow Var BUD YTD %]
VAR _worst = TOPN ( 1, VALUES ( DimEntity[Country] ), [EBITDA Var BUD YTD], ASC )
VAR _worstName = MAXX ( _worst, DimEntity[Country] )
VAR _worstGap = SUMX ( _worst, [EBITDA Var BUD YTD] )
VAR _dso = TOPN ( 1, VALUES ( DimEntity[Country] ), [DSO ACT], DESC )
VAR _dsoName = MAXX ( _dso, DimEntity[Country] )
VAR _dsoVal = MAXX ( _dso, [DSO ACT] )
VAR _dsoBud = MAXX ( _dso, [DSO BUD LFL] )
VAR _l1 =
    "Revenue " & FORMAT ( _r, "+0.0%;-0.0%" ) & " vs budget, EBITDA " & FORMAT ( _e, "+0.0%;-0.0%" )
        & IF ( _e < _r - 0.03, ": margin squeeze, costs are not flexing with volume.", "." )
VAR _l2 = "Largest EBITDA gap: " & _worstName & " (" & FORMAT ( _worstGap / 1000000, "+0.0;-0.0" ) & "M€ vs budget)."
VAR _l3 = "Highest DSO: " & _dsoName & " (" & FORMAT ( _dsoVal, "0" ) & " days vs " & FORMAT ( _dsoBud, "0" ) & " budgeted)."
VAR _l4 = "Full-year EBITDA outlook " & FORMAT ( _o, "+0.0%;-0.0%" ) & " vs budget; free cash flow "
        & FORMAT ( _f, "+0.0%;-0.0%" ) & " vs budget YTD."
RETURN
    "• " & _l1 & UNICHAR ( 10 ) & "• " & _l2 & UNICHAR ( 10 ) & "• " & _l3 & UNICHAR ( 10 ) & "• " & _l4
```

**EN:** The Overview page writes its own key messages. The text reacts to the period and entity slicers, finds the weakest entity and the worst collector, and only flags a "margin squeeze" when EBITDA falls clearly faster than revenue.
**FR :** La page Overview rédige elle-même ses messages clés. Le texte réagit aux filtres de période et d'entité, identifie l'entité la plus en retard et le pire délai d'encaissement, et ne signale un « margin squeeze » que si l'EBITDA baisse nettement plus vite que le chiffre d'affaires.

---

© 2026 Elio Torga – all rights reserved · *tous droits réservés*
