# C01 – Preserving Unknown Evidence in Zarr Audits

## Problem

In the previous audit path, a failed metadata or directory access could produce the same empty return values as confirmed absence or a confirmed empty structure. This applies to transport, permission, server, timeout, connection, and unsupported-listing failures. Downstream findings could therefore make a stronger negative claim than the available evidence supported.

C01 preserves the distinction between positive evidence, reliable evidence of absence, and an inconclusive access. It does **not** claim that this behavior caused data loss or that any Vesuvius dataset is damaged.

## Why this matters for Vesuvius

[ScrollPrize/villa issue #1760](https://github.com/ScrollPrize/villa/issues/1760) documents `zarr-pyramid-audit` as a possible header-only publish-time or nightly check for the public Vesuvius Zarr corpus.

The practical check used one public root from that workflow:

```text
fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr
```

Public resources:

- [Zarr root](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/)
- [Zarr v2 group header (`.zgroup`)](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/.zgroup)
- [OME multiscale attributes (`.zattrs`)](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/.zattrs)

A limited header capture on 2026-09-27 returned HTTP 200 for `.zgroup`, `.zattrs`, and the root listing. The `.zgroup` document identified a Zarr v2 group, and `.zattrs` contained `multiscales`. The capture downloaded 4,394 bytes and no array chunks or CT data.

## Reproduction

The test combined two separately identified evidence classes:

1. The real public header and listing structure of the Vesuvius Zarr root above.
2. A deterministically injected HTTP 503 while executing the same production audit chain: `HttpStore -> read_pyramid -> audit_one`.

The result was:

| Version | Finding | Evidence state |
|---|---|---|
| Unmodified baseline | `EMPTY_ZARR_DIR` | not represented |
| C01 | `ACCESS_UNKNOWN` | `UNKNOWN / SERVER_ERROR` |

**The HTTP 503 was deterministically injected. It was not observed as a live outage of the Vesuvius host.** The live metadata capture returned HTTP 200.

The focused public regression tests can be run from the contribution branch:

```bash
git clone --branch c01/preserve-evidence https://github.com/saschaivanov-lgtm/zarr-pyramid-audit.git
cd zarr-pyramid-audit
python -m pip install -r requirements.txt
python tests/test_evidence_semantics.py -v
```

These tests use the production HTTP/S3 store, metadata, and audit code paths. Only the transport or external filesystem surface is replaced with deterministic fixtures; they do not contact the public Vesuvius host.

## Negative control

C01 does not convert every negative result to `UNKNOWN`:

| Confirmed condition | Finding after C01 |
|---|---|
| Both metadata probes for a declared level return HTTP 404 | `LEVEL_MISSING` |
| Metadata probes are absent and a successful listing confirms an empty structure | `EMPTY_ZARR_DIR` |
| A successful level listing confirms no chunk keys | `LEVEL_NO_CHUNKS` |

The frozen contract also retained valid sparse/fill-value behavior without labeling missing physical chunks as a defect when at least one chunk was present.

## Evidence model

| Evidence | Meaning |
|---|---|
| `PRESENT` | Positive evidence exists. |
| `ABSENT` | The access method provides reliable evidence of non-existence. |
| `UNKNOWN` | The available access cannot establish presence or absence. |

`UNKNOWN` describes missing evidence, not a Zarr dataset defect.

## Tests

The final contribution was checked with:

- **17/17** focused standard-library `unittest` tests;
- **12/12** cases in the frozen C01 evidence contract;
- **42/42** cases in the readiness regression suite;
- **0 chunk GETs** in the relevant frozen contract;
- a conflict-free local combination with open [PR #1](https://github.com/sgsllc-jr/zarr-pyramid-audit/pull/1), with **23/23** combined unit tests and both external regression suites passing.

The 17 focused tests are included in the public contribution. The frozen 12-case contract and 42-case readiness suite were separate deterministic project regressions and are reported here as validation results; they are not added to the upstream PR diff.

## Scope and compatibility

- No new runtime dependency.
- Existing `try_json()` and `list_dir()` interfaces remain compatible tuple-returning wrappers.
- Legacy store adapters remain supported.
- Existing output fields are retained; `evidence_state`, `evidence_reason`, `chunk_evidence_state`, and `chunk_evidence_reason` are additive.
- HTTP and S3 use the same `PRESENT` / `ABSENT` / `UNKNOWN` evidence semantics.
- No additional chunk GETs are introduced in the tested header-audit paths.

S3 was tested through the production `S3Store` code with a mocked external filesystem surface. `FileNotFoundError` remains `ABSENT`; permission, service, and timeout failures remain `UNKNOWN`. No live S3 outage or outage frequency is claimed.

## Evidence classification

| Class | Statements on this page |
|---|---|
| **A — direct public-data evidence** | The named Vesuvius Zarr root, its v2 group metadata, its multiscale attributes, and its role in the audit workflow documented in villa issue #1760. |
| **B — deterministic fault-injection evidence** | The 503 before/after result, the negative controls, HTTP/S3 error semantics, compatibility checks, test totals, and zero chunk GETs. |
| **C — interpretation** | Replacing an unsupported `EMPTY_ZARR_DIR` conclusion with `ACCESS_UNKNOWN` can prevent a user from treating an access failure as evidence that data should be repaired or republished. This is an operational interpretation, not an observed incident report. |

## Limitations

- No HTTP 503 was observed from the live public Vesuvius host; the 503 was injected deterministically.
- The frequency of real transport or service failures was not measured.
- The demonstration makes no claim that Vesuvius data is corrupted or that a prior audit caused data loss.
- The practical demonstration evaluated one documented Vesuvius audit path and one public Zarr root.
- The contribution is an open pull request and is not necessarily merged upstream.
- S3 behavior was tested deterministically through the production store with a mocked filesystem surface, not against a failing live bucket.
- The public PR contains the focused regression tests. The larger frozen and readiness suites remain external validation results rather than files in the PR.

## Public contribution

- Upstream pull request: [sgsllc-jr/zarr-pyramid-audit PR #2](https://github.com/sgsllc-jr/zarr-pyramid-audit/pull/2)
- Contribution branch: [`saschaivanov-lgtm:c01/preserve-evidence`](https://github.com/saschaivanov-lgtm/zarr-pyramid-audit/tree/c01/preserve-evidence)
- Reviewed head commit: [`426a15a60207c34500017693150cb2b881d8f583`](https://github.com/saschaivanov-lgtm/zarr-pyramid-audit/commit/426a15a60207c34500017693150cb2b881d8f583)
- Focused regression tests: [`tests/test_evidence_semantics.py`](https://github.com/saschaivanov-lgtm/zarr-pyramid-audit/blob/c01/preserve-evidence/tests/test_evidence_semantics.py)

## Reproduction resources

- [Documented publish-time/nightly audit context — villa #1760](https://github.com/ScrollPrize/villa/issues/1760)
- [Public Vesuvius Zarr root](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/)
- [Public `.zgroup`](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/.zgroup)
- [Public `.zattrs`](https://dl.ash2txt.org/fragments/Frag3/PHercParis1Fr34.volpkg/volumes_standardized/88keV_3.24um_.zarr/.zattrs)
- [C01 implementation and review discussion — PR #2](https://github.com/sgsllc-jr/zarr-pyramid-audit/pull/2)
