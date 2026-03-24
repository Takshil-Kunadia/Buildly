# VekariyaInfra Construction Site Management App — Requirements & Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build a multi-tenant SaaS PWA for construction site management — installable on Android/iOS without App Store, with offline photo capture, role-based access, and full site/material/attendance/payroll management.

**Architecture:** Shared PostgreSQL database (Neon) with `tenant_id` on every table + PostgreSQL Row Level Security for tenant isolation. React PWA frontend on S3/CloudFront. Node.js/Express API on EC2 t4g.nano.

**Tech Stack:** React (Vite), Node.js + Express, PostgreSQL (Neon serverless), AWS S3 (photos), AWS EC2 t4g.nano (API), Firebase FCM (push notifications), react-i18next (i18n), JWT auth

---

## Infrastructure

| Service | Provider | Cost |
|---|---|---|
| API Server | AWS EC2 t4g.nano | ~$3.40/mo |
| Database | Neon serverless PostgreSQL (free tier) | $0/mo |
| File/Photo Storage | AWS S3 | ~$0.05-0.10/mo |
| PWA Hosting | AWS S3 + CloudFront | ~$0.50/mo |
| Push Notifications | Firebase FCM | Free |
| **Total** | | **~$4/mo** |

---

## Architecture Decisions

### Multi-Tenancy
- Strategy: **Shared database, tenant_id on every table**
- PostgreSQL Row Level Security (RLS) policies enforce isolation at DB layer — even a buggy query cannot leak cross-tenant data
- Every API middleware validates and sets tenant context (`SET app.current_tenant = $tenant_id`) before any query executes
- S3 paths: `/{tenant_id}/{module}/{year}/{filename}` — tenant isolation built into the path

### Auth
- JWT tokens, signed with RS256, containing `{ userId, tenantId, role, permissions }`
- Refresh token rotation (7-day sliding window)
- All API routes validate JWT and extract tenant context from it
- No cross-tenant API calls possible at the application layer

### Registration Code Flow
- Developer creates a company in the admin panel → system generates a **persistent multi-use code** (no expiry) identifying that company
- Any user can register with the code; they self-select role: **Owner** or **Employee**
- Developer can manually reassign any user's role via the admin panel at any time
- The code never expires — it's just a company identifier for onboarding

### Multi-Owner Behaviour
- Any Owner can approve or reject independently
- First Owner to act wins; a second Owner cannot undo the decision once actioned

### Offline Behaviour
- **Offline-capable** (IndexedDB queue, sync on reconnect): attendance selfie capture, material receipt photo capture, PO receipt photo, transfer receipt photo
- **Online-only**: all read screens, dashboards, reports, lists, form submissions (except the above)

### i18n
- All user-facing strings wrapped in `react-i18next` from day one
- Default locale: English
- Adding Gujarati/Hindi requires only a translation JSON file — zero code changes

### "Other" Dropdown Pattern
- Every `<select>` / dropdown across the app includes an "Other" option
- Selecting "Other" renders a freetext input inline
- This pattern is implemented as a single reusable `<SelectWithOther>` component

---

## User Roles

### Developer (Admin) — Platform level
- Creates companies (tenants), generates registration codes
- Manually assigns/changes user roles
- Can access any tenant for support
- Not involved in day-to-day operations

### Owner — Company level
- Full access to everything within their tenant
- Approves: material transfers, POs, payment requests, attendance, leave
- Manages: sites, employees, salary, vendors, permissions

### Employee — Default role
**Base permissions (always on):**
- Submit own attendance
- Apply for leave
- View own salary history

**Grantable by Owner (per-employee toggle):**
- Raise material transfer requests
- Raise purchase orders
- Raise payment requests
- View site-level reports
- Confirm material receipt

---

## Module Specifications

### 1. Auth Module
- Register with company code (self-select Owner or Employee)
- Login / logout
- JWT access + refresh token
- Role + permission management (Owner/Developer only)

### 2. Site Management
**States:** `not_started` → `ongoing` → `completed` → `closed`
- Owner creates/edits sites
- Site dashboard: inventory levels, recent activity, pending approvals
- Sites are the root entity — most other records reference a site

### 3. Stock / Material Register
- Fields: material name (dropdown + "Other" freetext), category, unit, quantity, type (consumable | permanent)
- Every entry: mandatory real-time photo + auto-captured GPS
- Linked to a specific site
- Photo captured offline → queued → synced when online

### 4. Site-to-Site Material Transfer
**Status flow:**
```
Transfer Requested → Transfer Approved → Transfer Dispatched → Transfer Received
                  ↘ Transfer Rejected
```
- Receiver raises request; Owner approves/rejects
- Dispatcher records dispatch with photo + GPS
- Receiver confirms receipt with photo + GPS (offline-capable)
- Receiving a transfer auto-updates inventory on both sites

### 5. Purchase Orders (from Vendors)
**Status flow:**
```
PO Requested → PO Approved → Order Placed → Order Dispatched → Order Received
             ↘ Order Cancelled
```
- Owner approves all POs
- **Edit audit trail**: every field change logs (user, field, old value, new value, timestamp) — visible to all users in tenant
- Receiving a PO auto-updates site inventory

### 6. Inventory Management
- Stock levels per site maintained automatically via incoming transfers + POs + consumption entries
- Site-wise view: materials at each site, quantities, low-stock alerts
- Manual adjustments by Owner require a mandatory reason/note

### 7. Payment Requests
**Status flow:** `Pending → Approved → Paid`
- Fields: amount, description, linked site, photo of QR code (primary method)
- Owner reviews QR, makes payment externally, uploads payment receipt
- Other payment methods supported (bank transfer, cash)

### 8. Attendance
- Check-in: mandatory real-time selfie + auto GPS
- Check-out: same
- Owner views all attendance across employees and sites
- Employees with granted permission view team attendance
- Selfie captured offline → queued → synced

### 9. Leave Management
- Employees can apply for any date, same-day allowed, no minimum notice
- Leave types: full day, partial (with time range)
- Owner approves/rejects
- Employee views own leave history; Owner views all

### 10. Salary Register
- Manual entry by Owner: employee, month, amount, paid/unpaid, paid date
- No auto-calculation
- Employee views only own salary history

### 11. Party / Vendor Ledger
- Register vendors with contact info, category
- Track supply history (linked to POs)
- Track payment history per vendor

### 12. Reports & Dashboard
**Dashboard widgets (role-filtered):**
- Material purchased per site
- Pending approvals (transfers, POs, payments, leave)
- Attendance summary
- PO status overview
- Current inventory levels per site

**Reports:**
- Material purchased per site (date range filter)
- Transfer history
- Attendance summary (per employee, per site)
- Payment summary
- Inventory levels per site
- Filters: site, date range, employee, category
- Export: PDF, Excel, WhatsApp share

### 13. Push Notifications (Firebase FCM)
- Owner notified: new approval requests (transfer, PO, payment, leave)
- Employee notified: approval decisions on their requests, attendance reminders

---

## Database Schema Overview

Every table includes: `id`, `tenant_id`, `created_at`, `updated_at`, `created_by`

RLS policy template on every table:
```sql
CREATE POLICY tenant_isolation ON <table>
  USING (tenant_id = current_setting('app.current_tenant')::uuid);
```

**Core tables:**
- `tenants` (id, name, registration_code, settings JSONB)
- `users` (id, tenant_id, name, email, phone, role, permissions JSONB, fcm_token)
- `sites` (id, tenant_id, name, location, state, meta JSONB)
- `materials` (id, tenant_id, name, category, unit, type) — master list per tenant
- `stock_entries` (id, tenant_id, site_id, material_id, quantity, photo_path, gps_lat, gps_lng, notes)
- `inventory` (id, tenant_id, site_id, material_id, current_quantity) — maintained by triggers
- `transfer_orders` (id, tenant_id, from_site_id, to_site_id, material_id, quantity, status, ...)
- `purchase_orders` (id, tenant_id, site_id, vendor_id, status, ...)
- `po_line_items` (id, po_id, material_id, quantity, unit_price)
- `po_audit_log` (id, tenant_id, po_id, user_id, field, old_value, new_value, changed_at)
- `payment_requests` (id, tenant_id, site_id, amount, description, status, qr_photo_path, receipt_path)
- `attendance` (id, tenant_id, user_id, site_id, date, check_in_time, check_out_time, selfie_path, gps_lat, gps_lng)
- `leaves` (id, tenant_id, user_id, date, type, start_time, end_time, status, reason)
- `salary_records` (id, tenant_id, user_id, month, year, amount, paid, paid_date)
- `vendors` (id, tenant_id, name, category, contact, address)

---

## Project Structure

### Monorepo layout
```
inventory-app/
├── frontend/          # React Vite PWA
│   ├── public/
│   │   ├── manifest.json
│   │   └── sw.js          # Service worker (offline queue)
│   ├── src/
│   │   ├── components/    # Shared UI (SelectWithOther, PhotoCapture, GPSField, etc.)
│   │   ├── features/      # Feature modules (auth, sites, materials, transfers, ...)
│   │   │   └── {feature}/
│   │   │       ├── api.ts       # API calls for this feature
│   │   │       ├── components/  # Feature-specific components
│   │   │       ├── hooks.ts     # React Query hooks
│   │   │       └── types.ts
│   │   ├── i18n/          # Translation files (en.json, gu.json, hi.json)
│   │   ├── lib/           # axios instance, queryClient, offline queue (IndexedDB)
│   │   ├── routes/        # React Router route definitions
│   │   └── store/         # Zustand (auth state, tenant context)
│   └── package.json
│
├── backend/           # Node.js + Express API
│   ├── src/
│   │   ├── config/        # env, db (Neon), s3, fcm
│   │   ├── middleware/    # auth (JWT), tenant (set RLS context), validation
│   │   ├── modules/       # Feature modules
│   │   │   └── {feature}/
│   │   │       ├── router.ts
│   │   │       ├── controller.ts
│   │   │       ├── service.ts
│   │   │       └── schema.ts   # Zod validation schemas
│   │   ├── db/
│   │   │   ├── migrations/    # SQL migration files
│   │   │   └── index.ts       # Neon connection pool
│   │   └── app.ts
│   └── package.json
│
├── infra/             # AWS CDK or Terraform for EC2 + S3 setup
└── docs/
    └── requirements.md  # This document (copy)
```

---

## Implementation Phases

### Phase 0 — Project Bootstrap
- [ ] Init monorepo (pnpm workspaces)
- [ ] Backend: Express + TypeScript, Zod, Neon pg client, JWT, middleware skeleton
- [ ] Frontend: Vite + React + TypeScript, react-router-dom, react-query, react-i18next, Tailwind
- [ ] Database: Neon project created, connection string in env
- [ ] S3: bucket created with `/{tenant_id}/` prefix policy
- [ ] EC2 t4g.nano provisioned, nginx reverse proxy, SSL via Let's Encrypt
- [ ] GitHub Actions: deploy backend to EC2, deploy frontend to S3/CloudFront
- [ ] **Verify**: `GET /health` returns 200, frontend loads at domain

### Phase 1 — Multi-Tenant Foundation + Auth
- [ ] DB migrations: `tenants`, `users` tables with RLS policies
- [ ] Tenant middleware (sets `app.current_tenant` on every request)
- [ ] Auth module: register (with company code), login, refresh token, logout
- [ ] Developer admin: create tenant, generate registration code, manage user roles
- [ ] Role + permission middleware
- [ ] `<SelectWithOther>` reusable component
- [ ] i18n setup with all strings for auth screens
- [ ] **Verify**: register two users on different tenants, confirm each can only see their own data

### Phase 2 — Sites + Inventory Foundation
- [ ] DB migrations: `sites`, `materials`, `inventory`, `stock_entries`
- [ ] Site CRUD (Owner), site state machine
- [ ] Material master list per tenant (with "Other" freetext)
- [ ] Stock entry form with `<PhotoCapture>` (real-time camera, no gallery) + GPS
- [ ] Offline queue: IndexedDB stores pending photo uploads + stock entries, syncs on reconnect
- [ ] Inventory view per site
- [ ] **Verify**: create stock entry offline, go online, confirm it syncs and inventory updates

### Phase 3 — Transfers + Purchase Orders
- [ ] DB migrations: `transfer_orders`, `purchase_orders`, `po_line_items`, `po_audit_log`, `vendors`
- [ ] Transfer order CRUD + status flow + Owner approval
- [ ] Transfer dispatch (photo + GPS) and receipt confirmation (offline-capable)
- [ ] Inventory auto-update on transfer received (DB trigger or service layer)
- [ ] PO CRUD + status flow + Owner approval
- [ ] PO edit audit trail
- [ ] PO receipt confirmation (offline-capable)
- [ ] Inventory auto-update on PO received
- [ ] Vendor ledger CRUD
- [ ] **Verify**: full transfer lifecycle, check inventory changes at both sites; full PO lifecycle; audit log entries

### Phase 4 — Attendance + Leave + Salary
- [ ] DB migrations: `attendance`, `leaves`, `salary_records`
- [ ] Attendance check-in/out with selfie (offline-capable) + GPS
- [ ] Leave application + Owner approval flow
- [ ] Salary register (Owner entry, employee read-own)
- [ ] Employee permission grants (Owner toggles per-employee capabilities)
- [ ] **Verify**: employee submits attendance offline, syncs, Owner approves leave, employee sees own salary

### Phase 5 — Payment Requests + Notifications
- [ ] DB migrations: `payment_requests`
- [ ] Payment request form (amount, description, QR photo upload)
- [ ] Owner approval + receipt upload flow
- [ ] Firebase FCM integration: register device tokens, send notifications on approval events
- [ ] Notification preferences per user
- [ ] **Verify**: raise payment request, Owner receives push notification, approves, requester notified

### Phase 6 — Reports + Dashboard
- [ ] Dashboard with role-filtered widgets (pending approvals, inventory, attendance summary, PO status)
- [ ] Reports: material purchased, transfer history, attendance summary, payment summary, inventory per site
- [ ] Date range + site + employee + category filters
- [ ] PDF export (pdfmake or puppeteer)
- [ ] Excel export (exceljs)
- [ ] WhatsApp share (Web Share API)
- [ ] Charts (Recharts: bar, pie, line)
- [ ] **Verify**: generate each report type, export to PDF and Excel, verify data accuracy

### Phase 7 — Polish + PWA
- [ ] `manifest.json` + service worker (Workbox): installable on Android + iOS
- [ ] Offline fallback screens
- [ ] Offline sync status indicator (banner when queued items pending)
- [ ] Full i18n audit: all strings in en.json, Gujarati translation file stubbed
- [ ] Responsive design audit (mobile-first, tested on iOS + Android)
- [ ] Error boundaries, empty states, loading skeletons
- [ ] **Verify**: install PWA on Android and iOS, test offline capture flow end-to-end

---

## Verification Checklist (end-to-end)

- [ ] Two different tenants registered — zero data leakage between them
- [ ] All three roles (Developer, Owner, Employee) behave correctly
- [ ] Employee with no granted permissions cannot access restricted actions
- [ ] Full transfer lifecycle (request → approve → dispatch → receive → inventory updated)
- [ ] Full PO lifecycle with audit trail
- [ ] Attendance selfie captured with no network, synced when network restored
- [ ] Push notification received on Owner's device when employee raises a request
- [ ] PWA installable and launchable from home screen on both Android and iOS
- [ ] PDF and Excel exports download correctly
- [ ] All dropdowns show "Other" option with freetext input

---

## Open Items (Resolved)

| Question | Decision |
|---|---|
| Tenant isolation strategy | Shared DB + `tenant_id` + PostgreSQL RLS |
| Database provider | Neon serverless PostgreSQL (free tier) |
| File storage | AWS S3 |
| API hosting | AWS EC2 t4g.nano |
| Frontend hosting | AWS S3 + CloudFront |
| Push notifications | Firebase FCM |
| Registration code | Multi-use, no expiry, identifies company only; roles self-selected at register; Developer reassigns manually |
| Multi-owner approval | Any Owner can act independently; first to act wins |
| Offline scope | Photo/GPS capture actions only; all reads are online-only |

## Remaining Open Items

- [ ] **Offline map**: full screen-by-screen breakdown of offline vs online-required (to be done in Phase 2 planning)
- [ ] **Domain name**: confirm domain for the PWA
- [ ] **AWS account**: confirm account setup and IAM roles for EC2 + S3
- [ ] **Neon project**: create Neon account + project, get connection string
- [ ] **Firebase project**: create Firebase project, get FCM credentials
