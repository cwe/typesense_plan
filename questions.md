# All questions

Generated from the chunk files (`### Q<chunk>.<n> — title` headings). Answer in [decisions.md](decisions.md). The tasks in [tasks/](tasks/README.md) cite these IDs.

## Chunk 0 — Baseline, scope and ground rules

- **Q0.1** Keep the current code as the base, or rewrite from the plan? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.2** What is the goal, and how is success measured? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.3** When exact parity is impossible, decline or approximate? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.4** Are changes to ResourceSpace core acceptable, and of what kind? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.5** What are the target environments? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.6** What is Typesense-only mode for? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.7** What scale and usage should the design assume? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.8** Working practices ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.9** Where do documents live, and which survive? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.10** In what order are the chunks tackled, and what is the first milestone? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.11** Confirm what is out of scope ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.12** What to do with the three unmerged pieces now ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.13** Keep the pipeline design for the rebuild? ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))
- **Q0.14** Approve the module layout ([00-baseline-and-ground-rules.md](chunks/00-baseline-and-ground-rules.md))

## Chunk 1 — Index schema and document model

- **Q1.1** Index only flagged fields, or every field's values? ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.2** Which representations per field type? ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.3** Fragments: keep, and should they be free-text searchable? ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.4** Value cleaning at index time ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.5** Stemming and tokenisation ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.6** Which extra fields go in now? ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.7** Resource-type names and contributor usernames as searchable words ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.8** Schema evolution ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.9** Typesense version and features ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.10** Document identity ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.11** Size budget ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.12** Complete-indexed fields ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.13** The 500-character cap ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.14** The registry as the extension contract ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))
- **Q1.15** The `0` sentinel in `nodes[]` and `populated_field_ids[]` ([01-index-schema-and-document-model.md](chunks/01-index-schema-and-document-model.md))

## Chunk 2 — Free-text keyword semantics

- **Q2.1** Target: identical sets to core, or sensible semantics? ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.2** Typo tolerance ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.3** Prefix and wildcards ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.4** Stop words ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.5** Phrases when stemming is on ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.6** Negated phrase `-"…"` ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.7** OR-groups `a;b` ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.8** Punctuation and commas ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.9** Type-name and contributor-name words ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.10** Related keywords ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.11** Bare years and numbers ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.12** Unknown words and suggestions ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.13** Which keyword quirks to raise upstream rather than copy ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.14** Pre-stem with ResourceSpace's own stemmer on both sides? ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.15** The full-text box: permanent veto? ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.16** Short wildcards ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.17** A wildcard inside a word ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))
- **Q2.18** Accent folding ([02-free-text-keyword-semantics.md](chunks/02-free-text-keyword-semantics.md))

## Chunk 3 — Field, date, numeric and node searches

- **Q3.1** Confirm the "decline rather than return everything" rule ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.2** Quoted field searches and search-bar text ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.3** `basicyear:` / `basicmonth:` / `basicday:` ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.4** Date-range semantics (open decision C) ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.5** Date-range fields (start/end pairs) ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.6** Field-specific searches on non-indexed fields ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.7** Numeric fields ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.8** Negation of field terms, dates, numbers and nodes inside OR groups ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.9** Node limits and toggles ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.10** Wildcard candidate caps on field searches ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.11** Field name lookup ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.12** Serve unresolved option text through the option names (`_ss`) ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.13** `basicyear` / `basicmonth` / `basicday`: integer components or `_q` tokens? ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.14** Two values and `;` on one date field ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.15** `$ignore_filters` ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.16** Non-indexed date and numeric fields as a slice of Q1.1 ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.17** The exact rules of the field-name match ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))
- **Q3.18** Is a backtick-quoted multi-word bare-colon value a phrase or an AND? ([03-field-date-numeric-and-node-searches.md](chunks/03-field-date-numeric-and-node-searches.md))

## Chunk 4 — Special searches

- **Q4.1** Which vetoed specials to serve, in what order? ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.2** Index fields for them: one batch or per special? ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.3** Argument parsing: replicate core exactly, or parse strictly and veto? ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.4** `restypes` on specials ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.5** Workflow states per special ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.6** Smart collections ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.7** `!unused` (resources in no collection) ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.8** `!empty<field>` ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.9** Unknown or misspelt specials ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.10** The form's `!properties` + `!list` collision ([04-special-searches.md](chunks/04-special-searches.md))
- **Q4.11** `!properties`: the key set and the unknown-key rule ([04-special-searches.md](chunks/04-special-searches.md))

## Chunk 5 — Collections

- **Q5.1** Access gate: core helper (G1) or plugin copy (G2)? ([05-collections.md](chunks/05-collections.md))
- **Q5.2** Hydrate the collection columns ([05-collections.md](chunks/05-collections.md))
- **Q5.3** Relevance order inside a collection ([05-collections.md](chunks/05-collections.md))
- **Q5.4** Upload-share sessions ([05-collections.md](chunks/05-collections.md))
- **Q5.5** Is searching *for* collections in scope now? ([05-collections.md](chunks/05-collections.md))
- **Q5.6** Freshness mechanism ([05-collections.md](chunks/05-collections.md))
- **Q5.7** Which collection kinds are indexed ([05-collections.md](chunks/05-collections.md))
- **Q5.8** `J` with `j*` and no `-j` ([05-collections.md](chunks/05-collections.md))
- **Q5.9** Reordering from the search page ([05-collections.md](chunks/05-collections.md))
- **Q5.10** Large collections and `fetchrows = -1` ([05-collections.md](chunks/05-collections.md))
- **Q5.11** `$smartsearch` ([05-collections.md](chunks/05-collections.md))

## Chunk 6 — Access, permissions and read-time validation

- **Q6.1** Validation strictness (open decision D1) ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.2** Gate uses that must not depend on the index ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.3** The small fixes ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.4** Editable-only searches ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.5** Typesense-only mode and access ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.6** `$heightmin` ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.7** NOT over hidden and inactive fields ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.8** Share keys and external access ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.9** Config parity for access options ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))
- **Q6.10** Validation and the sync plan ([06-access-permissions-and-validation.md](chunks/06-access-permissions-and-validation.md))

## Chunk 7 — Sorting, relevance and result windows

- **Q7.1** The relevance model (open decision D2) ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.2** Which further sorts to serve ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.3** The row limit and next/previous ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.4** Exact-ref-first for bare numbers ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.5** Page offset with collections above results ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.6** Row shape ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.7** Deep paging cost ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.8** Sort direction quirks ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))
- **Q7.9** Ordering by a metadata field that is not a join column ([07-sorting-relevance-and-result-windows.md](chunks/07-sorting-relevance-and-result-windows.md))

## Chunk 8 — Index sync and freshness

- **Q8.1** Re-confirm the thirteen taken decisions (P7–P17, P24) ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.2** Validation strictness ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.3** Group override ordering ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.4** Core hooks (Phase 2) ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.5** Phase 0 scope and order ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.6** Outage behaviour ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.7** Freshness target ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.8** Grants and memberships ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.9** Hit-count freshness ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.10** Rebuild window and capacity ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.11** Topology ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.12** Observability ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))
- **Q8.13** Which register rows must be immediate in M3? ([08-index-sync-and-freshness.md](chunks/08-index-sync-and-freshness.md))

## Chunk 9 — Reindexer, configuration and operations

- **Q9.1** Configuration ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.2** Secrets ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.3** Rebuild and resync from the UI ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.4** Performance work: now or after sync ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.5** Logging ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.6** Health, warnings and the badge ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.7** Deployment procedure and versioning ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.8** Several RS instances on one Typesense server ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.9** Test hosts ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))
- **Q9.10** The official Typesense PHP client, or a custom client? ([09-reindexer-configuration-and-operations.md](chunks/09-reindexer-configuration-and-operations.md))

## Chunk 10 — Testing and verification

- **Q10.1** Where the tests live ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.2** The verification method ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.3** Acceptance criteria ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.4** Test data ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.5** Credentials ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.6** Freshness, outage and rebuild tests ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.7** Performance tests ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.8** The live user matrix ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.9** Cadence ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.10** Core bug batteries ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.11** Reuse the existing harness code in the rebuild? ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.12** Battery naming tied to the catalogue ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.13** Fixture additions per package ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.14** Rows the API cannot send ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))
- **Q10.15** The combinations gate per package or per milestone ([10-testing-and-verification.md](chunks/10-testing-and-verification.md))

## Chunk 11 — Core changes and upstream

- **Q11.1** The policy, by category ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
- **Q11.2** The existing hook-return patch ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
- **Q11.3** Each core bug: raise, copy, or ignore ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
- **Q11.4** Tracking upstream ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
- **Q11.5** A cleaner provider contract ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
- **Q11.6** Is this plugin meant to go upstream? ([11-core-changes-and-upstream.md](chunks/11-core-changes-and-upstream.md))
