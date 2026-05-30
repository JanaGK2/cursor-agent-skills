---
name: apps-script-visualizations
description: Add interactive charts to Apps Script dashboards. Covers Google Charts, Chart.js, D3.js, Plotly, and more. Use when adding visualizations to web apps.
---

# Apps Script Visualizations

## When to Use

- Adding charts to Apps Script dashboard
- Need to choose between visualization libraries
- Implementing specific chart types (radar, heatmap, sankey, etc.)
- Selecting the right chart for your data structure

---

## Visualization Theory: Why Chart Choice Matters

A chart is a **model of reality** — it abstracts and simplifies data to make it intelligible. The designer's challenge is to abstract without sacrificing integrity. Understanding human visual perception ensures your charts communicate accurately.

### The Perceptual Hierarchy (Cleveland-McGill)

Human vision decodes visual encodings with varying accuracy. This hierarchy ranks perceptual tasks from most to least accurate:

| Rank | Quantitative Data | Ordinal Data | Nominal Data |
|------|-------------------|--------------|--------------|
| 1 (Best) | Position on common scale | Position on common scale | Position on common scale |
| 2 | Positions on nonaligned scales | Density/Shading | Color Hue |
| 3 | Length, direction, angle | Color Saturation | Texture |
| 4 | Area | Color Hue | Connection |
| 5 | Volume, curvature | Texture | Containment |
| 6 (Worst) | Shading, color saturation | Shape | Volume |

**Key insight**: Position-based encodings (bar charts, dot plots, line graphs) are most accurate. Area and color-based encodings (pie charts, bubble charts, choropleth maps) are least accurate.

### Stevens' Power Law: Why Pie Charts Deceive

The perceived magnitude of visual stimuli follows: `perceived = k × actual^α`

| Encoding | α (exponent) | Effect |
|----------|--------------|--------|
| Length | 1.0 | Unbiased (linear) |
| Area | ~0.7 | Systematic underestimation |
| Volume | ~0.5 | Severe underestimation |

This explains why:
- **Bubble charts** are prone to misinterpretation (area judgment)
- **Pie charts** are inaccurate (angle + curvature judgment)
- **Bar charts** are most reliable (length judgment)

### Cairo's Five Qualities of Great Visualizations

| Quality | Definition | Violation Example |
|---------|------------|-------------------|
| **Truthful** | No self-deception or data distortion | Truncated y-axis, misleading aggregation |
| **Functional** | Form follows function; enables the cognitive task | Chart can't be read accurately |
| **Beautiful** | Elegant simplicity that attracts attention | 3D effects, chartjunk, visual noise |
| **Insightful** | Exposes patterns hidden in tables | Merely replicating tabular data |
| **Enlightening** | Alters the viewer's mental model | Fails to address key questions |

---

## Chart Selection Decision Logic

Analyze your data to select the optimal chart:

- **D** = Dimension count (categorical columns)
- **M** = Metric count (quantitative columns)
- **C** = Cardinality (unique values in a category)
- **T** = Temporal marker (date/time present?)

```
IF T == True (Temporal Dimension):
  IF M == 1 AND C <= 12:
    → Column Chart (baseline at 0)
  ELSE:
    → Line Chart (max 6 lines, direct labeling)
    → If >6 series: Small Multiples Grid

ELSE IF M >= 2 AND D == 0 (Correlation):
  IF M == 2:
    → Scatter Plot with quadrant highlights
  IF M == 3:
    → Bubble Chart (map to AREA, not diameter)

ELSE IF D == 1 AND M == 1 (Category Comparison):
  IF C <= 15 AND labels are short:
    → Vertical Column Chart (sorted descending)
  ELSE:
    → Horizontal Bar Chart (sorted descending)

ELSE IF D == 2 AND M == 1 (Subcategory Comparison):
  IF parent_categories × child_categories <= 8:
    → Clustered Bar Chart
  ELSE:
    → Dot Plot with Grouping
```

---

## Design Standards (Emery-Evergreen Checklist)

### Structural Lines
- **Borders**: Remove chart borders entirely
- **Gridlines**: Mute to faint gray OR remove if data points are labeled
- **Dual Y-Axes**: PROHIBITED (suggests false correlations)
- **Tick marks**: Omit unless tracking precise intervals

### Geometric Proportions
- **Bar/Column baseline**: MUST start at exactly 0
- **Bar width**: Approximately 2× the gap between bars
- **Sorting**: Nominal → descending by value; Ordinal → natural order

### Text & Labels
- **Titles**: Action-oriented ("Sales Dropped 15% After Q2") not generic ("Sales Over Time")
- **Text direction**: Always horizontal (rotate chart, not labels)
- **Direct labeling**: Label data series on the chart, not in a separate legend

### Color Strategy
- **Action color**: Use ONE distinct color to highlight key insight
- **Supporting data**: Muted gray for comparison/context data
- **Accessibility**: Must work in grayscale; avoid red-green adjacent
- **Sequential palettes**: For continuous ranges
- **Diverging palettes**: For deviation from midpoint

---

## Library Selection Guide

| Library | Best For | CDN Size | Complexity |
|---------|----------|----------|------------|
| **Google Charts** | Basic charts, tight GAS integration | ~200KB | Low |
| **Chart.js** | Radar, polar, responsive charts | ~60KB | Low |
| **D3.js** | Custom visualizations, word clouds | ~90KB | High |
| **Plotly.js** | Scientific charts, heatmaps | ~3MB | Medium |
| **ApexCharts** | Modern dashboards, heatmaps | ~450KB | Low |
| **Cytoscape.js** | Network graphs, clustering | ~400KB | Medium |

## Quick Start: Load Libraries

Add to your HTML `<head>`:

```html
<!-- Google Charts -->
<script src="https://www.gstatic.com/charts/loader.js"></script>

<!-- Chart.js -->
<script src="https://cdn.jsdelivr.net/npm/chart.js"></script>

<!-- D3.js + word cloud -->
<script src="https://d3js.org/d3.v7.min.js"></script>
<script src="https://cdn.jsdelivr.net/npm/d3-cloud@1.2.5/build/d3.layout.cloud.min.js"></script>

<!-- Plotly -->
<script src="https://cdn.plot.ly/plotly-2.27.0.min.js"></script>

<!-- ApexCharts -->
<script src="https://cdn.jsdelivr.net/npm/apexcharts"></script>
```

## Integration Pattern

```html
<div id="chart"></div>

<script>
  // Load data from Apps Script
  google.script.run
    .withSuccessHandler(drawChart)
    .withFailureHandler(showError)
    .getChartData();
  
  function drawChart(data) {
    // Initialize and render chart with data
  }
  
  function showError(error) {
    document.getElementById('chart').innerHTML = 'Error: ' + error.message;
  }
</script>
```

---

## Responsive Charts

Google Charts are **not natively responsive**. You must handle window resize events manually.

### Basic Responsive Pattern

```html
<style>
  .chart-container {
    width: 100%;
    max-width: 800px;
  }
</style>

<div id="chart" class="chart-container"></div>

<script>
let chart, chartData, chartOptions;

google.charts.load('current', {'packages':['corechart']});
google.charts.setOnLoadCallback(initChart);

function initChart() {
  chartData = google.visualization.arrayToDataTable([
    ['Category', 'Value'],
    ['A', 100], ['B', 80], ['C', 60]
  ]);
  
  chartOptions = {
    title: 'My Chart',
    // Don't set fixed width - let container control it
    height: 400,
    legend: { position: 'bottom' }
  };
  
  chart = new google.visualization.BarChart(document.getElementById('chart'));
  drawChart();
}

function drawChart() {
  chart.draw(chartData, chartOptions);
}

// Redraw on window resize with debounce
let resizeTimer;
window.addEventListener('resize', function() {
  clearTimeout(resizeTimer);
  resizeTimer = setTimeout(drawChart, 250);
});
</script>
```

### Clear Chart Before Redraw (Fixes Shrink Issues)

If charts fail to shrink when window gets smaller:

```javascript
function drawChart() {
  chart.clearChart();  // Clear before redraw
  chart.draw(chartData, chartOptions);
}
```

### Chart.js Responsive Mode

Chart.js has built-in responsive support:

```javascript
new Chart(ctx, {
  type: 'bar',
  data: {...},
  options: {
    responsive: true,           // Enable responsive
    maintainAspectRatio: true,  // Maintain aspect ratio (optional)
  }
});
```

### Plotly Responsive Mode

```javascript
Plotly.newPlot('chart', data, layout, {
  responsive: true  // Built-in responsive support
});
```

### ApexCharts Responsive Breakpoints

```javascript
var options = {
  chart: {
    type: 'bar',
    height: 350
  },
  responsive: [{
    breakpoint: 768,
    options: {
      chart: { height: 300 },
      legend: { position: 'bottom' }
    }
  }, {
    breakpoint: 480,
    options: {
      chart: { height: 250 },
      legend: { show: false }
    }
  }]
};
```

---

## CSS Design System

### Design Tokens (CSS Variables)

Define a consistent color palette and typography:

```css
:root {
  /* Primary colors */
  --color-primary: #EE0000;
  --color-primary-dark: #C9190B;
  --color-black: #151515;
  --color-white: #FFFFFF;
  
  /* Grays */
  --color-grey: #6A6E73;
  --color-grey-light: #B8BBBE;
  --color-grey-dark: #3C3F42;
  --color-border: #D2D2D2;
  --color-bg: #F5F5F5;
  
  /* Accent colors */
  --color-teal: #009596;
  --color-yellow: #F0AB00;
  --color-green: #3E8635;
  --color-purple: #7B4397;
  
  /* Typography */
  --font-family: system-ui, -apple-system, 'Segoe UI', sans-serif;
  --font-size-base: 14px;
  --font-size-sm: 12px;
  --font-size-xs: 11px;
  --font-size-lg: 16px;
}

* { box-sizing: border-box; margin: 0; padding: 0; }
body { 
  font-family: var(--font-family); 
  background: var(--color-bg); 
  color: var(--color-black); 
  font-size: var(--font-size-base); 
}
```

---

## Dashboard UI Components

### Stat Cards (KPIs)

```html
<div class="stat-row">
  <div class="stat-card">
    <div class="val">1,234</div>
    <div class="lbl">Total Accounts</div>
  </div>
  <div class="stat-card teal">
    <div class="val">$5.2M</div>
    <div class="lbl">Pipeline Value</div>
  </div>
  <div class="stat-card yellow">
    <div class="val">87%</div>
    <div class="lbl">Conversion Rate</div>
  </div>
</div>
```

```css
.stat-row { 
  display: grid; 
  grid-template-columns: repeat(auto-fit, minmax(140px, 1fr)); 
  gap: 12px; 
  margin-bottom: 24px; 
}
.stat-card {
  background: var(--color-white);
  border-radius: 8px;
  padding: 16px;
  border-top: 4px solid var(--color-primary);
  box-shadow: 0 1px 4px rgba(0,0,0,.08);
}
.stat-card .val { font-size: 26px; font-weight: 700; }
.stat-card .lbl { font-size: var(--font-size-xs); color: var(--color-grey); margin-top: 2px; }

/* Color variants */
.stat-card.teal   { border-top-color: var(--color-teal); }
.stat-card.yellow { border-top-color: var(--color-yellow); }
.stat-card.green  { border-top-color: var(--color-green); }
.stat-card.grey   { border-top-color: var(--color-grey); }
```

### Chart Grid Layouts

```html
<div class="chart-grid chart-grid-2">
  <div class="chart-card">
    <h3>Revenue by Region</h3>
    <div class="chart-wrap" id="chart1"></div>
  </div>
  <div class="chart-card">
    <h3>Monthly Trend</h3>
    <div class="chart-wrap-tall" id="chart2"></div>
  </div>
</div>
```

```css
.chart-grid { display: grid; gap: 20px; margin-bottom: 24px; }
.chart-grid-2 { grid-template-columns: 1fr 1fr; }
.chart-grid-3 { grid-template-columns: 1fr 1fr 1fr; }
.chart-grid-auto { grid-template-columns: repeat(auto-fit, minmax(340px, 1fr)); }

.chart-card {
  background: var(--color-white);
  border-radius: 8px;
  padding: 20px;
  box-shadow: 0 1px 4px rgba(0,0,0,.08);
}
.chart-card h3 { 
  font-size: 13px; 
  font-weight: 700; 
  margin-bottom: 14px; 
}

/* Chart container heights */
.chart-wrap      { position: relative; height: 260px; }
.chart-wrap-tall { position: relative; height: 380px; }
.chart-wrap-sm   { position: relative; height: 200px; }

/* Responsive */
@media (max-width: 900px) {
  .chart-grid-2, .chart-grid-3 { grid-template-columns: 1fr; }
}
```

### Pill Badges (Status Indicators)

```html
<span class="pill pill-green">Active</span>
<span class="pill pill-red">At Risk</span>
<span class="pill pill-yellow">Pending</span>
<span class="pill pill-teal">New</span>
```

```css
.pill {
  display: inline-block;
  padding: 2px 8px;
  border-radius: 10px;
  font-size: 10px;
  font-weight: 600;
  white-space: nowrap;
}
.pill-red    { background: #FFDDD5; color: #7D1007; }
.pill-teal   { background: #BDE5E5; color: #003737; }
.pill-yellow { background: #FFF0B3; color: #5A3E00; }
.pill-grey   { background: #E8E8E8; color: #3C3F42; }
.pill-green  { background: #D4EDDA; color: #1A4731; }
.pill-purple { background: #E8D5F5; color: #3D1766; }
```

### Progress Bars (In-Table)

```html
<td class="bar-cell">
  <div>$2.5M</div>
  <div class="bar-bg">
    <div class="bar-fill" style="width: 75%"></div>
  </div>
</td>
```

```css
.bar-cell { min-width: 120px; }
.bar-bg { 
  background: #eee; 
  border-radius: 3px; 
  height: 10px; 
  overflow: hidden; 
  margin-top: 3px; 
}
.bar-fill { 
  height: 100%; 
  border-radius: 3px; 
  background: var(--color-teal); 
}
```

### Tab Navigation

```html
<div id="tab-bar">
  <button class="tab-btn active" onclick="showTab('overview')">Overview</button>
  <button class="tab-btn" onclick="showTab('details')">Details</button>
  <button class="tab-btn" onclick="showTab('charts')">Charts</button>
</div>

<div id="overview" class="tab-panel active">...</div>
<div id="details" class="tab-panel">...</div>
<div id="charts" class="tab-panel">...</div>
```

```css
#tab-bar {
  background: var(--color-white);
  border-bottom: 1px solid var(--color-border);
  padding: 0 24px;
  display: flex;
  gap: 0;
}
.tab-btn {
  padding: 12px 18px;
  border: none;
  background: none;
  cursor: pointer;
  font-size: 13px;
  font-weight: 500;
  color: var(--color-grey);
  border-bottom: 3px solid transparent;
  transition: all .15s;
}
.tab-btn:hover { color: var(--color-black); }
.tab-btn.active { 
  color: var(--color-primary); 
  border-bottom-color: var(--color-primary); 
  font-weight: 700; 
}

.tab-panel { display: none; }
.tab-panel.active { display: block; }
```

```javascript
function showTab(tabId) {
  document.querySelectorAll('.tab-btn').forEach(btn => btn.classList.remove('active'));
  document.querySelectorAll('.tab-panel').forEach(panel => panel.classList.remove('active'));
  
  document.querySelector(`[onclick="showTab('${tabId}')"]`).classList.add('active');
  document.getElementById(tabId).classList.add('active');
}
```

### Sticky Header and Filter Bar

```css
#header {
  background: var(--color-black);
  color: var(--color-white);
  padding: 14px 24px;
  position: sticky;
  top: 0;
  z-index: 200;
}

#filter-bar {
  background: var(--color-white);
  border-bottom: 3px solid var(--color-primary);
  padding: 10px 24px;
  display: flex;
  flex-wrap: wrap;
  gap: 8px;
  position: sticky;
  top: 64px;  /* Below header */
  z-index: 190;
}
```

### Loading Overlay with Spinner

```html
<div id="loading">
  <div class="spinner"></div>
  <p>Loading dashboard...</p>
  <div class="pct" id="load-pct">0%</div>
</div>
```

```css
#loading {
  position: fixed;
  inset: 0;
  background: rgba(255,255,255,.92);
  display: flex;
  flex-direction: column;
  align-items: center;
  justify-content: center;
  z-index: 999;
}
.spinner {
  width: 48px;
  height: 48px;
  border: 5px solid var(--color-border);
  border-top-color: var(--color-primary);
  border-radius: 50%;
  animation: spin .9s linear infinite;
}
@keyframes spin { to { transform: rotate(360deg); } }
#loading p { margin-top: 14px; color: var(--color-grey); }
#loading .pct { font-size: 22px; font-weight: 700; }
```

```javascript
function hideLoading() {
  document.getElementById('loading').style.display = 'none';
}

function updateLoadingProgress(percent) {
  document.getElementById('load-pct').textContent = percent + '%';
}
```

### Expandable Text Rows

```html
<tr>
  <td>Account Name</td>
  <td>
    <button class="expand-btn" onclick="toggleExpand(this, 'acc-123')">
      Show Details ▶
    </button>
    <div id="acc-123" class="text-block">
      <div class="text-section">
        <div class="lbl">Description</div>
        <p>Detailed account description goes here...</p>
      </div>
    </div>
  </td>
</tr>
```

```css
.expand-btn {
  background: none;
  border: none;
  color: var(--color-teal);
  cursor: pointer;
  font-size: var(--font-size-xs);
  font-weight: 600;
}
.expand-btn:hover { text-decoration: underline; }

.text-block { 
  max-height: 0; 
  overflow: hidden; 
  transition: max-height .3s; 
}
.text-block.open { max-height: 400px; }

.text-section { margin-top: 8px; }
.text-section .lbl { 
  font-size: 10px; 
  font-weight: 700; 
  color: var(--color-grey); 
  text-transform: uppercase; 
}
.text-section p { 
  font-size: var(--font-size-xs); 
  margin-top: 2px; 
  line-height: 1.5; 
}
```

```javascript
function toggleExpand(btn, id) {
  const block = document.getElementById(id);
  const isOpen = block.classList.toggle('open');
  btn.textContent = isOpen ? 'Hide Details ▼' : 'Show Details ▶';
}
```

### Pagination Controls

```html
<div class="pagination">
  <button class="pg-btn" onclick="goToPage(1)" disabled>« First</button>
  <button class="pg-btn" onclick="goToPage(currentPage-1)">‹ Prev</button>
  <span class="pg-info">Page <span id="pg-current">1</span> of <span id="pg-total">10</span></span>
  <button class="pg-btn" onclick="goToPage(currentPage+1)">Next ›</button>
  <button class="pg-btn" onclick="goToPage(totalPages)">Last »</button>
</div>
```

```css
.pagination { display: flex; gap: 6px; align-items: center; margin-top: 16px; }
.pg-btn {
  padding: 5px 12px;
  border: 1px solid var(--color-border);
  background: var(--color-white);
  border-radius: 4px;
  cursor: pointer;
  font-size: var(--font-size-sm);
}
.pg-btn.active { 
  background: var(--color-primary); 
  color: #fff; 
  border-color: var(--color-primary); 
}
.pg-btn:hover:not(.active):not(:disabled) { background: var(--color-bg); }
.pg-btn:disabled { opacity: .4; cursor: default; }
.pg-info { font-size: var(--font-size-sm); color: var(--color-grey); }
```

## Chart Type Reference

| Chart | Use Case | Data Structure | Best Practices | Cognitive Pitfalls |
|-------|----------|----------------|----------------|-------------------|
| **Bar/Column** | Compare magnitudes across categories | 1 Nominal + 1 Quantitative | Bar width = 2× gap; horizontal for long labels | Y-axis MUST start at 0; avoid 3D |
| **Line Chart** | Trends over time | 1 Temporal + 1 Quantitative | Max 6 lines; direct label; faint gridlines | Avoid dual y-axes; no gridlines if labeled |
| **Scatter Plot** | Correlations between 2 variables | 2 Quantitative | Add trend line or quadrant highlights | Outliers can distort scale |
| **Bubble Chart** | 3 quantitative variables | 3 Quantitative | Map to AREA, never diameter | Area estimation prone to underestimation (α≈0.7) |
| **Pie/Donut** | Simple ratios (use sparingly) | 1 Nominal (max 3) + 1 Quant | Only for very simple A vs B ratios | Replace with bar chart for accuracy |
| **Heatmap** | Matrix patterns, correlations | 2 Nominal + 1 Quantitative | Overlay numeric labels; accessible colors | Color saturation has low perceptual accuracy |
| **Radar Chart** | Multivariate profiles | 3+ Quantitative + 1 Nominal | Identical scales; high transparency; max 5 axes | Radial comparison is harder than Cartesian |
| **Treemap** | Nested hierarchical part-to-whole | Nested Nominal + 1 Quant | High-contrast borders; max 2-3 nesting levels | Non-adjacent rectangles hard to compare |
| **Sankey** | Flow/path relationships | Source → Target + Weight | Gradient colors for links | Flows can become tangled with high cardinality |
| **Word Cloud** | Qualitative frequency overview | Text → Frequency | Filter stop-words; use as hook not analysis | Longer words appear larger; context stripped |
| **Network** | Node relationships, clusters | Adjacency list (Source, Target) | Scale node/link by metrics; directed arrows | Easily becomes unreadable "hairball" |
| **Stacked Bar** | Part-to-whole over categories | 1 Nominal + 1 Ordinal + 1 Quant | Max 3-4 segments; key category at baseline | Non-baseline segments lack common reference |
| **Box Plot** | Statistical distributions | 1 Quantitative + optional grouping | Explain whisker meaning (1.5×IQR) | Low statistical literacy in exec audiences |
| **Histogram** | Continuous distribution | 1 Continuous (binned) | Uniform bin widths; no gaps between bars | Not for categorical data |
| **Slope Graph** | Change between 2 time points | 1 Nominal + 2 Ordinal + 1 Quant | Color-code increases vs decreases | Fails if intermediate fluctuations matter |

### Detailed Templates

- [charts/bar-charts.md](charts/bar-charts.md)
- [charts/radar-charts.md](charts/radar-charts.md)
- [charts/heatmaps.md](charts/heatmaps.md)
- [charts/sankey.md](charts/sankey.md)
- [charts/treemap.md](charts/treemap.md)
- [charts/network.md](charts/network.md)
- [charts/wordcloud.md](charts/wordcloud.md)
- [charts/quadrant.md](charts/quadrant.md)

## Common Implementations

### Google Charts Bar

```javascript
google.charts.load('current', {'packages':['corechart']});
google.charts.setOnLoadCallback(drawChart);

function drawChart() {
  var data = google.visualization.arrayToDataTable([
    ['Category', 'Value'],
    ['A', 100],
    ['B', 80],
    ['C', 60]
  ]);
  
  var options = {
    title: 'Values by Category',
    legend: { position: 'none' }
  };
  
  var chart = new google.visualization.BarChart(document.getElementById('chart'));
  chart.draw(data, options);
}
```

### Chart.js Radar

```javascript
const ctx = document.getElementById('radarChart').getContext('2d');
new Chart(ctx, {
  type: 'radar',
  data: {
    labels: ['Metric 1', 'Metric 2', 'Metric 3', 'Metric 4', 'Metric 5'],
    datasets: [{
      label: 'Series A',
      data: [85, 70, 60, 90, 45],
      backgroundColor: 'rgba(54, 162, 235, 0.2)',
      borderColor: 'rgb(54, 162, 235)'
    }]
  },
  options: {
    scales: {
      r: { suggestedMin: 0, suggestedMax: 100 }
    }
  }
});
```

### Plotly Heatmap

```javascript
Plotly.newPlot('heatmap', [{
  z: [[1, 20, 30], [20, 1, 60], [30, 60, 1]],
  x: ['A', 'B', 'C'],
  y: ['X', 'Y', 'Z'],
  type: 'heatmap',
  colorscale: 'RdYlGn'
}], {
  title: 'Correlation Matrix'
});
```

### Google Charts Sankey

```javascript
google.charts.load('current', {'packages':['sankey']});
google.charts.setOnLoadCallback(drawChart);

function drawChart() {
  var data = new google.visualization.DataTable();
  data.addColumn('string', 'From');
  data.addColumn('string', 'To');
  data.addColumn('number', 'Weight');
  data.addRows([
    ['A', 'X', 5],
    ['A', 'Y', 7],
    ['B', 'X', 3],
    ['B', 'Y', 9]
  ]);
  
  var chart = new google.visualization.Sankey(document.getElementById('chart'));
  chart.draw(data, { sankey: { link: { colorMode: 'gradient' } } });
}
```

## Chart Templates

Full code templates with styling are in the `charts/` folder:

- [charts/bar-charts.md](charts/bar-charts.md)
- [charts/radar-charts.md](charts/radar-charts.md)
- [charts/heatmaps.md](charts/heatmaps.md)
- [charts/sankey.md](charts/sankey.md)
- [charts/treemap.md](charts/treemap.md)
- [charts/network.md](charts/network.md)
- [charts/wordcloud.md](charts/wordcloud.md)
- [charts/quadrant.md](charts/quadrant.md)

---

## Executive Presentation Techniques

When presenting to executives, shift from data display to **visual argument**:

### Action-Oriented Titles

Replace generic headers with takeaways:

| Bad | Good |
|-----|------|
| "Sales Over Time" | "Sales Dropped 15% After Supply Chain Disruption" |
| "Revenue by Region" | "EMEA Outperforms All Regions by 23%" |
| "Q2 Results" | "Q2 Exceeded Target Despite Market Headwinds" |

### Direct Annotation Layers

Add text callouts directly on the chart canvas:

```
Y-Axis
│
│         *───────── "Fuel price spike triggers
│        /           margin compression"
│   *───*
│  /
│ *
└────────────────── X-Axis
```

### Selective Visual Focus

- **Gray** for supporting/comparison data
- **Action color** ONLY for the key insight
- Guides viewer's eye to the argument

### Cognitive Load Reduction

- Prioritize clarity over complexity
- Remove visual noise (excessive borders, gridlines, legends)
- Natural flow: high-level summary → detailed supporting data

---

## Related

- `apps-script-dashboard` skill - Building the dashboard framework
- `apps-script-limits` rule - Be aware of execution time limits when rendering charts

---

## References

Theoretical foundations based on:
- Cleveland & McGill (1984) - Graphical Perception Theory
- Cairo, Alberto - The Truthful Art
- Emery & Evergreen - Data Visualization Checklist
- Mackinlay - Expressiveness and Effectiveness criteria
