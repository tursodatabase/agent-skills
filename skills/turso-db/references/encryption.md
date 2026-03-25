# Encryption

Turso supports page-level encryption at rest.

**Status:** Experimental — requires `--experimental-encryption` flag.

## Getting Started

### 1. Generate a Key

Generate a secure 32-byte hex key:

```bash
openssl rand -hex 32
# produces a 64-character hex string
```

### 2. Create an Encrypted Database

```bash
tursodb --experimental-encryption database.db
```

Then set the cipher and key:

```sql
PRAGMA cipher = 'aegis256';
PRAGMA hexkey = '<your-64-char-hex-key>';
```

### 3. Reopen an Encrypted Database

**IMPORTANT:** To reopen an existing encrypted database, you MUST use URI format:

```bash
tursodb --experimental-encryption \
  "file:database.db?cipher=aegis256&hexkey=<your-64-char-hex-key>"
```

## URI Format

```
file:<path>?cipher=<cipher>&hexkey=<hex_key>
```

Parameters:
- `cipher` — cipher algorithm name (see table below)
- `hexkey` — encryption key in hexadecimal (32 or 64 hex characters depending on cipher)

## Supported Ciphers

| Cipher Name | Algorithm | Key Size |
|-------------|-----------|----------|
| `aes256gcm` | AES-GCM (256-bit) | 32 bytes (64 hex chars) |
| `aes128gcm` | AES-GCM (128-bit) | 16 bytes (32 hex chars) |
| `aegis256` | AEGIS-256 | 32 bytes (64 hex chars) |
| `aegis256x2` | AEGIS-256-X2 | 32 bytes (64 hex chars) |
| `aegis256x4` | AEGIS-256-X4 | 32 bytes (64 hex chars) |
| `aegis128l` | AEGIS-128L | 16 bytes (32 hex chars) |
| `aegis128x2` | AEGIS-128-X2 | 16 bytes (32 hex chars) |
| `aegis128x4` | AEGIS-128-X4 | 16 bytes (32 hex chars) |

## How It Works

- Each page is encrypted/decrypted individually
- A new random nonce is generated for every page write
- Authentication tag and nonce are stored in the page's reserved space
- Page 1 header (first 100 bytes) is NOT encrypted but IS authenticated (used as Additional Data)
- The key is never stored — every connection must provide it

## Example: PRAGMA-Based Setup

```sql
-- At database creation
PRAGMA cipher = 'aegis256';
PRAGMA hexkey = '<your-64-char-hex-key>';

-- Now use the database normally
CREATE TABLE secrets (id INTEGER PRIMARY KEY, data TEXT);
INSERT INTO secrets VALUES (1, 'sensitive information');
SELECT * FROM secrets;
```

## Important Notes

- The `--experimental-encryption` flag must be passed to `tursodb` CLI
- Opening an encrypted database without the correct key returns an error
- Key rotation requires rewriting the entire database
- Cipher information is stored in the database file header (replacing SQLite magic bytes)

## Remote Encryption (SDK)

SDKs support encrypting data synced to Turso Cloud via `remoteEncryption` configuration. Remote encryption uses the same cipher algorithms but key format varies by SDK:

- **Rust / Go**: hex-encoded key
- **JavaScript / WASM / React Native**: base64-encoded key

See SDK-specific documentation for configuration details.
