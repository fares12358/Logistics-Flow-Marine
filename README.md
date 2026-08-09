# Logistics & Shipping Operations SaaS

A full logistics and shipping-operations SaaS built to manage the entire commercial-to-operational lifecycle of container shipping — bookings, vessel rounds and voyages, capacity planning, invoicing, expenses, and financial/operational reporting.

Built solo, end-to-end: architecture, all 23 backend modules, database design, and ongoing security/performance hardening.

---

## Tech Stack

**Frontend:** Next.js 15, Tailwind CSS
**Backend:** Node.js, Express, MongoDB (Mongoose)
**Auth:** JWT
**Reporting / Exports:** pdfmake, exceljs
**Dashboards:** Chart.js
**API:** REST

---

## Architecture

The system is a deliberately layered monolith rather than a microservices split, matched to the team size and scale of the product:

- **Backend:** `Model → Service → Controller → Routes`, consistent across all 23 modules (`backend/src/modules/`). Controllers stay thin — parse the request, call the service, shape the response. Business logic lives entirely in `*.service.js`. No route calls a Mongoose model directly.
- **Frontend:** `services/*.service.js` (thin axios wrappers) → `hooks/use*.js` (React Query wrappers) → `modules/*/*.jsx` (feature components) → `app/(dashboard)/**/page.jsx` (route entries).
- **Cross-module aggregation** (dashboard stats, reports, exports) reads directly from multiple models where needed for read/aggregation paths — the accepted house style for a monolith of this shape, not an exception.

---

## Core Modules

### Dual-Layer Booking System
The core architectural decision of the app. **Layer 1 (ManualBooking)** captures commercial estimates before a shipment is confirmed; **Layer 2 (Booking)** tracks the real operational execution. Reports, the capacity engine, vessel stats, and exports all read from both layers wherever combined figures are needed, so commercial and operational views stay in sync.

### Vessel Rounds & Voyages
Manages shipping rounds and the voyages within them, with safe-delete guards that check for linked bookings, manual bookings, and expenses before allowing a round to be removed — preventing orphaned financial or operational records.

### Capacity Planning Engine
A read-only aggregation engine calculating available vs. booked capacity per voyage, combining data across bookings, manual bookings, and voyages in real time.

### Invoicing & Expenses
Full invoice and expense tracking tied to rounds, voyages, and agents, feeding directly into the financial reporting layer and dashboard KPIs.

### Reporting Module
Seven report types — including Round P&L, Voyage Performance, Revenue/Expense, and Booking Revenue Details — generated from cross-module aggregation and surfaced through a tiered dashboard widget system.

### Export Pipeline
Server-side PDF and Excel generation (pdfmake, exceljs) across bookings, invoices, and reports.

### RBAC & Permissions
Granular, role-based access control gating every one of the 25 backend route files, with a checkbox-style permission model so admins can fine-tune exactly what each role can see and do.

### Dashboard & Analytics
Real-time stats and fleet-utilisation widgets built with Chart.js.

---

## Project Structure

```
logistics/
├── backend/
│   ├── src/
│   │   ├── modules/                     # 23 modules, one folder per domain
│   │   │   ├── bookings/
│   │   │   │   ├── booking.model.js
│   │   │   │   ├── booking.service.js
│   │   │   │   ├── booking.controller.js
│   │   │   │   └── booking.routes.js
│   │   │   ├── manual-bookings/         # Layer 1 — commercial estimates
│   │   │   ├── rounds/
│   │   │   │   ├── round.service.js     # hardDelete cascade guards
│   │   │   │   └── round.routes.js
│   │   │   ├── voyages/
│   │   │   ├── capacity/
│   │   │   │   └── capacity.service.js  # read-only aggregation engine
│   │   │   ├── vessels/
│   │   │   │   └── vessel.service.js
│   │   │   ├── invoices/
│   │   │   │   └── invoice.service.js
│   │   │   ├── payments/
│   │   │   ├── expenses/
│   │   │   │   └── expense.service.js   # getAttachmentPath (upload handling)
│   │   │   ├── agents/
│   │   │   │   └── agent.service.js
│   │   │   ├── ports/
│   │   │   ├── voyage-ports/
│   │   │   ├── container-types/
│   │   │   ├── dashboard/
│   │   │   │   ├── dashboard.service.js
│   │   │   │   └── dashboard.routes.js  # GET /stats, GET /fleet-utilisation
│   │   │   ├── reports/
│   │   │   │   └── reports.service.js   # 7 report types
│   │   │   ├── export/
│   │   │   │   └── export.service.js    # PDF/Excel generation
│   │   │   ├── auth/
│   │   │   ├── roles/                   # RBAC / permissions
│   │   │   └── audit-log/
│   │   ├── middleware/
│   │   │   ├── rbac.js
│   │   │   ├── errorHandler.js
│   │   │   └── compressUpload.js        # sharp-based image compression
│   │   ├── utils/
│   │   │   ├── responseHelper.js
│   │   │   ├── mailer.js                # password reset / invite emails
│   │   │   └── cache.js                 # in-memory cache layer
│   │   └── server.js                    # CORS, helmet, gzip, cookie-parser
│   └── package.json
│
└── frontend/
    ├── src/
    │   ├── app/
    │   │   └── (dashboard)/
    │   │       ├── bookings/
    │   │       ├── vessels/
    │   │       ├── rounds/[id]/
    │   │       ├── agents/[id]/
    │   │       ├── agents/new/
    │   │       ├── roles/[id]/
    │   │       └── reports/
    │   ├── services/                    # thin axios wrappers, *.service.js
    │   ├── hooks/                       # React Query hooks, use*.js
    │   │   ├── useAgents.js
    │   │   ├── useVoyages.js
    │   │   ├── useServices.js
    │   │   ├── useRounds.js
    │   │   ├── useContainerTypes.js
    │   │   ├── usePorts.js
    │   │   └── useVoyagePorts.js
    │   ├── modules/                     # feature components, *.jsx
    │   │   └── BookingImport.jsx        # client-side xlsx import
    │   ├── context/
    │   │   └── AuthContext.jsx          # httpOnly-cookie JWT, no localStorage token
    │   ├── lib/
    │   │   └── axios.js
    │   └── components/
    │       └── PageGuard.jsx
    └── package.json
```

> Structure reconstructed from the module names, file paths, and service boundaries referenced throughout the internal system audit — not a literal `tree` export, so folder names for less-documented modules (auth, roles, audit-log, ports, container-types) are inferred from their role in the architecture rather than confirmed 1:1.

## Engineering Practices

- Consistent layered architecture verified across all modules — no route bypasses the service layer.
- Full RBAC coverage: every one of 25 backend route files is gated.
- No SQL/NoSQL injection via request bodies; hardened against NoSQL operator injection in query params.
- Regular dependency and security audits (`npm audit` across backend and frontend), with tracked remediation of CVEs in `next`, `xlsx`, `nodemailer`, and `sharp`.
- No empty-catch error swallowing; centralized error handling via a consistent `AppError` pattern.
- No path traversal, no `dangerouslySetInnerHTML`, JWT never stored in `localStorage` (httpOnly cookies only).

---

## Author

Built and maintained solo by **Fares Mohamed** — architecture, all backend modules, the dual-layer booking model, the reporting/aggregation engine, RBAC, and ongoing security/performance auditing.
