📊 Maven Toys Mexico: Strategic Inventory Optimization & Rebalancing Model
🎯 Project Objective
Analyze retail operations for a 50-store toy chain in Mexico over 8 quarters to identify systemic supply chain inefficiencies. This project focuses on isolating stockout constraints versus capital stagnation, designing an Excel-based Inventory Rebalancing Engine (POC) using Pareto principles, and formulating data-driven logistics strategies to recover trapped capital and minimize lost profit.

📈 Executive Summary & Core Challenges
A comprehensive BigQuery SQL analysis of Maven Toys Mexico’s operations revealed severe multi-node supply chain imbalances. While individual stores registered strong demand patterns, the network suffered from structural alignment failures between inventory positioning and real-time sales velocity.

The Problem Ecosystem
The Stockout Trap (Shortages): High-velocity products frequently dropped below critical buffer zones, leading to shelf depletion during peak operational windows and forcing customers to look elsewhere.

The Stagnant Capital Trap (Overstock): Slow-moving stock accumulated in low-yield locations, trapping cash liquidity in depreciating inventory while incurring unnecessary holding costs.

The Structural Friction: Traditional fixed-interval replenishment loops failed to adapt dynamically to distinct store footprints (e.g., Downtown locations vs. Residential hubs).

🛠️ The Tech Solution: The Predictive Inventory & Rebalancing Model
To address these challenges without changing the core corporate purchasing pipeline, I designed and executed a scalable Proof of Concept (POC) Inventory Management Model in Excel.

[ Excess Stock Detected ] ──► ( Rebalancing Engine ) ──► [ Inter-Store Shipment Route ]
                                        │
                                        └──► [ Target Store: Immediate Shortage Mitigated ]

![Excel Inventory Model Support](excel_model.png) 
                                        
Technical Workflow & Logic Archetype1. Demand Profiling & Target IsolationPareto Optimization (80/20 Rule): To validate the model's logic efficiently, I narrowed the data footprint down to Q3 2018, focusing exclusively on high-impact Downtown stores and the Top 8 primary profit-generating products (representing ~57% of total network profitability, led by Colorbuds and Action Figures).Dynamic Daily Run Rate (DRR): Replaced static historical averages with a calculated continuous daily demand factor per product per store:$$\text{Daily Sales Rate} = \frac{\text{Total Units Sold in Period}}{\text{Total Operational Days in Period}}$$2. Health Triage & Threshold LogicThe model continuous monitors stock depth using Days of Supply (DoS) metrics:$$\text{Days of Supply} = \frac{\text{Stock on Hand}}{\text{Daily Sales Rate}}$$Every product/store intersection is categorized automatically via logical thresholds:Shortage ($\text{DoS} < 3 \text{ days}$): Immediate critical trigger. Triggers an automated suggested order calculation to restore a healthy 7-day operational baseline:$$\text{Suggested Order} = (7 - \text{Days of Supply}) \times \text{Daily Sales Rate}$$Healthy ($3 \le \text{DoS} \le 7 \text{ days}$): Optimal balance of optimal turnover and buffer safety.Overstock ($\text{DoS} > 7 \text{ days}$): Cash stagnation trigger. Flags excessive holding volume.3. Inter-Store Rebalancing Engine (Logistics Matrix)Instead of recommending expensive bulk orders from central manufacturing hubs, the model utilizes an advanced Intra-Network Rebalancing Pivot Matrix.The engine checks for internal counter-matches: it pairs stores experiencing an urgent Shortage with nearby stores holding an Overstock for the exact same SKU.By shifting excess inventory horizontally between stores, the system directly rescues Capital Tied Up from overstocked shelves to satisfy uncaptured demand at zero secondary production cost.💰 Economic Impact & Quantifiable ValueBy applying the rebalancing model logic across the target network, the financial benefits are immediately clear:Lost Profit Recovery (Shortage Mitigation): Calculates potential revenue saved by eliminating stockouts of high-yield items:$$\text{Loss of Profit} = \text{Daily Run Rate} \times (\text{Price} - \text{Cost}) \times (7 - \text{Days of Supply})$$Trapped Capital Release (Overstock Liquidation): Quantifies cash freed up from over-shelved items:$$\text{Capital Tied Up} = (\text{Stock on Hand} - (\text{Daily Run Rate} \times 7)) \times \text{Cost}$$
-- ==============================================================================
-- SECTION 1: NETWORK DATA SANITY CHECK & INITIAL DISCOVERY
-- ==============================================================================

-- Checking primary constraints across the core tables
SELECT * FROM `toys.sales` LIMIT 100;
SELECT * FROM `toys.products` LIMIT 100;
SELECT * FROM `toys.inventory` LIMIT 100;
SELECT * FROM `toys.stores` LIMIT 100;

-- ==============================================================================
-- SECTION 2: DAILY RUN RATE & DEMAND MATRIX DEVELOPMENT
-- Takeaway: Calculates daily sales velocity benchmarks for Q3 across 2017 and 2018 
-- to establish baseline seasonal demand patterns.
-- ==============================================================================
WITH StoreProductMetrics AS (
    SELECT
        Store_ID, 
        Product_ID,
        SUM(CASE WHEN EXTRACT(YEAR FROM Date) = 2018 THEN Units ELSE 0 END) / 92.0 AS Avg_2018,
        SUM(CASE WHEN EXTRACT(YEAR FROM Date) = 2017 THEN Units ELSE 0 END) / 92.0 AS Avg_2017
    FROM `toys.sales`
    WHERE EXTRACT(QUARTER FROM Date) = 3
    GROUP BY Store_ID, Product_ID
)
SELECT 
    spm.Store_ID, 
    inv.product_id, 
    inv.stock_on, 
    Avg_2018 AS daily_demand_2018,
    CASE
        WHEN (spm.Avg_2018 / COALESCE(spm.Avg_2017, 0)) > 1 THEN (spm.Avg_2018 / COALESCE(spm.Avg_2017, 0))
        ELSE 1
    END AS stock_balance
FROM StoreProductMetrics spm 
JOIN `toys.inventory` inv ON spm.Store_ID = inv.store_id;

-- ==============================================================================
-- SECTION 3: REBALANCING LOGISTICS NODE (DOWNTOWN FOCUS)
-- Takeaway: Isolates current inventory depth specifically for Downtown locations. 
-- Maps live stock counts against daily sales rates to pinpoint runout windows.
-- ==============================================================================
WITH StoreProductMetrics AS (
    SELECT
        CAST(Store_ID AS INT64) AS Store_ID,
        CAST(Product_ID AS INT64) AS Product_ID,
        SUM(CASE WHEN EXTRACT(YEAR FROM Date) = 2018 THEN Units ELSE 0 END) / 92.0 AS Avg_2018,
        SUM(CASE WHEN EXTRACT(YEAR FROM Date) = 2017 THEN Units ELSE 0 END) / 92.0 AS Avg_2017
    FROM `toys.sales`
    WHERE EXTRACT(QUARTER FROM Date) = 3
    GROUP BY 1, 2
)
SELECT
    s.Store_Location,
    spm.Store_ID,
    spm.Product_ID,
    inv.Stock_On,
    spm.Avg_2017 AS daily_demand_2017,
    spm.Avg_2018 AS daily_demand_2018,
    -- Calculate specific operational runway (Days of Stock) handling division by zero constraints
    IFNULL(ROUND(CAST(inv.Stock_On AS FLOAT64) / NULLIF(spm.Avg_2018, 0), 2), 999) AS days_of_stock
FROM StoreProductMetrics spm
LEFT JOIN `toys.inventory` inv 
  ON spm.Store_ID = CAST(inv.Store_ID AS INT64) 
  AND spm.Product_ID = CAST(inv.Product_ID AS INT64)
LEFT JOIN `toys.stores` s 
  ON spm.Store_ID = CAST(s.Store_ID AS INT64)
WHERE s.Store_Location = 'Downtown'
ORDER BY days_of_stock ASC;
