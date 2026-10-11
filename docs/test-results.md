# Automated test results

[← 中文首页](../README.md) · [English home](../README_EN.md)

These results were produced locally from the private course repository on **11 October 2026**. They contain no user data, credentials, claim content, or private source code.

## Summary

| Suite | Result | Scope represented |
| --- | ---: | --- |
| Backend pytest | **18 passed** | Authentication flows and SSRF-aware URL fetching |
| Frontend Vitest | **17 passed across 5 files** | Auth pages and reusable UI/report components |
| Total | **35 passed** | Automated unit and integration-style checks in the repository |

## Backend run

```text
..................                                                       [100%]
18 passed, 2 warnings in 9.51s
```

Command:

```bash
DEBUG=false pytest -q
```

The two warnings were dependency deprecations for a legacy `python-json-logger` import path and the `langchain-community` package. They did not fail the tests but should be addressed during dependency maintenance.

## Frontend run

```text
Test Files  5 passed (5)
Tests      17 passed (17)
Duration   1.72s
```

Command:

```bash
npm test
```

## Interpretation and limits

The result demonstrates that the checked-in automated suites passed in a clean dependency installation. It does **not** claim complete system coverage.

The published count does not validate:

- live paid LLM or search-provider behaviour;
- PostgreSQL migration execution against a production-like database;
- FFmpeg, Whisper, ExifTool, yt-dlp, or C2PA integration;
- Docker Compose end-to-end startup;
- performance, load, or adversarial prompt testing; or
- statistical accuracy and calibration on a labelled fact-checking dataset.

Those areas would require a separate integration environment and evaluation dataset.
