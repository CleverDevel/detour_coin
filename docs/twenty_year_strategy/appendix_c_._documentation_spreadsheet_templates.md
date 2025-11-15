# Appendix C: Documentation Spreadsheet Templates

## Overview

This appendix provides comprehensive spreadsheet templates for tracking all aspects of DetourCoin operations, founder grants, performance metrics, and tax compliance. These templates are designed for use in Google Sheets or Excel and include detailed formulas, validation rules, and best practices.

**Critical Importance:**  
Meticulous documentation is essential for:
- Tax compliance (IRS reporting)
- Performance-based grant calculations
- Internal valuation justification
- Audit trail and transparency
- Long-term wealth tracking

---

## Table of Contents

1. [Master Tracking Workbook Structure](#master-tracking-workbook-structure)
2. [Sheet 1: Daily Emissions Log](#sheet-1-daily-emissions-log)
3. [Sheet 2: Founder Grant Tracking](#sheet-2-founder-grant-tracking)
4. [Sheet 3: Milestone Bonuses](#sheet-3-milestone-bonuses)
5. [Sheet 4: Performance Metrics](#sheet-4-performance-metrics)
6. [Sheet 5: Internal Valuations](#sheet-5-internal-valuations)
7. [Sheet 6: Tax Records](#sheet-6-tax-records)
8. [Sheet 7: Merchant Growth Tracker](#sheet-7-merchant-growth-tracker)
9. [Sheet 8: Transaction Volume Analytics](#sheet-8-transaction-volume-analytics)
10. [Sheet 9: Token Holdings Dashboard](#sheet-9-token-holdings-dashboard)
11. [Sheet 10: Gas Cost Tracking](#sheet-10-gas-cost-tracking)
12. [Automation & Integration](#automation--integration)
13. [Backup & Security](#backup--security)

---

## Master Tracking Workbook Structure

### File Organization

**Recommended Structure:**
```
DetourCoin_Master_Ledger_2025.gsheet
├── 1. Dashboard (Summary)
├── 2. Daily Emissions Log
├── 3. Founder Grant Tracking
├── 4. Milestone Bonuses
├── 5. Performance Metrics
├── 6. Internal Valuations
├── 7. Tax Records
├── 8. Merchant Growth Tracker
├── 9. Transaction Volume Analytics
├── 10. Token Holdings Dashboard
├── 11. Gas Cost Tracking
└── 12. Documentation & Notes
```

**Annual Archiving:**
- Create new workbook each year
- Archive previous year as read-only
- Maintain running totals across years

---

## Sheet 1: Daily Emissions Log

### Purpose
Track every daily emission execution for transparency, troubleshooting, and supply verification.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Date | Date | =TODAY() | Date of emission execution |
| B | Execution Day | Number | =ROW()-2 | Day number since deployment |
| C | Block Number | Number | Manual entry | Polygon block number |
| D | Transaction Hash | Text | Manual entry | Ethereum transaction hash |
| E | Amount Emitted | Number | Manual entry | DTC emitted (without decimals) |
| F | Days Elapsed | Number | Manual entry | Days since last emission |
| G | Cumulative Supply | Number | =SUM($E$2:E2) | Running total of all emissions |
| H | Gas Used | Number | Manual entry | Gas units consumed |
| I | Gas Price (Gwei) | Number | Manual entry | Gas price in Gwei |
| J | Gas Cost (MATIC) | Number | =H2*I2/1000000000 | Calculated gas cost |
| K | Gas Cost (USD) | Number | Manual entry | USD value of gas |
| L | Emission Rate | Number | Manual entry | Current emission rate setting |
| M | Status | Dropdown | SUCCESS, FAILED, SKIPPED | Execution status |
| N | Notes | Text | Manual entry | Any issues or observations |

### Data Validation Rules

**Column C (Block Number):**
```
Type: Number
Criteria: Greater than 0
```

**Column D (Transaction Hash):**
```
Type: Custom formula
Formula: =LEN(D2)=66
Error message: Must be valid Ethereum transaction hash (66 characters starting with 0x)
```

**Column E (Amount Emitted):**
```
Type: Number
Criteria: Between 50000 and 150000
Error message: Must be between 50,000 and 150,000 DTC
```

**Column M (Status):**
```
Type: List from range
Values: SUCCESS, FAILED, SKIPPED
```

### Conditional Formatting

**Status Column (M):**
```
Condition 1: Cell is equal to "SUCCESS" → Green background, dark green text
Condition 2: Cell is equal to "FAILED" → Red background, white text
Condition 3: Cell is equal to "SKIPPED" → Yellow background, dark text
```

**Days Elapsed (F):**
```
Condition: Cell value > 1 → Orange background
Reason: Highlights missed emissions (should always be 1)
```

**Amount Emitted (E):**
```
Condition: Cell value ≠ $L2 → Yellow background
Reason: Highlights when emitted amount differs from expected rate
```

### Summary Formulas (Top of Sheet)

```
Total Supply (as of today):     =G[last row]
Total Emissions Count:          =COUNTA(A2:A)
Failed Emissions:               =COUNTIF(M2:M,"FAILED")
Total Gas Cost (MATIC):         =SUM(J2:J)
Total Gas Cost (USD):           =SUM(K2:K)
Average Daily Gas Cost:         =AVERAGE(J2:J)
Current Emission Rate:          =L[last row]
Days Since Deployment:          =TODAY()-$A$2
Expected Supply:                =10000000+(B[last row]*50000)
Actual vs Expected Variance:    =G[last row]-[Expected Supply]
```

### Example Entries

| Date | Execution Day | Block # | TX Hash | Amount Emitted | Days Elapsed | Cumulative Supply | Gas Used | Gas Price | Gas Cost (MATIC) | Gas Cost (USD) | Rate | Status | Notes |
|------|---------------|---------|---------|----------------|--------------|-------------------|----------|-----------|------------------|----------------|------|--------|-------|
| 2025-01-15 | 0 | 52847362 | 0xabc123...def | 10000000 | 0 | 10000000 | 84235 | 50 | 0.0042 | $0.0014 | 50000 | SUCCESS | Initial deployment |
| 2025-01-16 | 1 | 52890145 | 0xdef456...ghi | 50000 | 1 | 10050000 | 78452 | 45 | 0.0035 | $0.0012 | 50000 | SUCCESS | First emission |
| 2025-01-17 | 2 | 52932891 | 0xghi789...jkl | 50000 | 1 | 10100000 | 78501 | 48 | 0.0038 | $0.0013 | 50000 | SUCCESS | Normal operation |

---

## Sheet 2: Founder Grant Tracking

### Purpose
Document all performance-based quarterly grants to founder for tax compliance and ownership tracking.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Quarter | Text | Q1 2025, Q2 2025, etc. | Quarter identifier |
| B | Grant Date | Date | Last day of quarter | Date grant executed |
| C | Merchant Target | Number | Manual entry | Quarterly merchant target |
| D | Merchants Achieved | Number | Manual entry | Actual merchants onboarded |
| E | Merchant % | Percentage | =D2/C2 | Achievement percentage |
| F | Platform Target | Percentage | Manual entry | Platform completion target |
| G | Platform Achieved | Percentage | Manual entry | Actual platform completion |
| H | Volume Target | Currency | Manual entry | Transaction volume target |
| I | Volume Achieved | Currency | Manual entry | Actual transaction volume |
| J | Volume % | Percentage | =I2/H2 | Achievement percentage |
| K | Strategic % | Percentage | Manual entry | Strategic milestone score |
| L | Overall Score | Percentage | =(E2*0.30)+(G2*0.25)+(J2*0.25)+(K2*0.20) | Weighted performance score |
| M | Base Grant | Number | 685000 | Quarterly base allocation |
| N | Performance Multiple | Percentage | =IF(L2>=0.6,L2,0) | Multiplier if ≥60% |
| O | Grant Amount | Number | =M2*N2 | Final grant amount |
| P | Transaction Hash | Text | Manual entry | Grant execution TX hash |
| Q | Internal Valuation | Currency | Manual entry | DTC value per token |
| R | Tax Basis | Currency | =O2*Q2 | Taxable compensation value |
| S | Tax Set Aside (37%) | Currency | =R2*0.37 | Estimated tax liability |
| T | Cumulative Grants | Number | =SUM($O$2:O2) | Running total of grants |
| U | Notes | Text | Manual entry | Performance commentary |

### Data Validation Rules

**Column A (Quarter):**
```
Type: Custom formula
Formula: =REGEXMATCH(A2,"Q[1-4] 20[2-9][0-9]")
Error message: Format must be "Q# YYYY" (e.g., Q1 2025)
```

**Column L (Overall Score):**
```
Type: Number
Criteria: Between 0 and 1
Custom format: 0.0%
```

**Column N (Performance Multiple):**
```
Type: Number
Criteria: Between 0 and 1
Note: Auto-calculated, shows 0 if score < 60%
```

### Conditional Formatting

**Overall Score (L):**
```
Condition 1: >= 0.85 → Dark green background (Exceptional)
Condition 2: >= 0.75 → Light green background (Strong)
Condition 3: >= 0.60 → Yellow background (Acceptable)
Condition 4: < 0.60 → Red background (Below threshold - no grant)
```

**Performance Multiple (N):**
```
Condition: = 0 → Red text, bold
Reason: Highlights quarters with no grant due to low performance
```

### Summary Formulas

```
Total Grants (All Time):        =SUM(O:O)
Average Performance Score:      =AVERAGE(L:L)
Grants Above 80% Performance:   =COUNTIF(L:L,">=0.8")
Total Tax Liability:            =SUM(S:S)
Current Year Grants:            =SUMIFS(O:O,B:B,">="&DATE(YEAR(TODAY()),1,1),B:B,"<="&DATE(YEAR(TODAY()),12,31))
Current Year Tax Liability:     =SUMIFS(S:S,B:B,">="&DATE(YEAR(TODAY()),1,1),B:B,"<="&DATE(YEAR(TODAY()),12,31))
```

### Example Entries

| Quarter | Grant Date | Merchant Target | Achieved | Merchant % | Platform Target | Platform Achieved | Volume Target | Volume Achieved | Volume % | Strategic % | Overall Score | Base Grant | Multiple | Grant Amount | TX Hash | Valuation | Tax Basis | Tax Set Aside | Notes |
|---------|------------|-----------------|----------|------------|-----------------|-------------------|---------------|-----------------|----------|-------------|---------------|------------|----------|--------------|---------|-----------|-----------|---------------|-------|
| Q1 2025 | 2025-03-31 | 75 | 50 | 67% | 80% | 80% | $3M | $2M | 67% | 67% | 73.5% | 685,000 | 73.5% | 503,475 | 0xabc... | $0.015 | $7,552 | $2,794 | Strong start, behind merchant targets |
| Q2 2025 | 2025-06-30 | 100 | 85 | 85% | 85% | 90% | $5M | $4.5M | 90% | 75% | 85.8% | 685,000 | 85.8% | 587,730 | 0xdef... | $0.018 | $10,579 | $3,914 | Exceeded most targets |

---

## Sheet 3: Milestone Bonuses

### Purpose
Track one-time achievement bonuses separate from quarterly performance grants.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Date Achieved | Date | Manual entry | When milestone was reached |
| B | Milestone | Text | Manual entry | Description of achievement |
| C | Category | Dropdown | List of categories | Type of milestone |
| D | Bonus Amount | Number | Manual entry | DTC bonus awarded |
| E | Transaction Hash | Text | Manual entry | Grant execution TX hash |
| F | Internal Valuation | Currency | Manual entry | DTC value at time of grant |
| G | Tax Basis | Currency | =D2*F2 | Taxable value |
| H | Tax Set Aside (37%) | Currency | =G2*0.37 | Estimated tax |
| I | Year | Number | =YEAR(A2) | Calendar year |
| J | Quarter | Text | ="Q"&ROUNDUP(MONTH(A2)/3,0)&" "&YEAR(A2) | Quarter achieved |
| K | Cumulative Bonuses | Number | =SUM($D$2:D2) | Running total |
| L | Evidence/Notes | Text | Manual entry | Supporting documentation |

### Milestone Categories (Dropdown)

```
- Regional Expansion
- Platform Milestone
- Technology Achievement
- Partnership Acquisition
- Financial Milestone
- User Milestone
- Regulatory/Compliance
- Strategic Achievement
```

### Pre-Defined Milestones Reference Table

**Insert as reference section at top of sheet:**

| Category | Milestone | Bonus Amount | Notes |
|----------|-----------|--------------|-------|
| Regional | First state operational | 500,000 DTC | One-time |
| Regional | 5 states operational | 1,000,000 DTC | Cumulative total |
| Regional | 10 states operational | 2,000,000 DTC | Cumulative total |
| Regional | National (all 50 states) | 5,000,000 DTC | Major achievement |
| Platform | First $1M transactions | 250,000 DTC | One-time |
| Platform | First $10M transactions | 500,000 DTC | One-time |
| Platform | First $100M transactions | 1,000,000 DTC | One-time |
| Platform | Platform profitability | 1,000,000 DTC | First profitable quarter |
| Technology | Mobile app launch | 500,000 DTC | Public release |
| Technology | 100K app downloads | 500,000 DTC | User adoption |
| Technology | 1M app downloads | 2,000,000 DTC | Major adoption |
| Technology | Smart contract audit completion | 500,000 DTC | Security milestone |
| Partnership | First major retail partner | 1,000,000 DTC | Defined as >500 locations |
| Partnership | First financial institution | 1,500,000 DTC | Bank or credit union |
| Partnership | Government/municipality adoption | 2,000,000 DTC | Public sector |
| Financial | First profitable month | 750,000 DTC | Platform operational |
| Financial | $1M ARR | 1,000,000 DTC | Annual recurring revenue |
| Financial | $10M ARR | 3,000,000 DTC | Scale milestone |
| User | 10,000 active users | 500,000 DTC | Monthly active |
| User | 100,000 active users | 1,500,000 DTC | Significant scale |
| User | 1M active users | 5,000,000 DTC | Mass adoption |

### Conditional Formatting

**Bonus Amount (D):**
```
Condition 1: >= 1,000,000 → Dark green background (Major milestone)
Condition 2: >= 500,000 → Light green background (Significant milestone)
Condition 3: < 500,000 → White background (Standard milestone)
```

### Summary Formulas

```
Total Milestone Bonuses:        =SUM(D:D)
Total Tax Liability:            =SUM(H:H)
Bonuses This Year:              =SUMIF(I:I,YEAR(TODAY()),D:D)
Bonuses by Category:            =SUMIF(C:C,"Regional Expansion",D:D)
Average Bonus Size:             =AVERAGE(D:D)
Largest Single Bonus:           =MAX(D:D)
```

### Example Entries

| Date | Milestone | Category | Bonus Amount | TX Hash | Valuation | Tax Basis | Tax Set Aside | Year | Quarter | Cumulative | Evidence/Notes |
|------|-----------|----------|--------------|---------|-----------|-----------|---------------|------|---------|------------|----------------|
| 2025-03-15 | First state operational (California) | Regional Expansion | 500,000 | 0xjkl... | $0.015 | $7,500 | $2,775 | 2025 | Q1 2025 | 500,000 | Launched in San Francisco Bay Area |
| 2025-03-28 | First $1M in transactions | Platform Milestone | 250,000 | 0xmno... | $0.015 | $3,750 | $1,388 | 2025 | Q1 2025 | 750,000 | Achieved March 28, 2025 |
| 2025-08-12 | Mobile app launch | Technology Achievement | 500,000 | 0xpqr... | $0.022 | $11,000 | $4,070 | 2025 | Q3 2025 | 1,250,000 | iOS and Android simultaneously |

---

## Sheet 4: Performance Metrics

### Purpose
Track quarterly performance against targets for grant calculation and strategic planning.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Quarter | Text | Q# YYYY format | Quarter identifier |
| B | Quarter End Date | Date | Last day of quarter | Quarter ending |
| C | **MERCHANT METRICS** | Header | | |
| D | Beginning Merchants | Number | =Previous Q ending | Starting count |
| E | New Merchants | Number | Manual entry | Merchants added |
| F | Lost Merchants | Number | Manual entry | Churned merchants |
| G | Ending Merchants | Number | =D2+E2-F2 | Net merchant count |
| H | Merchant Target | Number | Manual entry | Quarterly target |
| I | Merchant Achievement % | Percentage | =G2/H2 | Target achievement |
| J | Retention Rate | Percentage | =(D2-F2)/D2 | Quarterly retention |
| K | **TRANSACTION METRICS** | Header | | |
| L | Total Transactions | Number | Manual entry | Transaction count |
| M | Total Volume | Currency | Manual entry | Dollar volume |
| N | Avg Transaction Size | Currency | =M2/L2 | Per-transaction average |
| O | Volume Target | Currency | Manual entry | Quarterly target |
| P | Volume Achievement % | Percentage | =M2/O2 | Target achievement |
| Q | **PLATFORM METRICS** | Header | | |
| R | Feature Completion % | Percentage | Manual entry | Roadmap completion |
| S | System Uptime % | Percentage | Manual entry | Availability |
| T | Support Tickets | Number | Manual entry | Customer issues |
| U | Avg Resolution Time (hrs) | Number | Manual entry | Support efficiency |
| V | User Satisfaction | Rating | Manual entry | Average rating (1-5) |
| W | Platform Score | Percentage | Manual entry | Composite platform score |
| X | **STRATEGIC METRICS** | Header | | |
| Y | States Operational | Number | Manual entry | Geographic expansion |
| Z | Partnerships Added | Number | Manual entry | New partnerships |
| AA | Major Milestones | Number | Manual entry | Milestones achieved |
| AB | Strategic Score | Percentage | Manual entry | Composite strategic score |
| AC | **OVERALL PERFORMANCE** | Header | | |
| AD | Weighted Score | Percentage | =(I2*0.30)+(P2*0.25)+(W2*0.25)+(AB2*0.20) | Overall performance |
| AE | Grade | Text | =IFS(AD2>=0.9,"A",AD2>=0.8,"B",AD2>=0.7,"C",AD2>=0.6,"D",TRUE,"F") | Performance grade |

### Conditional Formatting

**Achievement Percentages (I, P, W, AB):**
```
Condition 1: >= 100% → Dark green background
Condition 2: >= 90% → Light green background
Condition 3: >= 75% → Yellow background
Condition 4: >= 60% → Orange background
Condition 5: < 60% → Red background
```

**Weighted Score (AD):**
```
Color scale: Red (0%) → Yellow (60%) → Green (100%)
```

**Grade (AE):**
```
Condition 1: = "A" → Dark green, bold
Condition 2: = "B" → Light green
Condition 3: = "C" → Yellow
Condition 4: = "D" → Orange
Condition 5: = "F" → Red, bold
```

**Retention Rate (J):**
```
Condition 1: >= 90% → Green (Excellent)
Condition 2: >= 85% → Light green (Good)
Condition 3: >= 80% → Yellow (Acceptable)
Condition 4: < 80% → Red (Concerning)
```

### Charts & Visualizations

**Chart 1: Merchant Growth Trend**
```
Type: Line chart
X-axis: Quarter
Y-axis: Ending Merchants
Series: Ending Merchants, Target
```

**Chart 2: Performance Score Breakdown**
```
Type: Stacked bar chart
Categories: Merchant %, Volume %, Platform %, Strategic %
Shows: Contribution of each component to overall score
```

**Chart 3: Quarterly Performance Grades**
```
Type: Column chart with color coding
X-axis: Quarter
Y-axis: Overall Score (%)
Color: Based on grade (A=green, F=red)
```

### Summary Formulas

```
Current Quarter Performance:    =[Last Row AD]
Average Performance (All Time): =AVERAGE(AD:AD)
Best Quarter:                   =MAX(AD:AD)
Worst Quarter:                  =MIN(AD:AD)
Quarters Above 80%:             =COUNTIF(AD:AD,">=0.8")
Current Merchant Count:         =[Last Row G]
Current Retention Rate:         =AVERAGE(J:J) (last 4 quarters)
YTD Transaction Volume:         =SUMIFS(M:M,B:B,">="&DATE(YEAR(TODAY()),1,1))
```

### Example Entries

| Quarter | End Date | Begin Merch | New | Lost | End Merch | Target | Achievement % | Retention | Transactions | Volume | Avg Size | Vol Target | Vol % | Feature % | Uptime | Tickets | Res Time | Satisfaction | Platform Score | States | Partners | Milestones | Strategic | Weighted | Grade |
|---------|----------|-------------|-----|------|-----------|--------|---------------|-----------|--------------|--------|----------|------------|-------|-----------|--------|---------|----------|--------------|----------------|--------|----------|------------|-----------|----------|-------|
| Q1 2025 | 2025-03-31 | 0 | 50 | 5 | 45 | 75 | 60% | 90% | 8,500 | $2.0M | $235 | $3.0M | 67% | 80% | 99.2% | 125 | 4.2 | 4.1 | 80% | 1 | 2 | 2 | 67% | 73.5% | C |
| Q2 2025 | 2025-06-30 | 45 | 50 | 7 | 88 | 100 | 88% | 84% | 18,200 | $4.8M | $264 | $5.0M | 96% | 87% | 99.5% | 95 | 3.8 | 4.3 | 90% | 2 | 1 | 1 | 75% | 88.0% | B |

---

## Sheet 5: Internal Valuations

### Purpose
Document quarterly internal valuation methodology and rationale for tax reporting and grant calculations.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Valuation Date | Date | End of quarter | When valuation determined |
| B | Quarter | Text | Q# YYYY | Quarter reference |
| C | Previous Valuation | Currency | =C[previous row] | Last quarter's value |
| D | **METHOD 1: TRANSACTION VOLUME** | Header | | |
| E | Annual Transaction Volume | Currency | Manual entry | Projected annual volume |
| F | Velocity Multiplier | Number | 10 | Turnover rate assumption |
| G | Required Float | Currency | =E2/F2 | Capital needed in circulation |
| H | Circulating Supply | Number | Manual entry | Total DTC in circulation |
| I | Base Value (Method 1) | Currency | =G2/H2 | Value per token |
| J | Private Discount | Percentage | 50% | Pre-launch discount |
| K | Method 1 Valuation | Currency | =I2*(1-J2) | Discounted value |
| L | **METHOD 2: PLATFORM COMPARABLE** | Header | | |
| M | Merchant Count | Number | Manual entry | Current merchants |
| N | Comparable Value/Merchant | Currency | Manual entry | Industry benchmark |
| O | Total Platform Value | Currency | =M2*N2 | Estimated platform worth |
| P | Token Supply | Number | Manual entry | Total token supply |
| Q | Base Value (Method 2) | Currency | =O2/P2 | Value per token |
| R | Crypto Discount | Percentage | 70% | Crypto + private discount |
| S | Method 2 Valuation | Currency | =Q2*(1-R2) | Discounted value |
| T | **METHOD 3: COST APPROACH** | Header | | |
| U | Development Investment | Currency | Manual entry | Capital invested |
| V | Time Investment (hours) | Number | Manual entry | Hours worked |
| W | Hourly Rate | Currency | $200 | Founder hourly value |
| X | Time Value | Currency | =V2*W2 | Value of time |
| Y | Total Investment | Currency | =U2+X2 | Combined investment |
| Z | Developmental Discount | Percentage | 40% | Early-stage discount |
| AA | Method 3 Valuation | Currency | =(Y2/P2)*(1-Z2) | Value per token |
| AB | **FINAL VALUATION** | Header | | |
| AC | Average of Methods | Currency | =AVERAGE(K2,S2,AA2) | Mean of three methods |
| AD | Selected Valuation | Currency | Manual entry | Final chosen value |
| AE | Change from Previous | Percentage | =(AD2-C2)/C2 | Quarter-over-quarter change |
| AF | Rationale | Text | Manual entry | Justification for valuation |
| AG | Key Metrics Summary | Text | Manual entry | Supporting data points |

### Data Validation Rules

**Column AD (Selected Valuation):**
```
Type: Number
Criteria: Must be within 20% of AC (Average of Methods)
Warning: "Selected value should be close to calculated average unless well-justified"
```

**Column AE (Change from Previous):**
```
Conditional warning: If >100% increase or >50% decrease in single quarter
Alert: "Large valuation change - ensure well-documented rationale"
```

### Conditional Formatting

**Change from Previous (AE):**
```
Condition 1: > 50% → Dark green (Strong growth)
Condition 2: > 25% → Light green (Good growth)
Condition 3: -10% to 25% → White (Normal variance)
Condition 4: -25% to -10% → Yellow (Concerning decline)
Condition 5: < -25% → Red (Significant decline)
```

**Method Valuations (K, S, AA):**
```
Highlight: Maximum value → Green border
Highlight: Minimum value → Red border
Purpose: Shows valuation range
```

### Summary Formulas

```
Current Valuation:              =[Last Row AD]
Initial Valuation:              =$AD$2
Lifetime Appreciation:          =([Current]-[Initial])/[Initial]
Average Quarterly Growth:       =GEOMEAN(1+AE:AE)-1
Highest Valuation:              =MAX(AD:AD)
Lowest Valuation:               =MIN(AD:AD)
Valuation Volatility:           =STDEV(AE:AE)
```

### Valuation Methodology Documentation

**Insert as reference at top of sheet:**

```
VALUATION METHODOLOGY GUIDE

DetourCoin internal valuations use a three-method approach to determine fair value for tax reporting and grant calculations.

METHOD 1: Transaction Volume Approach
- Projects annual transaction volume in DTC
- Applies velocity multiplier (how many times tokens turn over per year)
- Calculates required circulating supply
- Applies 50% discount for pre-launch, illiquid status
- Best for: Mature operations with established transaction patterns

METHOD 2: Platform Comparable Approach
- Values DetourMarket platform based on merchant count
- Uses comparable SaaS/platform valuations per merchant
- Allocates platform value across total token supply
- Applies 70% discount for crypto premium + private status
- Best for: Early stages with growing merchant base

METHOD 3: Cost Approach
- Sums actual capital invested + time invested (at $200/hour)
- Distributes total investment across token supply
- Applies 40% discount for developmental/pre-revenue stage
- Best for: Very early stage with limited operations

FINAL VALUATION:
- Take average of all three methods
- Select value within 20% of average (preferably the average itself)
- Document rationale for any deviation from average
- Conservative approach preferred for tax compliance

DISCOUNT RATIONALE:
- Pre-launch: No public trading = 30-50% discount
- Illiquidity: Cannot easily sell = 20-30% discount
- Developmental: Pre-revenue/early-stage = 20-40% discount
- Crypto premium: Volatility risk = 20-30% discount

Combined discounts prevent overvaluation and reduce tax liability.
```

### Example Entries

| Date | Quarter | Previous | Annual Vol | Velocity | Float | Supply | Base (M1) | Discount | Method 1 | Merchants | $/Merch | Platform Val | Supply | Base (M2) | Discount | Method 2 | Dev $ | Hours | Rate | Time $ | Total Inv | Discount | Method 3 | Average | Selected | Change | Rationale |
|------|---------|----------|------------|----------|-------|--------|-----------|----------|----------|-----------|---------|--------------|--------|-----------|----------|----------|-------|-------|------|--------|-----------|----------|----------|---------|----------|--------|-----------|
| 2025-03-31 | Q1 2025 | $0.010 | $8M | 10 | $800K | 10.5M | $0.076 | 50% | $0.038 | 45 | $5,000 | $225K | 10.5M | $0.021 | 70% | $0.006 | $50K | 500 | $200 | $100K | $150K | 40% | $0.009 | $0.018 | $0.015 | 50% | Strong Q1, merchant traction validates higher valuation. Conservative given early stage. |
| 2025-06-30 | Q2 2025 | $0.015 | $19M | 10 | $1.9M | 11.1M | $0.171 | 50% | $0.086 | 88 | $5,000 | $440K | 11.1M | $0.040 | 70% | $0.012 | $75K | 800 | $200 | $160K | $235K | 40% | $0.013 | $0.037 | $0.035 | 133% | Exceeded targets, doubled merchants, transaction volume strong. Platform value increasing. Conservative average selection. |

### Best Practices for Internal Valuations

**1. Consistency:**
- Use same methodology each quarter
- Document any methodology changes
- Maintain conservative approach

**2. Documentation:**
- Extensive notes in Rationale column
- Reference external comparables
- Cite market conditions

**3. Tax Compliance:**
- Prefer lower valuations when uncertain
- Document conservative assumptions
- Maintain audit trail

**4. Quarterly Review:**
- Reassess assumptions quarterly
- Update velocity/discount factors as appropriate
- Adjust for major milestones

**5. Annual Recalibration:**
- Major valuation review at year-end
- Consider CPA consultation
- Update methodology if business model shifts

---

## Sheet 6: Tax Records

### Purpose
Comprehensive tax tracking for IRS reporting and quarterly estimated payment calculations.

### Column Structure

| Column | Header | Type | Formula/Validation | Description |
|--------|--------|------|-------------------|-------------|
| A | Tax Year | Number | YYYY | Calendar year |
| B | Quarter | Text | Q# | Quarter if applicable |
| C | Date | Date | Manual entry | Income/expense date |
| D | Type | Dropdown | List | Income/expense type |
| E | Description | Text | Manual entry | Detailed description |
| F | DTC Amount | Number | Manual entry | Tokens received/spent |
| G | Valuation per DTC | Currency | Manual entry | Value at time of transaction |
| H | USD Value | Currency | =F2*G2 | Calculated dollar value |
| I | Category | Dropdown | List | Tax category |
| J | Form/Schedule | Text | Manual entry | IRS form reference |
| K | Deductible | Yes/No | Manual entry | Is this deductible? |
| L | Supporting Doc | Text | Manual entry | Document reference |
| M | Tax Liability (37%) | Currency | =IF(I2="Compensation Income",H2*0.37,0) | Estimated tax owed |
| N | Tax Paid | Currency | Manual entry | Actual payment made |
| O | Tax Payment Date | Date | Manual entry | When tax paid |
| P | Balance Due | Currency | =M2-N2 | Remaining liability |
| Q | Notes | Text | Manual entry | Additional context |

### Type Dropdown Options

```
- Performance Grant
- Milestone Bonus
- Token Sale
- Liquidity Pool Fee
- Dividend/Distribution
- Gas Fee
- Professional Services (Deductible)
- Development Costs (Deductible)
- Marketing Expense (Deductible)
- Legal/Compliance (Deductible)
- Other Income
- Other Expense
```

### Category Dropdown Options

```
- Compensation Income (Form 1040 Schedule C)
- Capital Gains - Long Term (Schedule D)
- Capital Gains - Short Term (Schedule D)
- Business Expense (Schedule C)
- Ordinary Income (1040)
- Tax Payment (1040-ES)
```

### Conditional Formatting

**Balance Due (P):**
```
Condition 1: > 0 → Red background (Tax owed)
Condition 2: = 0 → White background (Paid)
Condition 3: < 0 → Green background (Overpaid)
```

**Tax Paid (N) vs Tax Liability (M):**
```
Rule: If N < M and Date < (Quarter End + 15 days) → Orange highlight
Warning: Estimated payment due soon
```

### Summary Formulas (by Year)

```
Total Compensation Income:      =SUMIFS(H:H,A:A,2025,I:I,"Compensation Income")
Total Capital Gains:            =SUMIFS(H:H,A:A,2025,I:I,"Capital Gains*")
Total Deductible Expenses:      =SUMIFS(H:H,A:A,2025,K:K,"Yes")
Gross Income:                   =[Compensation]+[Capital Gains]
Adjusted Income:                =[Gross Income]-[Deductible Expenses]
Total Tax Liability:            =SUMIF(A:A,2025,M:M)
Total Tax Paid:                 =SUMIF(A:A,2025,N:N)
Balance Due:                    =[Tax Liability]-[Tax Paid]

Q1 Estimated Payment Due:       =SUMIFS(M:M,A:A,2025,B:B,"Q1")
Q2 Estimated Payment Due:       =SUMIFS(M:M,A:A,2025,B:B,"Q2")
Q3 Estimated Payment Due:       =SUMIFS(M:M,A:A,2025,B:B,"Q3")
Q4 Estimated Payment Due:       =SUMIFS(M:M,A:A,2025,B:B,"Q4")
```

### Tax Payment Schedule Reference

**Insert at top of sheet:**

```
QUARTERLY ESTIMATED TAX PAYMENT SCHEDULE

Q1 Income (Jan 1 - Mar 31):     Payment due April 15
Q2 Income (Apr 1 - Jun 30):     Payment due June 15
Q3 Income (Jul 1 - Sep 30):     Payment due September 15
Q4 Income (Oct 1 - Dec 31):     Payment due January 15 (next year)

FEDERAL TAX BRACKETS (2025 - Single Filer):
37% on income over $578,125
35% on income over $231,250
32% on income over $182,100
24% on income over $95,375
22% on income over $44,725
12% on income over $11,000

For tax planning purposes, founder assumes 37% bracket given total compensation.

IMPORTANT: This is estimated tax liability. Actual tax may vary based on:
- Other income sources
- Deductions and credits
- State and local taxes
- Filing status
- Professional tax preparation

RECOMMENDATION: Set aside 40% of all compensation to cover federal + state taxes.
```

### Example Entries

| Year | Q | Date | Type | Description | DTC Amount | Valuation | USD Value | Category | Form | Deductible | Supporting Doc | Tax Liability | Tax Paid | Payment Date | Balance Due | Notes |
|------|---|------|------|-------------|------------|-----------|-----------|----------|------|------------|----------------|---------------|----------|--------------|-------------|-------|
| 2025 | Q1 | 2025-03-31 | Performance Grant | Q1 2025 founder performance grant | 503,475 | $0.015 | $7,552 | Compensation Income | Sch C | No | Grant_Q1_2025.pdf | $2,794 | $3,000 | 2025-04-15 | -$206 | Paid via 1040-ES |
| 2025 | Q1 | 2025-03-31 | Milestone Bonus | First state operational | 500,000 | $0.015 | $7,500 | Compensation Income | Sch C | No | Milestone_CA.pdf | $2,775 | $0 | | $2,775 | Included in Q1 payment |
| 2025 | Q1 | 2025-01-20 | Development Costs | Smart contract audit (CertiK) | 0 | $0 | $5,000 | Business Expense | Sch C | Yes | Invoice_Certik.pdf | -$1,850 | $0 | | $0 | Deductible business expense |
| 2025 | Q2 | 2025-06-30 | Performance Grant | Q2 2025 founder performance grant | 587,730 | $0.018 | $10,579 | Compensation Income | Sch C | No | Grant_Q2_2025.pdf | $3,914 | $4,000 | 2025-06-15 | -$86 | Paid via 1040-ES |

---

## Sheet 7: Merchant Growth Tracker

### Purpose
Detailed tracking of merchant onboarding, retention, and engagement metrics.

### Column Structure

| Column | Header | Type | Description |
|--------|--------|------|-------------|
| A | Merchant ID | Text | Unique identifier |
| B | Merchant Name | Text | Business name |
| C | Category | Dropdown | Business type (Antique, Art, Craft, etc.) |
| D | Location (City, State) | Text | Physical location |
| E | Onboarding Date | Date | When they joined |
| F | Onboarding Quarter | Text | =Q&ROUNDUP(MONTH(E2)/3,0)&" "&YEAR(E2) |
| G | Subscription Tier | Dropdown | Free, Basic, Pro, Enterprise |
| H | Monthly Fee | Currency | Subscription revenue |
| I | Status | Dropdown | Active, Paused, Churned |
| J | Churn Date | Date | When they left (if applicable) |
| K | Churn Reason | Text | Why they left |
| L | Lifetime (days) | Number | =IF(I2="Active",TODAY()-E2,J2-E2) |
| M | First Transaction Date | Date | When they went live |
| N | Last Transaction Date | Date | Most recent activity |
| O | Days Since Last Transaction | Number | =TODAY()-N2 |
| P | Total Transactions (All Time) | Number | Cumulative transaction count |
| Q | Total Volume (All Time) | Currency | Cumulative dollar volume |
| R | Avg Transaction Size | Currency | =Q2/P2 |
| S | Transactions This Quarter | Number | Current quarter activity |
| T | Volume This Quarter | Currency | Current quarter volume |
| U | Engagement Score | Percentage | =(S2/DAYS_IN_QUARTER)*100 |
| V | Health Status | Calculated | =IFS(U2>10,"Healthy",U2>5,"At Risk",TRUE,"Churning") |
| W | Vendor Count | Number | Number of sub-vendors |
| X | Primary Contact | Text | Name and email |
| Y | Account Manager | Text | Who manages relationship |
| Z | Notes | Text | Important account details |

### Merchant Category Options

```
- Antique Mall
- Art Gallery/Collective
- Craft Market
- Farmers Market
- Flea Market
- Consignment Shop
- Vintage Store
- Maker Space
- Artist Co-op
- Multi-Vendor Boutique
- Other
```

### Subscription Tier Options

```
- Free (Beta/Trial)
- Basic ($29/month)
- Pro ($99/month)
- Enterprise ($299/month)
- Custom
```

### Status Options

```
- Active - Engaged (transacting regularly)
- Active - Low Activity (not transacting much)
- Paused (temporarily inactive)
- Churned (left platform)
```

### Conditional Formatting

**Health Status (V):**
```
Condition 1: "Healthy" → Green background
Condition 2: "At Risk" → Yellow background
Condition 3: "Churning" → Red background
```

**Days Since Last Transaction (O):**
```
Condition 1: > 90 days → Red (Churning risk)
Condition 2: > 30 days → Yellow (Follow up needed)
Condition 3: <= 30 days → Green (Active)
```

**Engagement Score (U):**
```
Color scale: Red (0%) → Yellow (5%) → Green (15%+)
```

### Summary Metrics Dashboard

**Insert at top of sheet:**

```
MERCHANT METRICS DASHBOARD (Current Quarter)

Total Merchants:                =COUNTIF(I:I,"Active*")
New This Quarter:               =COUNTIFS(G:G,CURRENT_QUARTER,I:I,"Active*")
Churned This Quarter:           =COUNTIFS(J:J,">="&QUARTER_START,J:J,"<="&QUARTER_END)
Net Growth:                     =[New]-[Churned]
Retention Rate:                 =([Beginning]+[New]-[Churned])/([Beginning]+[New])

Healthy Merchants:              =COUNTIF(V:V,"Healthy")
At Risk Merchants:              =COUNTIF(V:V,"At Risk")
Churning Merchants:             =COUNTIF(V:V,"Churning")

Average Revenue Per Merchant:   =AVERAGE(T:T)
Total Platform Revenue:         =SUM(T:T)
Average Engagement Score:       =AVERAGE(U:U)

By Tier:
  Free:                         =COUNTIF(G:G,"Free")
  Basic:                        =COUNTIF(G:G,"Basic")
  Pro:                          =COUNTIF(G:G,"Pro")
  Enterprise:                   =COUNTIF(G:G,"Enterprise")

By Category:
  Antique Malls:                =COUNTIF(C:C,"Antique Mall")
  Art Galleries:                =COUNTIF(C:C,"Art Gallery*")
  Craft Markets:                =COUNTIF(C:C,"Craft Market")
```

### Pivot Table Recommendations

**Pivot 1: Merchants by State**
- Rows: Location (State extracted)
- Values: Count of Merchant ID
- Purpose: Geographic distribution

**Pivot 2: Monthly Cohort Retention**
- Rows: Onboarding Quarter
- Columns: Months Since Onboarding
- Values: Count of Active Merchants
- Purpose: Cohort analysis

**Pivot 3: Revenue by Tier**
- Rows: Subscription Tier
- Values: Sum of Monthly Fee, Count of Merchants
- Purpose: Revenue composition

---

## Sheet 8: Transaction Volume Analytics

### Purpose
Track and analyze transaction patterns to inform tokenomics and growth strategies.

### Column Structure

| Column | Header | Type | Description |
|--------|--------|------|-------------|
| A | Date | Date | Transaction date |
| B | Week Number | Number | =WEEKNUM(A2) |
| C | Month | Text | =TEXT(A2,"MMM YYYY") |
| D | Quarter | Text | ="Q"&ROUNDUP(MONTH(A2)/3,0)&" "&YEAR(A2) |
| E | Merchant ID | Text | Reference to Merchant sheet |
| F | Merchant Name | Text | =VLOOKUP(E2,MerchantSheet,2,FALSE) |
| G | Transaction Count | Number | Number of transactions |
| H | Total Volume (USD) | Currency | Dollar volume |
| I | DTC Volume | Number | Transactions using DTC (post-launch) |
| J | DTC Value | Currency | Value of DTC transactions |
| K | DTC Adoption % | Percentage | =J2/H2 |
| L | Avg Transaction Size | Currency | =H2/G2 |
| M | Payment Method | Dropdown | Credit Card, Debit, DTC, Cash, Other |
| N | Commission Earned | Currency | Platform commission (if applicable) |
| O | Notes | Text | Anomalies or context |

### Summary Metrics

**Daily Aggregation (Auto-calculated):**
```
Total Daily Volume:             =SUMIF(A:A,TODAY(),H:H)
Daily Transaction Count:        =SUMIF(A:A,TODAY(),G:G)
DTC Adoption Today:             =SUMIF(A:A,TODAY(),K:K)/COUNTIF(A:A,TODAY())
```

**Weekly Aggregation:**
```
This Week Volume:               =SUMIFS(H:H,B:B,WEEKNUM(TODAY()))
Week-over-Week Growth:          =([This Week]-[Last Week])/[Last Week]
```

**Monthly Aggregation:**
```
MTD Volume:                     =SUMIFS(H:H,C:C,TEXT(TODAY(),"MMM YYYY"))
Monthly Target:                 =[From Performance Metrics Sheet]
Target Achievement:             =[MTD]/[Target]
```

**Quarterly Aggregation:**
```
QTD Volume:                     =SUMIFS(H:H,D:D,CURRENT_QUARTER)
Quarterly Target:               =[From Performance Metrics]
Days Remaining:                 =QUARTER_END-TODAY()
Required Daily Average:         =([Target]-[QTD])/[Days Remaining]
```

### Charts

**Chart 1: Daily Volume Trend**
```
Type: Area chart
X-axis: Date
Y-axis: Total Volume (USD)
Moving average: 7-day and 30-day
```

**Chart 2: DTC Adoption Over Time**
```
Type: Line chart with dual axis
X-axis: Month
Y-axis 1: DTC Adoption %
Y-axis 2: DTC Volume
```

**Chart 3: Volume by Merchant**
```
Type: Horizontal bar (top 20 merchants)
X-axis: Total Volume
Y-axis: Merchant Name
Color: By subscription tier
```

---

## Sheet 9: Token Holdings Dashboard

### Purpose
Real-time view of your token position, value, and strategic milestones.

### Live Data Section

| Metric | Value | Formula | Notes |
|--------|-------|---------|-------|
| **YOUR POSITION** | | | |
| Initial Allocation | 10,000,000 DTC | Fixed | From deployment |
| Grants Received (All Time) | | =SUM(GrantTracking!O:O) | From Grant Tracking |
| Milestone Bonuses (All Time) | | =SUM(MilestoneBonuses!D:D) | From Milestone Bonuses |
| Total Holdings | | =[Initial]+[Grants]+[Bonuses] | Your DTC balance |
| Holdings (Millions) | | =[Total]/1000000 | Formatted display |
| | | | |
| **SUPPLY METRICS** | | | |
| Circulating Supply | | =EmissionsLog!G[last row] | From Emissions Log |
| Max Supply | 1,000,000,000 DTC | Fixed | Hard cap |
| Supply Remaining | | =[Max]-[Circulating] | Available to emit |
| Supply Utilized % | | =[Circulating]/[Max] | Percentage issued |
| | | | |
| **OWNERSHIP** | | | |
| Your Ownership % | | =[Your Holdings]/[Circulating] | Current ownership |
| Your % of Max Supply | | =[Your Holdings]/[Max] | Diluted ownership |
| | | | |
| **VALUATION** | | | |
| Current Internal Valuation | | =InternalValuations!AD[last row] | Latest valuation |
| Your Position Value | | =[Holdings]*[Valuation] | Estimated worth |
| Your Position Value (M) | | =[Value]/1000000 | In millions |
| | | | |
| **MILESTONES** | | | |
| Target Year 5 Holdings | 28,700,000 DTC | Target | From strategy |
| Progress to Target | | =[Current Holdings]/[Target] | Achievement % |
| Grants Remaining to Target | | =[Target]-[Current] | Gap to close |
| Quarters Remaining | | =(2029-YEAR(TODAY()))*4+4-ROUNDUP(MONTH(TODAY())/3,0) | Time left |
| Required Avg Grant | | =[Gap]/[Quarters Remaining] | Per quarter needed |

### Historical Tracking Table

| Date | Total Holdings | Ownership % | Internal Valuation | Position Value | Notes |
|------|----------------|-------------|-------------------|----------------|-------|
| 2025-01-15 | 10,000,000 | 100.0% | $0.010 | $100,000 | Initial deployment |
| 2025-03-31 | 11,253,475 | 104.7% | $0.015 | $168,802 | Q1 grants complete |
| 2025-06-30 | 11,841,205 | 106.2% | $0.022 | $260,507 | Q2 strong performance |

*Auto-populate quarterly from other sheets*

### Progress to Strategic Goals

**Visual Progress Bars (use conditional formatting):**

```
YEAR 5 TARGET (28.7M DTC)
Current: ████████░░░░░░░░░░░░ 39%
[Insert progress bar using conditional formatting]

OWNERSHIP GOAL (28.3% at Launch)
Current: ████████████████████ 100% (pre-dilution)
Year 5 Target: ████████░░░░░░░░░░░░ 28.3%

POSITION VALUE TARGET ($5.74M at $0.20)
Current: ██░░░░░░░░░░░░░░░░░░ 4.4%
```

### Charts

**Chart 1: Holdings Growth Trajectory**
```
Type: Line chart with target line
X-axis: Date
Y-axis: Total Holdings (millions)
Series 1: Actual Holdings
Series 2: Target Trajectory (linear to 28.7M)
```

**Chart 2: Ownership Dilution Over Time**
```
Type: Area chart
X-axis: Date
Y-axis: Ownership %
Shows: Expected dilution as supply increases
```

**Chart 3: Position Value Growth**
```
Type: Column chart
X-axis: Quarter
Y-axis: Position Value ($)
Color: Green if increasing, Red if decreasing
```

---

## Sheet 10: Gas Cost Tracking

### Purpose
Monitor operational costs for emissions and contract interactions.

### Column Structure

| Column | Header | Type | Description |
|--------|--------|------|-------------|
| A | Date | Date | Transaction date |
| B | Operation | Dropdown | Type of operation |
| C | Transaction Hash | Text | Ethereum TX hash |
| D | Gas Used | Number | Gas units consumed |
| E | Gas Price (Gwei) | Number | Gas price |
| F | Gas Cost (MATIC) | Number | =D2*E2/1000000000 |
| G | MATIC Price (USD) | Currency | MATIC/USD rate |
| H | Gas Cost (USD) | Currency | =F2*G2 |
| I | Operation Success | Yes/No | Did it work? |
| J | Block Number | Number | Block included in |
| K | Notes | Text | Context |

### Operation Types

```
- Daily Emission
- Founder Grant Transfer
- Milestone Bonus Transfer
- Liquidity Pool Addition
- Contract Deployment
- Role Management
- Emergency Pause
- Contract Upgrade
- Other
```

### Summary Metrics

```
Total Gas Cost (MATIC):         =SUM(F:F)
Total Gas Cost (USD):           =SUM(H:H)
Average Cost Per Operation:     =AVERAGE(H:H)
Most Expensive Operation:       =MAX(H:H)

By Operation Type:
  Daily Emissions:              =SUMIF(B:B,"Daily Emission",H:H)
  Grants/Bonuses:               =SUMIF(B:B,"*Grant*",H:H)+SUMIF(B:B,"*Bonus*",H:H)
  Liquidity:                    =SUMIF(B:B,"*Liquidity*",H:H)

This Month Gas Cost:            =SUMIFS(H:H,A:A,">="&DATE(YEAR(TODAY()),MONTH(TODAY()),1))
This Year Gas Cost:             =SUMIFS(H:H,A:A,">="&DATE(YEAR(TODAY()),1,1))
```

---

## Automation & Integration

### Google Sheets Apps Script

**Auto-populate quarterly summaries:**

```javascript
function updateQuarterlyMetrics() {
  var ss = SpreadsheetApp.getActiveSpreadsheet();
  var metricsSheet = ss.getSheetByName('Performance Metrics');
  var merchantSheet = ss.getSheetByName('Merchant Growth Tracker');
  
  // Get current quarter
  var today = new Date();
  var quarter = 'Q' + Math.ceil((today.getMonth() + 1) / 3) + ' ' + today.getFullYear();
  
  // Count active merchants
  var merchantRange = merchantSheet.getDataRange();
  var merchantData = merchantRange.getValues();
  var activeMerchants = 0;
  
  for (var i = 1; i < merchantData.length; i++) {
    if (merchantData[i][8] === 'Active - Engaged' || merchantData[i][8] === 'Active - Low Activity') {
      activeMerchants++;
    }
  }
  
  // Find current quarter row in metrics
  var metricsRange = metricsSheet.getDataRange();
  var metricsData = metricsRange.getValues();
  
  for (var i = 1; i < metricsData.length; i++) {
    if (metricsData[i][0] === quarter) {
      // Update ending merchants
      metricsSheet.getRange(i + 1, 7).setValue(activeMerchants);
      break;
    }
  }
  
  Logger.log('Updated metrics for ' + quarter + ': ' + activeMerchants + ' active merchants');
}

// Run weekly
function createWeeklyTrigger() {
  ScriptApp.newTrigger('updateQuarterlyMetrics')
    .timeBased()
    .everyWeeks(1)
    .onWeekDay(ScriptApp.WeekDay.MONDAY)
    .create();
}
```

### Data Import from Blockchain

**Script to import emission data:**

```javascript
function importEmissionData() {
  // This would connect to Polygon RPC and fetch emission events
  // Requires Google Apps Script URL Fetch service
  
  var contractAddress = '0x...'; // Your DetourCoin address
  var rpcUrl = 'https://polygon-rpc.com';
  
  // Fetch TokensEmitted events
  var payload = {
    jsonrpc: '2.0',
    method: 'eth_getLogs',
    params: [{
      fromBlock: 'latest',
      toBlock: 'latest',
      address: contractAddress,
      topics: ['0x...'] // TokensEmitted event signature
    }],
    id: 1
  };
  
  var options = {
    method: 'post',
    contentType: 'application/json',
    payload: JSON.stringify(payload)
  };
  
  var response = UrlFetchApp.fetch(rpcUrl, options);
  var events = JSON.parse(response.getContentText());
  
  // Parse and insert into Emissions Log sheet
  // (Implementation details omitted for brevity)
}
```

---

## Backup & Security

### Backup Schedule

**Automated Backups:**
1. **Daily:** Google Sheets auto-saves continuously
2. **Weekly:** Export to Excel format, store in Google Drive backup folder
3. **Monthly:** Export to CSV + PDF, download locally
4. **Quarterly:** Full workbook backup to external drive
5. **Annually:** Archive entire year to multiple locations

### Access Control

**Sharing Settings:**
- **Owner:** You only
- **Editors:** Trusted CPA or business partner (optional)
- **Viewers:** None (keep private)
- **Link Sharing:** OFF

**Best Practices:**
- Enable 2FA on Google account
- Use strong, unique password
- Never share publicly
- Encrypt backups if stored on cloud services
- Consider Google Workspace for enhanced security

### Data Integrity

**Validation Rules:**
- Lock formula columns (prevent accidental edits)
- Protect summary sheets (view only)
- Use data validation on all input columns
- Enable "Notify me when anything changes" (Google Sheets)

**Monthly Reconciliation:**
```
□ Verify total supply matches on-chain data
□ Cross-check grant amounts with transaction hashes
□ Confirm tax calculations sum correctly
□ Validate merchant count against platform database
□ Review all conditional formatting still working
□ Check for broken formulas
□ Verify charts displaying correctly
```

---

## Conclusion

These spreadsheet templates provide a comprehensive tracking system for all aspects of DetourCoin operations. Key success factors:

1. **Consistency:** Update religiously (daily emissions, quarterly grants)
2. **Accuracy:** Double-check all manual entries
3. **Documentation:** Detailed notes in every record
4. **Backup:** Multiple copies, multiple locations
5. **Security:** Protect access, encrypt sensitive data
6. **Reconciliation:** Monthly verification against on-chain data

**Remember:** This documentation system is your defense in an IRS audit and your guide to building generational wealth. Treat it with the importance it deserves.

---