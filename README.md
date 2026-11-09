# PageWitness offline verifier

Check a [PageWitness](https://sentinelforensic.app) evidence pack (Case Pack) on your own computer:
offline, with one Python file, without trusting PageWitness.

`sentinel_verify.py` here is the same file the service hands out at
<https://sentinelforensic.app/verify/sentinel_verify.py>. To make sure, compare
`shasum -a 256 sentinel_verify.py` on both copies.

## What it checks

| Check | What it means |
|---|---|
| ZIP integrity, required files, package format | The pack is complete and well formed |
| Manifest hashes | Every evidence file still has the SHA-256 recorded in the signed manifest |
| Capture indexes, capture archives, capture outcome | The WARC, HAR, MHTML, PDF and indexes are consistent with each other |
| Evidence vs audit log | Every evidence file is the one the audit log recorded |
| Ed25519 signature | The pack was signed by the key you trust, and nothing changed after signing |
| Audit chain | The custody log has no gaps and was not reordered |
| RFC 3161 timestamps | Independent timestamp authorities saw these hashes at that time |
| Public audit checkpoints, Bitcoin anchor | The log matches what the issuer published |
| WORM proof | The evidence was stored write-once (object lock, compliance mode) |
| Derived files | Content review and post records are listed apart and never count as evidence |

## What it does not prove

It proves the integrity and provenance of what was captured. It does not prove that the content is
true, and whether evidence is admissible is for the court to decide.

## Use

Python 3.10 or later, and two libraries (the verifier refuses to run without them rather than skip
a check):

```bash
pip install cryptography asn1crypto
```

1. Take the issuer's public key from <https://sentinelforensic.app/verify>. Each PageWitness
   installation has its own key; without it the signature only shows the pack is consistent with
   itself.
2. Run:

```bash
python3 sentinel_verify.py casepack.zip \
    --public-key <issuer Ed25519 key, hex> \
    --trust-anchors trust_anchors/tsa-trust-anchors.pem
```

To try it, use the public sample pack:
<https://sentinelforensic.app/sample/PageWitness-sample-pack-en.zip>.

The issuer's public audit checkpoints can be checked on their own:

```bash
curl -O https://sentinelforensic.app/.well-known/sentinel-audit-checkpoints.json
python3 sentinel_verify.py --checkpoints sentinel-audit-checkpoints.json
```

Exit codes: `0` all checks passed, `1` broken or incomplete pack, `2` tampering (hash or signature),
`3` gap in the audit chain, `4` invalid timestamp.

`trust_anchors/tsa-trust-anchors.pem` holds the public root certificates of the timestamp
authorities (DigiCert, Sectigo, FreeTSA). The pack format is described in
[docs/CASE_PACK_FORMAT.md](docs/CASE_PACK_FORMAT.md).

## בעברית

כלי האימות בודק חבילת ראיות של PageWitness במחשב שלכם, בלי אינטרנט ובלי לסמוך עלינו. הוא
בודק שכל קובץ לא השתנה, שהחבילה חתומה במפתח של המנפיק, שחותמות הזמן תקינות ושיומן המשמורת
רציף. הוא מוכיח שלמות ומקור של מה שנלכד, לא שהתוכן נכון. השאלה אם ראיה קבילה נתונה להחלטת בית
המשפט. המפתח הציבורי וההוראות: <https://sentinelforensic.app/verify?lang=he>.

## License

Apache License 2.0, see [LICENSE](LICENSE). Copyright 2026 Moshe Wiesel (PageWitness).

The license covers only the files in this repository. The PageWitness service and the rest of its
source code are not open source and are not licensed by this repository.

## Security

Report a vulnerability to security@sentinelforensic.app. Policy:
<https://sentinelforensic.app/security>.
