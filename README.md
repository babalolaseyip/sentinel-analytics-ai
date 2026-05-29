# ThreatSentinel SaaS — Setup & Deployment Guide

Converting ThreatSentinel from a Streamlit demo into a production SaaS.  
**Total setup time: 2–3 days. Zero infrastructure expertise required.**

---

## Architecture Overview

```
sentinelanalyticsai.com (GitHub Pages)
├── index.html         ← Marketing site (already built)
├── platform.html
├── login.html         ← NEW: Supabase auth
├── signup.html        ← NEW: 14-day trial signup
├── dashboard.html     ← NEW: Full SaaS product UI
└── ...

api.sentinelanalyticsai.com (Railway / Render)
└── backend/
    ├── main.py        ← FastAPI app (all endpoints)
    ├── auth.py        ← Supabase JWT verification
    ├── billing.py     ← Stripe subscriptions
    └── threat_model.py ← Your ML model wrapper
```

---

## STEP 1 — Set up Supabase (auth + database)

**Time: 30 minutes · Cost: Free**

1. Go to [supabase.com](https://supabase.com) and create a new project
   - Project name: `sentinel-analytics`
   - Database password: save this securely
   - Region: `us-east-1` (closest to NJ/NY)

2. In your Supabase project → **Settings → API**, copy:
   - `Project URL` → `SUPABASE_URL`
   - `anon public` key → `SUPABASE_ANON_KEY`
   - `service_role` key → `SUPABASE_SERVICE_ROLE_KEY`
   - `JWT Secret` → `SUPABASE_JWT_SECRET`

3. In **Authentication → Settings**:
   - Enable **Email/Password** sign-in
   - Enable **Google OAuth** (optional — add Google OAuth credentials)
   - Set `Site URL` to `https://sentinelanalyticsai.com`
   - Add `https://sentinelanalyticsai.com/dashboard.html` to Redirect URLs

4. In both `login.html` and `signup.html`, replace:
   ```javascript
   const SUPABASE_URL = 'https://YOUR_PROJECT_ID.supabase.co';
   const SUPABASE_ANON_KEY = 'YOUR_ANON_KEY';
   ```
   with your actual credentials.

---

## STEP 2 — Set up Stripe (billing)

**Time: 45 minutes · Cost: 2.9% + 30¢ per transaction**

1. Go to [stripe.com](https://stripe.com) and create an account using your business email

2. In Stripe Dashboard → **Products**, create two products:
   
   **Starter — $299/month**
   - Name: `ThreatSentinel Starter`
   - Pricing: $299.00 / month, recurring
   - Copy the Price ID → `STRIPE_PRICE_STARTER`

   **Professional — $799/month**
   - Name: `ThreatSentinel Professional`
   - Pricing: $799.00 / month, recurring
   - Copy the Price ID → `STRIPE_PRICE_PROFESSIONAL`

3. In **Developers → API Keys**, copy:
   - Secret key → `STRIPE_SECRET_KEY`

4. In **Developers → Webhooks**, add endpoint:
   - URL: `https://api.sentinelanalyticsai.com/api/billing/webhook`
   - Events to listen for:
     - `checkout.session.completed`
     - `customer.subscription.updated`
     - `customer.subscription.deleted`
     - `invoice.payment_failed`
   - Copy the Webhook Signing Secret → `STRIPE_WEBHOOK_SECRET`

---

## STEP 3 — Connect your ML models

**Time: 1–2 hours**

Open `backend/threat_model.py` and find the `_load_models()` method.

If your models are saved as `.pkl` files (scikit-learn):
```python
import joblib

def _load_models(self):
    self.iso_forest = joblib.load("models/isolation_forest.pkl")
    self.oc_svm = joblib.load("models/oc_svm.pkl")
    self.pca_ae = joblib.load("models/pca_autoencoder.pkl")
    self.preprocessor = joblib.load("models/preprocessor.pkl")
    self.is_loaded = True
```

Then update `_real_scan()` to use your actual model:
```python
def _real_scan(self, scan_id, events, sensitivity):
    df = self.preprocessor.transform(pd.DataFrame(events))
    
    iso_scores = self.iso_forest.score_samples(df)    # lower = more anomalous
    svm_preds  = self.oc_svm.predict(df)              # -1 = anomaly
    ae_errors  = self.pca_ae.reconstruction_error(df) # higher = more anomalous
    
    # Ensemble
    ensemble = (
        0.40 * self._normalize(-iso_scores) +
        0.35 * (svm_preds == -1).astype(float) +
        0.25 * self._normalize(ae_errors)
    )
    
    anomaly_indices = np.where(ensemble > sensitivity)[0]
    # ... map to threat objects and return
```

---

## STEP 4 — Deploy the backend to Railway

**Time: 20 minutes · Cost: $5/month**

Railway is the simplest Python host. No Docker required.

1. Go to [railway.app](https://railway.app) and sign in with GitHub

2. Click **New Project → Deploy from GitHub repo**
   - Select your backend repository (or push the `backend/` folder to a new GitHub repo)

3. In Railway → **Variables**, add all your environment variables from `.env.example`

4. Railway auto-detects FastAPI. Add a `Procfile` in your backend root:
   ```
   web: uvicorn main:app --host 0.0.0.0 --port $PORT
   ```

5. In Railway → **Settings → Networking**, add a custom domain:
   - `api.sentinelanalyticsai.com`
   - Add the CNAME record Railway gives you to your Namecheap DNS

6. Your API is now live at `https://api.sentinelanalyticsai.com/api/docs`

**Alternative: Render.com** (also free tier available)
```yaml
# render.yaml
services:
  - type: web
    name: sentinel-api
    env: python
    buildCommand: pip install -r requirements.txt
    startCommand: uvicorn main:app --host 0.0.0.0 --port $PORT
```

---

## STEP 5 — Connect frontend to backend

In `dashboard.html`, update the API base URL:
```javascript
const API_BASE = 'https://api.sentinelanalyticsai.com';

// Example: load dashboard data on page load
async function loadDashboard() {
    const session = await supabase.auth.getSession();
    const token = session.data.session?.access_token;
    
    const resp = await fetch(`${API_BASE}/api/dashboard`, {
        headers: { 'Authorization': `Bearer ${token}` }
    });
    const data = await resp.json();
    // Update UI with real data
}
```

---

## STEP 6 — Update website nav links

In `login.html` and `signup.html`, the "Book a demo" button should point to your Calendly link:
```html
<a href="https://calendly.com/your-link" class="btn btn-white-sm">Book a demo</a>
```

---

## Testing Checklist

- [ ] Sign up at `/signup.html` → account created in Supabase dashboard
- [ ] Sign in at `/login.html` → redirects to `/dashboard.html`
- [ ] Dashboard loads → metrics, threat feed, PRs visible
- [ ] Click "Run scan" → scan executes, results appear
- [ ] Click "Merge" on a PR → status updates
- [ ] Click "Billing & Plan" → redirects to Stripe checkout
- [ ] Complete test payment (use Stripe card `4242 4242 4242 4242`)
- [ ] Plan updates in Supabase user metadata after payment
- [ ] Sign out → redirects to login page

---

## Security checklist before launch

- [ ] `SUPABASE_SERVICE_ROLE_KEY` is only used server-side, never in frontend HTML
- [ ] `STRIPE_SECRET_KEY` is only in backend `.env`, never exposed to browser
- [ ] CORS is restricted to your domain only (`ALLOWED_ORIGINS`)
- [ ] All protected API routes use `Depends(get_current_user)`
- [ ] Stripe webhook verifies signature before processing
- [ ] Rate limiting added to `/api/scan` endpoint (add `slowapi` package)
- [ ] PHI data is never logged in plaintext (check your ML preprocessing)

---

## Monthly cost summary (at launch)

| Service | Cost |
|---|---|
| Supabase (auth + database) | Free (up to 500 users) |
| Railway (backend hosting) | $5/month |
| GitHub Pages (frontend) | Free |
| Stripe (billing) | 2.9% + 30¢ per transaction |
| Google Workspace (email) | $6/month |
| Namecheap (domain) | $1.25/month |
| **Total fixed** | **~$12/month** |

Revenue from your **first paying customer** covers 25× your monthly infrastructure cost.

---

## Support

Questions about setup? Email: contact@sentinelanalyticsai.com
