# Umami Analytics + Firebase Views Setup

## Part 1: Umami on Proxmox NAS

### 1. Create an LXC container on Proxmox

In the Proxmox web UI:

1. Download a Debian 12 (or Ubuntu 22.04) container template if you don't have one
2. Create a new CT (container):
   - Template: Debian 12
   - Disk: 8 GB is plenty
   - Memory: 512 MB (1 GB if generous)
   - CPU: 1 core
   - Network: DHCP or static IP on your LAN
3. Start the container and open a shell

### 2. Install Docker inside the LXC

```bash
apt update && apt upgrade -y
apt install -y curl ca-certificates gnupg

# Add Docker repo
install -m 0755 -d /etc/apt/keyrings
curl -fsSL https://download.docker.com/linux/debian/gpg -o /etc/apt/keyrings/docker.asc
chmod a+r /etc/apt/keyrings/docker.asc
echo "deb [arch=$(dpkg --print-architecture) signed-by=/etc/apt/keyrings/docker.asc] https://download.docker.com/linux/debian $(. /etc/os-release && echo "$VERSION_CODENAME") stable" > /etc/apt/sources.list.d/docker.list

apt update
apt install -y docker-ce docker-ce-cli containerd.io docker-compose-plugin
```

> **Note:** If Docker fails to start in an unprivileged LXC, you may need to enable nesting. In Proxmox: CT > Options > Features > check "Nesting".

### 3. Deploy Umami with Docker Compose

```bash
mkdir -p /opt/umami && cd /opt/umami
```

Create `docker-compose.yml`:

```yaml
services:
  umami:
    image: ghcr.io/umami-software/umami:latest
    ports:
      - "3000:3000"
    environment:
      DATABASE_URL: postgresql://umami:umami@db:5432/umami
      APP_SECRET: CHANGE_THIS_TO_A_RANDOM_STRING
    depends_on:
      db:
        condition: service_healthy
    init: true
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "curl http://localhost:3000/api/heartbeat"]
      interval: 5s
      timeout: 5s
      retries: 5

  db:
    image: postgres:15-alpine
    environment:
      POSTGRES_DB: umami
      POSTGRES_USER: umami
      POSTGRES_PASSWORD: umami
    volumes:
      - umami-db-data:/var/lib/postgresql/data
    restart: always
    healthcheck:
      test: ["CMD-SHELL", "pg_isready -U $${POSTGRES_USER} -d $${POSTGRES_DB}"]
      interval: 5s
      timeout: 5s
      retries: 5

volumes:
  umami-db-data:
```

Generate a random APP_SECRET:

```bash
openssl rand -hex 32
```

Start it:

```bash
docker compose up -d
```

Umami is now running at `http://<container-ip>:3000`. Default login: `admin` / `umami`.

### 4. Expose via Cloudflare Tunnel

On your Cloudflare dashboard:

1. Go to **Zero Trust > Networks > Tunnels**
2. Create a new tunnel, name it (e.g., `nas-tunnel`)
3. Install the connector in your LXC:

```bash
curl -L --output cloudflared.deb https://github.com/cloudflare/cloudflared/releases/latest/download/cloudflared-linux-amd64.deb
dpkg -i cloudflared.deb
cloudflared service install <YOUR_TUNNEL_TOKEN>
```

4. In Cloudflare dashboard, add a public hostname:
   - Subdomain: `umami` (or whatever you want)
   - Domain: your domain (e.g., `piatrenka.com`)
   - Service: `http://localhost:3000`

Umami is now accessible at `https://umami.piatrenka.com` (or whatever subdomain you chose).

### 5. Configure Umami

1. Log in at your Umami URL (change the default password!)
2. Go to **Settings > Websites > Add website**
3. Enter your blog URL (e.g., `https://piatrenka.com`)
4. Copy the **Website ID** (a UUID like `a1b2c3d4-e5f6-...`)

### 6. Update blog config

In `config/_default/params.toml`, replace the placeholder values:

```toml
[umamiAnalytics]
  websiteid = "YOUR_WEBSITE_ID_HERE"
  domain = "umami.piatrenka.com"
  enableTrackEvent = true
```

---

## Part 2: Firebase Views Counter

### 1. Create a Firebase project

1. Go to https://console.firebase.google.com
2. Click **Add project**
3. Name it (e.g., `piatrenka-blog`)
4. Disable Google Analytics for the project (we're using Umami instead)
5. Click **Create project**

### 2. Get Firebase config

1. In the Firebase console, click the **gear icon > Project settings**
2. Under **Your apps**, click the web icon (`</>`) to add a web app
3. Name it (e.g., `blog`)
4. Copy the `firebaseConfig` object — you need these values:

```javascript
const firebaseConfig = {
  apiKey: "AIza...",
  authDomain: "piatrenka-blog.firebaseapp.com",
  projectId: "piatrenka-blog",
  storageBucket: "piatrenka-blog.firebasestorage.app",
  messagingSenderId: "123456789",
  appId: "1:123456789:web:abc123",
  measurementId: "G-XXXXXXX"
};
```

### 3. Enable Anonymous Authentication

1. In Firebase console: **Build > Authentication**
2. Click **Get started**
3. Go to **Sign-in method** tab
4. Click **Anonymous** and enable it

### 4. Create Firestore Database

1. In Firebase console: **Build > Firestore Database**
2. Click **Create database**
3. Select **Start in production mode**
4. Choose a location close to your audience (e.g., `europe-west1` for EU)
5. Click **Create**

### 5. Set security rules

In Firestore: **Rules** tab, replace the default rules with:

```
rules_version = '2';
service cloud.firestore {
  match /databases/{database}/documents {
    match /views/{document} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
                    && request.resource.data.keys().hasOnly(['views'])
                    && request.resource.data.views == 1;
      allow update: if request.auth != null
                    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['views'])
                    && request.resource.data.views == resource.data.views + 1;
    }

    match /likes/{document} {
      allow read: if request.auth != null;
      allow create: if request.auth != null
                    && request.resource.data.keys().hasOnly(['likes'])
                    && request.resource.data.likes == 1;
      allow update: if request.auth != null
                    && request.resource.data.diff(resource.data).affectedKeys().hasOnly(['likes'])
                    && (request.resource.data.likes == resource.data.likes + 1
                        || request.resource.data.likes == resource.data.likes - 1)
                    && request.resource.data.likes >= 0;
    }

    match /{document=**} {
      allow read, write: if false;
    }
  }
}
```

Click **Publish**.

### 6. Update blog config

In `config/_default/params.toml`, replace the placeholder values with your Firebase config:

```toml
[firebase]
  apiKey = "AIza..."
  authDomain = "piatrenka-blog.firebaseapp.com"
  projectId = "piatrenka-blog"
  storageBucket = "piatrenka-blog.firebasestorage.app"
  messagingSenderId = "123456789"
  appId = "1:123456789:web:abc123"
  measurementId = "G-XXXXXXX"
```

---

## Free tier limits

**Firestore (Spark plan):** 50K reads/day, 20K writes/day, 1 GB storage.
**Anonymous Auth:** 50K monthly active users.

For a personal blog, these limits are very generous.
