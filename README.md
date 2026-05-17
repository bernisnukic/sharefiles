# sharefiles

Tiny CLI to upload a file to an S3-compatible bucket (Cloudflare R2) and print a shareable URL.

## Features

- Single-file Bash script (no external dependencies beyond macOS defaults)
- AWS Signature V4 signing for S3-compatible endpoints
- TUI progress meter with percent, speed, and ETA
- **Multipart upload** for files larger than 4 GiB (configurable), with overall progress bar and clean abort on failure or Ctrl+C
- Optional metadata headers (Content-Type, Cache-Control, Content-Disposition, custom `x-amz-meta-*`)
- Preflight checks and retries for production use
- Copies the share URL to your clipboard (macOS `pbcopy`)
- Works correctly when symlinked into your `$PATH` (resolves the real script location to find `.env`)

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

Pass `-d` or `--debug` as the first argument to print the canonical request, string-to-sign, and computed signature for every signed request — handy when troubleshooting `SignatureDoesNotMatch`.

Examples:

```bash
./share ./report.pdf
./share "./my file.txt" my-file.txt
./share -d ./large.zip
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
| `SHARE_MULTIPART_THRESHOLD` | No | `4294967296` (4 GiB) | File size in bytes at which the script switches from single PUT to multipart upload. |
| `SHARE_PART_SIZE` | No | `104857600` (100 MiB) | Multipart chunk size in bytes. Minimum 5 MiB (S3 requirement); max ~5 GiB. |

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

Retries are enabled by default with `SHARE_RETRY_COUNT=3`. This helps with transient network errors and 5xx responses. (Retries apply to the single-PUT path; multipart uploads handle failure by aborting the upload — re-run the script to try again.)

## Large files (multipart upload)

Files larger than `SHARE_MULTIPART_THRESHOLD` (default 4 GiB) are uploaded using S3 multipart upload — the script splits the file into chunks of `SHARE_PART_SIZE` (default 100 MiB), uploads them sequentially, and completes the upload at the end. If anything fails or you interrupt the script with Ctrl+C, the in-flight multipart upload is aborted to avoid leaving orphaned parts in your bucket.

The progress bar shows overall progress across the whole file plus the current part number, updating every second as each chunk uploads:

```
██████████░░░░░░░░░░░░░░░░░░░░  34.3%  Part 19/54  1.8 GB / 5.3 GB  2.2 MB/s  ETA 26:51
```

Tuning notes:

- `SHARE_PART_SIZE` minimum is 5 MiB (an S3 requirement); maximum part size is ~5 GiB. Smaller chunks = more requests but a finer retry boundary; larger chunks = fewer requests but more wasted bandwidth on a single part failure.
- `SHARE_MULTIPART_THRESHOLD` controls when multipart kicks in. Single PUT works up to ~5 GiB on R2, so the default 4 GiB leaves comfortable headroom.

## Troubleshooting

- `AccessDenied`: token scope does not include the bucket in `SHARE_BUCKET`.
- `SignatureDoesNotMatch`: check region (`auto` for R2) and ensure clock drift is minimal.
- Public URL returns 403/404: verify the bucket or custom domain is publicly readable.

## License

MIT. See `LICENSE`.
