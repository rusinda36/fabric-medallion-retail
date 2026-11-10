##Microsoft Fabric Medallion Architecture — Retail Transactions
A practical Bronze–Silver–Gold data engineering experiment in Microsoft Fabric, using 1,067,371 real retail transaction rows. The goal was to understand what changes between raw ingestion and a business-facing metric—and how a small classification rule can materially change that metric.

The project uses a schema-enabled Fabric Lakehouse, Spark notebooks, Delta tables, and the UCI Online Retail II dataset. Monetary amounts are in GBP (£), as reported by the dataset; they are not USD.
The question
What happens when a transaction file contains missing customer IDs, cancellations, negative quantities, zero prices, and accounting adjustments?

Simply loading the file and summing Quantity * Price does not answer what the business means by sales. This experiment keeps the source available, makes the classification rules explicit, and shows their effect on the final reporting metric.
Architecture
UCI Online Retail II (.xlsx; two worksheets)
                    |
                    v
OneLake / Files/raw/online_retail_II.xlsx
                    |
                    v
BRONZE  bronze.retail_transactions
        1,067,371 rows; source values stored as text
        plus worksheet/file provenance
                    |
                    v
SILVER  silver.retail_transactions
        typed fields, transaction classification,
        quality flags, calculated line amounts
        1,067,371 rows retained
                    |
                    v
GOLD    gold.monthly_country_sales
        monthly and country-level sales metrics
        1,063,908 source rows included
                    |
                    +--> gold.monthly_country_sales_scenario
                         alternate business rule:
                         includes bad-debt adjustments
All three layers are implemented as schemas in one Lakehouse, RetailMedallion, rather than as separate Lakehouses. Gold tables are Delta tables queried with Spark SQL.
Project notebooks
Notebook	Purpose
NB_01_Bronze_Ingestion.ipynb	Read both Excel worksheets, retain source values and provenance, and create the Bronze Delta table.
NB_02_Silver_Transformation.ipynb	Convert types, classify transaction rows, record quality flags, and calculate LineAmount.
NB_03_Gold_Aggregation.ipynb	Build monthly/country Gold metrics, reconcile totals, and compare two business-rule scenarios.


The notebooks were developed and run in Microsoft Fabric. They expect the RetailMedallion Lakehouse to be attached as the default Lakehouse.
Bronze: what arrived?
Worksheet	Rows
Year 2009–2010	525,461
Year 2010–2011	541,910
Total	1,067,371


The Bronze ingestion does not deduplicate or remove questionable transactions. It reads source values as text and adds SourceSheet and SourceFile for provenance.

Initial data-quality profiling found:
Observation	Rows
Missing CustomerID	243,007
Negative Quantity	22,950
Zero Price	6,202
Invoice beginning with C	19,494
Missing Quantity	0
Missing Price	0


These categories can overlap; their counts must not be added together.
Silver: what do the records mean?
The Silver notebook converts quantities, prices, and dates to appropriate types and assigns TransactionClass. The final classifications are:
Class	Rows	Treatment
RegularSale	1,044,415	Included in the selected Gold sales metric
Cancelled	19,493	Included as negative amounts in the selected Gold metric
NegativeWithoutC	3,457	Kept in Silver; zero-priced in this dataset
BadDebtAdjustment	5	Kept separately from sales
CancellationAnomaly	1	Flagged for review, excluded from selected Gold metric
Total	1,067,371	No rows dropped from Silver


Notable findings
- C-prefixed invoices account for 19,493 negative-quantity rows, but one C-prefixed row has a positive quantity and line amount (C496350, Manual, £373.57).
- Another 3,457 rows have negative quantities without the C prefix. All have a zero price in this dataset. A negative quantity alone is therefore not enough to define a monetary return.
- Five records (StockCode = B, Description = Adjust bad debt) total −£158,676.14. Treating them as ordinary sales would distort the selected sales metric.
- Rows without a customer ID are retained. Missing customer identity does not automatically invalidate a transaction amount.
This is a classification exercise, not a claim that every row has been fully validated or that every accounting treatment has been approved by a finance team. The CancellationAnomaly remains unresolved.
Gold: how much does the business rule change the result?
The main Gold table aggregates by month and country. It includes only RegularSale and Cancelled transactions for the selected sales metric. The sums are calculated at full precision; rounding is applied when displaying results.
Metric	Value
Source rows contributing to Gold	1,063,908
Gross sales (positive line amounts)	£20,972,594.57
Cancellation amounts	−£1,527,041.43
Net sales under this rule	£19,445,553.14


The source-row count in Gold is the number of rows contributing to the aggregates, not the number of monthly/country summary rows or distinct orders.
Reconciliation across layers
Layer	Source rows represented	Sum of line amounts
Bronze	1,067,371	£19,287,250.57
Silver	1,067,371	£19,287,250.57
Main Gold	1,063,908	£19,445,553.14


The 3,463 excluded source rows are:
Class excluded from main Gold	Rows	Amount
NegativeWithoutC	3,457	£0.00
BadDebtAdjustment	5	−£158,676.14
CancellationAnomaly	1	£373.57
Total	3,463	−£158,302.57


The main Gold metric is £158,302.57 higher than the unfiltered Bronze/Silver sum, because the selected business definition excludes those amounts. This is not additional revenue or evidence of data loss.
Controlled business-rule experiment
A separate Gold scenario includes BadDebtAdjustment along with RegularSale and Cancelled—without changing Bronze, Silver, or the original Gold table.
Scenario	Source rows included	Net amount
Main Gold: exclude bad debt	1,063,908	£19,445,553.14
Alternative Gold: include bad debt	1,063,913	£19,286,877.00
Difference	+5	−£158,676.14


Five rows changed the result by £158,676.14. The point is not that one scenario is automatically correct: the point is that the definition of a business metric must be explicit and reviewable.
Reproduce the experiment
1. Obtain the Online Retail II Excel workbook from UCI. The raw dataset is intentionally not committed to this repository.
2. In Microsoft Fabric, create a schema-enabled Lakehouse named RetailMedallion.
3. Create the bronze, silver, and gold schemas. The Lakehouse may also contain its default dbo schema.
4. Upload the workbook to Files/raw/ in the Lakehouse.
5. Attach RetailMedallion as the default Lakehouse for each Notebook.
6. Run the notebooks in order: Bronze ingestion → Silver transformation → Gold aggregation.
7. Validate row counts, classification totals, the main Gold figures, and the alternative-scenario difference against the tables above.
The notebooks require a functioning Microsoft Fabric Spark environment and access to the Lakehouse. The Bronze notebook expects one .xlsx workbook in Files/raw/. Its load strategy overwrites the first worksheet's table, then appends the second; it is not an atomic multi-sheet ingestion transaction.
Scope and limitations
- The experiment demonstrates classification, aggregation, and reconciliation, not a production orchestration or incremental-load framework.
- Rows have not been deduplicated. Repeated-looking retail lines are not necessarily ingestion duplicates.
- The source data are denominated in GBP. No currency conversion was performed.
- The chosen Gold metric is a defined analytical view, not independently certified accounting revenue.
- Zero-priced adjustments, missing customer IDs, bad-debt entries, and the anomalous cancellation require business context before a production policy is adopted.
Source and attribution
Online Retail II, UCI Machine Learning Repository, contributed by Daqing Chen.
- Dataset: https://archive.ics.uci.edu/dataset/502/online+retail+ii
- DOI: https://doi.org/10.24432/C5CG6D
- Microsoft Fabric Lakehouse schemas: https://learn.microsoft.com/en-us/fabric/data-engineering/lakehouse-schemas
