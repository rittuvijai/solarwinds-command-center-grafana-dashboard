# SolarWinds Command Center: Enterprise NOC Infrastructure Observability Dashboard for Grafana

[![Grafana Version](https://img.shields.io/badge/Grafana-12.0%2B-orange?logo=grafana)](https://grafana.com/)
[![Datasource](https://img.shields.io/badge/DataSource-MSSQL-blue?logo=microsoftsqlserver)](https://www.microsoft.com/sql-server/)
[![Platform](https://img.shields.io/badge/SolarWinds-Orion%20NPM%20%2F%20SAM-yellow)](#)
[![License: MIT](https://img.shields.io/badge/License-MIT-green.svg)](LICENSE)

An enterprise-grade **NOC Command Center Grafana dashboard** engineered for direct, real-time infrastructure observability across distributed multi-region networks[cite: 1]. This dashboard queries the **SolarWinds Orion (`SolarWindsOrion`) MSSQL database directly**[cite: 1], bypassing Orion Web Console latency to deliver instant telemetry for over 2,600+ managed nodes, core uplinks, and regional sites[cite: 1].

![SolarWinds Command Center Grafana Infrastructure Monitoring Dashboard](solarwinds-command-center-grafana-infrastructure-monitoring-dashboard.jpg)

---

## ⚡ Architecture & Key Performance Indicators (KPIs)

* **Direct SQL Query Pipeline:** Sub-second dashboard refresh cycles querying `dbo.NodesData`, `dbo.AlertActive`, and `dbo.Interfaces` without Orion web API overhead[cite: 1].
* **Global Availability & Estate Triage:** Real-time uptime scoring, latency tracking, and tiered breakdown of Active Down, Critical, Warning, and Unmanaged assets[cite: 1].
* **Multi-Region WAN Health:** Automated SQL casing maps custom property site codes (`DXB`, `KSA`, `OMN`, `EGY`, `LKA`, `PAK`) into consolidated regional health status groups[cite: 1].
* **NOC Core Uplink Watchdog:** High-speed interface filter tracking down states exclusively on core links, backbone circuits, and uplinks exceeding 1 Gbps[cite: 1].
* **Alert Fatigue & Noise Suppression:** Isolates actionable, unacknowledged Severity 2 alerts (`AlertConfigurations.Severity = 2`) with live active-minute duration gauges[cite: 1].
* **Capacity & Grey Failure Detection:** Identifies critical hardware sensor anomalies and drives with fixed disk capacity exceeding 90%[cite: 1].

---

## 📊 Dashboard Panel Architecture

| Section / Panel | Visualization | Target SolarWinds Orion Tables | Operational Focus |
| :--- | :--- | :--- | :--- |
| **Estate Summary KPIs** | Stat Cards | `Nodes`, `AlertActive`, `AlertConfigurations` | Node count, latency (ms), regional outages, global availability %, and active critical alerts[cite: 1]. |
| **Active Issues Breakdown** | Stat List | `NodesData`, `Interfaces`, `Volumes`, `ResponseTime_CS_Detail_hist` | Categorizes nodes down, critical, high latency (>500ms), and packet loss (>10%)[cite: 1]. |
| **Nodes Down by Region** | Bargauge (LCD) | `NodesData`, `NodesCustomProperties` | Granular regional breakdown of hard node outages[cite: 1]. |
| **Warnings & Degraded Links**| Bargauge (LCD) | `NodesData`, `Interfaces`, `ResponseTime_CS_Detail_hist` | Detects degraded operations (flapping interfaces, packet drops)[cite: 1]. |
| **Global Infrastructure Map**| Geomap | `Nodes`, custom GPS coordinates | Spatial visualization of site health and node concentration[cite: 1]. |
| **Vendor Risk Score** | Horizontal Bar | `Nodes` | Identifies problematic hardware platforms by vendor impact[cite: 1]. |
| **NOC Operational Overview** | Multi-column Table| `NodesData`, `NodesCustomProperties`, `Interfaces` | Real-time queue for NOC engineers prioritizing downtime and packet loss[cite: 1]. |
| **Down Core Interfaces** | Table | `Interfaces`, `NodesData` | Filters down interfaces rated >= 1 Gbps or tagged Uplink/Core[cite: 1]. |
| **NetPath Business Services**| Table | `NetPath_EndpointServices`, `NetPath_EndpointServiceAssignments` | End-to-end multi-hop probe connectivity status[cite: 1]. |

---

## 🛠️ Installation & Setup

### Prerequisites
* **Grafana:** Version 11.x or 12.x[cite: 1]
* **SolarWinds Database:** Read-only access to the `SolarWindsOrion` Microsoft SQL Server instance[cite: 1]
* **Datasource:** Configured Grafana MSSQL native plugin pointed to Orion[cite: 1]

### Least-Privilege MSSQL Security Configuration
Run this setup script on your SQL Server instance to grant minimal read access[cite: 1]:

```sql
USE [SolarWindsOrion];
CREATE LOGIN [grafana_noc_reader] WITH PASSWORD = 'StrongPasswordHere!';
CREATE USER [grafana_noc_reader] FOR LOGIN [grafana_noc_reader];

GRANT SELECT ON [dbo].[Nodes] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[NodesData] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[NodesCustomProperties] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[Interfaces] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[Volumes] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[AlertActive] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[AlertObjects] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[AlertConfigurations] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[AlertHistory] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[ResponseTime_CS_Detail_hist] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[HWH_HardwareItem] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[NetPath_EndpointServices] TO [grafana_noc_reader];
GRANT SELECT ON [dbo].[NetPath_EndpointServiceAssignments] TO [grafana_noc_reader];
```

### Dashboard Deployment
1. Download or clone this repository.
2. Inside your Grafana web console, go to **Dashboards** > **New** > **Import**.
3. Upload `dashboard.json` (or paste its content).
4. Select your **MSSQL** datasource when prompted.
5. Save and open the dashboard.

---

## 🔍 Core Telemetry Logic Example

Example of how the dashboard eliminates false alerts by isolating genuine non-up states and interface-level packet degradation simultaneously[cite: 1]:

```sql
WITH LatestMetrics AS (
    SELECT NodeID, ResponseTime, PercentLoss,
    ROW_NUMBER() OVER (PARTITION BY NodeID ORDER BY Timestamp DESC) as Rank
    FROM ResponseTime_CS_Detail_hist
),
InterfaceIssues AS (
    SELECT NodeID, COUNT(InterfaceID) as DownInt 
    FROM Interfaces 
    WHERE Status = 2 
    GROUP BY NodeID
)
SELECT
    n.Caption AS [NODE],
    ISNULL(CAST(cp.Site_ID AS NVARCHAR(100)), 'N/A') AS [SITE],
    CASE 
        WHEN RTRIM(n.Status) = '2' THEN 'DOWN'
        WHEN RTRIM(n.Status) = '14' THEN 'CRITICAL'
        WHEN RTRIM(n.Status) = '3' THEN 'WARNING'
        WHEN RTRIM(n.Status) = '1' AND (ISNULL(i.DownInt, 0) > 0 OR ISNULL(m.PercentLoss, 0) > 0) THEN 'DEGRADED'
        ELSE 'OTHER'
    END AS [HEALTH_STATE],
    ISNULL(CAST(m.ResponseTime AS VARCHAR) + ' ms', 'N/A') AS [RESPONSE TIME],
    ISNULL(m.PercentLoss, 0) AS [PERCENT LOSS],
    ISNULL(i.DownInt, 0) AS [DOWN INT]
FROM NodesData n
LEFT JOIN NodesCustomProperties cp ON n.NodeID = cp.NodeID
LEFT JOIN LatestMetrics m ON n.NodeID = m.NodeID AND m.Rank = 1
LEFT JOIN InterfaceIssues i ON n.NodeID = i.NodeID
WHERE (n.Status NOT IN ('1', '9', '11'))
   OR (n.Status = '1' AND (ISNULL(i.DownInt, 0) > 0 OR ISNULL(m.PercentLoss, 0) > 0))
ORDER BY CASE WHEN n.Status = '2' THEN 0 WHEN n.Status = '14' THEN 1 ELSE 2 END, m.PercentLoss DESC;
```

---

## 📄 License

Distributed under the MIT License. See [`LICENSE`](LICENSE) for details.