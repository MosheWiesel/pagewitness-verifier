# Sentinel Case Pack 1.5

A Case Pack is the signed ZIP produced for one captured event. `manifest.json` is
authoritative: every evidence item lists its evidence type, source URL, declared and
detected MIME types, SHA-256, byte size, capture time, WORM object version, sanitized
capture metadata and (since 1.2) its exact `export_path` inside the archive.

## Verifying who signed

The signature is checked against a trusted key. Without `--public-key` the key inside the
package is all there is, so anyone could have made it: the result is `SIGNER UNKNOWN`, exit 6.
`--allow-unpinned-key` accepts that for a test. Take the issuer key from the issuer's `/verify`
page, not from the package.

The verifier also compares each file with the hash the audit log recorded at capture, warns
when a claimed time is more than 48 hours before its external timestamp or has none, and
fails when it claims a time after that timestamp. The external time is the earliest date a
package can prove.

## Chain of custody

`custody_report.pdf` is written for each Case Pack, in the language of its report, and the
signature covers it. It states what was captured, who captured it and with which system, the
audit chain in plain words, the signing key, the timestamps, the storage lock and every file
with its full SHA-256. `manifest.json` stays authoritative: if they ever differ, the manifest
and the audit chain are the record.

`source_type` is `SINGLE_URL` for a one-off capture and the source type (`RSS`,
`TELEGRAM_PUBLIC`) otherwise.

With `DECLARATION_TEMPLATES_ENABLED` on, the pack also holds `declarations/declaration_en.txt`
and `declarations/tazhir_he.txt`: drafts filled in with this pack's identifiers. The setting
stays off until a lawyer has reviewed the wording.

## What changed from 1.4

1.5 adds `custody_report.pdf`, the optional `declarations/` files and `SINGLE_URL` as a source
type. A verifier older than 1.5 reports these files as undeclared, so take the verifier from the
issuer's `/verify` page. The 1.5 verifier still reads every older pack.

## Bitcoin proofs

`audit_chain.json` can carry `bitcoin_proofs`, the OpenTimestamps proof of each public checkpoint
in `public_checkpoints`, by sequence. A proof covers the SHA-256 of the checkpoint as published:
`json.dumps(record, indent=2, sort_keys=True) + "\n"`. The chain from a file to Bitcoin is: file
hash, audit entry, anchor batch, public checkpoint, proof, block.

A checkpoint is published once a day and a proof completes hours later, so a pack exported soon
after a capture has no proof. Export it again a day or two later to include one. The verifier
fails a proof that covers anything else. It works offline, so it prints each block and merkle root
for comparison with any Bitcoin explorer.

## What changed from 1.3

1.4 adds `capture_outcome` to the manifest: what this capture really preserved.

    "capture_outcome": {"status": "DEGRADED", "reasons": ["browser_failed"]}

`CAPTURED`: the page or file and everything planned for it are stored. `DEGRADED`: the
page or file is stored, `reasons` says which part is missing. `FAILED`: the target itself
was not preserved, `reasons` gives the fetch failure (`http_503`, `timeout`...). A URL
that points directly at a file (PDF, image, video) is preserved as evidence of the type
its bytes show, with no page artifacts. `HTML_SNAPSHOT` is therefore not required.

The verifier fails closed (exit 2) when the outcome contradicts the evidence: `CAPTURED`
or `DEGRADED` with nothing preserved or with a fetch failure record, `CAPTURED` with
reasons, `DEGRADED` without them. Packs older than 1.4 do not declare an outcome.

The reason `investigator_browser` marks a capture submitted from the investigator's own
browser: its HTML and screenshots come from that browser, not from the capture server, so
the pack proves who submitted these bytes and when, not that the site served them. Such a
capture is always `DEGRADED` and holds no raw response, WARC or HAR.

The reason `investigator_screen` marks a capture from a tab the investigator shared with the
site: screenshots and, if the investigator kept it, a silent WebM recording (`VIDEO` evidence with
the variant `screen_recording`), always `DEGRADED`, with no HTML at all. In `evidence/metadata/extraction.json`
`address_source` is `typed_by_the_investigator`, because a shared tab does not tell the site its
address; for `investigator_browser` it is `reported_by_the_browser`. The list `screen_times` gives
the moment the browser reported for each screen.

## What changed from 1.2

1.3 keeps the `evidence-tree/1` layout and adds five evidence types for complete page
preservation. A 1.2 verifier still hash-checks these files (they are ordinary
`export_path` items); the 1.3 verifier additionally checks their structure.

| Type | Path | Content |
| --- | --- | --- |
| `WARC_ARCHIVE` | `evidence/network/archive.warc.gz` | WARC 1.1, one gzip member per record: every request/response of the isolated browser |
| `HAR_ARCHIVE` | `evidence/network/archive.har` | sanitized HAR 1.2 (headers, timings; bodies are in the WARC) |
| `MHTML_SNAPSHOT` | `evidence/page/snapshot.mhtml` | single-file page copy (`Page.captureSnapshot`, `format=mhtml`) |
| `PAGE_PDF` | `evidence/page/print.pdf` | A4 print of the rendered page, backgrounds on, print media |
| `PAGE_METADATA` | `evidence/metadata/page-metadata.json` | structured page facts (`sentinel-page-metadata/1`) |

`media-index.json` entries may now also carry `preferred_url` (smaller srcset versions that
were listed, not downloaded), `loaded_by_browser` and `network_archive`
(`captured`/`truncated`: the browser's own copy of that URL is in the WARC), and the summary
adds `skipped_smaller_versions` and `in_network_archive`.

## What changed from 1.1

| 1.1 | 1.2 |
| --- | --- |
| evidence files under `media/<evidence-id>.<ext>` | evidence files under the `evidence/` tree at the manifest's `export_path` |
| no layout field | `manifest.evidence_layout = "evidence-tree/1"` (mirrored in `package_metadata.json`) |
| page HTML, extracted text, extraction metadata | + raw HTTP response, rendered DOM, screenshots, media, media index, crawl map, network log |

Packs from 1.0/1.1 (and 1.2 packs that use the old `media/` layout) are still verified
with the legacy rules; 1.2 packs (no page archives) pass the 1.3 archive check trivially.

## Archive layout

```text
manifest.json              signed manifest (signature block excluded from the payload hash)
package_metadata.json      format/version/layout, evidence_files_included
audit_chain.json           hash-chained audit entries + RFC 3161 proofs (batch anchors, legacy per-entry tokens)
worm_proof.json            per-object bucket, key, version id, retention mode/until, etag, size
report.pdf                 human-readable summary (not authoritative)
timestamps.json            RFC 3161 token index (added after signing)
timestamp_tokens/          payload-hash and audit-chain tokens (DER)
evidence/
  page/
    source.html            HTML_SNAPSHOT: response body decoded per Content-Encoding
    rendered.html          RENDERED_DOM: DOM serialized after isolated-browser rendering
    extracted.txt          TEXT_EXTRACT: main text (trafilatura), when extractable
    screenshot-viewport.png  SCREENSHOT variant=viewport (1440x900)
    screenshot-full.png      SCREENSHOT variant=full_page (height-capped, see limits)
    snapshot.mhtml         MHTML_SNAPSHOT: single-file page copy (1.3)
    print.pdf              PAGE_PDF: A4 print of the rendered page (1.3)
    copies/<id>.<ext>      further MHTML/PDF items
  media/
    images/<id>.<ext>      IMAGE
    video/<id>.<ext>       VIDEO (incl. HLS segments within limits)
    audio/<id>.<ext>       AUDIO
  documents/<id>.<ext>     DOCUMENT (PDF/Office/ODF/CSV/TXT/ZIP, subtitle tracks, HLS/DASH playlists)
  network/
    transactions.json      HTTP_TRANSACTION_JSON: complete network trail of the capture
    raw-responses/<id>.bin RAW_HTTP_RESPONSE: page body exactly as received on the wire
    archive.warc.gz        WARC_ARCHIVE: browser network archive (1.3)
    archive.har            HAR_ARCHIVE: sanitized HAR (1.3)
    archives/<id>.<ext>    further WARC/HAR items
  metadata/
    extraction.json        EXTRACTION_META: fetch/extraction summary incl. full_capture block
    media-index.json       MEDIA_INDEX: every detected media item and its outcome
    crawl-map.json         CRAWL_MAP: root page + same-origin linked pages
    page-metadata.json     PAGE_METADATA: structured page facts (1.3)
    failures/<id>.json     FETCH_FAILURE records
  crawl/<id>.<ext>         raw response / source HTML / text of crawled linked pages
  ingest/<id>.txt          text recorded at ingest (feed summary or capture request), not page text
  other/<id>.<ext>         any other evidence type
```

Fixed names are used for the first item of each root-page type; further items of the same
type (or crawled pages, `capture_metadata.crawl_depth >= 1`) use their evidence UUID, so
paths never collide. Paths are built only from fixed directory names, the evidence UUID
and a whitelisted extension; they are part of the signed manifest.

## Index formats

`media-index.json` (`format: sentinel-media-index/1`):

```json
{
  "page_url": "...", "final_url": "...",
  "summary": {"detected": 31, "captured": 18, "failed": 2, "skipped": 11, "captured_bytes": 1234567},
  "items": [
    {"url": "...", "relation": "img:src", "source_page_url": "...", "expected_kind": "IMAGE",
     "status": "captured", "evidence_id": "<uuid>", "evidence_type": "IMAGE",
     "sha256": "<hex>", "size": 12345, "mime_declared": "image/jpeg", "mime_detected": "image/jpeg",
     "http_status": 200, "final_url": "..."},
    {"url": "...", "status": "skipped", "reason": "asset_count_limit"},
    {"url": "...", "status": "failed", "reason": "not_media_content"}
  ]
}
```

Statuses: `captured` (preserved, has `evidence_id` + `sha256`), `failed` (download or
validation failed: `http_404`, `asset_too_large`, `mime_mismatch`, `not_media_content`,
`blocked_by_egress_policy`, `timeout`, ...), `skipped` (a limit was reached:
`asset_count_limit`, `total_size_limit`, `capture_time_limit`, `hls_segment_limit`,
`hls_variant_not_selected`, `encrypted_stream_not_decrypted`, `live_stream_not_recorded`;
or a policy applied: `cross_site_document_link`, `srcset_smaller_candidate` and
`srcset_larger_candidate_preferred` - a larger version of the same image, named in
`preferred_url`, was preserved instead).
HLS entries carry `stream_role` (`playlist`, `variant_playlist`, `segment`),
`playlist_url` and `sequence`.

`crawl-map.json` (`format: sentinel-crawl-map/1`): `root_url`, `final_url`,
`same_origin_only`, `max_depth`, `max_pages`, `robots_policy`, `status`
(`disabled`, `completed`, `page_limit_reached`, `capture_time_limit`, `total_size_limit`)
and `pages[]` with `url`, `depth`, `parent_url`, `status`
(`captured`/`blocked`/`failed`/`skipped`), `reason` and `evidence_ids`.

`transactions.json` (`format: sentinel-network-log/1`): `page_fetch` (request/response
headers, redirects, body SHA-256), `asset_downloads[]` (one per preserved media item, with
`evidence_id` and `response_body_sha256`), `browser.requests[]` (every response/failure the
isolated browser observed), `browser.blocked_by_interception[]`, and `egress` — the
connection log of both egress proxies (destination, resolved and connected address,
allowed/blocked decision and reason, bytes). Cookies, authorization and other
credential-bearing headers are never recorded.

`archive.warc.gz` (WARC 1.1): record 1 is `warcinfo` (`software`, `format`, `conformsTo`,
`isPartOf`, `sentinel-capture-id` = event id, page URL, browser version, body representation,
removed headers, limits). Then, per HTTP(S) exchange, a `response` record followed by a
`request` record whose `WARC-Concurrent-To` names the response. Every record has
`WARC-Record-ID`, `WARC-Date` (request start, microseconds, UTC), `WARC-Block-Digest`;
request/response records have `WARC-Target-URI`, `WARC-Payload-Digest` (both
`sha256:<base32>`), `WARC-Protocol` (`http/1.1`, `h2`, ...), `WARC-IP-Address` when known
and `WARC-Sentinel-Resource-Type` (document, stylesheet, script, image, font, fetch, media,
...). A body cut by a limit carries `WARC-Truncated: length`. Bodies are the payload the
browser decoded; the response's original `Content-Encoding`/`Transfer-Encoding`/
`Content-Length` are kept as `X-Archive-Orig-*` and `Content-Length` is the stored length.
`Cookie`, `Authorization`, `Proxy-Authorization` and `Set-Cookie` headers are removed
(`WARC-Sentinel-Removed-Headers` lists them on affected records). Failed requests (no
response) and refusal pages generated by Sentinel's egress proxy (`X-Sentinel-Egress-Block`)
have no records - a refusal is not the site's response; both are listed in the network log
summary (`failed_samples`, `egress_blocked_samples`).

`archive.har`: Playwright's HAR with `cookies` emptied, the same headers removed and response
bodies dropped; each entry's `_sentinel` block gives `warc_record_id`, `payload_sha256`,
`archived_bytes`, `body_size`, `truncated` (or `archived: false` and a reason).

`page-metadata.json` (`format: sentinel-page-metadata/1`): `extracted_from` (live rendered
DOM, serialized DOM or source HTML), `final_url`, `title`, `language` (lang/dir/charset),
`meta` (description, author, keywords, robots, generator), all `meta_tags`, `canonical_url`,
`alternates`, `feeds` (RSS/Atom/JSON feed links), `opengraph`, `twitter_card`, `jsonld` (each
block parsed or with its error; size-capped), `dates` and `authors` (every candidate with the
exact source it came from; ISO-8601 normalization only when the value states a timezone),
`links` (href, text, rel, internal/external/non_http), `social_links` (telegram, x/twitter,
facebook, instagram, tiktok, youtube, linkedin, whatsapp; with `sources`, `occurrences` and
`is_share_link` for share buttons), `social_handles` (twitter:site/creator), `iframes`,
`forms` (action and method only), `media` (images with largest srcset candidate and the
browser's `current_src`, picture sources, video/audio sources, posters, subtitle tracks),
`counts` (found vs collected, list caps) and a `summary`. It records what the page states;
it does not attribute identity or verify dates.

## Audit timestamp proofs (`audit_chain.json`)

Audit entries are written without a network call. The worker anchors new entries in
batches (default every 5 minutes): it builds a `sentinel-merkle-v1` tree over their entry
hashes and obtains one RFC 3161 token over the root.

* leaf = SHA-256(`0x00` || entry_hash bytes); node = SHA-256(`0x01` || left || right);
  an unpaired node at the end of a level is promoted unchanged.
* `timestamp_anchors.<batch_id>` = `{tree_alg, merkle_root, leaf_count, token_base64}`.
* `entries[].timestamp_proof` = `{batch_id, leaf_index, path: [{position: "left"|"right", hash}]}`.
  Folding the entry hash up the path must give `merkle_root`, and the token's message
  imprint must equal `merkle_root`.

Only sibling hashes leave the server, never other entries' content. Entries written before
batching keep `timestamp_token_base64` (imprint = entry hash). Entries the worker had not yet
anchored when the pack was built carry no proof; the package-level token
(`casepack_payload_hash`) still covers them.

## Case packages (`sentinel-case-bundle/2.0`)

A case export wraps one complete event Case Pack per capture; nothing is re-encoded.

```text
manifest.json              package_format=sentinel-case-bundle, package_scope=case, case summary,
                           events[]: {event_id, path, sha256, size, payload_hash_hex, degraded, ...},
                           signature (Ed25519 over the case payload hash, same scheme as event packs)
package_metadata.json      format / version / scope / case_id (must match the manifest)
timeline.json              case timeline (captures, evidence, audit, timestamps, exports)
report.pdf                 case summary with the issuer key fingerprint (not authoritative)
events/<event_id>.zip      the event's own Case Pack, signed and timestamped by itself
timestamps.json, timestamp_tokens/payload_timestamp.der   RFC 3161 token over the case payload hash
```

`sentinel_verify.py` detects `package_scope: "case"` and checks, in order: the case manifest
(format, metadata, every declared `events/<id>.zip` present with the recorded SHA-256 and size,
no undeclared files), the case signature (pinned with `--public-key`), the case payload
timestamp, and then the full event verification of every nested pack. Each nested pack must
carry the event id and signed payload hash the case manifest records and be signed by the same
key. The export fails closed: if any event cannot be packaged, no case package is produced.

`POST /api/v1/cases/{id}/exports` only queues the job (202, `job_id`); the worker builds it -
each event package in its own transaction - and the client polls `/api/v1/exports/{job_id}`.
A second request while a case job is pending returns the same job.

## Verification

```bash
python sentinel_verify.py casepack.zip \
    --public-key <issuer Ed25519 key, hex> \
    --trust-anchors trust_anchors/tsa-trust-anchors.pem
```

For 1.2 (`evidence-tree/1`) packs the verifier additionally fails closed when:

- an `export_path` is not a plain relative path below `evidence/` (absolute, `..`, `.`,
  empty segments, backslashes, drive letters, NUL) or resolves outside the extraction
  directory (links are rejected during extraction and again here);
- a stored item has no `export_path`, two items declare the same path, or a file inside the
  archive is not declared in the manifest;
- `media-index.json`, `crawl-map.json` or `transactions.json` is malformed, references an
  evidence id that is not in the manifest, or records a SHA-256 that differs from the
  manifest; a captured media item is missing from the media index;
- a screenshot is not a PNG or its dimensions differ from its capture metadata.

For 1.3 page archives it also fails closed (exit 2) when:

- `archive.warc.gz` is not a sequence of gzip members each holding one WARC/1.x record, the
  first record is not `warcinfo`, a record lacks `WARC-Record-ID`/`WARC-Date`/`WARC-Type` (or
  `WARC-Target-URI` for requests/responses), a block length differs from `Content-Length`, a
  recomputed `WARC-Block-Digest` or `WARC-Payload-Digest` differs, a response has no payload
  digest, or the record count differs from the item's `capture_metadata.records`;
- `archive.har` has no `log.entries` list or its entry count differs from `capture_metadata.entries`;
- `snapshot.mhtml` has no `MIME-Version` / `multipart/related` header;
- `print.pdf` does not start with `%PDF-` or lacks `%%EOF`;
- `page-metadata.json` is not a `sentinel-page-metadata/1` JSON object.

The WARC check is self-contained (no warcio needed) and decompression is bounded per record.

All existing checks (manifest hashes, pinned Ed25519 signer, RFC 3161 tokens bound to the
signed payload, audit chain, WORM proof covering every stored item) still apply.

Audit timestamp proofs are checked by the RFC 3161 step: every `timestamp_proof` and every
legacy `timestamp_token_base64` that is present must verify (Merkle path, CMS signature,
timeStamping EKU, chain to `--trust-anchors`), otherwise the pack fails with exit 4 - even
with `--allow-no-timestamps`, which only tolerates a missing package-level token. The report
states how many entries carry a verified proof.
