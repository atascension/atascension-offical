# ConvertKit (Kit) API Integration Guide

A complete reference for wiring the Architech Ascension site's email signup form to ConvertKit's API so that anyone who submits their name and email on the front end is automatically added to a Kit form and enrolled in the connected email automation.

---

## Table of Contents

1. [How It Works — Big Picture](#1-how-it-works--big-picture)
2. [ConvertKit's Current API — Key Facts](#2-convertkits-current-api--key-facts)
3. [Step 1 — Get Your API Key](#3-step-1--get-your-api-key)
4. [Step 2 — Get Your Form ID](#4-step-2--get-your-form-id)
5. [Why You Can't Call the API Directly from the Browser](#5-why-you-cant-call-the-api-directly-from-the-browser)
6. [The Two-Call API Flow](#6-the-two-call-api-flow)
7. [Option A — Netlify Functions (Recommended)](#7-option-a--netlify-functions-recommended)
8. [Option B — Vercel Serverless Functions](#8-option-b--vercel-serverless-functions)
9. [Option C — Cloudflare Workers](#9-option-c--cloudflare-workers)
10. [Frontend Implementation — Button, Modal Form & Fetch](#10-frontend-implementation--button-modal-form--fetch)
11. [Storing Your API Key Securely](#11-storing-your-api-key-securely)
12. [Testing the Integration End-to-End](#12-testing-the-integration-end-to-end)
13. [Error Handling Reference](#13-error-handling-reference)
14. [Rate Limits](#14-rate-limits)
15. [Quick-Reference Cheat Sheet](#15-quick-reference-cheat-sheet)

---

## 1. How It Works — Big Picture

```
User clicks button
      ↓
Modal form appears (name + email)
      ↓
User submits form (JavaScript fetch)
      ↓
Your serverless proxy function receives the data
      ↓
Proxy → POST /v4/subscribers   (creates or updates contact)
Proxy → POST /v4/forms/{id}/subscribers  (adds them to your Kit form)
      ↓
Kit fires the email automation attached to that form
      ↓
Proxy returns success → modal shows "You're in!"
```

The serverless proxy is a ~20-line function that lives on the same hosting provider as your static site. It holds your secret API key on the server side so it is never exposed in the browser.

---

## 2. ConvertKit's Current API — Key Facts

| Detail | Value |
|---|---|
| Brand name | **Kit** (formerly ConvertKit) |
| Current version | **v4** |
| Base URL | `https://api.kit.com/v4/` |
| Auth header | `X-Kit-Api-Key: YOUR_KEY` |
| Content-Type | `application/json` |
| Rate limit (API key) | 120 requests / 60 seconds |
| CORS policy | **No browser access** — requires a server-side proxy |

> **Note:** The old v3 base URL was `api.convertkit.com/v3/`. V4 uses `api.kit.com/v4/`. V3 still works but is considered legacy. Build against v4.

---

## 3. Step 1 — Get Your API Key

1. Log in to [app.kit.com](https://app.kit.com)
2. Go to **Account Settings → Developer** (or visit `https://app.kit.com/account_settings/developer_settings`)
3. Click **"Add a new key"**
4. Give it a name (e.g., `atascension-site`)
5. **Copy the key immediately** — Kit shows it only once

Store the key in your hosting provider's environment variables (see [Section 11](#11-storing-your-api-key-securely)).

---

## 4. Step 2 — Get Your Form ID

1. In Kit, go to **Grow → Landing Pages & Forms**
2. Open the form you want people to be added to
3. Look at the URL — it will be something like `https://app.kit.com/forms/1234567/edit`
4. The number in the URL (`1234567`) is your **Form ID**

You will use this Form ID in the API endpoint URL.

---

## 5. Why You Can't Call the API Directly from the Browser

Browsers enforce **CORS (Cross-Origin Resource Sharing)**. When JavaScript on `atascension.com` tries to call `api.kit.com`, the browser first sends a preflight `OPTIONS` request. Because Kit's API does not return the `Access-Control-Allow-Origin` header allowing your domain, the browser blocks the request entirely.

**The fix:** A serverless function on *your* server receives the form data from the browser, then makes the Kit API call server-to-server (no browser CORS restriction applies). The function returns the result back to your page.

```
Browser ←→ Your Proxy Function ←→ Kit API
  (same origin)      (server-to-server, no CORS)
```

---

## 6. The Two-Call API Flow

Adding someone to a Kit form via v4 requires two sequential API calls.

### Call 1 — Create or Update the Subscriber

**Endpoint:** `POST https://api.kit.com/v4/subscribers`

This is an **upsert** — if the email already exists, it updates the first name; if not, it creates a new contact.

**Request:**
```json
{
  "email_address": "user@example.com",
  "first_name": "Jane",
  "state": "active"
}
```

**Response (201 = new, 200 = updated):**
```json
{
  "subscriber": {
    "id": 353,
    "first_name": "Jane",
    "email_address": "user@example.com",
    "state": "active",
    "created_at": "2024-01-01T00:00:00Z",
    "fields": {}
  }
}
```

---

### Call 2 — Add the Subscriber to Your Form

**Endpoint:** `POST https://api.kit.com/v4/forms/{FORM_ID}/subscribers`

Replace `{FORM_ID}` with the numeric ID from Step 2.

**Request:**
```json
{
  "email_address": "user@example.com"
}
```

**Response (201 = newly added, 200 = already on form):**
```json
{
  "subscriber": {
    "id": 353,
    "first_name": "Jane",
    "email_address": "user@example.com",
    "state": "active",
    "created_at": "2024-01-01T00:00:00Z",
    "added_at": "2024-06-15T12:00:00Z",
    "fields": {}
  }
}
```

After this call succeeds, Kit triggers the email automation attached to that form.

---

## 7. Option A — Netlify Functions (Recommended)

If your site is (or will be) hosted on Netlify, this is the simplest path. No separate backend needed.

### File Structure

```
atascension-official/
├── index.html
├── netlify.toml          ← add this
└── netlify/
    └── functions/
        └── subscribe.js  ← add this
```

### `netlify.toml`

```toml
[build]
  functions = "netlify/functions"
```

### `netlify/functions/subscribe.js`

```javascript
exports.handler = async (event) => {
  if (event.httpMethod !== 'POST') {
    return { statusCode: 405, body: 'Method Not Allowed' };
  }

  const { email, firstName } = JSON.parse(event.body);

  if (!email) {
    return { statusCode: 400, body: JSON.stringify({ error: 'Email is required' }) };
  }

  const API_KEY = process.env.KIT_API_KEY;
  const FORM_ID = process.env.KIT_FORM_ID;
  const headers = {
    'Content-Type': 'application/json',
    'X-Kit-Api-Key': API_KEY,
  };

  // Step 1: Create/upsert subscriber
  const createRes = await fetch('https://api.kit.com/v4/subscribers', {
    method: 'POST',
    headers,
    body: JSON.stringify({
      email_address: email,
      first_name: firstName || '',
      state: 'active',
    }),
  });

  if (!createRes.ok) {
    const err = await createRes.json();
    return { statusCode: 500, body: JSON.stringify({ error: 'Failed to create subscriber', details: err }) };
  }

  // Step 2: Add subscriber to form
  const formRes = await fetch(`https://api.kit.com/v4/forms/${FORM_ID}/subscribers`, {
    method: 'POST',
    headers,
    body: JSON.stringify({ email_address: email }),
  });

  if (!formRes.ok) {
    const err = await formRes.json();
    return { statusCode: 500, body: JSON.stringify({ error: 'Failed to add to form', details: err }) };
  }

  return {
    statusCode: 200,
    headers: { 'Access-Control-Allow-Origin': '*' },
    body: JSON.stringify({ success: true }),
  };
};
```

### Set Environment Variables in Netlify

In the Netlify dashboard → **Site Settings → Environment Variables**, add:

| Key | Value |
|---|---|
| `KIT_API_KEY` | Your v4 API key |
| `KIT_FORM_ID` | Your form's numeric ID |

### Endpoint URL Your Frontend Will Call

```
/.netlify/functions/subscribe
```

---

## 8. Option B — Vercel Serverless Functions

If you prefer Vercel, the approach is nearly identical.

### File Structure

```
atascension-official/
├── index.html
└── api/
    └── subscribe.js   ← add this
```

### `api/subscribe.js`

```javascript
export default async function handler(req, res) {
  if (req.method !== 'POST') {
    return res.status(405).json({ error: 'Method not allowed' });
  }

  const { email, firstName } = req.body;

  if (!email) {
    return res.status(400).json({ error: 'Email is required' });
  }

  const API_KEY = process.env.KIT_API_KEY;
  const FORM_ID = process.env.KIT_FORM_ID;
  const headers = {
    'Content-Type': 'application/json',
    'X-Kit-Api-Key': API_KEY,
  };

  // Step 1: Create/upsert subscriber
  const createRes = await fetch('https://api.kit.com/v4/subscribers', {
    method: 'POST',
    headers,
    body: JSON.stringify({
      email_address: email,
      first_name: firstName || '',
      state: 'active',
    }),
  });

  if (!createRes.ok) {
    return res.status(500).json({ error: 'Failed to create subscriber' });
  }

  // Step 2: Add subscriber to form
  const formRes = await fetch(`https://api.kit.com/v4/forms/${FORM_ID}/subscribers`, {
    method: 'POST',
    headers,
    body: JSON.stringify({ email_address: email }),
  });

  if (!formRes.ok) {
    return res.status(500).json({ error: 'Failed to add to form' });
  }

  return res.status(200).json({ success: true });
}
```

### Set Environment Variables in Vercel

In the Vercel dashboard → **Project Settings → Environment Variables**, add:

| Key | Value |
|---|---|
| `KIT_API_KEY` | Your v4 API key |
| `KIT_FORM_ID` | Your form's numeric ID |

### Endpoint URL Your Frontend Will Call

```
/api/subscribe
```

---

## 9. Option C — Cloudflare Workers

If your site is behind Cloudflare and you want globally distributed edge functions.

### `worker.js`

```javascript
export default {
  async fetch(request, env) {
    if (request.method !== 'POST') {
      return new Response('Method Not Allowed', { status: 405 });
    }

    const { email, firstName } = await request.json();
    const headers = {
      'Content-Type': 'application/json',
      'X-Kit-Api-Key': env.KIT_API_KEY,
    };

    // Step 1: Create/upsert subscriber
    const createRes = await fetch('https://api.kit.com/v4/subscribers', {
      method: 'POST',
      headers,
      body: JSON.stringify({ email_address: email, first_name: firstName || '', state: 'active' }),
    });

    if (!createRes.ok) {
      return new Response(JSON.stringify({ error: 'Failed to create subscriber' }), { status: 500 });
    }

    // Step 2: Add subscriber to form
    const formRes = await fetch(`https://api.kit.com/v4/forms/${env.KIT_FORM_ID}/subscribers`, {
      method: 'POST',
      headers,
      body: JSON.stringify({ email_address: email }),
    });

    if (!formRes.ok) {
      return new Response(JSON.stringify({ error: 'Failed to add to form' }), { status: 500 });
    }

    return new Response(JSON.stringify({ success: true }), {
      headers: { 'Content-Type': 'application/json', 'Access-Control-Allow-Origin': '*' },
    });
  },
};
```

Set `KIT_API_KEY` and `KIT_FORM_ID` as **Secrets** in the Cloudflare Workers dashboard.

---

## 10. Frontend Implementation — Button, Modal Form & Fetch

This is the HTML/CSS/JS you'll add to `index.html`. Adjust the styling to match your existing design system.

### HTML — Button Trigger

Place this inside your `<footer class="flyer-foot">` section or wherever you want the button to appear:

```html
<button class="cta-btn" id="joinBtn">Join the Waitlist</button>
```

### HTML — Modal

Place this just before the closing `</body>` tag:

```html
<div class="modal-overlay" id="joinModal" role="dialog" aria-modal="true" aria-labelledby="modalTitle" hidden>
  <div class="modal-box">
    <button class="modal-close" id="closeModal" aria-label="Close">&times;</button>
    <h2 id="modalTitle">Join the Waitlist</h2>
    <p>Be first to know when we launch. No spam — ever.</p>
    <form id="joinForm" novalidate>
      <label for="firstName">First Name</label>
      <input type="text" id="firstName" name="firstName" placeholder="Jane" autocomplete="given-name" />
      <label for="emailInput">Email Address <span aria-hidden="true">*</span></label>
      <input type="email" id="emailInput" name="email" placeholder="jane@example.com" required autocomplete="email" />
      <button type="submit" id="submitBtn">Get Early Access</button>
      <p class="form-status" id="formStatus" aria-live="polite"></p>
    </form>
  </div>
</div>
```

### JavaScript — Modal Open/Close + Form Submit

Add this inside your existing `<script>` block or in a new one:

```javascript
(function () {
  // ── Change this URL to match your hosting provider ──────────────────────
  // Netlify:   '/.netlify/functions/subscribe'
  // Vercel:    '/api/subscribe'
  // Cloudflare Worker: 'https://your-worker.your-subdomain.workers.dev'
  var SUBSCRIBE_URL = '/.netlify/functions/subscribe';
  // ────────────────────────────────────────────────────────────────────────

  var modal   = document.getElementById('joinModal');
  var openBtn = document.getElementById('joinBtn');
  var closeBtn = document.getElementById('closeModal');
  var form    = document.getElementById('joinForm');
  var status  = document.getElementById('formStatus');
  var submitBtn = document.getElementById('submitBtn');

  function openModal() {
    modal.hidden = false;
    modal.querySelector('input').focus();
  }

  function closeModal() {
    modal.hidden = true;
    form.reset();
    status.textContent = '';
  }

  openBtn.addEventListener('click', openModal);
  closeBtn.addEventListener('click', closeModal);

  // Close on overlay click
  modal.addEventListener('click', function (e) {
    if (e.target === modal) closeModal();
  });

  // Close on Escape key
  document.addEventListener('keydown', function (e) {
    if (e.key === 'Escape' && !modal.hidden) closeModal();
  });

  form.addEventListener('submit', async function (e) {
    e.preventDefault();

    var email     = document.getElementById('emailInput').value.trim();
    var firstName = document.getElementById('firstName').value.trim();

    if (!email) {
      status.textContent = 'Please enter your email address.';
      return;
    }

    submitBtn.disabled = true;
    submitBtn.textContent = 'Sending…';
    status.textContent = '';

    try {
      var res = await fetch(SUBSCRIBE_URL, {
        method: 'POST',
        headers: { 'Content-Type': 'application/json' },
        body: JSON.stringify({ email: email, firstName: firstName }),
      });

      var data = await res.json();

      if (res.ok && data.success) {
        status.textContent = "You're in! Check your inbox for a welcome email.";
        form.reset();
        submitBtn.textContent = 'Done!';
      } else {
        status.textContent = 'Something went wrong. Please try again.';
        submitBtn.disabled = false;
        submitBtn.textContent = 'Get Early Access';
      }
    } catch (err) {
      status.textContent = 'Network error. Please check your connection and try again.';
      submitBtn.disabled = false;
      submitBtn.textContent = 'Get Early Access';
    }
  });
})();
```

---

## 11. Storing Your API Key Securely

**Never put your API key in `index.html`, any `.js` file in your repo, or commit it to git.**

Anyone who views your page source or your GitHub repo would have full access to your Kit account.

The correct approach is **environment variables** on your hosting provider:

| Provider | Where to set | Variable names |
|---|---|---|
| Netlify | Site Settings → Environment Variables | `KIT_API_KEY`, `KIT_FORM_ID` |
| Vercel | Project Settings → Environment Variables | `KIT_API_KEY`, `KIT_FORM_ID` |
| Cloudflare Workers | Workers → Settings → Variables & Secrets | `KIT_API_KEY`, `KIT_FORM_ID` |

The serverless function reads `process.env.KIT_API_KEY` (Node.js) or `env.KIT_API_KEY` (Cloudflare) at runtime — the value is injected by the platform and never appears in your code.

---

## 12. Testing the Integration End-to-End

### Step 1 — Test the proxy function directly with curl

```bash
curl -X POST https://your-site.com/.netlify/functions/subscribe \
  -H "Content-Type: application/json" \
  -d '{"email": "test@yourdomain.com", "firstName": "Test"}'
```

Expected response: `{"success":true}`

### Step 2 — Check Kit dashboard

1. Go to **Grow → Subscribers** in Kit
2. Search for `test@yourdomain.com`
3. Confirm the subscriber exists and is tagged to your form

### Step 3 — Verify the automation fired

1. In Kit, go to **Automate → Sequences** or **Visual Automations**
2. Confirm the test subscriber entered the automation sequence
3. Check **Activity** on the subscriber's profile to see emails sent

### Step 4 — Test the full UI flow

1. Open your site
2. Click the button
3. Fill in name + email with a real address you can check
4. Submit
5. Verify you receive the welcome/sequence email

---

## 13. Error Handling Reference

| HTTP Status | Meaning | What to do |
|---|---|---|
| `200` | Subscriber already existed, updated | Success — treat same as 201 |
| `201` | New subscriber created | Success |
| `400` | Bad request — missing or malformed field | Check `email_address` is present and valid |
| `401` | Invalid or missing API key | Verify `X-Kit-Api-Key` header and key value |
| `404` | Form ID not found | Double-check the Form ID |
| `409` | Duplicate conflict | Already subscribed — can show "you're already on the list" |
| `422` | Unprocessable — invalid parameter value | Check field types match the schema |
| `429` | Rate limit exceeded | Add retry with exponential backoff |
| `500` | Kit server error | Retry after a delay |

---

## 14. Rate Limits

- **120 requests per 60-second rolling window** with API key authentication
- Exceeding the limit returns `429 Too Many Requests`
- At scale, consider batching with the bulk endpoints or upgrading to OAuth (600 req/min)
- A typical sign-up form will never approach this limit for a personal/community site

---

## 15. Quick-Reference Cheat Sheet

```
API Base URL:      https://api.kit.com/v4/
Auth Header:       X-Kit-Api-Key: YOUR_KEY
Content-Type:      application/json

── Create/upsert subscriber ────────────────────────────────────
POST /v4/subscribers
Body: { "email_address": "...", "first_name": "...", "state": "active" }

── Add subscriber to form ──────────────────────────────────────
POST /v4/forms/{FORM_ID}/subscribers
Body: { "email_address": "..." }

── Where to find your Form ID ──────────────────────────────────
Kit dashboard → Grow → Landing Pages & Forms → open form → URL number

── Where to find your API Key ──────────────────────────────────
app.kit.com → Account Settings → Developer → Add a new key

── Frontend calls ──────────────────────────────────────────────
Netlify:    POST /.netlify/functions/subscribe
Vercel:     POST /api/subscribe
CF Worker:  POST https://your-worker.workers.dev

── Environment variables needed ────────────────────────────────
KIT_API_KEY   = your v4 API key
KIT_FORM_ID   = numeric form ID (e.g. 1234567)
```
