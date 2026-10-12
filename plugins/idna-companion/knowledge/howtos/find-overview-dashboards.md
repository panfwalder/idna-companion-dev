---
id: howtos/find-overview-dashboards
title: Find the Overview dashboards and pages
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/consolidation]
related:
  - concepts/remediation-classification
  - concepts/focus-db
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08
kbase_sources:
  - metabase-10000000-start-page
  - metabase-10000004-scope-focus-funnel
  - metabase-10000005-remediation-overview
  - metabase-10000033-user-licensing-optimization
  - metabase-10000034-deny-access-group-memberships
  - portal-environment-overview
  - portal-inventory
  - portal-remediation-overview
  - portal-scope-focus-funnel
  - portal-start-page
  - portal-status-dashboard
  - portal-user-licensing-optimization
last_reviewed: 2026-10-12
---

# Find the Overview dashboards and pages

The Overview section of the Portal holds the estate-wide summaries: what iDNA has collected, how large the environment is and what its database inventory consists of, how far the project scope can be narrowed, which remediation paths the applications fall into, and where user licensing can be optimized. One Metabase-only dashboard on deny-access group memberships belongs with them. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### Start Page

- **Where:** Overview › Start Page, `{{idna_server}}/#/mbhome`.
- **What you get:** the Portal's entry dashboard: three key figures (replica sets with very low or no recent usage, how far the project scope is reduced, inactive users), each with a link to the analysis behind it. A row of charts groups Focus DB replica sets into usage categories, applications into lifecycle stages and users by the time frame of their last activity; there are no filters.
- **Fits questions like:** is there a one-page summary of my application estate? Where do I see at a glance how many applications are consolidation candidates? How many of our users are inactive?

### Status Dashboard

- **Where:** Overview › Status Dashboard, `{{idna_server}}/#/home`.
- **What you get:** key figures on what iDNA has collected and analyzed: database instances discovered and used, Focus DBs identified, used and unlocked in the current license, Focus DB designs collected and analyzed, Focus DBs with Insights findings in their source code, and registered versus active users. Charts add web versus client usage, the usage history of the Focus DBs and their design complexity classes; there are no filters.
- **Fits questions like:** is iDNA collecting data from all of my databases? How many Focus DB designs has iDNA collected and analyzed? How many Focus DBs does my current license unlock?

### Environment Overview

- **Where:** Overview › Environment Overview, `{{idna_server}}/#/deo`.
- **What you get:** one read-only diagram of the analyzed Domino environment with a count for each element: directories, users and groups, ACL entries, servers, databases and their storage, Domino releases and operating systems, and views with their indexes. Emphasized boxes mark the subsets that matter most, such as active users, analyzed servers and databases with usage; there are no filters.
- **Fits questions like:** how big is our Domino environment? How many of the discovered servers does iDNA analyze? Is there one picture of the environment I can show to management?

### Inventory

- **Where:** Overview › Inventory, `{{idna_server}}/#/inventoryhistory`.
- **What you get:** a table and donut charts that break your databases down by type (applications, mail-in databases, user mail, system databases, directories and more), with DB instances, replica sets and disk usage each shown as total, used in recorded history and percentage; there are no filters. Despite its route, this native page is not App Lifecycle › Inventory History, the dashboard that tracks the inventory across time.
- **Fits questions like:** how many of our databases are applications and how many are mail? How much disk space do applications take compared with other databases? How many replica sets have been used at all?

### Scope Focus Funnel

- **Where:** Overview › Scope Focus Funnel, `{{idna_server}}/#/dashboard/10000004`.
- **What you get:** a funnel that narrows the estate to the databases worth a project's effort, removing databases at each stage by usage, design and similarity. Next to it are key figures for Project Scope, Replica Sets (Apps) in Focus and Replica Sets (Apps) Used This Month; there are no filters.
- **Fits questions like:** how far can the project scope be reduced? How much of the estate really matters for a migration project? How many applications are in focus?

### Remediation Overview

- **Where:** Overview › Remediation Overview, `{{idna_server}}/#/dashboard/10000005`.
- **What you get:** a bubble chart that places groups of applications (replica sets) by usage and design complexity and colors each group by its potential remediation path. Beside it are counts for Consolidation Potential, Quick Win Candidates and Business Critical; there are no filters.
- **Fits questions like:** how many applications are quick-win candidates for migration? Which remediation paths do our applications fall into? How many applications look business critical?

### User Licensing Optimization

- **Where:** Overview › User Licensing Optimization, `{{idna_server}}/#/dashboard/10000033`.
- **What you get:** license optimization candidates among your users, derived from HCL Domino License Analysis Utility (DLAU) results together with iDNA's activity and security information: charts by user category and by the authorization removal action required, plus a users list. You can filter by optimization category and by flags such as agent signer, ACL manager, locked-out ID, deny-access memberships and available DLAU data; the dashboard depends on DLAU results being integrated into iDNA.
- **Fits questions like:** how many users might no longer need a Domino license? Which authorization changes would lower our license count? Is there a dashboard that uses DLAU results?

### Deny Access Group Memberships

- **Where:** `{{idna_server}}/metabase/dashboard/10000034`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **What you get:** a single table, Deny Access Group Membership Details, showing users' memberships in deny-access groups; its only filter is User. User Licensing Optimization has a related Deny Access Memberships filter.
- **Fits questions like:** where do I find the Deny Access Group Memberships dashboard? Which users are members of deny-access groups?
