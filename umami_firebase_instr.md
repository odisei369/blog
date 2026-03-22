# Umami Analytics + Firebase Views Setup

## Part 1: Umami on Proxmox NAS

### 1. Create Umami LXC with tteck script

From the Proxmox host shell:

```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/umami.sh)"
```

Follow the prompts. The script creates a dedicated LXC with Umami + PostgreSQL pre-configured.

Once done, note the container's IP address (e.g., `10.0.0.50`). Umami runs on port `3000`.
Default login: `admin` / `umami`.

### 2. Create Cloudflare Tunnel LXC with tteck script

A dedicated LXC for `cloudflared` keeps concerns separated — Umami stays untouched, and this tunnel can serve any future services on your LAN.

From the Proxmox host shell:

```bash
bash -c "$(wget -qLO - https://github.com/tteck/Proxmox/raw/main/ct/cloudflared.sh)"
```

Follow the prompts. The script creates a lightweight LXC with `cloudflared` pre-installed.

### 3. Create the tunnel on Cloudflare

1. Go to **Zero Trust > Networks > Tunnels**
2. Click **Create a tunnel**
3. Name it (e.g., `nas-tunnel`)
4. Copy the tunnel token

In the cloudflared LXC shell, install the token:

```bash
cloudflared service install <YOUR_TUNNEL_TOKEN>
```

Add a public hostname in the Cloudflare dashboard:
- Subdomain: `umami` (or your choice)
- Domain: your domain (e.g., `piatrenka.com`)
- Service: `http://10.0.0.50:3000` (the Umami LXC's LAN IP)

### 4. Block everything except tracking endpoints with WAF

The tunnel exposes all of Umami by default. Use a Cloudflare WAF rule to only allow what the blog needs.

1. Go to your domain in Cloudflare dashboard
2. Navigate to **Security > WAF > Custom rules**
3. Create a rule:
   - **Rule name:** `Block Umami dashboard`
   - **Expression:**
     ```
     (http.host eq "umami.piatrenka.com" and not http.request.uri.path eq "/script.js" and not http.request.uri.path eq "/api/send")
     ```
   - **Action:** **Block**
4. Click **Deploy**

Result:
- `umami.piatrenka.com/script.js` — allowed (blog visitors load the tracking script)
- `umami.piatrenka.com/api/send` — allowed (blog visitors send analytics data)
- Everything else — blocked (dashboard, login, API endpoints all return 403)

To access the dashboard, use Umami directly on your LAN at `http://10.0.0.50:3000`.

### 5. Configure Umami

1. Access the dashboard on your LAN: `http://10.0.0.50:3000`
2. Change the default password immediately
3. Go to **Settings > Websites > Add website**
4. Enter your blog URL (e.g., `https://piatrenka.com`)
5. Copy the **Website ID** (a UUID like `a1b2c3d4-e5f6-...`)

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
**Cloudflare WAF custom rules:** Free plan includes 5 custom rules.

For a personal blog, these limits are very generous.
