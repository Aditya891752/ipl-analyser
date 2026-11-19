# IPL Cricket Performance Dashboard
Multi-Season Sports Performance & Match Intelligence Analysis built in Power BI with a multi-table Star Schema.

## STEP 1 — Load All 3 Tables

**Get Data → Text/CSV** → Load each file:
- `ipl_matches.csv` → rename to `Matches`
- `ipl_batting.csv` → rename to `Batting`
- `ipl_bowling.csv` → rename to `Bowling`

**Power Query transformations:**

For `Matches`:
- `Season` → Whole Number
- `Team1Score`, `Team2Score`, `MarginValue` → Whole Number

For `Batting`:
- `Runs`, `Balls`, `Fours`, `Sixes`, `Dismissed` → Whole Number
- `StrikeRate` → Decimal Number

For `Bowling`:
- `Overs`, `RunsConceded`, `Wickets` → appropriate numeric types
- `Economy`, `BowlingAverage` → Decimal Number

---

## STEP 2 — Data Model (Star Schema)

Go to **Model View**. Create relationships:

```
Matches[MatchID]  ──1──→──*── Batting[MatchID]
Matches[MatchID]  ──1──→──*── Bowling[MatchID]
Matches[Season]   ──1──→──*── Batting[Season]
Matches[Season]   ──1──→──*── Bowling[Season]
```

This is a **star schema** — mention this explicitly in your interview. Recruiters love when freshers know data modeling terminology.

---

## STEP 3 — All DAX Measures

Create `_IPL_Measures` table and add:

```dax
-- ── MATCH-LEVEL ───────────────────────────────────────────────

Total Matches = COUNTROWS(Matches)

Total Seasons = DISTINCTCOUNT(Matches[Season])

Highest Team Score = MAX(Matches[Team1Score])

Average Match Score = 
AVERAGEX(Matches, (Matches[Team1Score] + Matches[Team2Score]) / 2)

-- ── TEAM PERFORMANCE ──────────────────────────────────────────

Team Wins = 
CALCULATE(
    COUNTROWS(Matches),
    USERELATIONSHIP(Matches[Winner], Matches[Winner])
)

Win % by Team = 
VAR TeamName = SELECTEDVALUE(Matches[Winner])
VAR Wins = CALCULATE(COUNTROWS(Matches), Matches[Winner] = TeamName)
VAR TotalGames = 
    CALCULATE(COUNTROWS(Matches), 
        Matches[Team1] = TeamName || Matches[Team2] = TeamName)
RETURN DIVIDE(Wins, TotalGames) * 100

Toss Win to Match Win % = 
DIVIDE(
    CALCULATE(COUNTROWS(Matches), Matches[TossWinner] = Matches[Winner]),
    [Total Matches]
) * 100

Home Advantage Score = 
CALCULATE(
    COUNTROWS(Matches),
    Matches[TossDecision] = "bat"
)

-- ── BATTING ANALYTICS ─────────────────────────────────────────

Total Runs Scored = SUM(Batting[Runs])

Average Strike Rate = AVERAGE(Batting[StrikeRate])

Total Sixes = SUM(Batting[Sixes])

Total Fours = SUM(Batting[Fours])

Boundary % = 
DIVIDE(
    ([Total Fours] * 4 + [Total Sixes] * 6),
    [Total Runs Scored]
) * 100

Top Scorer (Season) = 
TOPN(1,
    SUMMARIZE(Batting, Batting[Batter], "TotalRuns", SUM(Batting[Runs])),
    [TotalRuns], DESC
)

Batting Average = 
DIVIDE(
    [Total Runs Scored],
    CALCULATE(COUNTROWS(Batting), Batting[Dismissed] = 1)
)

Century Count = 
CALCULATE(COUNTROWS(Batting), Batting[Runs] >= 100)

Half Century Count = 
CALCULATE(COUNTROWS(Batting), Batting[Runs] >= 50, Batting[Runs] < 100)

-- ── BOWLING ANALYTICS ─────────────────────────────────────────

Total Wickets = SUM(Bowling[Wickets])

Avg Economy Rate = AVERAGE(Bowling[Economy])

Best Economy Bowler = 
TOPN(1,
    SUMMARIZE(Bowling, Bowling[Bowler], "AvgEcon", AVERAGE(Bowling[Economy])),
    [AvgEcon], ASC
)

Bowling Strike Rate = 
DIVIDE(
    SUM(Bowling[Overs]) * 6,
    [Total Wickets]
)

Wickets per Match = 
DIVIDE([Total Wickets], [Total Matches])

-- ── SEASON TREND ──────────────────────────────────────────────

YoY Run Rate Change = 
VAR CurrentSeason = MAX(Matches[Season])
VAR AvgNow = CALCULATE(AVERAGE(Batting[StrikeRate]), Batting[Season] = CurrentSeason)
VAR AvgPrev = CALCULATE(AVERAGE(Batting[StrikeRate]), Batting[Season] = CurrentSeason - 1)
RETURN DIVIDE(AvgNow - AvgPrev, AvgPrev) * 100
```

---

## STEP 4 — Pages & Visuals

### PAGE 1: "Season Overview" (Landing Page)

**Color Palette (IPL-inspired):**
- Background: `#0A0A0A`
- Primary: `#F59E0B` (Gold/IPL trophy color)
- Secondary: `#1D4ED8` (Blue)
- Text: `#FFFFFF`
- Accent: `#EF4444` (Red for danger/records)

**Top KPI Row (5 cards):**
Total Matches | Total Runs | Total Wickets | Avg Match Score | Toss→Win %

**Main visuals:**

1. **Clustered Column Chart**
   - X: `Season`
   - Y: Count of Matches (from Matches table)
   - Secondary Y-axis: `[Avg Economy Rate]`
   - Title: "IPL Season Overview (2019–2024)"
   - This shows your dual-axis skill

2. **Bar Chart (horizontal)**
   - Axis: `Winner` (team names)
   - Values: Count of MatchID (wins)
   - Sort: Descending
   - Color: Gold gradient `#F59E0B` → `#D97706`
   - Title: "All-Time Win Leaderboard"

3. **Pie/Donut Chart**
   - Values: Count of TossDecision
   - Labels: Bat vs Field
   - Title: "Toss Decision Split"

4. **Line Chart**
   - X: Season
   - Y: `[Average Match Score]`
   - Title: "Is T20 Scoring Increasing Season by Season?"

---

### PAGE 2: "Batting Analysis" (Player Performance)

1. **Scatter Chart** ← Most impressive visual on this page
   - X-axis: `StrikeRate`
   - Y-axis: `Runs`
   - Size: `Sixes`
   - Legend: `Team`
   - Tooltip: Add `Batter`, `Balls`, `Fours`
   - Title: "Strike Rate vs Runs — Identifying Power Hitters"
   - Add reference lines: SR=130 (vertical), Runs=50 (horizontal) → creates 4 quadrants: Anchors, Power Players, Impact Hitters, Finishers

2. **Top N Bar Chart**
   - Axis: `Batter`
   - Values: `SUM(Batting[Runs])`
   - Apply Top N filter: Top 10 by Total Runs
   - Title: "Top 10 Run Scorers (All Seasons)"

3. **Stacked Bar Chart**
   - X: `Batter` (Top 8)
   - Values: `Fours × 4` and `Sixes × 6` stacked
   - Title: "Boundary Runs Breakdown — Fours vs Sixes"

4. **Matrix/Table**
   - Rows: Batter
   - Columns: Season
   - Values: SUM of Runs
   - Conditional formatting: Data bars
   - Title: "Batter Performance Across Seasons"

---

### PAGE 3: "Bowling Intelligence"

1. **Bubble Chart / Scatter**
   - X: `Economy`
   - Y: `Wickets`
   - Size: `BowlingAverage`
   - Title: "Economy vs Wickets — Finding the Best Bowlers"
   - Add reference lines: Economy ≤ 8 (good), Wickets ≥ 10 (impact)

2. **Bar Chart**
   - Top 10 Wicket Takers (SUM of Wickets, Top N filter)
   - Title: "Leading Wicket Takers"

3. **Line Chart**
   - X: Season
   - Y: `[Avg Economy Rate]` and `[Wickets per Match]`
   - Dual axis
   - Title: "Bowling Trends: Are Batsmen Dominating?"

4. **KPI Cards row:**
   Best Economy | Most Wickets | Lowest Bowling Average | Highest SR

---

### PAGE 4: "Head-to-Head & Venue Analysis"

1. **Matrix (Head-to-Head)**
   - Rows: `Team1`
   - Columns: `Team2`
   - Values: Count of MatchID
   - Conditional formatting: Heatmap (more matches = darker color)
   - Title: "Head-to-Head Match Count Matrix"
   - **This is unique** — almost no fresher shows a matrix-heatmap

2. **Bar Chart**
   - Axis: `Venue`
   - Values: Count of MatchID (matches played)
   - Title: "Most Used IPL Venues"

3. **Stacked Bar**
   - Axis: `Venue`
   - Values: Wins per team (Winner field)
   - Title: "Venue-Wise Winner Distribution"

4. **100% Stacked Bar**
   - Axis: `TossDecision` (Bat/Field)
   - Values: Count of Winner vs Loser
   - Title: "Does Winning the Toss Matter?"

---

## STEP 5 — Drill-Through Setup

This is what separates intermediate from beginner dashboards:

1. Create a new page → name it "Player Card"
2. Add `Batter` field to **Drill-through filters** (right panel)
3. On this page, build:
   - Card: Total Runs, Strike Rate, Centuries, Half-Centuries
   - Bar Chart: Runs by Season
   - Line Chart: Strike Rate trend across seasons
   - Table: Match-by-match performance

4. Go back to Page 2 → right-click any batter name → Drill Through → Player Card
5. Add a **Back button** (Insert → Buttons → Back)

Now do same for `Bowler` field → "Bowler Card" page

---

## STEP 6 — Bookmarks & Navigation

1. **Insert → Buttons → Blank** → Add text "🏏 Batting" / "🎯 Bowling" / "🏟 Venues"
2. Create Bookmarks (View → Bookmarks) for each page state
3. Assign bookmark to each button via Action → Bookmark
4. This creates a professional navigation bar — use same position on all pages

---

## STEP 7 — Tooltips (Advanced)

1. Create a new page → Page Information → set as **Tooltip**
2. Size: 320px × 240px
3. Add mini-charts: Player's last 5 match runs as a sparkline
4. Go to any player chart → Format → Tooltip → Report Page → select your tooltip page
5. Now hovering over a player name shows a pop-up mini-dashboard

---

## Resume Description (copy-paste ready)

> **IPL Cricket Performance Analytics Dashboard** | Power BI | DAX | Multi-Table Star Schema
> Designed a 4-page interactive sports analytics dashboard analyzing 356 IPL matches across 6 seasons (2019–2024) using 3 related tables (3,900+ rows). Built 25+ DAX measures covering batting averages, bowling economy, toss influence, and YoY scoring trends. Implemented drill-through navigation for individual player cards, report-page tooltips, and a head-to-head heatmap matrix. Data model follows star schema design with 3 fact-dimension relationships. Featured insight: Toss winner converted to match winner 54% of the time — debunking the "toss = advantage" myth.

---
---

---

## Skills Demonstrated
* **Data Architecture:** Star Schema Data Model (1-to-Many relationships between Matches fact table and Batting/Bowling dimension tables)
* **DAX & Calculations:** Batting/Bowling strike rates, Economy rates, Top N filtering, Dynamic KPI cards
* **Interactive UX:** Drill-Through navigation for player cards, Report-Page Tooltips, Bookmarks, Dual-Axis visual analytics
