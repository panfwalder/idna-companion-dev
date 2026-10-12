---
id: howtos/find-content-dashboards
title: Find the Content dashboards and pages
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/content-age]
related:
  - concepts/document-age-timestamps
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08
kbase_sources:
  - metabase-10000040-retention-catalog
  - metabase-10000041-retention-overview
  - portal-data-retention-analysis
  - portal-retention-catalog
  - portal-retention-overview
last_reviewed: 2026-10-09
---

# Find the Content dashboards and pages

The Content section of the Portal is about what the databases hold: which retention settings apply, whether the document age collection succeeded, and how much content still lies inside its retention window. Two dashboards present this as charts and lists; one native page plots it against usage. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### Retention Overview

- **Where:** Content › Retention Overview, `{{idna_server}}/#/dashboard/10000041`.
- **What you get:** pie charts of the databases by retention mode and by collection status, the count of unique replica sets, document volume per retention mode and a database list; you can filter by Retention Mode and Collection Status and leave out databases that have no retention configuration. The picture is complete only when content retention settings are configured for the databases.
- **Fits questions like:** how many of our databases have a retention configuration? How are our documents spread across the retention modes? Which databases lack usable document age data?

### Retention Catalog

- **Where:** Content › Retention Catalog, `{{idna_server}}/#/dashboard/10000040`.
- **What you get:** the database-level retention list on its own: the databases with their content-retention summary. Unlike the list on Retention Overview, you can narrow it to a database or a replica ID, group it by replica set, and filter by Retention Mode and Collection Status.
- **Fits questions like:** what retention mode and collection status does this database have? How do the replicas of one application compare on retention? Which databases use a particular retention mode?

### Data Retention Analysis

- **Where:** Content › Data Retention Analysis, `{{idna_server}}/#/content_age_db_bubble`.
- **What you get:** a bubble chart of the focus databases that sets usage (user access days) against average document age, colored by the share of content still inside the retention window, while grey marks databases that have no retention configuration; it is the chart view of the data the two retention dashboards list. You can set the period the usage figures cover and decide whether each replica set appears with all its instances or with just one.
- **Fits questions like:** which unused databases have no content left within retention and could be deleted? Which unused databases must be archived rather than deleted? How does document age relate to usage across our databases?
