# MiracardiacHealth – Telemedicine + Online Hospital + Pharmacy Platform

Complete upgrade of the MiracardiacHealth static PWA into a full telemedicine, appointment, support and pharmacy marketplace platform for Nigeria.

## Tech Stack
- Plain HTML + Tailwind CSS (CDN) + Vanilla JS (ES modules)
- Firebase Auth (Google + Email), Firestore, Storage
- Paystack (inline popup + server-side verification via Cloud Functions)
- Jitsi Meet (optional video calls)
- Progressive Web App (PWA) ready for GitHub Pages

## Project Structure
```
miracardiachealth/
├── index.html                 # Landing page
├── Dashboard.html             # Role-based dashboard
├── admin.html                 # Admin panel
├── doctors.html               # Find a Doctor (public)
├── doctor-register.html       # Doctor registration
├── chat.html                  # Doctor-patient chat
├── appointments.html          # Appointment booking
├── customer-care.html         # Support center + tickets
├── support-agent.html         # Support agent queue
├── pharmacy-search.html       # Drug search across pharmacies
├── cart.html                  # Cart & checkout
├── pharmacist.html            # Pharmacy & drug management
├── student-hub.html           # Medical student past questions & courses
├── js/
│   └── config.js              # Shared Firebase + helpers
├── css/
│   └── styles.css             # Shared mobile-first styles
├── sw.js                      # Service Worker (PWA)
├── manifest.json              # PWA manifest
├── firestore.rules            # Complete security rules
├── storage.rules              # Storage security rules
├── functions/
│   └── index.js               # Paystack verify + webhook + expiry
└── README.md
```

## Setup Steps (do these in order)

### 1. Firebase Console
1. Open project **miracardiachealth**
2. Enable **Authentication** → Sign-in methods: Email/Password + Google
3. Create **Firestore** (production mode) → paste the entire `firestore.rules` → **Publish**
4. Enable **Storage** → paste `storage.rules` → Publish
5. Create any composite indexes the console prompts for
6. Your Firebase config is already in `js/config.js` (keep it)

### 2. Admin emails (already configured)
The following emails are allowed to claim the admin role automatically on login:
- `onyekamiracle08@gmail.com`
- `onyekaudechukwum@gmail.com`
- `onyekamiracle08@yahoo.com`
- `onyekaudechukwum@yahoo.com`
- `admin@miracardiachealth.com`

**How to use:** Just sign in with Google (or email) using one of these addresses. The app will automatically set `role: "admin"`.  
To add more emails later: edit the `adminEmails()` function inside `firestore.rules` and re-publish the rules.

### 3. Cloudinary (image storage – REQUIRED for logo & uploads)
1. Go to https://cloudinary.com and sign up / log in (same account you used for past projects)
2. Dashboard → copy your **Cloud name**
3. Settings → Upload → Upload presets → **Add upload preset**
   - Signing mode: **Unsigned**
   - Name it e.g. `miracardiac_unsigned`
4. Open `js/config.js` and replace:
   ```js
   const CLOUDINARY_CLOUD_NAME = "YOUR_CLOUD_NAME";
   const CLOUDINARY_UPLOAD_PRESET = "YOUR_UNSIGNED_PRESET";
   ```
5. Save. Logo upload and any image upload will now go to Cloudinary.

### 4. Paystack – WHERE TO PUT YOUR KEYS
**Public key (client-side – safe to expose):**
- Open `js/config.js`
- Find the line:
  ```js
  const PAYSTACK_PUBLIC_KEY = "pk_test_xxxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx";
  ```
- Replace with your real key from https://dashboard.paystack.com/#/settings/developers  
  (use `pk_test_...` while testing, `pk_live_...` for production)

**Secret key (server-side only – NEVER put in the HTML/JS):**
```bash
firebase functions:config:set paystack.secret="sk_live_xxxx"
cd functions && npm install firebase-functions firebase-admin cors
firebase deploy --only functions
```
Optional: set the Paystack webhook URL to your deployed `paystackWebhook` function URL.

### 5. Platform Logo (Admin page)
1. Sign in as one of the admin emails → open **Admin Dashboard**
2. Go to the **Settings** tab
3. Upload your logo image → click **Upload & Save Logo**
4. The logo is stored on Cloudinary and automatically appears on every page that uses the shared branding helper.

### 6. Deploy to GitHub Pages
1. Push the whole folder to your repo
2. GitHub Pages → source = root (or /docs)
3. In Firebase Auth → Settings → Authorized domains, add your GitHub Pages domain (e.g. `username.github.io`)

## Roles
| Role        | Default | How to get it                          |
|-------------|---------|----------------------------------------|
| patient     | Yes     | Automatic on register                  |
| doctor      | No      | Register via doctor-register.html → Admin approves |
| pharmacist  | No      | Create pharmacy in pharmacist.html → Admin approves |
| support     | No      | Register + Admin changes role + approves |
| admin       | No      | Sign in with one of the listed admin emails (automatic) |

## Key Flows
1. **Telemedicine**: Patient finds doctor → pays via Paystack → status `paid_pending` → Admin approves → status `active` (7 days) → Chat opens
2. **Appointments**: Patient books slot → status `requested` → Doctor/Admin confirms
3. **Pharmacy**: Pharmacist adds drugs → Patient searches → Cart (one pharmacy) → Order → Pharmacist updates status
4. **Support**: Patient creates ticket → Support agent replies & changes status
5. **Student Hub**: Browse courses → Pay → Admin approves purchase → Access granted to questions

## Security Notes
- Role enforcement is in **Firestore rules**, not only UI
- Paystack secret key never appears in client code
- Health data readable only by owner, assigned doctor, admin
- File uploads limited by type and size in Storage rules
- Student course questions require sign-in; full access controlled by `studentAccess` documents

## Important Fixes Applied in this version
- Admin access fixed: listed emails (5 total) can claim admin role without requiring email_verified first
- Cloudinary integrated for all image storage (logo, profiles, drugs, chat, etc.)
- Platform logo can be uploaded from Admin → Settings and appears site-wide via `applyPlatformBranding()`
- Paystack public key location clearly marked in `js/config.js` and README
- Full rebrand from AreaHealth-NG → MiracardiacHealth
- Fixed filename casing, missing Firestore rules for student features, service worker location, etc.

## Testing Checklist
- [ ] Register / Login (email + Google)
- [ ] Doctor registration → appears in Admin Approvals
- [ ] Approve doctor → appears on Find a Doctor
- [ ] Pay for consultation → appears in Admin Consultations
- [ ] Approve consultation → chat becomes active
- [ ] Send text + image in chat
- [ ] Book appointment
- [ ] Create support ticket → agent can reply
- [ ] Pharmacist adds drug → patient can search & add to cart
- [ ] Place order
- [ ] Student Hub: browse courses, purchase, admin approve, access questions

## Notes for Phone Users
All files are complete and paste-ready. After downloading the zip, extract and upload the contents to your GitHub repo (or use GitHub web editor / Codespaces if available).

---
Built for MiracardiacHealth · Mobile-first · NDPR-aware · Not a substitute for emergency care.

## Login → Dashboard fix (this build)

**Root cause:** Firestore rules only allowed `status == "pending"` when creating a `users/{uid}` document. Login/Register tried `status: "active"` for patients first, which was denied. If the fallback also failed (rules not deployed), the Auth user existed but had **no Firestore profile**. Dashboard called `requireAuth()` → got `null` profile → never called `boot()` and never redirected → screen stayed on loading skeleton.

**What we fixed:**
1. `firestore.rules` – patients may create with `status` `active` **or** `pending`; other roles still require `pending`. Added `slotLocks`, safer `hasUserDoc()`, and `tickets` alias.
2. `js/config.js` – `requireAuth` / `getCurrentUserProfile` auto-create a patient profile if missing; use relative `Login.html` redirects (works on GitHub Pages).
3. `Login.html` / `Register.html` – robust profile ensure + clear error if rules block create.

### After downloading this zip
1. Paste `firestore.rules` into Firebase Console → Firestore → Rules → **Publish**.
2. Paste `storage.rules` if not already.
3. In Authentication → Settings → **Authorized domains**, add your GitHub Pages domain (e.g. `yourname.github.io`) and custom domain.
4. Enable Google + Email/Password sign-in methods.
5. Replace Paystack public key in `js/config.js` when ready.

Then Google / email login will create the user doc and redirect to **Dashboard.html**.

