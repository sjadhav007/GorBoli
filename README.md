# Secure Multi-App Hosting Documentation

**Goal:** Host multiple internal applications (Streamlit & Flask) under a single secure domain (`https://brokeagebiapp-dev.rxo.com`) using Microsoft Azure AD for Single Sign-On (SSO).

### 1. Architecture Overview

1. **Nginx (Reverse Proxy):** Handles SSL encryption and forwards valid traffic. It acts as the single entry point.
2. **OAuth2-Proxy (Authentication):** Intercepts traffic to enforce Microsoft Azure Login. It injects user identity headers (`X-Forwarded-Email`) into requests.
3. **Application Routing:**
* `https://.../`  **Streamlit** (Main Dashboard) on Port `5050`.
* `https://.../sales/`  **Flask** (Sales App) on Port `8080`.



---

To generate keys and certificates to a specific path, you use the `-out` flag (and `-keyout` for the key) followed by the full file path.

Ensure the target directory exists before running these commands, as OpenSSL will not create folders for you.

### 1. The "All-in-One" Command (Best for Local Dev)

This generates a **Private Key** and a **Self-Signed Certificate** in a single step without a password (using `-nodes`).

```bash
openssl req -x509 -newkey rsa:2048 -keyout /path/to/folder/private.key -out /path/to/folder/certificate.crt -days 365 -nodes

```

* **`-keyout`**: Where to save the private key.
* **`-out`**: Where to save the certificate.
* **`-nodes`**: Skips password protection for the key (easier for automated scripts/servers).

### 2. Generate Only a Private Key

If you just need the key file:

```bash
openssl genrsa -out /path/to/folder/private.key 2048

```

### 3. Generate a CSR (Certificate Signing Request)

If you already have a key and need a CSR to send to a Certificate Authority (CA):

```bash
openssl req -new -key /path/to/folder/private.key -out /path/to/folder/request.csr

```


### 2. Configuration Files

#### A. Orchestrator (`multiAppRun.bat`)

**Purpose:** One-click startup script that stops old processes and launches all services in the correct order.
**Location:** `D:\selfServedReporting_20260120\multiAppRun.bat`

```batch
@echo off
echo ========================================================
echo STOPPING OLD SERVICES
echo ========================================================
taskkill /F /IM nginx.exe /T >nul 2>&1
taskkill /F /IM oauth2-proxy.exe /T >nul 2>&1
taskkill /F /IM python.exe /T >nul 2>&1

echo.
echo ========================================================
echo STARTING FLASK (Sales App - Port 8080)
echo ========================================================
start "" cmd /k "cd /d C:\PythonScripts\salessummarizer_nginx_20260204 && call D:\PyEnvs\vertexAISalesEnv\Scripts\activate.bat && python flaskApp.py"

echo.
echo ========================================================
echo STARTING STREAMLIT (Main App - Port 5050)
echo ========================================================
start "" cmd /k "cd /d D:\selfServedReporting_20260120 && call D:\PyEnvs\vertexAISalesEnv\Scripts\activate.bat && streamlit run main.py --server.port 5050 --server.headless true"

echo.
echo ========================================================
echo STARTING OAUTH2-PROXY (Port 4180)
echo ========================================================
start "" cmd /k "cd /d D:\selfServedReporting_20260120 && .\oauth2-proxy-windows\oauth2-proxy.exe --config=oauth2-proxy.cfg"

echo.
echo ========================================================
echo STARTING NGINX (Ports 80 and 443)
echo ========================================================
cd /d C:\Users\sunil.jadhav\Downloads\nginx-1.29.4
start nginx

echo.
echo ========================================================
echo SYSTEM READY
echo 1. Main App: https://brokeagebiapp-dev.rxo.com/
echo 2. Sales App: https://brokeagebiapp-dev.rxo.com/sales/
echo ========================================================
pause

```

#### B. Traffic Controller (`oauth2-proxy.cfg`)

**Purpose:** Defines routing rules and Azure AD connection settings.
**Location:** `D:\selfServedReporting_20260120\oauth2-proxy.cfg`

```ini
## OAuth2 Proxy Config
provider = "azure"
client_id = "YOUR_CLIENT_ID"
client_secret = "YOUR_CLIENT_SECRET"
oidc_issuer_url = "https://login.microsoftonline.com/YOUR_TENANT_ID/v2.0"

email_domains = ["*"]
http_address = "0.0.0.0:4180"
cookie_secret = "YOUR_COOKIE_SECRET"
cookie_secure = true
cookie_domains = ["brokeagebiapp-dev.rxo.com"]

# Routing Rules (Specific paths MUST be listed before generic ones)
upstreams = [
    "http://127.0.0.1:8080/sales/",   # Flask App
    "http://127.0.0.1:5050/"          # Streamlit App (Catch-all)
]

pass_access_token = true
pass_authorization_header = true
set_xauthrequest = true

```

#### C. Front Door (`nginx.conf`)

**Purpose:** SSL termination and header buffer management.
**Location:** `C:\Users\sunil.jadhav\Downloads\nginx-1.29.4\conf\nginx.conf`

```nginx
worker_processes 1;
events { worker_connections 1024; }

http {
    include       mime.types;
    default_type  application/octet-stream;

    # Fix for large Azure Auth Headers (Prevents 400 Bad Request)
    client_header_buffer_size 64k;
    large_client_header_buffers 4 128k;
    proxy_buffer_size   128k;
    proxy_buffers       4 256k;
    proxy_busy_buffers_size 256k;

    # HTTP Redirect
    server {
        listen 80;
        server_name brokeagebiapp-dev.rxo.com 127.0.0.1;
        return 301 https://brokeagebiapp-dev.rxo.com$request_uri;
    }

    # HTTPS Server
    server {
        listen 443 ssl;
        server_name brokeagebiapp-dev.rxo.com;

        ssl_certificate      "path/to/cert.pem";
        ssl_certificate_key  "path/to/key.pem";

        location / {
            proxy_pass http://127.0.0.1:4180;
            proxy_set_header Host $host;
            proxy_set_header X-Real-IP $remote_addr;
            proxy_set_header X-Forwarded-Proto https;
            
            # WebSocket Support
            proxy_http_version 1.1;
            proxy_set_header Upgrade $http_upgrade;
            proxy_set_header Connection "upgrade";
        }
    }
}

```

---

### 3. Application Code Modifications

#### A. Flask App (`flaskApp.py`)

**Changes Made:**

1. **Middleware:** Added `PrefixMiddleware` to strip the `/sales` prefix so the app handles routes as `/`.
2. **Routes:** Defined as standard root paths (e.g., `/api/chat`, NOT `/sales/api/chat`).
3. **Auth:** Updated `authenticate_user_flask` to read `X-Forwarded-Email` headers instead of URL parameters.

```python
# Middleware Implementation
class PrefixMiddleware(object):
    def __init__(self, app, prefix=''):
        self.app = app
        self.prefix = prefix

    def __call__(self, environ, start_response):
        if environ['PATH_INFO'].startswith(self.prefix):
            environ['PATH_INFO'] = environ['PATH_INFO'][len(self.prefix):]
            environ['SCRIPT_NAME'] = self.prefix
            return self.app(environ, start_response)
        else:
            return self.app(environ, start_response)

# Apply Middleware
app.wsgi_app = PrefixMiddleware(app.wsgi_app, prefix='/sales')

if __name__ == '__main__':
    app.run(host='0.0.0.0', port=8080)

```

#### B. JavaScript (`app.js`)

**Changes Made:**

1. **API Paths:** Updated all `fetch` calls to include the `/sales` prefix so Nginx routes them correctly.

```javascript
// Old
fetch('/api/chat', ...)

// New
fetch('/sales/api/chat', ...)

```

---

### 4. Troubleshooting Guide

| Issue | Symptom | Fix |
| --- | --- | --- |
| **502 Bad Gateway** | "No connection could be made" in logs. | Flask or Streamlit is not running. check if the batch file windows closed unexpectedly. |
| **404 Not Found** | API call fails. | 1. Check `oauth2-proxy.cfg` order (Sales first).<br>

<br>2. Ensure Flask routes do **not** have `/sales` prefix.<br>

<br>3. Ensure JS `fetch` calls **do** have `/sales` prefix. |
| **400 Bad Request** | Browser error on login. | Nginx buffer size is too small. Verify `large_client_header_buffers 4 128k;` is in `nginx.conf`. |
| **Login Loop** | Repeats Microsoft login. | `client_secret` is invalid or expired in Azure AD. |

---

### 5. Deployment Steps

1. **Update Paths:** Ensure all paths in `multiAppRun.bat` match your actual folder structure.
2. **Certificates:** Place SSL `cert.pem` and `key.pem` in the Nginx `conf/ssl/` folder.
3. **Run:** Double-click `multiAppRun.bat`.
4. **Verify:** Open `https://brokeagebiapp-dev.rxo.com/` (Streamlit) and `https://brokeagebiapp-dev.rxo.com/sales/` (Flask).
