# end-to-end-bi-project-markit
BI Dashboard solving complex data challenges: search-to-sale correlation, advanced DAX (Top N + Subtotal), and ETL in Power Query
MarkIt C2C Platform: End-to-End BI Dashboard

Project Overview
This project involved the end-to-end design, development, and delivery of a Business Intelligence dashboard in Power BI for MarkIt, a fictional C2C sales platform.
The core business problem was to translate raw platform data into actionable insights for stakeholders, primarily the VP of Sales. The goal was to understand the complete sales funnel, from initial user search to final purchase, and to quantify the listing lifecycle.
This solution delivers a dynamic, interactive dashboard that visualizes key performance indicators (KPIs), analyzes search behavior, tracks the sales lifecycle over time, and uncovers the direct correlation between search volume and sales completed.
<img width="1283" height="726" alt="MarkIt_dashboard_v1_2025-10-22" src="https://github.com/user-attachments/assets/8271c965-fa40-4489-b2b7-eb82b539d2cb" />
<img width="1283" height="726" alt="MarkIt_dashboard_v1_2025-10-22" src="https://github.com/user-attachments/assets/5490b4a6-a8f6-4716-88ef-9279c3e1aae1" />


1. Business Requirements & Strategy

Based on stakeholder interviews and requirement documents, the project was built to answer three core business questions:

Listing Lifecycle: "What is the flow of new listings, sales, and deletions over time?"

Time-to-Sale: "How long does it take for a product to sell, and how does this differ by category?"

Search-to-Purchase Funnel: "What are users searching for, and how does search volume correlate with sales?"


<iframe title="Project MarkIt Docs_Final" width="600" height="373.5" src="https://app.powerbi.com/view?r=eyJrIjoiMDVhMmEyODItYTZiNC00M2JkLWE3NWItYWVjYmFhODAzNzlkIiwidCI6IjZjMDMxZjk0LWM0MDItNDMzYS05MmQyLTJkM2NlODUxNmRhMyIsImMiOjF9" frameborder="0" allowFullScreen="true"></iframe>


2. Data Transformation (ETL) in Power Query

The raw data (F_Listings.csv, F_SearchLogs.csv) was un-pivoted and required significant cleaning and transformation.

Date Extraction: Created dedicated date columns (SearchDate_Date from timestamp) to build relationships.

Search Term Normalization: Created a new TermKey column from the raw search_term (or term_norm) by trimming whitespace and converting to lowercase. This was critical for accurate grouping.

Error Handling: Managed and corrected M query syntax errors (e.g., Token Eof expected, Column not found) by rebuilding and validating the query steps, ensuring the F_SearchLogs table loaded correctly.


3. Data Modeling (Star Schema)

A robust Star Schema was implemented to optimize performance and ensure correct DAX calculations:

Facts Tables: F_Listings (listing/sale events) and F_SearchLogs (search events).

Dimension Tables:

D_Date (Generated in DAX)

D_ItemCategory (From source)

D_Terms (Generated in DAX)


Key Relationships:

D_Date[Date] -> F_Listings[CreatedAt_Date], F_Listings[SoldDate], etc.

D_Date[Date] -> F_SearchLogs[SearchDate_Date]

D_Terms[TermKey] -> F_SearchLogs[TermKey] (This was the key relationship that enabled the search funnel analysis).


4. Technical Challenges & DAX Solutions

The primary challenge was solving the "Hardcore Mode" questions from the project brief.

Challenge 1: The "Top 10 + Total" Problem

Problem: The Popular Search Terms table needed to show the Top 10 products and a correct Subtotal line. Power BI's default "Top N" filter breaks the "Total" row, and adding a Category <> BLANK filter (to hide "cheap", "sale") also incorrectly filtered out the "Total" row.

Solution: A complex, "all-in-one" DAX measure was built. This measure uses ISINSCOPE to differentiate between a product row and the total row. For product rows, it applies the Top 10 rank. For the total row, it uses SUMX over a virtual table of only the Top 10 items, calculating a true subtotal.

Challenge 2: The Search-to-Sale Correlation

Problem: The data did not provide a key to link a specific search_event_id to a specific listing_id. It was impossible to calculate a "Conversion Rate per Term."

Solution: The strategy was shifted. Instead of attribution, we proved correlation. A Scatter Plot (Sales Completed vs. Search Volume) was built, plotting sales against searches by date. The resulting positive-slope Trend Line visually proved that days with more searches directly correlate with days with more sales.

Challenge 3: Misaligned Time-Series Data

Problem: The Listings, Sales & Deletions line chart looked broken because Listings data stopped on 10/07/2025, while Sales continued to 11/28/2025.

Solution: A simple, non-destructive visual-level filter was applied to the chart, clipping the x-axis for all three metrics at 10/07/2025, creating a clean, aligned visual for analysis.


5. Glossary: Key Metrics & Formulas

This glossary defines the final DAX measures used to power the dashboard.

General notes:

D_Date is the single date table, joined to F_Listings[CreatedDate] (active) and F_Listings[SoldDate] (inactive; activated via USERELATIONSHIP), and to F_SearchLogs[SearchDate] for searches.

TermKey = normalized term (lower+trim) derived in Power Query (as per the PRD—pre-processing).

Counting base
Listings Posted :=
CALCULATE(
    DISTINCTCOUNT( F_Listings[listing_id] ),
    USERELATIONSHIP( D_Date[Date], F_Listings[CreatedDate] )
)

Sales Completed :=
CALCULATE(
    DISTINCTCOUNT( F_Listings[listing_id] ),
    USERELATIONSHIP( D_Date[Date], F_Listings[SoldDate] ),
    NOT ISBLANK( F_Listings[SoldDate] )
)

Search Events :=
DISTINCTCOUNT( F_SearchLogs[search_event_id] )

Time
Avg Time-to-Sale (Days) :=
AVERAGEX(
    FILTER( F_Listings, NOT ISBLANK( F_Listings[SoldDate] ) ),
    DATEDIFF( F_Listings[CreatedDate], F_Listings[SoldDate], DAY )
)

Conversion Rates (global)
CR – Search Traffic :=
DIVIDE( [Sales Completed], [Search Events] )

CR – Listings :=
DIVIDE( [Sales Completed], [Listings Posted] )

Terms (term/category context)
-- searches in the context of the selected term (Top-N/table/word cloud)
Searches (Term Context) :=
VAR _t = SELECTEDVALUE( D_Terms[TermKey] )
RETURN CALCULATE( [Search Events], TREATAS( {_t}, F_SearchLogs[TermKey] ) )

-- sales for the term by category (maps term → listing category)
Sales Completed (Term Category) :=
VAR _t = SELECTEDVALUE( D_Terms[TermKey] )
VAR _cat = LOOKUPVALUE( D_SearchMap[category], D_SearchMap[search_term], _t )
RETURN
CALCULATE(
    [Sales Completed],
    TREATAS( {_cat}, F_Listings[category] )
)

-- per-term CR (with fallback to global)
CR (Smart) :=
VAR hasTerm = ISFILTERED( D_Terms[TermKey] )
VAR crTerm  = DIVIDE( [Sales Completed (Term Category)], [Searches (Term Context)] )
RETURN IF( hasTerm, crTerm, [CR – Search Traffic] )

Top-10 and Total (side table)
Searches Rank :=
RANKX( ALL( D_Terms[TermKey] ), [Searches (Term Context)], , DESC )

Top10 Searches (Sum) :=
VAR _top = TOPN( 10, ALL( D_Terms[TermKey] ), [Searches (Term Context)], DESC )
RETURN CALCULATE( [Searches (Term Context)], KEEPFILTERS( _top ) )


Why Word Cloud = Table:
The PRD requires normalized terms and the removal of PII/stop-words; we applied TermKey in Power Query and used the same event-count measure in both the Word Cloud and the ranked Table—preventing generic terms (e.g., sale/cheap) from inflating the cloud.

Metric (DAX Measure)	 	Definition (The "Why")			Final DAX Formula
Listings Posted			Total count of unique listings created. DISTINCTCOUNT('F_Listings'[listing_id])
Sales Completed			Total count of unique listings sold.	CALCULATE(DISTINCTCOUNT('F_Listings'[listing_id]), 'F_Listings'[status] = "sold")
Avg. Time-to-Sale		The average number of days from listing creation to sale. AVERAGE('F_Listings'[Time-to-Sale])
Searches (Master KPI) 		Total count of unique search events.	DISTINCTCOUNT('F_SearchLogs'[search_event_id])
CR - Search Traffic (Global CR) The site-wide conversion rate.		DIVIDE( [Sales Completed], [Searches] )
Searches (Products Only)	A version of [Searches] that filters out non-product terms (like 'cheap', 'sale') by checking the D_Terms[Category].							CALCULATE( [Searches], 'D_Terms'[Category] <> BLANK() )
Searches Rank(Helper Measure) 	Ranks TermKey based only on product-related searches. RANKX( ALLSELECTED( 'D_Terms'[TermKey] ), [Searches (Products Only)], , DESC, Dense )
Searches (Top 10 Subtotal)	(The "Hardcore" Solution) The final measure used in the table. It dynamically shows the Top 10 product searches and calculates the correct subtotal for only those 10 items.	VAR _rank = [Searches Rank] VAR _sp = [Searches (Products Only)] RETURN IF ( ISINSCOPE ( 'D_Terms'[TermKey] ), IF ( _rank <= 10, _sp, BLANK() ), SUMX( FILTER( ALLSELECTED('D_Terms'[TermKey]), [Searches Rank] <= 10 ), _sp ) )


6. Relationships & Filters (model summary)

D_Date 1:* F_Listings[CreatedDate] (active) and D_Date 1:* F_Listings[SoldDate] (inactive; activated in sales/time measures).

D_Date 1:* F_SearchLogs[SearchDate] (active).

D_Terms 1:* F_SearchLogs[TermKey] (for term context).

D_SearchMap (term→category) and D_ItemCategory (category→group) to link search → listing.

Daily/quarterly/annual granularities as per Strategy/SRD.


7. QA, accessibility & privacy (acceptance checklist)

Performance < 3s and coverage ≥95% before sign-off (with QA report).

Accessibility: enlarged fonts, high contrast, TTS and keyboard navigation validated (checklist included).

Privacy: removal of PII in terms; data-use note in the Metric Dictionary; Governance approval.
