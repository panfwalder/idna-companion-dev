---
id: concepts/shipped-insight-catalog
title: Shipped insight catalog - every ruleset and insight iDNA ships
type: concept
status: verified
tags: [domain/idna, topic/modernization]
related:
  - concepts/insights-and-rulesets
  - howtos/find-code-patterns
  - concepts/notes-64bit-findings-taxonomy
  - concepts/nomad-web-mobile-findings-taxonomy
  - concepts/compatibility-ruleset-routing
  - schema/tables/design-insight
  - schema/tables/design-insight-ruleset
  - schema/tables/design-insight-ruleset-member
requires_capability: none
source: docs/reference/insight-catalog/shipped-insight-catalog.generated.json (ETL repo dda_ci/080-085 at 1c5d046, 2026-10-18; the 22 pattern corrections of ETL plan dda-ci-regex-fixes) + docs/reference/insight-catalog/insight-meanings.json (meanings derived from the patterns)
kbase_sources: [legacy-design-complexity-ruleset, portal-custom-insights-manage]
last_reviewed: 2026-10-18
---

# Shipped insight catalog

## What it is

The complete list of the code insights panagenda ships with iDNA Applications: 9 rulesets, 110 insights. Use it to answer "does iDNA already detect X in our code?" before suggesting a code search or tooling outside iDNA; [[howtos/find-code-patterns]] then shows where the results of an insight are. How rulesets, insights and patterns work is on [[concepts/insights-and-rulesets]].

**Version.** This is the shipped catalog as of the ETL reference data of 2026-10-18 (commit 1c5d046). An estate's live catalog is `cqx_data.design_insight` with `cqx_data.design_insight_ruleset`; it may lag or lead this page (older iDNA builds ship fewer insights, other severities or other patterns, and customers add their own). For one server, the live list wins. The 2026-10-18 reference data corrected the patterns of 22 detections: @DbExists Formula, the definite and possible dependencies on other databases, External C Calls, Hardcoded IP Addresses, ODBC Connectivity, Non-Domino @Commands, What's new in Notes 10.0.1 and 8.5.3, Addressbook related code, Calendar related code, Mail Send and its three Sign / Encrypt combinations, Working with files, Working with other programs, Java Imports, Special LotusScript Includes, Con: Code not supported on the web, Paper Printing and Pro: Javascript and HTML tags. On older builds these insights find less or match more noise than described here - for example What's new in Notes 10.0.1 and Calendar related code find only the calls without arguments, and External C Calls misses Declare Public / Private.

## How to read it

- **Severity** is the shipped default (Insignificant, Low, Medium, High, Very High, Exceptional).
- **What it flags** says what the insight's patterns look for, not what the code does; the matches need review. The patterns themselves are in Custom Insights › Manage.
- **Same detection as** marks an insight whose patterns are identical to an insight in another ruleset. Both report the same code: count within one ruleset, or deduplicate by code location - never add the two.
- The two compatibility rulesets (Notes 64bit Readiness, Nomad Web, iOS, Android Compatibility) are listed by name; their meaning and routing are on their own pages.

## Cloud Readiness (19)

What ties an application to its current environment: other databases, hard-coded names, addresses and IDs, external data interfaces, native calls, and methods of newer Notes releases. Earlier releases named it IBM Cloud Readiness. 13 of its 19 insights repeat a detection of another ruleset.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| @DbExists Formula | Exceptional | Tests whether another database exists - the code depends on a database outside the application. | Dependencies on other Databases: @DbExists Formula |
| Dependencies on other databases (definite) | High | Opens or looks up another database whose server or file name is given, at least in part, as a quoted literal - a dependency on a specific other application. | Dependencies on other Databases: Definite dependencies on other databases |
| Dependencies on other databases (possible) | Medium | Opens or looks up another database whose name is given by variables, expressions or \|...\| / {...} strings - a likely dependency; the target has to be read in the code. | Dependencies on other Databases: Possible dependencies on other databases |
| External C Calls | Very High | Declares functions in native libraries (Declare [Public / Private] Function / Sub ... Lib) - ties the code to the platform and to 32 / 64-bit. | Operating System Dependencies: External C Calls |
| Hardcoded Filenames | High | A hard-coded Notes database or template file name. | Specific Code Findings: Hardcoded Filenames |
| Hardcoded IP Addresses | Very High | A hard-coded IPv4 address directly after a quote, apostrophe, slash or backslash (string start, URL, UNC path) - the code depends on a specific host. | Specific Code Findings: Hardcoded IP Addresses |
| Hardcoded Replica IDs | High | A hard-coded replica ID - the code depends on one specific database. | Specific Code Findings: Hardcoded Replica IDs |
| Interfaces: DB2 | High | Code tied to Domino's DB2 storage integration. | Special Interfaces: Working with DB2 |
| Interfaces: Lotus Connector | Very High | Connections to external data systems through the Lotus Connectors (LC LSX). | Special Interfaces: Working with Lotus Connector |
| Interfaces: LotusScript Extensions (UseLSX) | Very High | Loads a LotusScript extension (LSX) - a native add-in that must be installed wherever the code runs. | Nomad Web, iOS, Android Compatibility: LotusScript Extensions (UseLSX); Special Interfaces: LotusScript Extensions (UseLSX) |
| Interfaces: ODBC Connectivity | Very High | Relational database access through ODBC. | Special Interfaces: ODBC Connectivity |
| Non-Domino @Commands | Very High | @DbCommand calls to a non-Domino data source (typically ODBC). | Special Interfaces: Non-Domino @Commands |
| Non-Notes @DBLookups/@DbColumns | Very High | Lookups into a non-Notes data source (typically ODBC). | Specific Code Findings: Non-Notes @DBLookups/@DbColumns |
| What's new in Notes 10.0 | Medium | Uses methods the insight's name assigns to Notes 10.0; the code needs a client and server of that release or later. |  |
| What's new in Notes 10.0.1 | Medium | Calls CreateJSONNavigator, a method the insight's name assigns to Notes 10.0.1; the code needs a client and server of that release or later. |  |
| What's new in Notes 8.0 | Medium | Uses methods the insight's name assigns to Notes 8.0. |  |
| What's new in Notes 8.5 | Medium | Uses a method the insight's name assigns to Notes 8.5. |  |
| What's new in Notes 8.5.3 | Medium | Uses a method the insight's name assigns to Notes 8.5.3. |  |
| What's new in Notes 9.0.1FP9 | Medium | Uses a method the insight's name assigns to Notes 9.0.1 FP9. |  |

## Dependencies on other Databases (3)

Code that reaches into other Notes databases.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| @DbExists Formula | Exceptional | Tests whether another database exists - the code depends on a database outside the application. | Cloud Readiness: @DbExists Formula |
| Definite dependencies on other databases | High | Opens or looks up another database whose server or file name is given, at least in part, as a quoted literal - a dependency on a specific other application. | Cloud Readiness: Dependencies on other databases (definite) |
| Possible dependencies on other databases | Medium | Opens or looks up another database whose name is given by variables, expressions or \|...\| / {...} strings - a likely dependency; the target has to be read in the code. | Cloud Readiness: Dependencies on other databases (possible) |

## Mail Dependencies (11)

Code that sends mail or depends on the Domino Directory, the calendar or the to-do features.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| Addressbook Fields | Very High | Reads mail routing data of people from the Domino Directory. |  |
| Addressbook related code | Very High | Reads the Domino Directory through its hidden lookup views. |  |
| Calendar related code | High | Drives the Notes calendar. |  |
| MIME related code | High | Works with MIME content - reads, builds or converts MIME entities. |  |
| Mail Recipient Fields | Medium | Works with mail recipient fields. |  |
| Mail Send | High | Sends, forwards or opens mail, or reads mail settings and addresses. |  |
| Mail Send and Encrypt | Very High | Mail sending and encryption in the same code block. |  |
| Mail Send and Sign | High | Mail sending and signing in the same code block. |  |
| Mail Send, Sign and Encrypt | Exceptional | Mail sending, signing and encryption in the same code block. |  |
| Todo / Tasks related code | High | Uses the mail file's to-do features. |  |
| Working with Document Links | High | Creates or handles document links. |  |

## Nomad Web, iOS, Android Compatibility (16)

A compatibility target: what each insight means, and how to route the work, is on [[concepts/nomad-web-mobile-findings-taxonomy]] and [[concepts/compatibility-ruleset-routing]]. Insights (shipped default severity):

- ActivateApp (Very High)
- Chdir (Medium)
- CreateObject (Very High)
- Date/Time (Medium)
- Declare function (Very High)
- Edit/Find (Nomad Android Only) (Medium)
- Get/Set FileAttribute (Medium)
- GetObject (Very High)
- LS2J (unsupported in Nomad) (Very High)
- Lock/Unlock (Very High)
- LotusScript Extensions (UseLSX) (Medium)
- NotesHTTPRequest (Nomad Web Only) (Medium)
- NotesHTTPRequest unsupported methods (Nomad Web Only) (High)
- OSLoadProgram (Nomad iOS Only) (Medium)
- SendKeys (Medium)
- Shell Commands (Medium)

## Notes 64bit Readiness (17)

A compatibility target: what each insight means, and how to route the work, is on [[concepts/notes-64bit-findings-taxonomy]] and [[concepts/compatibility-ruleset-routing]]. Insights (shipped default severity):

- Contains 'win' or 'windows' (Medium)
- Contains /32 without /64 (Medium)
- Contains OS Architecture or Name (Medium)
- GetAllEntriesByKey (mixed 32/64 compile-runtime risk) (Low)
- Missing Java EE/CORBA classes (Java 17 / Notes 14) (High)
- Missing V12 Java classes (High)
- Missing V14 Java classes (High)
- References Program Files (x86) (Medium)
- References platform string 'Windows/32' (High)
- Use of CreateObject (Medium)
- Use of Declare Function accepts or returns long (Medium)
- Use of Declare Function that returns long (Very High)
- Use of EmbedObject (Medium)
- Use of RegQueryValue (Medium)
- Use of isDefined (High)
- Use of loadlibrary (Medium)
- Use of type block with long/string (Medium)

## Operating System Dependencies (6)

Code tied to the operating system of the client or server: files, directories, other programs, OLE, native calls, the system clock.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| External C Calls | Very High | Declares functions in native libraries (Declare [Public / Private] Function / Sub ... Lib) - ties the code to the platform and to 32 / 64-bit. | Cloud Readiness: External C Calls |
| Working with OLE | Very High | OLE / COM automation of Windows programs. |  |
| Working with directories | High | Creates, changes or removes file-system directories. |  |
| Working with files | High | Reads or writes local files. |  |
| Working with other programs | High | Starts or drives other programs, or reads the operating-system environment or registry. |  |
| Working with system date/time (or old code) | Exceptional | Sets the system date or time - or old code that assigns variables named Date or Time. |  |

## Special Interfaces (8)

Connections to systems outside Notes: ODBC, DB2, the Lotus Connectors, LotusScript extensions, Sametime instant messaging, command shells and scripts, custom data.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| Instant Messaging | Medium | Sametime instant-messaging integration. |  |
| LotusScript Extensions (UseLSX) | Very High | Loads a LotusScript extension (LSX) - a native add-in that must be installed wherever the code runs. | Cloud Readiness: Interfaces: LotusScript Extensions (UseLSX); Nomad Web, iOS, Android Compatibility: LotusScript Extensions (UseLSX) |
| Non-Domino @Commands | Very High | @DbCommand calls to a non-Domino data source (typically ODBC). | Cloud Readiness: Non-Domino @Commands |
| ODBC Connectivity | Very High | Relational database access through ODBC. | Cloud Readiness: Interfaces: ODBC Connectivity |
| Working with DB2 | High | Code tied to Domino's DB2 storage integration. | Cloud Readiness: Interfaces: DB2 |
| Working with DOS and similar | High | Calls Windows command shells or scripts. |  |
| Working with Lotus Connector | Very High | Connections to external data systems through the Lotus Connectors (LC LSX). | Cloud Readiness: Interfaces: Lotus Connector |
| Working with custom data | High | Reads or writes item custom data. |  |

## Specific Code Findings (19)

Individual code patterns worth knowing about: hard-coded values, locking, computed code, includes, Option Base 1, old formula functions, and informational counts (script libraries, rich text, XML, Java imports).

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| Agent Synchronization (CreateLock) | High | Agent synchronization with LotusScript locks. |  |
| Developer Warning: Option Base 1 | Exceptional | Arrays declared without an explicit lower bound start at 1 instead of 0 in that module - code moved into or out of it behaves differently. |  |
| Execution of computed LotusScript Code | Medium | Runs LotusScript text built at runtime, which static analysis cannot see. |  |
| Hardcoded Filenames | High | A hard-coded Notes database or template file name. | Cloud Readiness: Hardcoded Filenames |
| Hardcoded IP Addresses | Very High | A hard-coded IPv4 address directly after a quote, apostrophe, slash or backslash (string start, URL, UNC path) - the code depends on a specific host. | Cloud Readiness: Hardcoded IP Addresses |
| Hardcoded Internet Email Addresses | Very High | A hard-coded internet e-mail address. |  |
| Hardcoded Replica IDs | High | A hard-coded replica ID - the code depends on one specific database. | Cloud Readiness: Hardcoded Replica IDs |
| Java Imports | Insignificant | Imports Java packages other than lotus.domino - standard Java and other Domino / XPages packages count too. |  |
| Locking of Documents and Design Elements | High | Locks documents or design elements. |  |
| Non-Notes @DBLookups/@DbColumns | Very High | Lookups into a non-Notes data source (typically ODBC). | Cloud Readiness: Non-Notes @DBLookups/@DbColumns |
| Old code | Insignificant | Formula functions kept for backward compatibility. |  |
| Special LotusScript Includes | Medium | Includes LotusScript source files from the file system at compile time. |  |
| String Conversion | Insignificant | Converts strings between character widths or cases. |  |
| Use of Script Libraries | Insignificant | Loads LotusScript script libraries. |  |
| Working with NotesInternational | Insignificant | Reads the operating system's regional settings (date, time, number formats) through NotesInternational. |  |
| Working with Rich Text | Insignificant | Processes rich text. |  |
| Working with XML | Insignificant | Processes XML or DXL. |  |
| Working with notes.ini | Low | Reads or writes notes.ini variables. |  |
| beep | Insignificant | Sounds the client's beep. |  |

## Web Design (11)

Readiness of the design for browser access. Name prefixes: "Con:" blocks the web, "Pro:" is web-ready, "Fun:" is informational; insights without a prefix are review items.

| Insight | Severity | What it flags | Same detection as |
|---|---|---|---|
| Con: Code interacting with Notes client UI | Very High | Needs the Notes client user interface; does not run in a browser. |  |
| Con: Code not supported on the web | Very High | Formula code that does not run on the web. |  |
| Con: LS Buttons and ActionHotSpots | Very High | LotusScript in button and action-hotspot click events - it does not run on the web. |  |
| Fun: LS Buttons and ActionHotSpots without code | Low | Button and action-hotspot click events without LotusScript code - informational. |  |
| Internet and Notes Protocols | High | Uses or links to internet or Notes URLs. |  |
| Occurrences of other product names | Insignificant | Mentions other products. |  |
| Paper Printing | Medium | Prints from the Notes client or from JavaScript (window.print()). |  |
| Pro: Agent Redirection | Insignificant | Web agents that redirect the browser - a web-ready pattern. |  |
| Pro: Domino Commands | Insignificant | @DbCommand("Domino"; ...) web view navigation - a web-ready pattern. |  |
| Pro: Formulas | Insignificant | Formula and LotusScript code written for the web - a web-ready pattern. |  |
| Pro: Javascript and HTML tags | Insignificant | Code that names JavaScript or contains HTML / script tags - a web-ready pattern. |  |
