# PBIP / TMDL templates

All TMDL below uses **tab** indentation – keep it that way when copying. Replace every `lineageTag`
with a freshly generated GUID (PowerShell: `[guid]::NewGuid()`, bash: `uuidgen`).

## Contents
1. Project layout
2. Entry file and definition files
3. Parameters and shared source (expressions.tmdl)
4. Model file (model.tmdl)
5. Import table (dimension or fact)
6. Date table generated in Power Query
7. Relationships (active and inactive)
8. Measure table and measures
9. Minimal report with theme

---

## 1. Project layout

```
<Name>.pbip
<Name>.Report/
    definition.pbir
    report.json                                   # Desktop may convert this to definition/pages/...
    StaticResources/SharedResources/BaseThemes/<theme>.json
<Name>.SemanticModel/
    definition.pbism
    definition/
        database.tmdl
        model.tmdl
        expressions.tmdl
        relationships.tmdl
        tables/
            dim_customers.tmdl
            dim_products.tmdl
            dim_date.tmdl
            fact_sales.tmdl
            _Measures.tmdl
.gitignore                                        # **/.pbi/localSettings.json, **/.pbi/cache.abf
```

## 2. Entry file and definition files

`<Name>.pbip`
```json
{
  "version": "1.0",
  "artifacts": [ { "report": { "path": "<Name>.Report" } } ],
  "settings": { "enableAutoRecovery": true }
}
```

`<Name>.Report/definition.pbir`
```json
{
  "version": "1.0",
  "datasetReference": { "byPath": { "path": "../<Name>.SemanticModel" }, "byConnection": null }
}
```

`<Name>.SemanticModel/definition.pbism`
```json
{ "version": "4.2", "settings": {} }
```

`definition/database.tmdl`
```
database
	compatibilityLevel: 1600
```

## 3. Parameters and shared source (expressions.tmdl)

```
/// Supabase session pooler host (Supabase → Connect → Session pooler). Port 5432 is the default.
expression PgServer = "aws-0-REGION.pooler.supabase.com" meta [IsParameterQuery = true, Type = "Text", IsParameterQueryRequired = true]
	lineageTag: 00000000-0000-0000-0000-000000000001
	queryGroup: Parameters

	annotation PBI_ResultType = Text

expression PgDatabase = "postgres" meta [IsParameterQuery = true, Type = "Text", IsParameterQueryRequired = true]
	lineageTag: 00000000-0000-0000-0000-000000000002
	queryGroup: Parameters

	annotation PBI_ResultType = Text

/// Shared connection. Sign in with Database credentials: powerbi_reader.<project-ref>
expression GoldSource =
		let
			Source = PostgreSQL.Database(PgServer, PgDatabase)
		in
			Source
	lineageTag: 00000000-0000-0000-0000-000000000003
	queryGroup: Parameters

	annotation PBI_ResultType = Table
```

## 4. Model file (model.tmdl)

```
model Model
	culture: en-US
	defaultPowerBIDataSourceVersion: powerBI_V3
	sourceQueryCulture: en-US
	dataAccessOptions
		legacyRedirects
		returnErrorValuesAsNull

queryGroup Parameters

	annotation PBI_QueryGroupOrder = 0

queryGroup Gold

	annotation PBI_QueryGroupOrder = 1

annotation __PBI_TimeIntelligenceEnabled = 0

ref table dim_customers
ref table dim_products
ref table fact_sales
ref table dim_date
ref table _Measures
```

Turning off auto date/time (`__PBI_TimeIntelligenceEnabled = 0`) avoids hidden date tables once you
have a proper `dim_date`.

## 5. Import table (dimension or fact)

```
table fact_sales
	lineageTag: 00000000-0000-0000-0000-000000000010

	column order_number
		dataType: string
		lineageTag: 00000000-0000-0000-0000-000000000011
		summarizeBy: none
		sourceColumn: order_number

		annotation SummarizationSetBy = Automatic

	column customer_key
		dataType: int64
		isHidden
		formatString: 0
		lineageTag: 00000000-0000-0000-0000-000000000012
		summarizeBy: none
		sourceColumn: customer_key

		annotation SummarizationSetBy = Automatic

	column order_date
		dataType: dateTime
		formatString: yyyy-mm-dd
		lineageTag: 00000000-0000-0000-0000-000000000013
		summarizeBy: none
		sourceColumn: order_date

		annotation SummarizationSetBy = Automatic

		annotation UnderlyingDateTimeDataType = Date

	column sales_amount
		dataType: int64
		isHidden
		formatString: #,0
		lineageTag: 00000000-0000-0000-0000-000000000014
		summarizeBy: sum
		sourceColumn: sales_amount

		annotation SummarizationSetBy = Automatic

	partition fact_sales = m
		mode: import
		queryGroup: Gold
		source =
				let
					Source = GoldSource,
					Data = Source{[Schema = "gold", Item = "fact_sales"]}[Data],
					Selected = Table.SelectColumns(Data, {"order_number", "customer_key", "order_date", "sales_amount"}),
					Typed = Table.TransformColumnTypes(Selected, {{"order_number", type text}, {"customer_key", Int64.Type}, {"order_date", type date}, {"sales_amount", Int64.Type}})
				in
					Typed

	annotation PBI_ResultType = Table
```

Hide key and raw measure columns that users should reach through measures.

## 6. Date table generated in Power Query

```
/// Calendar covering whole years from the first to the last date in fact_sales.
table dim_date
	lineageTag: 00000000-0000-0000-0000-000000000020
	dataCategory: Time

	column date_key
		dataType: dateTime
		isKey
		formatString: yyyy-mm-dd
		lineageTag: 00000000-0000-0000-0000-000000000021
		summarizeBy: none
		sourceColumn: date_key

		annotation SummarizationSetBy = Automatic

		annotation UnderlyingDateTimeDataType = Date

	column year
		dataType: int64
		formatString: 0
		lineageTag: 00000000-0000-0000-0000-000000000022
		summarizeBy: none
		sourceColumn: year

		annotation SummarizationSetBy = Automatic

	column month_no
		dataType: int64
		isHidden
		formatString: 0
		lineageTag: 00000000-0000-0000-0000-000000000023
		summarizeBy: none
		sourceColumn: month_no

		annotation SummarizationSetBy = Automatic

	column month_name
		dataType: string
		lineageTag: 00000000-0000-0000-0000-000000000024
		summarizeBy: none
		sourceColumn: month_name
		sortByColumn: month_no

		annotation SummarizationSetBy = Automatic

	column month_start
		dataType: dateTime
		formatString: mmm yyyy
		lineageTag: 00000000-0000-0000-0000-000000000025
		summarizeBy: none
		sourceColumn: month_start

		annotation SummarizationSetBy = Automatic

		annotation UnderlyingDateTimeDataType = Date

	partition dim_date = m
		mode: import
		queryGroup: Gold
		source =
				let
					Dates = List.RemoveNulls(List.Combine({fact_sales[order_date], fact_sales[shipping_date], fact_sales[due_date]})),
					Start = #date(Date.Year(List.Min(Dates)), 1, 1),
					End = #date(Date.Year(List.Max(Dates)), 12, 31),
					List = List.Dates(Start, Duration.Days(End - Start) + 1, #duration(1, 0, 0, 0)),
					Table = Table.FromList(List, Splitter.SplitByNothing(), type table [date_key = date]),
					Year = Table.AddColumn(Table, "year", each Date.Year([date_key]), Int64.Type),
					MonthNo = Table.AddColumn(Year, "month_no", each Date.Month([date_key]), Int64.Type),
					MonthName = Table.AddColumn(MonthNo, "month_name", each Date.ToText([date_key], [Format = "MMM", Culture = "en-US"]), type text),
					MonthStart = Table.AddColumn(MonthName, "month_start", each Date.StartOfMonth([date_key]), type date)
				in
					MonthStart

	annotation PBI_ResultType = Table
```

### Complete-month flag

Decision R3-2, option B. Add to the date table so time visuals can show complete months only while
totals keep every row. Add these steps before `in` in the `dim_date` partition (and reference the last
step after `in`):

```
					DataStart = List.Min(List.RemoveNulls(fact_sales[order_date])),
					DataEnd = List.Max(List.RemoveNulls(fact_sales[order_date])),
					Complete = Table.AddColumn(MonthStart, "is_complete_month", each Date.StartOfMonth([date_key]) >= DataStart and Date.EndOfMonth([date_key]) <= DataEnd, type logical)
```

and the column:

```
	column is_complete_month
		dataType: boolean
		lineageTag: 00000000-0000-0000-0000-000000000026
		summarizeBy: none
		sourceColumn: is_complete_month

		annotation SummarizationSetBy = Automatic
```

Filter trend visuals (or the page) on `dim_date[is_complete_month] = TRUE`. In PBIR that is a visual
filter in the visual's `filterConfig`; validate it with the report tool. The rule only affects the
edges: the first month is complete only if data starts on its 1st, the last month only if data runs
to its last day. It does not detect an incomplete extract inside a full calendar month – that is a
separate check in decision R3-2.

## 7. Relationships (active and inactive)

No `///` descriptions in this file – relationships do not support them.

```
relationship 00000000-0000-0000-0000-000000000030
	fromColumn: fact_sales.customer_key
	toColumn: dim_customers.customer_key

relationship 00000000-0000-0000-0000-000000000031
	fromColumn: fact_sales.product_key
	toColumn: dim_products.product_key

relationship 00000000-0000-0000-0000-000000000032
	fromColumn: fact_sales.order_date
	toColumn: dim_date.date_key

relationship fact_sales_shipping_date_dim_date
	isActive: false
	fromColumn: fact_sales.shipping_date
	toColumn: dim_date.date_key
```

## 8. Measure table and measures

```
/// All measures. The column is a hidden placeholder required for a calculated table.
table _Measures
	lineageTag: 00000000-0000-0000-0000-000000000040

	measure Revenue = SUM ( fact_sales[sales_amount] )
		formatString: #,0
		displayFolder: Sales
		lineageTag: 00000000-0000-0000-0000-000000000041

	measure 'Total Orders' = DISTINCTCOUNT ( fact_sales[order_number] )
		formatString: #,0
		displayFolder: Sales
		lineageTag: 00000000-0000-0000-0000-000000000042

	measure 'Active Customers' = DISTINCTCOUNT ( fact_sales[customer_key] )
		formatString: #,0
		displayFolder: Customers
		lineageTag: 00000000-0000-0000-0000-000000000043

	/// Point-in-time unit cost stored on the fact (decision R2-1, option C).
	/// If the user chose current cost instead: SUMX ( fact_sales, fact_sales[quantity] * RELATED ( dim_products[cost] ) )
	measure Cost = SUMX ( fact_sales, fact_sales[quantity] * fact_sales[unit_cost] )
		formatString: #,0
		displayFolder: Sales
		lineageTag: 00000000-0000-0000-0000-000000000044

	measure 'Gross Margin' = [Revenue] - [Cost]
		formatString: #,0
		displayFolder: Sales
		lineageTag: 00000000-0000-0000-0000-000000000045

	measure 'Margin %' = DIVIDE ( [Gross Margin], [Revenue] )
		formatString: 0.0%
		displayFolder: Sales
		lineageTag: 00000000-0000-0000-0000-000000000046

	measure 'Revenue YTD' = TOTALYTD ( [Revenue], dim_date[date_key] )
		formatString: #,0
		displayFolder: Time Intelligence
		lineageTag: 00000000-0000-0000-0000-000000000047

	measure 'Revenue LY' = CALCULATE ( [Revenue], SAMEPERIODLASTYEAR ( dim_date[date_key] ) )
		formatString: #,0
		displayFolder: Time Intelligence
		lineageTag: 00000000-0000-0000-0000-000000000048

	measure 'Revenue YoY %' =
			VAR _ly = [Revenue LY]
			RETURN
			    DIVIDE ( [Revenue] - _ly, _ly )
		formatString: 0.0%
		displayFolder: Time Intelligence
		lineageTag: 00000000-0000-0000-0000-000000000049

	measure 'Revenue by Ship Date' = CALCULATE ( [Revenue], USERELATIONSHIP ( fact_sales[shipping_date], dim_date[date_key] ) )
		formatString: #,0
		displayFolder: Time Intelligence
		lineageTag: 00000000-0000-0000-0000-000000000050

	/// Last day of the latest COMPLETE month with sales. Skips a partial final month.
	measure 'Latest Complete Month End' =
			VAR _max = CALCULATE ( MAX ( fact_sales[order_date] ), REMOVEFILTERS ( dim_date ) )
			RETURN
			    IF ( _max = EOMONTH ( _max, 0 ), _max, EOMONTH ( _max, -1 ) )
		formatString: yyyy-mm-dd
		isHidden
		displayFolder: Dashboard
		lineageTag: 00000000-0000-0000-0000-000000000051

	measure 'Revenue Current Month' =
			VAR _end = [Latest Complete Month End]
			RETURN
			    CALCULATE ( [Revenue], REMOVEFILTERS ( dim_date ),
			        dim_date[date_key] > EOMONTH ( _end, -1 ), dim_date[date_key] <= _end )
		formatString: #,0
		displayFolder: Dashboard
		lineageTag: 00000000-0000-0000-0000-000000000052

	measure 'Revenue Previous Month' =
			VAR _end = [Latest Complete Month End]
			RETURN
			    CALCULATE ( [Revenue], REMOVEFILTERS ( dim_date ),
			        dim_date[date_key] > EOMONTH ( _end, -2 ), dim_date[date_key] <= EOMONTH ( _end, -1 ) )
		formatString: #,0
		displayFolder: Dashboard
		lineageTag: 00000000-0000-0000-0000-000000000053

	measure 'Revenue Month Label' =
			VAR _cm = [Revenue Current Month]
			VAR _pm = [Revenue Previous Month]
			RETURN
			    "Previous month: " & FORMAT ( _pm, "#,0" ) & " (" & FORMAT ( DIVIDE ( _cm - _pm, _pm ), "+0.0%;-0.0%;0.0%" ) & ")"
		displayFolder: Dashboard
		lineageTag: 00000000-0000-0000-0000-000000000054

	/// Category with the most orders in the current filter context.
	measure 'Top Category' =
			VAR _items = FILTER ( VALUES ( dim_products[category] ), NOT ISBLANK ( dim_products[category] ) )
			VAR _top1 = TOPN ( 1, _items, [Total Orders], DESC )
			RETURN
			    CONCATENATEX ( _top1, dim_products[category], ", " )
		displayFolder: Dashboard
		lineageTag: 00000000-0000-0000-0000-000000000055

	column Placeholder
		isHidden
		lineageTag: 00000000-0000-0000-0000-000000000056
		summarizeBy: none
		isNameInferred
		sourceColumn: [Placeholder]

		annotation SummarizationSetBy = Automatic

	partition _Measures = calculated
		mode: import
		source = ROW ( "Placeholder", BLANK () )
```

Do not name a DAX variable `LastDate` – it collides with the `LASTDATE` function.

## 9. Minimal report with theme

`<Name>.Report/report.json` – one empty page plus a base theme. The theme file must exist at
`StaticResources/SharedResources/BaseThemes/<theme>.json`; copy it from any project saved by a
current Power BI Desktop (it is not distributed with this skill) and use its `name` below.

```json
{
  "id": 0,
  "resourcePackages": [
    {
      "resourcePackage": {
        "name": "SharedResources",
        "type": 2,
        "items": [ { "type": 202, "path": "BaseThemes/<theme>.json", "name": "<theme>" } ],
        "disabled": false
      }
    }
  ],
  "sections": [
    {
      "id": 0,
      "name": "ReportSection",
      "displayName": "Page 1",
      "filters": "[]",
      "ordinal": 0,
      "visualContainers": [],
      "config": "{}",
      "displayOption": 1,
      "width": 1280,
      "height": 720
    }
  ],
  "config": "{\"version\":\"5.73\",\"themeCollection\":{\"baseTheme\":{\"name\":\"<theme>\",\"type\":2}},\"activeSectionIndex\":0,\"settings\":{\"useNewFilterPaneExperience\":true,\"allowChangeFilterTypes\":true,\"useDefaultAggregateDisplayName\":true}}",
  "layoutOptimization": 0
}
```

### Dark KPI tiles with `cardVisual`
Container background alone is not enough; the card's inner tile covers it. Add to the visual's
`objects`:

```json
"fillCustom": [
  {
    "properties": { "transparency": { "expr": { "Literal": { "Value": "100D" } } } },
    "selector": { "id": "default" }
  }
]
```
