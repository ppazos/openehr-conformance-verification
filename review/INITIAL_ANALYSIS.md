# openEHR Conformance Verification Framework: Initial Analysis

| Item      | Value |
|-----------|-------|
| Date      | 2026-11-19 |
| Scope     | `conformance_testing_specification/*.md`, `conformance_testing_specification/datasets/**`, `conformance_test_suites/**` (Groovy/Spock REST tests), and the framework design PDFs (checked only for consistency with the test specs) |
| Goal      | Find inconsistencies, incomplete parts and likely errors that would get in the way of using this material for **conformance verification and software certification through the openEHR REST API** |
| Method    | Read every spec document by hand. Ran scripts over all `expected` tables (looking for typos in the result values, results that contradict their own "constraints violated" column, and duplicate rows), recomputed every date/time *validity-kind* table from its inputs, checked that every JSON/XML/OPT dataset is well-formed, and checked the Groovy tests against the spec |

> Statements about the openEHR RM/AOM/ITS-REST come from the published specifications as I know them. Items marked **(verify)** should be checked against the exact release the framework will target (for example RM 1.0.4 vs 1.1.0, ITS-REST 1.0.2 vs 1.0.3 vs development).

## Severity legend

| Level | Meaning |
|-------|---------|
| **S1 – Blocker** | Prevents objective, reproducible certification use (no defined pass/fail, untestable through REST, contradicts the openEHR specs) |
| **S2 – Major** | A wrong expected result, a missing test area, or a test case that cannot be executed as written |
| **S3 – Minor** | Editorial issues: typos, numbering, broken formatting, naming. Some of these still matter if the tables are parsed automatically |

---

## 0. Executive summary

1. **No binding to the REST API (S1).** Every test case is written against abstract Service Model (SM) operations ("invoke has COMPOSITION service"), and expected results are vague ("result should be negative, related to …"). There is no SM-to-REST mapping, no HTTP method/endpoint, and no expected HTTP status code, header or response body. Several SM operations used by the suites have no ITS-REST equivalent at all (see §1.1). Two certifiers could get different results for the same product.
2. **Outcomes that are not normative (S1).** Many expected results rest on the author's opinion ("IMO should fail"), on open SEC discussions, or are left open ("could be an error or an empty result", "depending on the implementation"). A certification test needs a single expected outcome, or an outcome tied to an option the vendor declares in the Conformance Statement. Today the Conformance Statement is not wired into any test case (§1.3).
3. **Preconditions that clash with the RM (S1/S2).** Several cases need "an EHR with no CONTRIBUTIONs", but creating an EHR creates `VERSION<EHR_STATUS>`, and `VERSION.contribution` is mandatory, so the server must already hold a CONTRIBUTION. The "list contributions returns one item" checks depend on this as well (§1.4).
4. **Wrong expected results in the data-validation tables (S2).** Examples: `DV_BOOLEAN` with `true_valid=false` is expected to be *accepted*; in `DV_DATE_TIME` range tests a 2021 value is checked against a range on 1900-03-13; some `DV_INTERVAL<DV_TIME>` rows contradict each other; several "constraints violated" lists are wrong. There are also contradictory rules on comparing partial dates (§5, §6).
5. **Missing coverage (S2).** There is no AQL/QUERY test suite, although AQL datasets and expected results are already in `datasets/query`. ADL, Terminology and Demographic suites are absent or placeholders. Structural validation stops at section 6.5 (sections 6.6–7.11 are empty headings). Optimistic-locking/version-conflict, `is_modifiable`/`is_queryable` effects, and cross-EHR access are never tested (§1.5).
6. **The test implementation does not assert what it seems to (S1).** In Spock, comparisons inside `when:` blocks, and inside `if` blocks within `then:`, are not assertions. Most `client.lastResponseCode == …` checks are therefore never evaluated (§8).
7. **Dataset problems (S2).** A malformed OPT sits in `valid_templates`, compositions and contributions are stored under template folders, and nothing maps dataset files to spec rows (§9).

---

## 1. Cross-cutting / framework-level findings

### 1.1 No SM → ITS-REST binding (S1)

Verification is meant to run over REST, but every suite describes abstract SM calls. Needed: a normative table per suite with *SM operation → HTTP method + path + required headers (`Prefer`, `If-Match`, `openEHR-VERSION`, `openEHR-AUDIT_DETAILS`, `Content-Type`/`Accept`) → expected status codes, plus the error-body expectations*.

SM operations used by the suites that have **no direct ITS-REST 1.0.x endpoint (verify)**:

| Suite | Operation used in spec | REST situation |
|-------|------------------------|----------------|
| KNOWLEDGE C.1 | validate OPT | no validate endpoint |
| KNOWLEDGE C.2.d / C.3.d / C.5 | versioned ADL 1.4 OPT, delete OPT | ADL 1.4 template endpoints have no version parameter and no DELETE (the `version` parameter exists only for ADL2). The link in KNOWLEDGE C.2.c (`definitions-adl-1.4-template`) does not support the claim that "this is the solution on the openEHR REST API" for 1.4 |
| EHR B.2 | has EHR | only GET (no HEAD defined) |
| EHR C.2–C.5 | set/clear queryable/modifiable | only by PUT of a full EHR_STATUS with `If-Match` |
| EHR_COMPOSITION B.1 | has COMPOSITION | none |
| EHR_CONTRIBUTION F, G | list CONTRIBUTIONs, has CONTRIBUTION | none (only `GET /ehr/{id}/contribution/{uid}`) |
| DIRECTORY C, D, J, L | has directory, has path, has directory version, get versioned directory | none (only GET/POST/PUT/DELETE on `/ehr/{id}/directory`) |

Each of these needs either a documented REST substitute (for example "has X" = `GET` → 200 vs 404) or a statement that the case is out of scope for REST-based verification.

### 1.2 Expected results are not machine-checkable (S1)

The flows say "result should be positive / negative, related to …" or "return information about the errors". For certification each step needs:
- the exact expected status code (or a closed set of them),
- which response fields must be present or equal (for example `uid` format `<uuid>::<system_id>::<n>`),
- what counts as "content check" (B.2/B.3/B.4 in EHR_COMPOSITION). The server is allowed to add or alter `uid`, `feeder_audit`, terminology rubrics, date/time formatting (basic vs extended), decimal formatting and element order. A canonical comparison rule is needed (for example RM-level semantic equality, ignoring server-assigned fields).

### 1.3 Non-normative or open-ended outcomes; Conformance Statement not wired in (S1)

Examples of expected results that are opinions, or depend on the implementation:
- `IMO should fail` rows in DV_INTERVAL tables (DATATYPES 3.9.1, 3.10.1, 3.11.1, 3.12.1), backed only by discourse threads.
- Partial date/time comparability (3.10.1, 3.11.1, 4.4.3 note) is an opinion, and section 4.5.3 contradicts it (§6.3).
- Persistent-composition uniqueness (EHR_COMPOSITION B.6.c, EHR_CONTRIBUTION C.9, B.2.2 B) is acknowledged as "under debate".
- Rejecting `incomplete` lifecycle for EHR_STATUS (CONTRIBUTION B.3.1) depends on open issue SPECPR-368.
- Template-id mismatch on update (EHR_COMPOSITION B.7.d), `creation` + `lifecycle_state=deleted` rejection (CONTRIBUTION B.2.2): no normative reference is given.
- DIRECTORY F.1, G.1, G.2, L.1 and KNOWLEDGE C.2.c/C.2.d say "empty result **or** error" or "depending on the implementation".
- DV_DURATION magnitude calculation (4.2.3) is left to the SUT.

The design document (§1.4 of *Conformance Framework Design 2023-11-21*) says suites are "adapted" per vendor based on the Conformance Statement. For certification this needs to be **formalised**: define named *conformance options/profiles* (versioning scheme, persistent-composition policy, delete behaviour, template versioning, supported formats, RM version, …). Each affected test case should then state its expected result per option. Otherwise the results are not comparable across vendors, and ad-hoc "adaptation" is hard to audit.

Also note: the 2023-11-08 revision of the design document says it *"removed software certification from scope"*. That conflicts with this repository's stated certification goal and should be reconciled.

### 1.4 Preconditions inconsistent with the RM / with each other (S1/S2)

- **"EHR with no CONTRIBUTIONs"**: EHR creation creates `VERSIONED_EHR_STATUS` v1 (and EHR_ACCESS), and `ORIGINAL_VERSION.contribution` is mandatory. So a newly created EHR already has at least one CONTRIBUTION in a conformant server. Affected: EHR_COMPOSITION B.1.b, B.2.b, B.3.c ("has no CONTRIBUTIONS"); EHR_CONTRIBUTION F.2 (expects an empty list), G.2, H.2; and F.1/F.4/F.5, which expect a list "with one item". Suggest rewording to "no COMPOSITIONs", and counting contributions relative to a baseline read after EHR creation.
- **"The server should be empty (no EHRs, no commits, no OPTs)"** appears as a precondition almost everywhere. The REST API has no reset/truncate operation, and no cleanup is specified (the README mentions a CLEAN phase, but no test case defines one). The KNOWLEDGE "implementation note" (one PRE/POST per dataset item) cannot be met for C.2.a, whose postcondition loads an OPT. Needed: either a documented vendor-provided reset hook (as part of the conformance setup), or rewriting tests to be state-independent (unique template ids/subject ids, relative counts).
- **"random ehr_id / version uid"** in "doesn't exist" cases must be *well-formed*, otherwise a correct server returns 400 rather than 404 (the Groovy tests use `'non-existing-id'`, `'xxxxxxxx'`, `'xxx.yyy.v1'`). Malformed identifiers should be a separate negative test with their own expected code.
- **Time-based flows** (EHR_COMPOSITION B.3.a/B.3.e, DIRECTORY G.*) use "current time" and "t0/t1", which are client times. Client/server clock skew and sub-second commits make these flaky. Use `commit_audit.time_committed` returned by the server, and specify the required time precision.
- COMPOSITION category vs context: RM invariant `Is_persistent_validity: is_persistent implies context = Void`. The structural tables in COMPOSITION_VALIDATION_STRUCTURE §2 combine "context present" with no stated category, so for a persistent composition the expected "accepted" would be wrong. Fix the category to `event`, or add the persistent variants.

### 1.5 Coverage gaps (S2)

- **AQL / QUERY**: no spec document, although `datasets/query` has queries in groups A–D, data loads and expected results (empty/loaded DB), and AQL is central to CDR conformance. Not listed among the suites in README or the design document.
- **KNOWLEDGE**: ADL 1.4/2 and Terminology are placeholders. OPT 2 is mentioned, but every dataset is OPT 1.4.
- **Demographic** (`I_DEMOGRAPHIC*`): no suite, yet the Groovy project carries demographic resources and tests.
- **Versioning semantics missing from every suite**:
  - update/delete with a `preceding_version_uid` that is not the latest (REST: `412 Precondition Failed`);
  - update/delete of an already deleted composition;
  - commit to an EHR with `is_modifiable=false`, and the effect of `is_queryable=false` on AQL (this is the only observable effect of C.2–C.5 in the EHR suite);
  - accessing a composition/contribution through the wrong `ehr_id`;
  - EHR_STATUS versioning (`/ehr_status` GET at time / at version, versioned EHR_STATUS);
  - revision history (`/versioned_composition/{id}/revision_history`), get version at time / by version id;
  - `change_type` and `lifecycle_state` actually stored and returned (postconditions mention them, but no flow verifies them);
  - `openEHR-AUDIT_DETAILS` / committer handling;
  - `IMPORTED_VERSION`, `attestation`.
- **EHR B.1**: invalid data sets are listed (missing/empty `is_queryable`…), but no flow uses them. There is no "create EHR invalid EHR_STATUS" case, no "get EHR by subject" when two namespaces share the same id, and no `PUT /ehr/{ehr_id}` with a client-chosen id vs `POST`.
- **Representations**: no tests for canonical JSON vs XML vs flat/structured formats, `Accept` negotiation, `Prefer: return=minimal|representation`, or `Location`/`ETag` headers.
- **RM 1.1.0** (verify): `EHR.folders` (multiple folders), `DV_SCALE`, and the optional DV_IDENTIFIER attributes are not reflected consistently. Coverage of RM-version-dependent behaviour should be tied to the declared RM version.

### 1.6 Traceability and versioning of assets (S2)

- Test case IDs are not globally unique. `B.1.a` exists in both EHR and EHR_COMPOSITION, and the Groovy tests are named `B.3.a`, `B.4.a`, `B.6.a`… without the suite. Use `<SUITE>-<case>-<dataset>` IDs (for example `EHR.B.1.a#ds12`).
- Nothing maps dataset files to spec rows (for example the 32 EHR_STATUS combinations vs 8 files in `datasets/ehr/valid`).
- Asset versioning: design §1.3.7 requires strict versioning, but versions differ per document (README 1.0/2021, EHR 0.4.2/2023, STRUCTURE 0.2/2021, …), there is no release tag, and datasets are unversioned.
- Terminology clash: the design document uses "accepted/rejected" for a *test result* (expected met / not met), and the specs use "accepted/rejected" for *what the server did with the data*. A test row with expected `rejected` that "passes" would be reported as "accepted". Use pass/fail for test results.
- No LICENSE file (relevant if the material is meant to become an openEHR asset).

---

## 2. README.md

- S2: README index and the design document list 5 suites plus validation. QUERY is missing (§1.5).
- S2: `COMPOSIITON_VALIDATION_DATATYPES_INTERVAL_DATE.md` (misspelled name) is not linked anywhere and is a **diverging copy** of DATATYPES §3.10. For example, at line 19 the orphan has `2021 | 2022-10`, while DATATYPES line 944 has `2021 | 2021-10`, and the "TBD" text and the 3.10.3 title also differ. Remove it or merge it.
- S2: The "Composition Validation" note says the component *excludes* schema (syntactic) validation, but STRUCTURE §3/§5 and DATATYPES include many "RM/Schema mandatory" rejection rows (for example OBSERVATION.data absent). Either drop those rows or state that they are syntactic checks run through the commit endpoints.
- S3: Precondition 2 (an auth mechanism) is not tied to any test. Auth is out of openEHR scope, so it should be a setup requirement, not a precondition.
- S3: TOC-generator comments in every file point to `../ehrbase/doc/conformance_testing` (left over from the original location).

## 3. KNOWLEDGE.md

- S1: Validate, versioned 1.4 upload/retrieve, and delete cannot be done through ITS-REST 1.0.x for ADL 1.4 (§1.1) **(verify)**. C.1.a's fallback ("upload then delete") depends on a delete that does not exist.
- S1: C.2.c and C.2.d are mutually exclusive, depending on a server capability, but nothing selects between them (§1.3).
- S2: "stored as it was uploaded" (C.2.a) / "exactly the same as the uploaded one" (C.3.a): byte equality is unrealistic (re-serialisation, namespaces, whitespace). Define XML-canonical or OPT-semantic equivalence.
- S2: The datasets don't match the list. The spec says "invalid OPT (empty template_id)", while the folder is `removed_template_id` and contains a stray `invalid_readme.txt`. `alien_tags/` and `nested/` exist but are not in the spec. "maximal valid OPT (all types in the RM)" has no obvious single file.
- S2: B.1 mentions "deactivation" and C.5 says delete is only allowed when no data references the OPT, but no test covers delete/deactivate with existing data, or committing against a deactivated OPT.
- S2: C.5.a: the postcondition is "None" although delete changes state. The note "for step 1, exclude versioned OPT" is unclear. C.5.b's title says "delete **last version**", but its notes say delete without a version removes **all** versions.
- S3: C.3 numbering jumps from "Alternative flow 1" to "flow 3" and "flow 4". The C.3.d note was copied from C.3.c ("to be sure the **last** version was returned", line 330; it should be "the requested version").
- S3: A.1 "OPT 1.4 if more frequent" should be "is more frequent" (the same typo is repeated in EHR, EHR_COMPOSITION and EHR_CONTRIBUTION).
- S3: Validation preconditions require an empty server, which validation does not need.

## 4. EHR.md

- S2: A.2 requires **XML** support ("we don't have official tools to validate JSON"), while EHR_COMPOSITION/EHR_CONTRIBUTION A.2 accept XML **or** JSON, and EHR B.1 note 5 validates EHR_STATUS against the JSON schemas. The ITS-JSON schemas now exist. Align the format requirement across suites.
- S2: B.1 "Any other data set should be treated as invalid": no flow uses the invalid set, and no expected status is given (§1.5).
- S2: B.1 note 2 asks that "EHR.system_id is set to a known value", but nothing verifies it. Note 3 states defaults (`is_modifiable/is_queryable=true`, `PARTY_SELF` without `external_ref`), but C.1.a only checks "rules", not those defaults explicitly.
- S2: B.1.c depends on the server enforcing "one EHR per subject". Is that normative or a policy? It needs to be a declared option. It also doesn't say whether `subject.external_ref.namespace` is part of the identity (two namespaces, same id).
- S2: C.2–C.5 (set/clear flags) are only checked by reading the flag back. The behavioural effect (commits rejected when not modifiable, AQL excludes non-queryable EHRs) is never tested.
- S2: B.2.b/B.3.b ("by subject_id") don't say which namespace to use. REST requires `subject_id` **and** `subject_namespace`.
- S3: Typos: "strucrures", "providing and EHR_STATUS", "exter_ref", "realted", "relato".

## 5. EHR_COMPOSITION.md

- S1: B.1 (has COMPOSITION) has no REST equivalent (§1.1).
- S2: "Content check" is undefined (§1.2). B.2.a requires two versions in the precondition, but nothing explains how to create them.
- S2: B.2.b/B.3.c "result must be **empty, with an error**" contradicts itself. For REST it should be one specific code.
- S2: B.5.a: "VERSIONED_COMPOSITION … referencing the VERSION<COMPOSITION> it contains". The RM `VERSIONED_OBJECT` does not serialise its versions. REST exposes them through `/revision_history` and `/version`. Specify what to check.
- S2: B.6.a/B.6.b say "version number should be 1" but don't say where to read it (the `uid` suffix `::1`), and don't check `commit_audit.change_type = creation (249)`.
- S2: B.7.a postcondition uses `CREATE`/`MODIFY`, which are not openEHR terms. They should be `creation (249)` / `modification (251)`, and no flow step verifies them. The same applies to B.8 postcondition 3 (`lifecycle_state = 523 deleted`): no step reads it back. After delete, the expected `GET` behaviour (REST: `204 No Content` for a deleted composition **(verify)**) is not specified.
- S2: Missing cases: update/delete with a stale `preceding_version_uid` (412), delete twice, update after delete, commit when `is_modifiable=false`, composition ehr_id mismatch (§1.5).
- S2: B.6.c and B.7.d rely on non-normative rules (§1.3).
- S3: Line 713: "two create", "physical deleted". Lines 189 and 272: "ehd_uid", and `ehr_uid` vs `ehr_id` used interchangeably. B.3.d says "VERSIONED_OBJECT uid", while siblings say "VERSIONED_COMPOSITION uid".

## 6. EHR_CONTRIBUTION.md

- S1: F (list) and G (has) have no REST endpoints (§1.1). F.1/F.4/F.5 ("one item") and F.2/G.2/H.2 (no contributions) conflict with the RM (§1.4).
- S2: **H.1–H.4 all say "Invoke *has* CONTRIBUTION service"** (lines 932, 949, 965, 982). This is a copy-paste error; it should be *get*. **H.4 is truncated**: it has no step 2 / expected result (the file ends at line 982).
- S2: B.3.1 accepted-combinations table: rows 317–319 and 320–322 are identical (`false | true` repeated). The intended combination is probably different (the `true/false` and `false/false` blocks are already present), so either one block is spurious or a combination is missing (for example `subject` variations other than `external_ref`).
- S2: B.2.2 one-commit table: the `lifecycle_state = incomplete` cases are only covered by a note, and there is no row for `creation + incomplete`. "creation + deleted → rejected" has no normative source.
- S2: B.4 (FOLDER) table: `deleted/deleted → accepted` and `amendment/modification → accepted` don't say which preceding state they need (first commit vs existing directory). The `incomplete` lifecycle is accepted for FOLDER but rejected for EHR_STATUS, with no rationale given.
- S2: C.1 postcondition "version ID matches the change_type (creation = 1, modification/amendment = 2, 3 …)" is imprecise: the version number follows the version tree (a `deleted` version is also n+1), not the change type. Flows check only "positive", not the postconditions.
- S2: C.4 requires transactional behaviour ("no CONTRIBUTIONS or VERSIONS should be created"), but the flow never verifies it.
- S2: C.7 does not state the `lifecycle_state` of the deleting version, while EHR_COMPOSITION B.8 expects `523 deleted`. Align them.
- S2: C.8/C.9 postcondition "just one VERSION in the EHR" ignores the EHR_STATUS version. It should be "one VERSION of that COMPOSITION".
- S2: D.3 uses change types `create` and `delete`, but the expected error ("EHR_STATUS already existing") only fits `creation`. A `deleted` change type needs its own expected error.
- S2: Missing: a single CONTRIBUTION mixing COMPOSITION + EHR_STATUS + FOLDER; two versions of the same object in one contribution; contributions with stale `preceding_version_uid`; contribution `audit` validation (missing committer); `IMPORTED_VERSION`.
- S3: Flow numbering: E.1–E.4 are labelled "Alternative flow 12–15" (there are no flows 10–11, and E.1 is effectively a main flow). Line 742 "!." should be "1.". "lis CONTRIBUTIONS", "hould", "aree", "EHR_STATUE", "change_tyoe", "inscludes", "directry". `delete` vs `deleted` in B.2. The `<style>` blocks don't render on GitHub, so the colours are lost.

## 7. DIRECTORY.md

- S1: C, D, J, L have no REST equivalent (§1.1). F.1/G.1/G.2/L.1 allow "empty structure **or** error".
- S2: **I.3 (section at line 631) calls "get directory"** instead of *delete*. **L.3 calls "get directory"** instead of *get versioned directory* (copy-paste errors).
- S2: D.3 path data mixes FOLDER names with COMPOSITION names (`/emergency/episode-x/summary-composition-x`). `FOLDER.items` are `OBJECT_REF`s without names, and the annex itself explains this is ill-defined. "/" as the root name is a convention, not normative. Specify the path grammar (REST `path` parameter) or drop the item-level paths.
- S2: G.5 "a time before EHR.time_created → empty structure" conflicts with the REST behaviour, which is probably 404 **(verify)**. Clock-based flows have the same issue as in §1.4.
- S2: H.1 does not check the version increment or the content. I.2 says the directory "exists as a new deleted version" but nothing verifies it. Delete/update require `If-Match` (preceding version) in REST, and no stale-version case is defined.
- S2: A. General Requirements is "TBD". A "WIP" marker is left in the middle of the document (line 529).
- S2: The Annex (lines 806+) contains internal implementation discussion: a link to a private `wiki.vitagroup.ag`, named individuals' comments ("Luis: …"), and EHRbase-specific decisions ("in EHRbase", "Primary Key in the Database"). That material doesn't belong in a vendor-neutral conformance specification; move it to a separate design note or remove it.
- S2: RM 1.1.0 `EHR.folders` (multiple directories) is not considered **(verify)**.
- S3: E.2 has steps "1, 3". G.7 is labelled "flow 7" (there is no flow 6). J titles say "has directory" for "has directory version". Typos: "ehd_id", "paylaods", "drectory", "Consdidering", "jsut", "exsinting".

## 8. COMPOSITION_VALIDATION_STRUCTURE.md

- S2: **Incomplete**: §§6.6–7.11 are empty headings (ITEM_TREE/LIST/TABLE cardinalities, CLUSTER, ELEMENT value/null_flavour). Section numbers 7.8/7.9 are used twice (lines 1578/1581). The TOC stops at 6.5.
- S2: The intro promises COMPOSITION, EHR_STATUS, FOLDER and PARTY top-level classes, and SECTION/ENTRY/ITEM/DATA_VALUE internals. Only COMPOSITION, OBSERVATION, HISTORY, EVENT and ITEM_STRUCTURE types exist. Not covered: SECTION, EVALUATION, INSTRUCTION/ACTIVITY, ACTION/ISM_TRANSITION, ADMIN_ENTRY, archetype slots, `archetype_node_id`/`name` matching, C_OBJECT occurrences (as opposed to container cardinality), ordered/unique cardinality, RM invariants (language/territory/category codes, `ENTRY.subject`, `encoding`).
- S2: Border cases are incomplete despite §1 requiring "all border cases": for `3..5` (2.6, 2.12, 4.7, 4.13) the upper border (5 accepted, 6 rejected) is never tested, and for `0..1` the first failing value (2) is skipped.
- S2: COMPOSITION category vs context invariant (§1.4).
- S2: §3/§5 include RM/schema "data absent" rows, which contradicts the README scope note (§2).
- S3: §4 intro says "combined with the test data sets for HISTORY defined in section 3" (line 948); it should be OBSERVATION. §6 tables use the column header `event` instead of `structure`. "no event" vs "no events". "ITEM_STRUCRTURE", "manualy". UML image links point to auto-generated `diagram_Diagrams___18_1_…svg` URLs that are likely to break.

## 9. COMPOSITION_VALIDATION_DATATYPES.md

**Wrong expected results (S2):**

| Location | Problem |
|----------|---------|
| §1.2.3 (line 203) | `value=true`, `true_valid=false` → expected **accepted**, while the violated column says `C_BOOLEAN.true_valid`. Should be **rejected** |
| §3.11.3 (line 1045) | `T10..null`, `upper_unbounded=true` → **accepted** with violated `C_TIME.range (upper)` (self-contradictory). Line 1044, the symmetric case `null..T11` with `lower_unbounded=true`, is **rejected**. The two rows need a single rule for how range constraints apply to unbounded limits (and §3.7.2 says limits are optional) |
| §4.5.3 (lines ~2376–2399) | Value `2021-10-24T10` is checked against ranges on **1900-03-13** (`1900-03-13T00..1900-03-13T23` → *accepted*, `>=1900-03-13T11` → *rejected*). Taken literally, every "accepted" row should be rejected and the `>=` rows accepted. The ranges were probably meant to use 2021-10-24 |
| §3.9.2 (lines 833, 842, 843, 850, 851) | "constraints violated" lists `minute_validity (upper)` where minute (upper) is `optional`. Line 833 omits `millisecond_validity (upper)` although millisecond (upper) is `mandatory` (and that constraint combination also breaks the C_DATE_TIME invariant the note assumes) |
| §3.6.1 (lines 580, 585) | `fraction/integer fraction` with numerator 10.5 and precision 1 is attributed to `is_integral_validity`. With precision ≠ 0 `is_integral` is false, so the violated invariant is `Fraction_validity` (fraction requires integral). The real `is_integral_validity` case (precision 0 with a non-integer value) is missing, as are `Precision_validity` and `precision = -1` |
| §2.3.3 | Data uses `terminology_id = SNOMED-CT` while `constraint_bindings = [SNOMED_CT]` and the row expects **accepted**. The identifiers must match, or an equivalence rule must be stated |
| §6.3.1 | `application/dicom, size 123` → accepted without `data` or `uri`. DV_MULTIMEDIA requires inline data or an external URI (invariant `Not_empty`) **(verify)**. The table needs those columns |

**Contradictions (S1/S2):**
- §3.10.1 (line 944), §3.11.1 and the §4.4.3 note (line 2099) say values of different precision are **not comparable** (→ rejected). §4.5.3 (lines 2372–2373) explicitly compares `2021-05` with `1900..2021` and `2021` with `2020-07..2022-03` and expects **accepted**, interpreting them as ranges of instants. Choose one normative rule (ideally from BASE `Iso8601_date` comparison semantics) and apply it everywhere.
- §1.3 DV_IDENTIFIER: `issuer/assigner/type = NULL` is "rejected by C_STRING.pattern". A string constraint does not imply existence (that is the attribute's `existence`). In recent RM releases these attributes are optional **(verify)**, while in RM 1.0.2 they are mandatory, so the outcome depends on the RM version and needs separate existence columns.
- Pattern semantics are unspecified: is `C_STRING.pattern` matched against the whole string or as a substring? (For example §6.2.2 `a*` against `xxx`: a substring regex engine finds an empty match and **accepts**.) State that the match is anchored.

**Incomplete (S2):** `DV_STATE`, `DV_PARAGRAPH`, `DV_GENERAL/PERIODIC_TIME_SPECIFICATION` are skipped or "TBD". §3.11.2 has "TBD: combinations". `DV_SCALE` constraints use a non-existent `C_DV_SCALE` (SPECPR-381), which should be marked "pending spec" rather than certified. DV_QUANTITY never tests `precision` or the lower magnitude border (5.0) for `[cm 5.0..10.0]`. DV_ORDINAL never tests a mismatched pair (symbol of one item + value of another). DV_DATE never tests calendar validity (`2021-02-30`, `2020-02-29`). DV_DURATION 3.12.1 has a duplicate row (lines 1060/1061). DV_CODED_TEXT tables omit `value` (mandatory). DV_PROPORTION `C_REAL.list` is not tested. DV_DURATION 4.2.3 depends on the SUT's own `magnitude()`. DV_EHR_URI: accepting `ehr:/<uuid>` (single slash) has no normative basis **(verify)**. Leap second `:60` is treated as invalid; ISO 8601 allows it, so state an openEHR rule.

**Format note (S2):** test data uses `T10:30` time literals with a leading `T`. openEHR `Iso8601_time` and most canonical JSON/XML instances use `10:30:00` without `T` **(verify)**. Define the canonical form the datasets use, and whether both forms must be accepted.

**Editorial (S3):** `reejcted` ×4 (lines 610, 622, 646, 658); this breaks automated parsing. Heading "1.3" is used twice (DV_IDENTIFIER and DV_STATE). Inconsistent labels `month_val.` / `month_validity` / `timezone__val.` / `seoncd_val.`. `C_DURATION.lower` vs `C_DURATION.range.lower`, and "months_allowed. C_DURATION.lower" (period used as separator). "Datatyps", "Test ccase", "untis", "atribute", "constrainted", "this DB is not used" (should be "this DV"), "COSNTRAINT_REF".

---

## 10. Test implementation (`conformance_test_suites`)

- **S1: Most status-code checks are not assertions.** In Spock, implicit conditions are only the top-level expressions of `then:`/`expect:` blocks. Every `client.lastResponseCode == …` in a `when:` block (lines 215, 219, 236, 262–270, 296, 300, 341–350, 381–390) is evaluated and its result discarded. `[201, 409].contains(...)` is likewise never asserted. Conditions nested in `if` inside `then:` (line 161) are not asserted either. Wrap them with `assert`, or move them into `then:`/`verifyAll`. As written, B.3.c and B.4.b pass even when the server returns 500.
- S2: Line 161 dereferences `ehr.ehr_status.subject…`, although the comment above says `ehr_status` is an OBJECT_REF. If evaluated, this either fails or tests nothing.
- S2: `create_ehr` sends an EHR_STATUS referencing template `ehr_status_any_en_v1`, but the OPT is never uploaded (`FIXME`, line 414). The outcome depends on server leniency.
- S2: Malformed identifiers are used where the spec says "random/non-existent" (§1.4).
- S2: No cleanup: `truncateServer()` is commented out everywhere, so the "empty server" preconditions are not met and runs are order-dependent. Accepting `409` for template upload hides state leaks.
- S2: Very low coverage relative to the spec: EHR B.1.a/b, B.3.a/c and COMPOSITION B.4.a–c, B.6.a, B.7.a. "Content check", version-number and change-type postconditions are not implemented. Test names don't carry the suite (`B.4.a` is ambiguous).
- S2: Only RM 1.0.2 classes and JSON are used. There is no report output matching the design's "Conformance Test Report" (test case × data set → pass/fail).
- S3: `application.properties` commits credentials (`admin@cabolabs.com`/`admin`, a commented JWT). Use env-only configuration plus a template file.
- S3: `.gradle/` caches are committed despite `.gitignore`. `build.gradle` uses `jcenter()` (sunset), which can break dependency resolution. The README is titled `openehr-rest-client` (the wrong project).

## 11. Datasets (`conformance_testing_specification/datasets`)

- S2: `valid_templates/cache/case.opt` and `case2.opt` are **not well-formed XML** (line 49: `xmlns:xsi="http //www.w3.org/…"`, a missing colon) but sit under *valid* templates.
- S2: Files in the wrong folders: `invalid_templates/minimal_persistent/*.composition…invalid.xml` and `invalid_templates/nested/*.xml` are compositions, not templates. `valid_templates/invalid_contributions/` holds contributions.
- S2: Vendor- or project-specific artefacts (`test-ism.vitagroup.de.v1.opt`, `SSIAD PRIeSM.opt`, `ehrn_*`, `000_ehr_status_ecis.json`, filenames with spaces) and a reviewer note committed as data (`removed_template_id/invalid_readme.txt`: "@Pablo I'd suggest…").
- S2: `datasets/ehr/valid` has 8 files for 32 specified combinations. It includes `ehr_can_not_be_modifyable.json` (sic), which sounds like a negative case. There's no mapping file (§1.6).
- S2: `datasets/validation` has only a few ADL/OPT files and no data instances for the STRUCTURE/DATATYPES tables, so those tables cannot be executed yet.
- S2: The `query/` datasets exist without a specification (§1.5). `queries_working_file.js` looks like a scratch file.
- Intentionally malformed files (`empty_file.opt`, `*__invalid_wrong_structure.*`) are fine, but should be flagged as intentionally malformed in a manifest.

---

## 12. Recommended next steps (prioritised)

1. **Define the REST binding** (§1.1/§1.2): for each test case, give the endpoint, headers, expected status codes and response assertions. Remove or substitute the SM-only operations.
2. **Introduce conformance options/profiles** tied to the Conformance Statement (§1.3), and replace every "IMO", "under debate" or "depends on implementation" outcome with either a normative reference or a profile-dependent expected result. Raise the open points with the openEHR SEC (DV_INTERVAL invariants, partial date comparison, persistent compositions, EHR_STATUS lifecycle).
3. **Fix the RM-inconsistent preconditions** (§1.4) and define the environment reset/isolation strategy.
4. **Correct the wrong expected results** listed in §9, and add a CI lint (similar to the one used for this review) that checks every table for valid `expected` values, consistency with "constraints violated", duplicate rows, and recomputable outcomes such as validity-kind.
5. **Fill the coverage gaps**: AQL suite, versioning/conflict cases, behavioural effects of EHR_STATUS flags, the missing structural sections.
6. **Make the tests assert** (§10) and generate a per-case/per-dataset report. Add stable IDs and a dataset manifest (§1.6).
7. **Clean up the repository**: remove the orphan/diverging file, the vendor-internal annex, committed credentials and build caches; fix the dataset placement; add a LICENSE and release tags.

## 13. Open questions for the author

1. Which ITS-REST release (and RM release) is the certification baseline?
2. Is a vendor-provided "reset/truncate" hook acceptable as part of the certification setup, or must tests be state-independent?
3. Should certification cover only the CDR product class for now (as the suites suggest), with AQL included?
4. For behaviour the specs don't settle, should the framework hold back the test until the SEC decides, or certify against declared options?
