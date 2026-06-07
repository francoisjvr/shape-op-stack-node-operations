# Large Snapshot Staging and Extraction

Use this note when a Shape mainnet Reth rebuild depends on a very large snapshot archive and the naive `curl | tar` path has already proven flaky.

## Why this exists
For the June 2026 mainnet reseed, the latest official archive was about 192.4 GiB (`206574228078` bytes). A streamed restore failed because the HTTP transfer cut out mid-extract. The durable fix was to separate **download/staging** from **decompression/extraction**.

## Recommended pattern
1. Confirm the current official snapshot name/URL from Shape docs or the provider page.
2. Check whether the full archive already exists on the VPS.
3. If the local file exists, validate the expected byte size before redownloading.
4. If incomplete, resume with `aria2c -c` and preserve the `.aria2` file.
5. Stop stale readers against the archive (old `tar -tf`, `pigz`, checksum probes, abandoned downloaders).
6. Extract from the completed local file with local decompression.
7. Only after extraction succeeds, continue promote/config/start verification.

## Good operator choices
- Prefer a verified local archive over redownloading.
- Prefer resumable download tooling (`aria2c -c`) over fresh `curl` retries.
- Prefer `pigz -dc <archive> | tar -xf -` for local extraction of gzip tarballs.
- Keep a clear separation between:
  - snapshot cache/upload directory
  - staging extraction directory
  - promoted live datadir

## Avoid
- `curl <url> | tar -xz` or similar one-shot network-stream extraction for ~200 GB archives.
- deleting a giant archive before confirming it is corrupt or incomplete.
- running archive inspection and real extraction against the same file at the same time.
- bouncing unrelated containers just because the mainnet rebuild is in progress.

## Minimal verification checklist
- archive file exists at the expected path
- archive byte size matches expected size
- no stale downloader/reader still holds the file
- extraction process is visibly running (`pigz`/`tar` or equivalent)
- staging directory size is increasing during extraction
- only after extraction completes: resume book flow and verify repeated head movement

## Session-specific anchor
Observed working path in this session:
- reuse local archive already present on the VPS
- fall back to resumable `aria2c` only if the local archive is missing or undersized
- extract offline from the local file instead of streaming from HTTP
