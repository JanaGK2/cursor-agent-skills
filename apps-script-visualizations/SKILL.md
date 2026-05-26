---
name: apps-script-visualizations
description: Add interactive charts to Apps Script dashboards. Covers Google Charts, Chart.js, D3.js, Plotly, and more. Use when adding visualizations to web apps.
---

# Apps Script Visualizations

## When to Use

- Adding charts to Apps Script dashboard
- Need to choose between visualization libraries
- Implementing specific chart types (radar, heatmap, sankey, etc.)

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

### Clear Chart Before Redraw

If charts fail to shrink when window gets smaller:

```javascript
function drawChart() {
  chart.clearChart();
  chart.draw(chartData, chartOptions);
}
```

### Chart.js Responsive Mode

```javascript
new Chart(ctx, {
  type: 'bar',
  data: {...},
  options: {
    responsive: true,
    maintainAspectRatio: true
  }
});
```

### Plotly Responsive Mode

```javascript
Plotly.newPlot('chart', data, layout, { responsive: true });
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

### Bar/Column Charts → Google Charts or ApexCharts

See [charts/bar-charts.md](charts/bar-charts.md)

### Radar/Spider Charts → Chart.js or ApexCharts

See [charts/radar-charts.md](charts/radar-charts.md)

### Heatmaps → Plotly or ApexCharts

See [charts/heatmaps.md](charts/heatmaps.md)

### Sankey/Flow Diagrams → Google Charts

See [charts/sankey.md](charts/sankey.md)

### Treemaps → Google Charts

See [charts/treemap.md](charts/treemap.md)

### Network Graphs → Cytoscape.js

See [charts/network.md](charts/network.md)

### Word Clouds → D3.js + d3-cloud

See [charts/wordcloud.md](charts/wordcloud.md)

### Quadrant/Scatter → D3.js or Plotly

See [charts/quadrant.md](charts/quadrant.md)

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

## Related

- `apps-script-dashboard` skill - Building the dashboard framework
- `apps-script-limits` rule - Be aware of execution time limits when rendering charts
