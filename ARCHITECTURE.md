# Architecture Overview

This document explains how the OCI Metrics Report Generator works - from data fetching to report generation.

## Quick Summary

| Component | Technology | Purpose |
|-----------|------------|---------|
| **Backend** | Python + Flask | REST API server, OCI SDK integration |
| **Frontend** | Vanilla JavaScript | Single-page application UI |
| **Data Source** | OCI Monitoring API | Fetches metrics via MQL queries |
| **Visualization** | Chart.js | Time-series charts with gap detection |
| **Export** | html2canvas + jsPDF | PNG screenshots, PDF reports |

---

## System Architecture

```
┌─────────────────────────────────────────────────────────────────────────┐
│                         USER'S BROWSER                                   │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │                    index.html (SPA)                                │  │
│  │  ┌─────────────┐  ┌─────────────┐  ┌─────────────────────────────┐│  │
│  │  │ Query       │  │ Chart.js    │  │ Export Tools                ││  │
│  │  │ Builder UI  │  │ Visualization│  │ (html2canvas, jsPDF)        ││  │
│  │  └─────────────┘  └─────────────┘  └─────────────────────────────┘│  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ HTTP/JSON
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                        FLASK BACKEND (app.py)                            │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ REST API Endpoints                                                 │  │
│  │  /api/compartments  /api/namespaces  /api/metrics  /api/query     │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ OCIRegionClientManager        │  OCIMonitoringClient              │  │
│  │ (manages clients per region)  │  (executes OCI API calls)         │  │
│  └───────────────────────────────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────────────────────────────┐  │
│  │ Authentication: Config File | Instance Principal | Security Token │  │
│  └───────────────────────────────────────────────────────────────────┘  │
└──────────────────────────────────┬──────────────────────────────────────┘
                                   │ OCI Python SDK
                                   ▼
┌─────────────────────────────────────────────────────────────────────────┐
│                    ORACLE CLOUD INFRASTRUCTURE                           │
│  ┌────────────────────────┐  ┌────────────────────────────────────────┐ │
│  │ Identity Service       │  │ Monitoring Service                     │ │
│  │ - List compartments    │  │ - List metrics/namespaces              │ │
│  │ - List regions         │  │ - Execute MQL queries                  │ │
│  └────────────────────────┘  └────────────────────────────────────────┘ │
└─────────────────────────────────────────────────────────────────────────┘
```

---

## Data Flow: How Metrics Are Fetched

### Step 1: User Builds a Query
```
User selects:
  → Compartment(s): "Production", "Development"
  → Region(s): "us-ashburn-1", "uk-london-1"
  → Namespace: "oci_computeagent"
  → Metric: "CpuUtilization"
  → Interval: "1h"
  → Statistic: "mean()"
```

### Step 2: Frontend Builds MQL Query
```javascript
// MQL (Monitoring Query Language) format:
"CpuUtilization[1h].mean()"

// With dimension filter:
"CpuUtilization[1h]{resourceDisplayName = \"web-server-1\"}.mean()"
```

### Step 3: API Request to Backend
```javascript
// POST /api/query-unified
{
  "regions": ["us-ashburn-1", "uk-london-1"],
  "compartment_ids": ["ocid1...", "ocid1..."],
  "namespace": "oci_computeagent",
  "query": "CpuUtilization[1h].mean()",
  "start_time": "2024-01-01T00:00:00Z",
  "end_time": "2024-01-02T00:00:00Z"
}
```

### Step 4: Backend Executes OCI API Calls
```python
# For each region × compartment combination:
monitoring_client.summarize_metrics_data(
    compartment_id=compartment_id,
    summarize_metrics_data_details=SummarizeMetricsDataDetails(
        namespace="oci_computeagent",
        query="CpuUtilization[1h].mean()",
        start_time=start_time,
        end_time=end_time
    )
)
```

### Step 5: Results Returned with Source Metadata
```json
{
  "results": [
    {
      "region": "us-ashburn-1",
      "compartment_name": "Production",
      "metric_data": [
        {
          "name": "CpuUtilization",
          "dimensions": {"resourceDisplayName": "web-server-1"},
          "datapoints": [
            {"timestamp": "2024-01-01T00:00:00Z", "value": 45.2},
            {"timestamp": "2024-01-01T01:00:00Z", "value": 52.1}
          ]
        }
      ]
    }
  ]
}
```

### Step 6: Chart.js Renders Visualization
```
┌─────────────────────────────────────────────────────────────┐
│  CPU Utilization                                             │
│  100% ┤                                                      │
│   75% ┤           ╭───╮                                      │
│   50% ┤    ╭──────╯   ╰────╮     ╭─────                      │
│   25% ┤────╯               ╰─────╯                           │
│    0% ┼──────────────────────────────────────────────────    │
│        00:00    06:00    12:00    18:00    00:00             │
└─────────────────────────────────────────────────────────────┘
  Legend: ● web-server-1 [us-ashburn-1] (Production)
          ● web-server-2 [uk-london-1] (Development)
```

---

## Report Generation: How Exports Work

### PNG Export
```
Chart DOM Element
      ↓
html2canvas.js captures canvas
      ↓
Canvas converted to data URL
      ↓
Browser downloads as .png file
```

### PDF Export
```
Chart DOM Element(s)
      ↓
html2canvas.js captures each chart
      ↓
jsPDF creates PDF document
  - Adds title page (if "Export All")
  - Adds chart images to pages
  - Adds headers, timestamps, metadata
      ↓
Browser downloads as .pdf file
```

### CSV Export
```
Raw metric data from API
      ↓
JavaScript formats to CSV string
  - Headers: timestamp, value, series_name, region, compartment
  - One row per datapoint
      ↓
Browser downloads as .csv file
```

---

## Key Components Explained

### Backend: `app.py`

| Component | Responsibility |
|-----------|----------------|
| `OCIMonitoringClient` | Wraps OCI SDK for metric queries |
| `OCIRegionClientManager` | Caches SDK clients per region |
| `detect_auth_type()` | Auto-detects authentication method |
| `get_signer()` | Creates OCI SDK signer for auth |
| Flask routes (`/api/*`) | REST API endpoints |

### Frontend: `static/index.html`

| Component | Responsibility |
|-----------|----------------|
| `state` object | Global application state |
| `runQuery()` | Executes query and updates chart |
| `renderChart()` | Creates Chart.js visualization |
| `exportChartAsPDF()` | Generates PDF using jsPDF |
| Multi-select UI | Manages compartment/region chips |

---

## Authentication Flow

```
Startup
   │
   ├─→ Check OCI_CLI_AUTH=instance_principal? ──→ Use Instance Principal
   │
   ├─→ Check OCI_RESOURCE_PRINCIPAL_VERSION? ──→ Use Resource Principal
   │
   ├─→ Check OCI_CLI_CLOUD_SHELL? ─────────────→ Use Security Token
   │
   └─→ Default ─────────────────────────────────→ Use Config File (~/.oci/config)
```

---

## API Endpoints Summary

| Endpoint | Method | Purpose |
|----------|--------|---------|
| `/api/auth-info` | GET | Current auth type & tenancy |
| `/api/compartments` | GET | List all compartments |
| `/api/regions` | GET | List subscribed regions |
| `/api/namespaces` | GET | Metric namespaces in compartment |
| `/api/metrics` | GET | Metrics in a namespace |
| `/api/dimensions` | GET | Dimension names/values for metric |
| `/api/query` | POST | Execute single MQL query |
| `/api/query-unified` | POST | Execute across regions/compartments |

---

## MQL Query Examples

```sql
-- Basic CPU utilization (hourly mean)
CpuUtilization[1h].mean()

-- Filter by specific instance
CpuUtilization[1h]{resourceDisplayName = "web-server-1"}.mean()

-- Group by resource ID
CpuUtilization[1h].groupBy(resourceId).max()

-- 95th percentile memory usage
MemoryUtilization[5m].percentile(0.95)

-- Multiple dimension filters
CpuUtilization[1h]{availabilityDomain = "AD-1", faultDomain = "FD-1"}.mean()
```

---

## File Structure

```
metricreport/
├── app.py                  # Flask backend + OCI SDK integration
├── generate_report.py      # CLI tool for standalone reports
├── requirements.txt        # Python dependencies
├── run.sh                  # Startup script
├── cloudshell_report.sh    # CloudShell wrapper
├── static/
│   └── index.html          # Single-page web application (JS + CSS)
├── images/                 # Screenshots for documentation
├── README.md               # User guide & installation
└── ARCHITECTURE.md         # This file
```

---

## Dependencies

### Python (Backend)
```
flask>=2.0.0           # Web framework
flask-cors             # CORS support for API
oci>=2.90.0            # Oracle Cloud SDK
python-dateutil        # Date parsing
```

### JavaScript (Frontend - loaded via CDN)
```
Chart.js 4.4.1         # Time-series visualization
chartjs-adapter-date-fns # Date axis adapter
html2canvas 1.4.1      # Screenshot capture for PNG
jsPDF 2.5.1            # PDF generation
```

---

## Design Patterns Used

1. **Client Pooling** - `OCIRegionClientManager` caches clients per region
2. **Lazy Initialization** - Clients created on-demand, not at startup
3. **Partial Failure Handling** - Multi-region queries return successful results even if some fail
4. **State Persistence** - Frontend state saved when switching between queries
5. **MQL Generation** - UI dynamically builds MQL from form inputs

---

## Quick Start

```bash
# 1. Install dependencies
pip install -r requirements.txt

# 2. Start the server
python app.py

# 3. Open browser
open http://localhost:8080
```

For CLI usage:
```bash
python generate_report.py -c <compartment_ocid> -n oci_computeagent -m CpuUtilization
```
