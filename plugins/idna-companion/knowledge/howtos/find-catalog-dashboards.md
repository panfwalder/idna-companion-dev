---
id: howtos/find-catalog-dashboards
title: Find the Catalog dashboards and pages
type: howto
status: verified
tags: [domain/idna, topic/dashboards, audience/customer, audience/consultant, topic/consolidation]
related:
  - concepts/about-replica-sets
  - concepts/database-instance
  - concepts/replica-id
requires_capability: none
source: Portal and Metabase inventory, iDNA 3.5.0, 2026-09-08
kbase_sources:
  - metabase-10000006-replica-sets-filterable
  - metabase-10000017-consolidation-potential
  - metabase-10000018-database-acl-search
  - metabase-10000039-profile-documents
  - portal-consolidation-potential
  - portal-database-acl-search
  - portal-databases
  - portal-discrepancies
  - portal-instance-details
  - portal-profile-documents
  - portal-replica-sets
  - portal-servers
last_reviewed: 2026-10-09
---

# Find the Catalog dashboards and pages

The Catalog section of the Portal is the inventory of the estate: the database instances on each server, the replica sets they form, the servers iDNA collects data from, and a details page per instance. It also shows replica sets whose instances differ, unused applications that are candidates for consolidation, the entries of database ACLs and the content of profile documents; a filterable replica-set list sits outside the Portal menu, in Metabase. Names and navigation paths are as seen in iDNA 3.5.0; older versions may differ. In links, replace `{{idna_server}}` with `dashboard_base_url` from `get_idna_session` when it is present; otherwise show the path with the placeholder and say that it stands for the customer's iDNA host.

### Databases

- **Where:** Catalog › Databases, `{{idna_server}}/#/catalog`.
- **What you get:** a grid with one row per database instance, that is per database file on a given server, and switchable column sets such as Usage, Scores, Source Code, Content Age, Insights, Notes 64bit Readiness, Nomad Web/iOS/Android Compatibility, Cloud Readiness and Custom DB Instance properties. You can search across all fields, filter per column and export to CSV or Excel; a database's title opens its Application Details.
- **Fits questions like:** where can I see each database instance with its server, file name and template? Is there a per-database list of 64-bit or Nomad readiness? Can I export the database inventory to Excel?

### Application Details (per instance)

- **Where:** the per-instance details page, `{{idna_server}}/#/replicadetails/<database_id>`.
- **Opened from:** Catalog › Databases, click a database's title.
- **What you get:** the detailed record of a single database instance on its server: key figures for active users, web or client usage, usage and design rank, design complexity and focus-DB status, followed by tabs for the Application properties (Database, Replication, Access Control List), Content Analysis, Content Age and Custom DB Instance properties. A banner warns when design components could not be collected, and the page lets you queue a design re-scan or start a content scan of the instance.
- **Fits questions like:** what does iDNA know about this one database instance? Why is the design data for this database incomplete? How are replication and the ACL set up for this instance?

### Replica Sets

- **Where:** Catalog › Replica Sets, `{{idna_server}}/#/masterlistreplicasets`.
- **What you get:** the application list, one row per replica set (the instances with the same replica ID), with remediation classification, last access, usage categorization, active users and design complexity among the default columns; search, column filters, column sets and CSV or Excel export work as on Catalog › Databases, and a replica set's title opens its Application Insights dashboard. This is the native Portal grid; Replica Sets (Filterable) is a separate Metabase dashboard with a similar list.
- **Fits questions like:** where is the list of our applications rather than individual database files? Which applications have which remediation classification? Where do I open the details of one application across all its replicas?

### Replica Sets (Filterable)

- **Where:** `{{idna_server}}/metabase/dashboard/10000006`. Not in the Portal menu; it opens in Metabase and depends on your Metabase access.
- **What you get:** a single Metabase table of applications with key columns on usage, design complexity and remediation classification, which you can filter by Database Type, Remediation Class, User Activity, Last Used and Design Complexity and explore further in Metabase. It is distinct from Catalog › Replica Sets, the native Portal grid with column sets and search.
- **Fits questions like:** is there a replica-set list in Metabase that I can filter by remediation class and design complexity? Which complex applications have not been used recently?

### Servers

- **Where:** Catalog › Servers, `{{idna_server}}/#/serverlist`.
- **What you get:** the state of data collection on each Domino server: whether iDNA can reach the server and, for each data source it reads (such as Catalog, Domlog and Data Files), the access status and the last collection date. A status icon and an Issues column flag servers with failing sources, and outdated collection dates stand out by color.
- **Fits questions like:** which servers have data collection problems? When did iDNA last collect data from each server? Are catalog and web log collection working on all servers?

### Discrepancies

- **Where:** Catalog › Discrepancies, `{{idna_server}}/#/discrepancies`.
- **What you get:** the replica sets whose instances do not match each other, counted in five tiles (Document Count, Design Age, Template Inheritance, Template Claims, Code Base) that act as combinable filters on a list showing which kinds of difference apply to each replica set. Not every difference is a fault; some are by design.
- **Fits questions like:** which applications have replicas that are out of sync? Do the replicas of an application inherit from different templates? Where do replicas of the same application carry different code?

### Consolidation Potential

- **Where:** Catalog › Consolidation Potential, `{{idna_server}}/#/dashboard/10000017`.
- **What you get:** the replica sets that saw little or no use over the recorded history and none on any instance in the last 90 days, the likeliest applications to sunset or consolidate; you can filter the list by Lifecycle Vector. The dashboard advises waiting until usage has been collected for three months or more, since some applications are used only at certain times of the year.
- **Fits questions like:** which applications does nobody use any more? Is there a list of consolidation or sunset candidates? Which unused applications fall into a particular lifecycle vector?

### Database ACL Search

- **Where:** Catalog › Database ACL Search, `{{idna_server}}/#/dashboard/10000018`.
- **What you get:** a table of the ACL entries of each database as they stand in the ACL, with groups not expanded into their members; you can filter by User Name, Access Level, Server Name, Database and Is FocusDB.
- **Fits questions like:** which databases grant Manager access, and to which ACL entries? How are the ACLs of the focus databases on one server set up? In which database ACLs does a particular group appear?

### Profile Documents

- **Where:** Catalog › Profile Documents, `{{idna_server}}/#/dashboard/10000039`.
- **What you get:** a search over the collected profile documents of the databases, by field value, field name, document name or all of them at once, optionally narrowed to one database (filters Search Field Value, Search Field Name, Search Doc Name, Search All, Database). Profile document collection is off by default and has to be enabled; where a database holds several profile documents with the same name, only the newest appears.
- **Fits questions like:** which databases store a particular value in their profile documents? What settings does this database keep in its profile documents? Which databases use a profile document with a given name?
