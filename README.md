# QuickInfo CRM

A serverless, cost-optimized CRM for lead generation using Google Places API. Search for businesses, manage leads through a sales pipeline, and visualize them on an interactive map with **real-time multi-user sync**.

## Features

- 🔍 **Smart Search**: Search businesses by category and city
- 🎯 **Area Search**: Draw circles on map to find hidden businesses (bypasses 60-result limit)
- 💾 **DB-First Strategy**: Minimizes API calls by storing basic info permanently
- 🗺️ **Interactive Map**: Visualize leads with color-coded status pins
- 📊 **Lead Management**: Track status (New → Contacted → Interested → Won/Lost)
- 📝 **Notes**: Add notes to each lead
- 🔐 **Google Authentication**: Secure sign-in with Google OAuth
- ⚡ **Real-Time Sync**: Multi-user collaboration - see changes instantly
- 🔔 **Live Notifications**: Get notified when teammates find new leads
- 🎛️ **Status Filters**: Filter map pins by lead status
- 💰 **Cost Optimized**: ~$10-15/month for low traffic

## Tech Stack

- **Frontend**: Next.js 15, React 19, Tailwind CSS
- **Map**: @vis.gl/react-google-maps
- **Database**: PostgreSQL (Cloud SQL with auto-pause)
- **Real-Time**: Firebase Firestore (serverless)
- **Authentication**: Firebase Auth (Google OAuth)
- **Cache**: Firebase Firestore (replaces Redis)
- **API**: Google Places API (Text Search + Place Details)
- **Hosting**: Google Cloud Run (scales to zero)
- **CI/CD**: GitHub Actions

## Architecture

```
┌─────────────────────────────────────────────────────────────┐
│                         USER BROWSER                         │
│  (React 19 + Next.js 15 + Google Maps + Tailwind CSS)       │
│  • Firebase Auth (Google sign-in)                           │
│  • Firestore real-time subscriptions                        │
└────────────────────────┬────────────────────────────────────┘
                         │ HTTPS
                         ▼
┌─────────────────────────────────────────────────────────────┐
│                    CLOUD RUN (Serverless)                    │
│  • Next.js App (SSR + API Routes)                           │
│  • Firebase Admin SDK (server-side Firestore)               │
│  • Auto-scales: 0 → 10 instances                            │
└──────┬──────────────┬──────────────┬───────────────────────┘
       │              │              │
       ▼              ▼              ▼
┌──────────┐   ┌──────────┐   ┌──────────────┐
│ Cloud SQL│   │Firestore │   │ Secret Mgr   │
│(Postgres)│   │(Real-time│   │ (API Keys)   │
│          │   │ + Cache) │   │              │
│Auto-pause│   │Serverless│   │Serverless    │
└──────────┘   └──────────┘   └──────────────┘
```

## Prerequisites

- Node.js 18+
- Docker & Docker Compose (for local development)
- Google Cloud Platform account
- Firebase project (same GCP project)

## Quick Start (Local Development)

### 1. Clone and Install

```bash
git clone https://github.com/ChrisPham03/Quick-Info-2.0.git
cd Quick-Info-2.0
npm install
```

### 2. Environment Variables

Create `.env.local` file:

```bash
# Database
DATABASE_URL="postgresql://quickinfo:localdev123@localhost:5433/quickinfo_crm"

# Google Maps API Keys
GOOGLE_MAPS_SERVER_KEY="your-server-api-key"
NEXT_PUBLIC_GOOGLE_MAPS_API_KEY="your-client-api-key"

# Firebase (Client-side)
NEXT_PUBLIC_FIREBASE_API_KEY="your-firebase-api-key"
NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN="your-project.firebaseapp.com"
NEXT_PUBLIC_FIREBASE_PROJECT_ID="your-project-id"
NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET="your-project.appspot.com"
NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID="your-sender-id"
NEXT_PUBLIC_FIREBASE_APP_ID="your-app-id"

# Firebase Admin (Server-side) - JSON key as single line
FIREBASE_ADMIN_KEY='{"type":"service_account","project_id":"..."}'
```

### 3. Start Docker Services

```bash
docker-compose up -d
```

### 4. Initialize Database

```bash
npx prisma generate
npx prisma db push
```

### 5. Run Development Server

```bash
npm run dev
```

Open [http://localhost:3000](http://localhost:3000)

## Commands

| Command | Description |
|---------|-------------|
| `npm run dev` | Start development server |
| `npm run build` | Build for production |
| `npm run start` | Start production server |
| `docker-compose up -d` | Start PostgreSQL |
| `docker-compose down` | Stop services |
| `npx prisma studio` | Open database GUI |
| `npx prisma db push` | Push schema changes |

## Real-Time Collaboration

### How It Works

1. **User A** searches for "Nails in Montreal" → Results saved to PostgreSQL + synced to Firestore
2. **User B** searches for same → Subscribes to Firestore real-time updates
3. **User A** does area search, finds 50 more businesses → Synced to Firestore
4. **User B** sees notification: "50 new leads available" → Clicks to refresh
5. **User A** changes lead status → **User B sees it instantly** (no refresh needed)

### What Syncs in Real-Time

| Data | Real-Time? | Notes |
|------|------------|-------|
| Lead status changes | ✅ Yes | Instant sync |
| Lead notes | ✅ Yes | Instant sync |
| New leads from area search | 🔔 Notification | User clicks to load |
| Map position | ❌ No | Each user controls their own view |

### Architecture

```
Browser (User A)                    Browser (User B)
      │                                   │
      │ 1. Update lead status             │
      ▼                                   │
   API Route ──────────────────────────────
      │                                   │
      │ 2. Save to PostgreSQL             │
      │ 3. Sync to Firestore              │
      │    (via Admin SDK)                │
      ▼                                   │
   Firestore ─────────────────────────────▶ 4. Real-time
      │                                      subscription
      │                                      fires
      │                                   │
      │                                   ▼
      │                              UI Updates
      │                              instantly!
```

## Authentication

### Google Sign-In Flow

1. User visits app → Redirected to login page
2. Click "Continue with Google" → Google OAuth popup
3. Authenticate → Firebase creates session
4. User can now access the app

### Security

- All routes protected by `AuthProvider`
- Firestore rules require authentication
- Server-side operations use Firebase Admin SDK (bypasses rules)
- API keys restricted by domain/API

## API Optimization Strategy

| Action | API Calls | Data Source |
|--------|-----------|-------------|
| First search (new city/category) | 1-3 | Google API |
| Repeat search (same city/category) | **0** | Database |
| Area search (circle on map) | 1-3 | Google API |
| View lead details | 1 | Google API (cached 30 days) |

## Cloud Deployment

### Production URL

https://quickinfo-prod-670354371508.northamerica-northeast1.run.app

### CI/CD Pipeline

Every push to `main` triggers:
1. Build Docker image with all environment variables
2. Push to Google Artifact Registry
3. Deploy to Cloud Run
4. Auto-scaling from 0-10 instances

### Required GitHub Secrets

| Secret | Description |
|--------|-------------|
| `GCP_SA_KEY` | Service account JSON for deployment |
| `GCP_PROJECT_ID` | Google Cloud project ID |
| `GOOGLE_MAPS_SERVER_KEY` | Server-side Maps API key |
| `NEXT_PUBLIC_GOOGLE_MAPS_API_KEY` | Client-side Maps API key |
| `NEXT_PUBLIC_FIREBASE_*` | Firebase configuration (6 secrets) |

### Required GCP Secrets (Secret Manager)

| Secret | Description |
|--------|-------------|
| `firebase-admin-key` | Firebase Admin SDK service account |

## Cost Estimation

| Service | Monthly Cost |
|---------|--------------|
| Cloud Run | $0-5 (scales to zero) |
| Cloud SQL | $10 (with auto-pause) |
| Firestore | $0 (free tier) |
| Google Maps API | $0-10 (within $200 credit) |
| **Total** | **~$10-15** |

## Project Structure

```
src/
├── app/
│   ├── api/leads/           # API routes
│   ├── page.tsx             # Main page
│   └── layout.tsx           # Root layout with AuthProvider
├── components/
│   ├── AuthProvider.tsx     # Authentication wrapper
│   ├── LoginPage.tsx        # Google sign-in page
│   ├── UserMenu.tsx         # User avatar & sign-out
│   ├── RealtimeIndicator.tsx# "Live sync" status
│   ├── CircleSearch.tsx     # Area search UI
│   ├── MapController.tsx    # Map bounds control
│   └── ...
├── hooks/
│   └── useAuth.ts           # Authentication hook
├── lib/
│   ├── firebase.ts          # Firebase client SDK
│   ├── firebase-admin.ts    # Firebase Admin SDK
│   ├── prisma.ts            # Database client
│   └── leads-client.ts      # Frontend API client
├── services/
│   ├── leads.ts             # Lead business logic
│   └── places.ts            # Google Places API
└── types/
    └── index.ts             # TypeScript types
```

## Troubleshooting

### "Missing or insufficient permissions" (Firestore)

- Check Firestore rules in Firebase Console
- Ensure user is authenticated
- Server operations use Admin SDK (bypasses rules)

### Real-time sync not working

- Check both users are searching **exact same city/category** (watch for trailing spaces!)
- Check browser console for subscription logs
- Verify "Live sync" indicator is green

### "Service Unavailable" on area search

- Cloud Run may need more memory
- Run: `gcloud run services update quickinfo-prod --memory 1Gi`

### Database enum errors

- Add missing enum values in Cloud SQL Studio:
  ```sql
  ALTER TYPE "LeadStatus" ADD VALUE 'CLOSED_WON';
  ALTER TYPE "LeadStatus" ADD VALUE 'CLOSED_LOST';
  ```

## Documentation

- [Deployment Guide](docs/DEPLOYMENT.md) - Full deployment instructions
- [Real-Time Sync](docs/REALTIME_SYNC.md) - How real-time collaboration works

## License

MIT
