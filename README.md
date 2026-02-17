# Self-Host Sync Setup Changes

This document summarizes all changes made to enable self-hosted Logseq sync with custom AWS Cognito credentials.

## Files Modified

### 1. `src/main/frontend/config.cljs`

**Purpose:** Configure Cognito credentials and sync server URLs for self-hosted setup.

**Changes:**
- Replaced the conditional `if ENABLE-FILE-SYNC-PRODUCTION` block with a simple `do` block containing custom credentials
- Added custom Cognito credentials:
  ```clojure
  (def LOGIN-URL "https://<your-domain>.auth.<region>.amazoncognito.com/login?client_id=<client-id>&response_type=code&scope=email+openid+phone&redirect_uri=logseq%3A%2F%2Fauth-callback")
  (def COGNITO-IDP "https://cognito-idp.<region>.amazonaws.com/")
  (def COGNITO-CLIENT-ID "<your-client-id>")
  (def REGION "<your-region>")
  (def USER-POOL-ID "<your-user-pool-id>")
  (def IDENTITY-POOL-ID "<your-identity-pool-id>")
  (def OAUTH-DOMAIN "<your-oauth-domain>")
  ```
- Changed sync server URLs to use HTTPS with Tailscale hostname:
  ```clojure
  (defonce db-sync-ws-url "wss://<hostname>.<tailnet>.ts.net:8787/sync/%s")
  (defonce db-sync-http-base "https://<hostname>.<tailnet>.ts.net:8787")
  ```

### 2. `src/main/frontend/handler/user.cljs`

**Purpose:** Bypass the user group check that restricts remote sync to Logseq's paid users.

**Changes:**
- Modified `rtc-group?` function to always return `true`:
  ```clojure
  (defn rtc-group?
    []
    true)
  ```
- Original code checked for "team" or "rtc_2025_07_10" groups from Logseq's API server

### 3. `src/main/frontend/components/repo.cljs`

**Purpose:** Improve error handling and logging for remote graph creation.

**Changes:**
- Added console logging and user-facing error notifications in `new-db-graph` component
- Added logging to `ensure-e2ee-rsa-key-for-cloud!` function for debugging
- Fixed issue where `set-creating-db?` wasn't being reset when cloud sync was disabled

### 4. `deps/db-sync/src/logseq/db_sync/node/config.cljs`

**Purpose:** Add SSL configuration support for HTTPS.

**Changes:**
- Added SSL key and cert path configuration:
  ```clojure
  :ssl-key (env-value env "SSL_KEY_PATH")
  :ssl-cert (env-value env "SSL_CERT_PATH")
  ```
- Added `:ssl-key` and `:ssl-cert` to allowed config keys

### 5. `deps/db-sync/src/logseq/db_sync/node/server.cljs`

**Purpose:** Add HTTPS server support with SSL certificates.

**Changes:**
- Added imports for `https` and `fs` modules
- Modified `start!` function to:
  - Check for SSL key and cert files
  - Create HTTPS server when SSL is configured, HTTP otherwise
  - Use appropriate scheme (`https` or `http`) in request handling
  - Log the server URL on startup

## Files Created

### 1. `run-selfhost.sh`

**Purpose:** Start both the sync server and webapp with HTTPS using Tailscale certificates.

**Features:**
- Uses Tailscale-provided Let's Encrypt certificates
- Starts sync server with SSL environment variables
- Starts web server with HTTPS using `serve --ssl-cert --ssl-key`

**Usage:**
```bash
./run-selfhost.sh [port]  # Default port is 3000
```

### 2. `rebuild-selfhost.sh`

**Purpose:** Rebuild both the sync server and webapp after code changes.

**Usage:**
```bash
./rebuild-selfhost.sh
```

### 3. `certs/` directory

**Purpose:** Store SSL certificates from Tailscale.

**Contents:**
- `<hostname>.<tailnet>.ts.net.crt` - Let's Encrypt certificate (from Tailscale)
- `<hostname>.<tailnet>.ts.net.key` - Private key for the certificate

## Prerequisites

### AWS Cognito Setup

1. **User Pool** - Create a user pool with email sign-in
2. **App Client** - Create an app client WITHOUT a client secret (browser apps can't keep secrets)
3. **App Client Settings** - Configure:
   - Callback URLs: `logseq://auth-callback` and `https://<hostname>.<tailnet>.ts.net:<port>`
   - Allowed OAuth flows: Authorization code grant
   - Scopes: `email`, `openid`, `phone`
4. **Identity Pool** - Create an identity pool linked to your user pool
5. **Domain** - Configure a domain for OAuth (e.g., `your-app.auth.region.amazoncognito.com`)

### Tailscale Setup (for HTTPS)

1. **Install Tailscale** on your Windows host
2. **Enable HTTPS certificates** in Tailscale admin console (Settings → HTTPS)
3. **Generate certificates** on Windows:
   ```powershell
   tailscale cert
   ```
4. **Copy certificates** to WSL `certs/` directory:
   - `<hostname>.<tailnet>.ts.net.crt`
   - `<hostname>.<tailnet>.ts.net.key`

### Port Forwarding (Windows → WSL)

Forward these ports from Windows to WSL:
- `<web-port>` (default: 3000) - Web app
- `8787` - Sync server

Example (run in Windows PowerShell as Admin):
```powershell
netsh interface portproxy add v4tov4 listenport=3000 listenaddress=0.0.0.0 connectport=3000 connectaddress=<wsl-ip>
netsh interface portproxy add v4tov4 listenport=8787 listenaddress=0.0.0.0 connectport=8787 connectaddress=<wsl-ip>
```

## Build Requirements

1. **Install dependencies:**
   ```bash
   yarn install --frozen-lockfile
   cd deps/db-sync && yarn install --frozen-lockfile
   ```

2. **Build the sync server:**
   ```bash
   cd deps/db-sync
   COGNITO_ISSUER=https://cognito-idp.<region>.amazonaws.com/<user-pool-id> \
   COGNITO_CLIENT_ID=<client-id> \
   COGNITO_JWKS_URL=https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/jwks.json \
   DB_SYNC_PORT=8787 \
   yarn build:node-adapter
   ```

3. **Build the webapp:**
   ```bash
   ENABLE_DB_SYNC_LOCAL=true yarn release
   ```

## Running

### Manual Start

```bash
./run-selfhost.sh [port]  # Default port is 3000
```

### Systemd Service (Auto-start on Boot)

Create a systemd service for automatic startup:

```bash
sudo tee /etc/systemd/system/logseq-sync.service > /dev/null << 'EOF'
[Unit]
Description=Logseq Self-Host Sync Server
After=network.target

[Service]
Type=simple
User=<your-username>
WorkingDirectory=/home/<your-username>/logseq
Environment=PATH=/home/<your-username>/.nvm/versions/node/<node-version>/bin:/usr/local/sbin:/usr/local/bin:/usr/sbin:/usr/bin:/sbin:/bin
Environment=COGNITO_ISSUER=https://cognito-idp.<region>.amazonaws.com/<user-pool-id>
Environment=COGNITO_CLIENT_ID=<client-id>
Environment=COGNITO_JWKS_URL=https://cognito-idp.<region>.amazonaws.com/<user-pool-id>/.well-known/jwks.json
Environment=DB_SYNC_PORT=8787
ExecStart=/home/<your-username>/logseq/run-selfhost.sh 9005
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
EOF
```

**Important:** Replace:
- `<your-username>` - Your WSL username
- `<node-version>` - Your Node.js version (check with `node --version`, e.g., `v25.2.1`)
- Cognito credentials with your values

**Enable and start:**
```bash
sudo systemctl daemon-reload
sudo systemctl enable logseq-sync
sudo systemctl start logseq-sync
```

**Useful commands:**
```bash
# Check status
sudo systemctl status logseq-sync

# View logs
sudo journalctl -u logseq-sync -f

# Restart
sudo systemctl restart logseq-sync

# Stop
sudo systemctl stop logseq-sync
```

### Access

Open `https://<hostname>.<tailnet>.ts.net:<port>` in your browser

Create an account and sign in, then create a new graph with "Use Logseq Sync" checked.

## HTTPS Requirements

HTTPS is required for:
- **OPFS (Origin Private File System)** - Used by SQLite for database storage
- **Secure Context** - Required by browsers for certain APIs
- **Remote access** - Accessing from other devices (Tailscale, etc.)

### Why Tailscale Certificates?

Tailscale provides **real Let's Encrypt certificates** for your tailnet machines:
- ✅ No browser certificate warnings
- ✅ Trusted by all devices
- ✅ Auto-renewal handled by Tailscale
- ✅ Works from any device on your tailnet

## Architecture

```
[Other Devices] ──Tailscale──> [Windows: <hostname>.ts.net] ──port forward──> [WSL: sync server + webapp]
```

## Troubleshooting

### Certificate Not Found Error
- Run `tailscale cert` on Windows to generate certificates
- Copy both `.crt` and `.key` files to the `certs/` directory
- Ensure filenames match the hostname in `config.cljs`

### SQLite/OPFS Errors
- Ensure you're using HTTPS (not HTTP)
- Check that the hostname in the URL matches the certificate
- Verify you're accessing via the Tailscale hostname

### CORS Errors
- Ensure both webapp and sync server are using the same hostname
- Check that HTTPS is used for both

### JWT Verification Failures
- Check that Cognito credentials match between the webapp and sync server
- Verify the User Pool ID and Client ID are correct

### RSA Key Errors
- E2EE requires the sync server to have correct Cognito credentials
- Check server logs for authentication errors

### Graph Creation Silently Fails
- Check browser console for error messages
- Verify sync server is running at `https://<hostname>.ts.net:8787/health`

### Connection Refused
- Verify both servers are running
- Check that port forwarding is configured (Windows → WSL)
- Ensure Tailscale is connected
- Verify firewall allows the ports
