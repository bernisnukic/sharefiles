# sharefiles

Tiny CLI to upload a file to an S3-compatible bucket (Cloudflare R2) and print a shareable URL.

## Features

- Single-file Bash script (no external dependencies beyond macOS defaults)
- AWS Signature V4 signing for S3-compatible endpoints
- TUI progress meter with percent, speed, and ETA
- Optional metadata headers (Content-Type, Cache-Control, Content-Disposition, custom `x-amz-meta-*`)
- Preflight checks and retries for production use
- Copies the share URL to your clipboard (macOS `pbcopy`)

## Requirements

- macOS with `bash`, `curl`, `openssl` (defaults)
- `file` (optional, used to auto-detect Content-Type)

## Install

Clone the repo and make the script executable:

```bash
chmod +x share
```

Optionally add it to your PATH:

```bash
ln -s "$(pwd)/share" /usr/local/bin/share
```

## Usage

```bash
./share <path-to-file> [object-key]
```

Examples:

```bash
./share ./report.pdf
./share "./my file.txt" my-file.txt
```

Object keys are URL-encoded in the output URL.

## Configuration

Create a `.env` at the repo root (parsed, not executed) and set your values. Use `.env.example` as a starting point.

```bash
SHARE_BUCKET=your-bucket-name
SHARE_ENDPOINT=https://<accountid>.r2.cloudflarestorage.com
SHARE_BASE_URL=https://your-share-domain.example
SHARE_ACCESS_KEY_ID=YOUR_ACCESS_KEY_ID
SHARE_SECRET_ACCESS_KEY=YOUR_SECRET_ACCESS_KEY
```

Defaults are placeholders; set every required value for the script to run.
The share URL is constructed from `SHARE_BASE_URL` plus the object key, so it must point to a public bucket or custom domain that serves the uploaded objects.

### Environment variables

| Variable | Required | Default | Description |
| --- | --- | --- | --- |
| `SHARE_BUCKET` | Yes | `your-bucket-name` | Bucket name. Must match token scope. |
| `SHARE_ENDPOINT` | Yes | `https://<accountid>.r2.cloudflarestorage.com` | S3 endpoint URL. |
| `SHARE_BASE_URL` | Yes | `https://your-share-domain.example` | Public base URL used to build the share link. |
| `SHARE_REGION` | No | `auto` | Region for signing (R2 uses `auto`). |
| `SHARE_STYLE` | No | `path` | Request style: `path` or `virtual`. |
| `SHARE_ACCESS_KEY_ID` | Yes | - | R2 Access Key ID. |
| `SHARE_SECRET_ACCESS_KEY` | Yes | - | R2 Secret Access Key. |
| `SHARE_SESSION_TOKEN` | No | - | Session token for temporary credentials. |
| `SHARE_PREFLIGHT` | No | `1` | Run preflight checks (`0` to skip). |
| `SHARE_CLIPBOARD` | No | `1` | Copy share URL to clipboard (`0` to skip). |
| `SHARE_CONTENT_TYPE` | No | `auto` | Content-Type header (`none` to disable). |
| `SHARE_CACHE_CONTROL` | No | - | Cache-Control header for the uploaded object. |
| `SHARE_CONTENT_DISPOSITION` | No | - | Content-Disposition header for the uploaded object. |
| `SHARE_META` | No | - | Custom metadata in `key=value,key2=value2` form (maps to `x-amz-meta-*`). |
| `SHARE_RETRY_COUNT` | No | `3` | Retry attempts for transient failures (`0` to disable). |
| `SHARE_RETRY_DELAY` | No | `1` | Seconds between retries. |
| `SHARE_RETRY_MAX_TIME` | No | `30` | Max seconds for all retries. |

## Metadata

- `SHARE_CONTENT_TYPE` defaults to `auto`, which uses `file` to detect a MIME type (if `file` is missing, the header is skipped). Set it to `none` to skip.
- `SHARE_CACHE_CONTROL` and `SHARE_CONTENT_DISPOSITION` are passed as headers on upload.
- `SHARE_META` entries become `x-amz-meta-<key>: <value>` (keys are lowercased and must match `a-z0-9._-`; values should not contain commas).

Example:

```bash
export SHARE_CONTENT_TYPE=image/png
export SHARE_CACHE_CONTROL="public, max-age=31536000"
export SHARE_CONTENT_DISPOSITION='inline; filename="hero.png"'
export SHARE_META="author=bernis,source=cli"
```

## Preflight checks

When `SHARE_PREFLIGHT=1`, the script:

- Validates the S3 endpoint is reachable
- Performs a signed GET to confirm bucket access
- Warns if the public base URL is unreachable or blocked

## Retries

Retries are enabled by default with `SHARE_RETRY_COUNT=3`. This helps with transient network errors and 5xx responses.

## Troubleshooting

- `AccessDenied`: token scope does not include the bucket in `SHARE_BUCKET`.
- `SignatureDoesNotMatch`: check region (`auto` for R2) and ensure clock drift is minimal.
- Public URL returns 403/404: verify the bucket or custom domain is publicly readable.

## License

MIT. See `LICENSE`.
