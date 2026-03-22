# SKILL 04-H — Secure File Uploads in Any Backend
Category: Backend Development
Applies to: Node.js, Python, Go; S3, Google Cloud Storage, Cloudflare R2; multipart/form-data

## What this skill covers
File upload endpoints are one of the most commonly exploited attack vectors: path traversal, malicious file execution, storage exhaustion, and metadata leaking are all real risks. This skill covers how to implement file uploads securely regardless of backend language: validating file type by content (not extension), enforcing size limits, scanning for malware, storing safely in object storage (not local disk), and generating secure access URLs.

## When to activate this skill
- Implementing an upload endpoint for images, documents, or any binary files
- Adding file upload to a mobile or web app
- Reviewing an existing upload handler for security issues
- Migrating uploads from local disk storage to cloud object storage

## Core principles
1. **Never trust file extensions.** Validate file content using magic bytes (file signature), not the filename or Content-Type header from the client.
2. **Never store user uploads on the application server's local filesystem.** Use object storage (S3, GCS, R2). Local storage leaks on container restarts, doesn't scale, and creates path traversal risk.
3. **Enforce size limits before reading the stream.** Check `Content-Length` and hard-limit the streaming read to prevent storage exhaustion.
4. **Generate a new name for every upload.** Never use the client-supplied filename. Generate a UUID-based key. Store the original filename as metadata only.
5. **Use pre-signed URLs for browser uploads.** Direct-to-S3 uploads via pre-signed URL skip the app server entirely for large files. The app server only validates and records the upload.

## Step-by-step guide
1. **Configure size limit on the body parser:**
   - Express: `multer({ limits: { fileSize: 10 * 1024 * 1024 } })`
   - FastAPI: check `Content-Length` header before processing; `UploadFile.size` in Python 3.12+
   - Go: `http.MaxBytesReader(w, r.Body, 10<<20)` before parsing
2. **Validate MIME type by magic bytes:**
   - Node.js: `file-type` package reads the first bytes and returns the real mime type
   - Python: `python-magic` or `filetype` package
   - Go: `net/http.DetectContentType()` on the first 512 bytes
3. **Generate a storage key:** `${userId}/${randomUUID()}.${extension}` — never use `req.file.originalname`.
4. **Upload to object storage:**
   - S3: `PutObjectCommand` with `ContentType`, `Metadata: { originalName: ... }`, `ACL: 'private'`
   - For large files: use multipart upload or pre-signed URL pattern
5. **For direct browser-to-S3 uploads:** app server generates a pre-signed URL (`PutObject` permission, 5-minute TTL). Browser uploads directly. App server receives a callback (webhook or polling) to record the completed upload.
6. **Serving files to users:** generate a `GetObject` pre-signed URL with short TTL (15 min) on each serve request. Never make buckets public.
7. **Optional: virus scanning.** Scan uploads via ClamAV (self-hosted) or cloud-based AV scanning on an S3 event trigger before marking the file as available.

## The right way vs the wrong way

| Wrong | Right |
|-------|-------|
| `fs.writeFile(req.file.originalname, ...)` | Generate UUID key, upload to S3 with `ContentType` from magic byte check |
| Accept any Content-Type header from client | Validate actual file content using magic bytes regardless of declared type |
| Store uploads in `public/uploads/` directory | Private S3 bucket; serve via pre-signed URLs with TTL |
| No file size limit | Hard limit on body parser AND object storage max file size |
| Return upload path directly to client | Return the S3 key/ID; generate pre-signed URL on each access request |

## Stack-specific notes
**Node.js:** `multer` handles multipart parsing. Use `multer({ storage: multer.memoryStorage() })` to get a Buffer for magic-byte check before streaming to S3 via `@aws-sdk/client-s3`.
**Python (FastAPI):** `UploadFile` provides `.file` stream. Read first 512 bytes for magic-byte check, then stream the full file to S3 using `boto3` or `aiobotocore`.
**Go:** `r.ParseMultipartForm(10 << 20)` for in-memory handling. Use `r.FormFile("file")` to get a `multipart.File`. Check `http.DetectContentType()` on first 512 bytes.

## Common mistakes
1. **Trusting the `Content-Type` header.** A malicious user can send `Content-Type: image/png` with an executable file. Always check magic bytes after reading the first bytes of the stream.
2. **No size limit on streaming.** Without a size limit, a client can stream many gigabytes of data through the server, exhausting memory or disk. Apply limits before streaming.
3. **Public S3 bucket for user uploads.** Every uploaded file is publicly accessible — including files from other users. Always use private buckets with pre-signed URLs for access.
4. **Allowing SVG without sanitization.** SVG files can contain JavaScript (`<script>` or event handlers). Either reject SVG or sanitize with `DOMPurify` / `svg-sanitizer` before storing.

## Checklist
- [ ] File size limit enforced at body parser level before the stream is read
- [ ] File type validated via magic bytes (not Content-Type header or file extension)
- [ ] Client-supplied filename discarded; UUID-based storage key generated
- [ ] Files stored in a private S3/GCS/R2 bucket — not on local filesystem
- [ ] Pre-signed URLs generated for file access with short TTL (< 1 hour)
- [ ] Allowed file type whitelist defined (not a blacklist)
- [ ] SVG uploads either rejected or sanitized before storage
- [ ] Upload metadata (original filename, uploader ID, timestamp) stored in DB
- [ ] Large file uploads use pre-signed PUT URL or multipart upload
- [ ] Virus scanning configured (if handling untrusted documents)
