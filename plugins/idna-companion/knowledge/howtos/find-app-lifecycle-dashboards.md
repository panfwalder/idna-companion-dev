---
id: howtos/find-app-lifecycle-dashboards
title: Find the App Lifecycle dashboards
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/consolidation, topic/modernization, topic/usage]
related:
  - concepts/lifecycle-vector
  - playbooks/application-lifecycle/lifecycle-routing
  - playbooks/application-lifecycle/sunset-funnel
  - playbooks/application-lifecycle/keep-funnel
  - playbooks/application-lifecycle/stakeholder-cohort
  - concepts/about-modernization-plan
  - concepts/about-modernization-vector
  - concepts/about-template-lineage
  - concepts/modernization-dashboard-labels
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08 + Metabase capture 2026-10-09 (re-captured 2026-10-12) of 10000043 / 10000045 / 10000048 / 10000049 / 10000050 and product-owner input (D-8, 2026-10-07 / 2026-10-09; `dashboard-texts-draft.md`)
kbase_sources:
  - metabase-10000001-inventory-history
  - metabase-10000012-department-location-activity-lookup
  - metabase-10000015-top-user-lookup
  - metabase-10000021-app-activity-drill-down
  - metabase-10000042-lifecycle-overview
  - metabase-10000043-modernization-overview
  - metabase-10000044-application-insights
  - metabase-10000045-modernization-planning
  - metabase-10000046-modernization-overview-by-work-session
  - metabase-10000048-modernization-readiness
  - metabase-10000049-template-lineage
  - metabase-10000050-modernization-session-work-list
  - portal-app-activity-drill-down
  - portal-department-location-activity-lookup
  - portal-inventory-history
  - portal-lifecycle-overview
  - portal-modernization-overview
  - portal-modernization-planning
  - portal-modernization-readiness
  - portal-replica-sets
  - portal-top-user-lookup
last_reviewed: 2026-10-12
---

# Find the App Lifecycle dashboards

The App Lifecycle section of the iDNA Portal follows applications through their life: how the inventory develops, which applications to keep or sunset, how ready the kept ones are for Notes 64-bit or Nomad and how the modernization work is planned, and who works with an application. This page also lists the Metabase-only dashboards that belong to this area. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### Inventory History

- **Where:** App Lifecycle › Inventory History, `{{idna_server}}/#/dashboard/10000001`.
- **What you get:** the Focus DB inventory over time (instances and replica sets), how many replica sets were used this month, a Consolidation Potential figure for little-used replica sets, and how Focus DB replica sets divide by usage level over the last 90 days, the last 365 days and all recorded history; the dashboard has no filters. Not to be confused with Overview › Inventory (`{{idna_server}}/#/inventoryhistory`), a separate native Portal page despite its route name.
- **Fits questions like:** how has our application inventory developed over time? How many of our applications are still in use? How much of the estate is a consolidation candidate?

### Lifecycle Overview

- **Where:** App Lifecycle › Lifecycle Overview, `{{idna_server}}/#/dashboard/10000042`.
- **What you get:** which Lifecycle Vector each application (replica set) is on, as a Sankey chart from keep or sunset down to the disposition and its reason, plus a replica set list and a tab that explains the vector. You can filter by Lifecycle Vector and by Title or Replica ID; Show Details adds the chart's finest level.
- **Fits questions like:** which of our applications are sunset candidates? Which applications should we keep and modernize? Is there a dashboard for the lifecycle vector?

### Modernization Readiness

- **Where:** App Lifecycle › Modernization Readiness, `{{idna_server}}/#/dashboard/10000048`.
- **What you get:** the readiness of the kept applications for one modernization topic: how many are ready and how many still need fixes, by usage in the last 90 days, and how their share of applications compares with their share of usage. A second tab makes the same comparison per modernization vector and explains each vector; choose the topic in the Modernization Topic filter.
- **Fits questions like:** how ready are our applications for Notes 64-bit? Are the applications that still need fixes the ones people use most? Where does a small fix effort help the most users?

### Modernization Overview

- **Where:** App Lifecycle › Modernization Overview, `{{idna_server}}/#/dashboard/10000043`.
- **What you get:** the modernization vector of each kept application for one modernization topic, as a Sankey chart from ready or needing fixes down to the kind of fix path, plus a replica set list and a tab that explains the vectors. You can filter by Modernization Topic, Lifecycle Vector, Modernization Vector, Title or Replica ID and Only Show Active Since; Show Details adds the chart's finest level.
- **Fits questions like:** which modernization vector does this application have? How do our applications split between ready and needing fixes for Nomad? How should we go about making this application ready?

### Modernization Planning

- **Where:** App Lifecycle › Modernization Planning, `{{idna_server}}/#/dashboard/10000045`.
- **What you get:** the modernization plan for one modernization topic: a summary of the applications that still need fixes and of the work by kind of session, optionally limited to applications used in the last 90 days, then the ranked work sessions with the applications each affects and its step-by-step procedure, linking to the session's code work and to the master's code changes. Choose the topic in the Modernization Topic filter and a session by clicking its title.
- **Fits questions like:** which modernization work pays off first? Where is the remaining code work, and how much of it is for applications in use? What are the steps of this session?

### Modernization Overview by Work Session

- **Where:** `{{idna_server}}/metabase/dashboard/10000046`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **What you get:** a single list of modernization work sessions that you can narrow by Modernization Topic and Modernization Vector; one session usually covers several applications, so sessions and applications do not add up one to one. Not the same as the Modernization Session Work List, which itemizes the code work inside one session.
- **Fits questions like:** which work sessions does the plan hold for a given modernization vector? Is there a plain list of all modernization sessions?

### Template Lineage

- **Where:** `{{idna_server}}/metabase/dashboard/10000049`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **Opened from:** App Lifecycle › Modernization Planning, the Drill-Down on a session's re-apply step.
- **What you get:** this dashboard compares the code of one master template with the standard template it was derived from: which code-carrying design documents were modified or added, and for each finding whether the code survives a template update. Also the ancestor, its template family and the newest version; the comparison covers code only, and it is less complete when the ancestor is iDNA's bundled reference rather than a template scanned in your estate.
- **Fits questions like:** which code did we customize in this master? Which findings disappear when we update the template? Which standard template is this master derived from?

### Modernization Session Work List

- **Where:** `{{idna_server}}/metabase/dashboard/10000050`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **Opened from:** App Lifecycle › Modernization Planning, the Drill-Down on a session's code-block steps.
- **What you get:** the code blocks of one modernization session: what to do with each, where its fix is written, and whether it is HCL template code. Also the design documents and code elements per application, with a link into Domino Designer; you choose the modernization topic and the session in the filters.
- **Fits questions like:** what exactly do we fix in this session? Which design elements are affected, in which applications? Is this code ours or HCL's?

### Department/Location Activity Lookup

- **Where:** App Lifecycle › Department/Location Activity Lookup, `{{idna_server}}/#/dashboard/10000012`.
- **What you get:** the departments and locations that work with an application: for the replica sets matching your search, their activity per department and per location and their usage history across all recorded history. You choose the applications by Title or Replica ID.
- **Fits questions like:** which departments use this application? Where are the users of this database located? Which business units should we talk to before retiring this application?

### Top User Lookup

- **Where:** App Lifecycle › Top User Lookup, `{{idna_server}}/#/dashboard/10000015`.
- **What you get:** the most active users of one application, ranked by several usage measures (document reads and writes from the Notes client, user access days, session count); you choose the application by Title or Replica ID.
- **Fits questions like:** which people use this application most? Who are the key users to involve before we change this application?

### App Activity Drill-down

- **Where:** App Lifecycle › App Activity Drill-down, `{{idna_server}}/#/dashboard/10000021`.
- **What you get:** a table of user activity per application (replica set) within a time period; you can narrow it by Time Period, DB Type and Title or Replica ID.
- **Fits questions like:** is anyone still using this application? Which users would be affected if we retired this database?

### Application Insights

- **Where:** `{{idna_server}}/metabase/dashboard/10000044`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **Opened from:** Catalog › Replica Sets and App Lifecycle › Modernization Planning, a replica set's title.
- **What you get:** one application (replica set) at a glance: its general and lifecycle details, when it was last used, its usage history and its replicas, and on a second tab its code findings. Links lead on to Top User Lookup, Department/Location Activity Lookup, App Activity Drill-down and the Retention Catalog for the same application.
- **Fits questions like:** where can I see everything about one application? When was this application last used, and does it have code findings?
