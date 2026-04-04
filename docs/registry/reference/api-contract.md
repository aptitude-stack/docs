# API Contract

## Current Registry Paths

The resolver currently reads from the live registry through these runtime paths:

- `POST /discovery`
- `GET /skills/{slug}`
- `GET /skills/{slug}/{version}`
- `GET /resolution/{slug}/{version}`
- `GET /skills/{slug}/{version}/content`

## Runtime Assumptions

- the server is the source of immutable metadata and artifact facts
- final ranking, version choice, dependency solving, and lock generation remain local resolver behavior
- `sync --lock` must replay an existing lock without calling discovery or dependency solving

## Checksum Contract

- checksum algorithm: `sha256`
- checksum verification happens during materialization
- mismatch must fail fast as `ContentChecksumMismatchError`
