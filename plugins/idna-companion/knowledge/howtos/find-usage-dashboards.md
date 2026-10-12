---
id: howtos/find-usage-dashboards
title: Find the usage dashboards and pages
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/usage, topic/organization]
related:
  - concepts/usage-tier
  - concepts/usage-time-windows
  - concepts/user-access-days
  - howtos/find-most-used-applications
  - howtos/analyze-usage-by-department
  - schema/topics/department-and-location
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08
kbase_sources:
  - metabase-10000007-user-activity-overview
  - metabase-10000008-department-location-overview
  - metabase-10000009-department-activity
  - metabase-10000010-usage-category-kpis
  - metabase-10000011-location-activity
  - metabase-10000013-user-activity-overview-focus-dbs
  - metabase-10000014-user-activity-lookup
  - metabase-10000020-user-activity-drill-down
  - portal-department-activity
  - portal-department-location-overview
  - portal-location-activity
  - portal-notes-client-usage
  - portal-sessions-vs-complexity
  - portal-usage-category-kpis
  - portal-usage-pattern-similarity
  - portal-user-activity-drill-down
  - portal-user-activity-lookup
  - portal-user-activity-overview
  - portal-user-activity-overview-focus-dbs
  - portal-web-browser-usage
  - portal-web-vs-client-usage
last_reviewed: 2026-10-09
---

# Find the usage dashboards and pages

The Usage section of the iDNA Portal shows how the Notes estate is used: activity of users, departments and locations, the usage categories of replica sets, Notes client versus web browser access, usage set against design complexity, and groups of applications that share their users. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### User Activity Overview

- **Where:** Usage › User Activity Overview, `{{idna_server}}/#/dashboard/10000007`.
- **What you get:** a list of users with activity KPIs, next to pie charts of when users were last active and how many database instances each of them has used. Every database type counts as activity here; the User Activity Overview (Focus DBs) variant counts only Focus DBs and Focus Replica Sets.
- **Fits questions like:** when were our users last active? How many databases do our users work with? Is there an overview of user activity for license planning?

### User Activity Overview (Focus DBs)

- **Where:** Usage › User Activity Overview (Focus DBs), `{{idna_server}}/#/dashboard/10000013`.
- **What you get:** the same three views as User Activity Overview, limited to activity on Focus DBs and Focus Replica Sets: a user list with activity KPIs, when users were last active on Focus DBs, and how many Focus Replica Sets each of them has used. It is the variant to choose when access to system databases or other non-Focus databases should not count as activity.
- **Fits questions like:** when were our users last active on business applications? How many applications does a typical user work with? How many users have not used a Focus DB for a long time?

### User Activity Lookup

- **Where:** Usage › User Activity Lookup, `{{idna_server}}/#/dashboard/10000014`.
- **What you get:** activity of individual users on Focus Replica Sets: the user names that match the name filter, an activity history chart, and the replica sets those users accessed.
- **Fits questions like:** is there a dashboard that shows which applications individual users work with? Where can I see per-user activity on Focus Replica Sets?

### User Activity Drill-down

- **Where:** Usage › User Activity Drill-down, `{{idna_server}}/#/dashboard/10000020`.
- **What you get:** the full user list with activity in a chosen time period, which you can filter by time period, database type, department and location. It reads raw activity data, so it can take longer to load than other dashboards.
- **Fits questions like:** which users were active in a given period? Is there a complete user list I can narrow down by department or location?

### Department / Location Overview

- **Where:** Usage › Department / Location Overview, `{{idna_server}}/#/dashboard/10000008`.
- **What you get:** activity by department and by location, with no filters to set: pie charts of when departments and locations last accessed a Focus DB, and the most active departments and locations ranked by user access days.
- **Fits questions like:** which departments use Notes applications the most? Are there locations that have stopped using Focus DBs? How is activity spread across our departments and locations?

### Department Activity

- **Where:** Usage › Department Activity, `{{idna_server}}/#/dashboard/10000009`.
- **What you get:** which replica sets departments use: a department list ranked by user access days, a weekly history of active users per department, and the replica sets accessed by the departments you select with the Department Name filter.
- **Fits questions like:** which applications does a given department use? How has a department's activity developed over recent months? Which departments have the most activity?

### Location Activity

- **Where:** Usage › Location Activity, `{{idna_server}}/#/dashboard/10000011`.
- **What you get:** which replica sets are used from which locations: a location list ranked by user access days, a weekly history of active users per location, and the replica sets accessed from the locations you select with the Location Name filter.
- **Fits questions like:** which applications are used at a given location? How has activity at a location changed over recent months? Which locations are the most active?

### Usage Category KPIs

- **Where:** Usage › Usage Category KPIs, `{{idna_server}}/#/dashboard/10000010`.
- **What you get:** KPIs per usage category, from No Usage to Very High, for Focus DB replica sets and for Focus DB instances over all recorded history: pie charts of how many fall into each category, and tables with the values per category. The categories derive from user access days; the dashboard has no filters.
- **Fits questions like:** how many of our applications fall into each usage category? What does High usage mean in numbers for our environment? How are our replica sets distributed across the usage tiers?

### Notes Client Usage

- **Where:** Usage › Notes Client Usage, `{{idna_server}}/#/usage_client_bubble`.
- **What you get:** a bubble chart of Focus DBs by the data read against the data written through the Notes client, which shows which databases are mainly looked up and which carry heavy write traffic. You can show or hide the database types Application, Mailin DB and System DB.
- **Fits questions like:** which databases are mostly read and rarely written in the Notes client? Which applications get the most write traffic from Notes clients?

### Web Browser Usage

- **Where:** Usage › Web Browser Usage, `{{idna_server}}/#/usage_web_bubble`.
- **What you get:** the browser counterpart of Notes Client Usage: Focus DBs with web traffic, by data downloaded against data uploaded over HTTP, with the same database-type toggles. Only servers whose Domino web log is collected contribute to it.
- **Fits questions like:** which applications are used through a web browser? Is the web traffic on a database read-only or interactive?

### Web vs. Client Usage

- **Where:** Usage › Web vs. Client Usage, `{{idna_server}}/#/usage_webpct_bubble`.
- **What you get:** for each Focus DB, where its sessions come from on a scale from Pure Notes Client through Mixed Access to Pure Web Access, plotted against its total sessions; database types can be shown or hidden. It shows which applications already depend on browser access and which are used only in the Notes client.
- **Fits questions like:** which applications are used only in the Notes client? Which applications are accessed both through the web and the Notes client? Which applications already rely on browser access?

### Sessions vs. Complexity

- **Where:** Usage › Sessions vs. Complexity, `{{idna_server}}/#/usage_complexity_sessions_bubble`.
- **What you get:** a bubble chart of Focus DBs by sessions against design complexity score, both on logarithmic scales, with databases that have no complexity score on a separate baseline; Application and Mailin DB can be shown or hidden. It is the per-database view of the data that Overview › Remediation Overview groups into the remediation classification.
- **Fits questions like:** which heavily used applications are also complex? Where are the quick-win candidates among our applications? Which databases have no complexity score yet?

### Usage Pattern Similarity

- **Where:** Usage › Usage Pattern Similarity, `{{idna_server}}/#/usagecluster`.
- **What you get:** replica sets grouped into usage clusters by the people who use them, shown as a bubble map of the top clusters and a grid with the users, replica sets and ratio of users per replica set for each cluster. Clusters in which a small group of people uses many replica sets point to consolidation candidates; the grouping comes from activity, not from design (design overlap is under Design › Similarity).
- **Fits questions like:** which applications are used by the same group of people? Where could overlapping applications be consolidated? Which applications serve our largest user groups?
