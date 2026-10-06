# ADR 010: Object Storage

**Status:** Accepted (specification-mandated; upload path and limits are architectural recommendations) · **Date:** 2026-10-05 · **Related:** [File Storage](../16-file-storage.md), [Security](../17-security.md)

# Context

Students upload PDFs, presentations, documents and images. Large files must not live in PostgreSQL. Uploads are untrusted input and personal data.

# Problem

Choose storage technology, upload architecture, access control and deletion semantics.

# Options Considered

1. Files in PostgreSQL (bytea or large objects).
2. Local disk/volume on application servers.
3. **S3-compatible object storage** with backend-proxied upload and signed download URLs.
4. S3-compatible storage with direct browser upload via pre-signed POST.

# Decision

S3-compatible object storage (S3, R2, MinIO locally) accessed through a storage adapter. MVP uses option 3: Frontend → Backend → validation → object storage → metadata in PostgreSQL, per the specification. Rules: private bucket, opaque keys (`users/{user_id}/files/{file_id}`), 25 MiB per file, per-user quotas, extension allowlist plus magic-byte MIME validation, no SVG/HTML/macro files, malware scan before `available`, signed 5-minute download URLs with attachment disposition, soft delete then purge after 30 days, reconciliation job for orphans.

# Reasoning

Backend-proxied uploads let validation happen before data is stored and keep control of quotas, ownership and scanning in one place. Object storage separates large-binary durability and scaling from the database. Signed URLs avoid proxying downloads.

# Trade-offs

Proxied uploads consume API bandwidth and memory/disk for temporary files and cap practical file size; pre-signed direct upload scales better but validation must happen after upload.

# Consequences

Edge must enforce request-size limits; API streams to a bounded temporary file; scan worker and quarantine flow are required; backups of the database do not include file bytes; storage lifecycle rules handle exports and incomplete uploads.

# Future Migration Path

Move to direct-to-storage uploads (pre-signed POST with size and type conditions) where the upload lands in a quarantine prefix, then the worker validates, scans and promotes it to the final key, with the same metadata states already defined. Add thumbnails and text extraction as worker jobs. The storage adapter allows switching providers without application changes.
