# CredGuard — System Architecture

> **B.Tech Mini-Project**: Secure Academic Credential Verification using Blockchain + AI/OCR

---

## 1. High-Level Overview

CredGuard is a **multi-service monorepo** with four independently deployable components that talk to each other over HTTP/RPC:

| Service | Technology | Default Port |
|---|---|---|
| `frontend` | Next.js 15 (App Router) | 3000 |
| `backend` | Node.js + Express | 4000 |
| `ocr-service` | FastAPI + Tesseract | 8000 |
| `blockchain` | Hardhat / Polygon Mumbai | – |

---

## 2. System Context Diagram

```
                          ┌─────────────────────────────────────────────────┐
                          │                   Internet / Browser             │
                          └─────────────────────────────────────────────────┘
                                         │
                          ┌──────────────▼──────────────────────────────┐
                          │           Next.js Frontend                  │
                          │   (Landing, Issue, Verify, Student, Admin,  │
                          │    Explorer pages + 3D animated UI)         │
                          └──────────────────────────────────────────────┘
                                         │  REST API calls (JSON)
                          ┌──────────────▼──────────────────────────────┐
                          │           Express Backend (Node.js)         │
                          │  Routes → Controllers → Services             │
                          └────────┬───────────────────┬────────────────┘
                                   │                   │
              ┌────────────────────▼──┐    ┌───────────▼────────────────┐
              │      MongoDB           │    │   Supabase Auth             │
              │ (Credentials, Users,   │    │ (Google OAuth + Email/PW)   │
              │  Logs, Institutions)   │    └────────────────────────────┘
              └────────────────────────┘
                                   │
         ┌─────────────────────────┴──────────────────────────┐
         │                                                     │
┌────────▼──────────────────┐                    ┌────────────▼───────────────┐
│  Polygon Mumbai Testnet    │                    │   FastAPI OCR Service       │
│  CredentialRegistry.sol    │                    │  (Tesseract + OpenCV)       │
│  (Solidity Smart Contract) │                    │  POST /analyze              │
└────────────────────────────┘                    └────────────────────────────┘
```

---

## 3. Component Deep-Dives

### 3.1 Frontend — `frontend/`

Built with **Next.js 15 App Router** using TypeScript + TailwindCSS.

#### Pages & Responsibilities

| Route | Page | Actor | Purpose |
|---|---|---|---|
| `/` | Landing | Public | 3D animated hero, intro to platform |
| `/issue` | Issue Credential | Admin | Upload PDF, fill metadata, issue on-chain |
| `/verify` | Verify Credential | Employer | Upload PDF or enter ID to verify |
| `/student` | Student Dashboard | Student | View own credentials, download, share QR |
| `/admin` | Admin Dashboard | University | Manage issued credentials, analytics |
| `/explorer` | Blockchain Explorer | Public | Look up any credential by ID on-chain |
| `/about` | About | Public | Project info |

#### Frontend Architecture

```
src/
├── app/               # Next.js App Router pages
│   ├── page.tsx       # Landing (3D scene)
│   ├── issue/
│   ├── verify/
│   ├── student/
│   ├── admin/
│   ├── explorer/
│   └── layout.tsx     # Root layout (Supabase provider, fonts)
├── components/        # Reusable UI components
│   ├── 3D/            # React Three Fiber scenes
│   └── ui/            # Buttons, cards, modals
├── lib/               # Supabase client, API helpers, utils
└── styles/            # Global CSS
```

#### Key Libraries

| Library | Purpose |
|---|---|
| `react-three-fiber` | 3D animated background scenes |
| `framer-motion` | Page transitions, card animations |
| `gsap` | Timeline-based scroll animations |
| `@supabase/ssr` | Server-side Supabase auth |
| `tailwindcss` | Utility-first styling |

---

### 3.2 Backend — `backend/`

**Express.js** REST API in TypeScript following a layered architecture:

```
Route → Controller → Service → [MongoDB | Blockchain | OCR]
```

#### Directory Layout

```
src/
├── server.ts          # App entry point, middleware mount
├── routes/            # Express routers (auth, credentials, ocr)
├── controllers/       # Request handlers — parse input, call service, respond
├── services/          # Business logic
│   ├── blockchain.ts  # Ethers.js wrapper around CredentialRegistry
│   ├── ocr.ts         # HTTP client to FastAPI /analyze
│   └── qr.ts          # QR code generation
├── models/            # Mongoose schemas
│   ├── Credential.ts
│   ├── User.ts
│   └── Institution.ts
├── middleware/        # JWT auth guard, error handler, multer upload
├── config/            # DB connection, env validation
└── utils/             # PDF hashing (SHA-256), formatters
```

#### API Design

| Method | Endpoint | Auth | Description |
|---|---|---|---|
| `POST` | `/api/auth/login` | Public | Authenticate user |
| `POST` | `/api/credentials/issue` | Bearer (Admin) | Issue a new credential |
| `GET` | `/api/credentials/:id` | Bearer | Fetch credential by ID |
| `POST` | `/api/credentials/verify` | Public | Verify certificate hash |
| `GET` | `/api/credentials/student/:userId` | Bearer | List student credentials |
| `POST` | `/api/ocr/analyze` | Bearer | Proxy to OCR microservice |

#### Credential Issuance Flow

```
1. Admin uploads PDF
2. Backend computes SHA-256 hash of the PDF
3. Metadata (student name, degree, date, institution) is sent to backend
4. Backend calls CredentialRegistry.issueCertificate() via Ethers.js
5. Transaction hash is saved to MongoDB with status "confirmed"
6. QR code URL is generated and returned to frontend
```

---

### 3.3 Blockchain — `blockchain/`

**Hardhat** project targeting Polygon Mumbai (EVM-compatible testnet).

#### Smart Contract: `CredentialRegistry.sol`

```solidity
// Simplified representation
struct Certificate {
    string studentName;
    string degree;
    string institution;
    bytes32 documentHash;    // SHA-256 of the PDF
    uint256 issuedAt;
    address issuer;
    bool isValid;
}

mapping(string => Certificate) public certificates;
```

#### Contract Functions

| Function | Visibility | Description |
|---|---|---|
| `issueCertificate(id, name, degree, inst, hash)` | External (Issuer only) | Writes cert to chain |
| `storeHash(id, newHash)` | External (Issuer only) | Updates the PDF hash |
| `getCertificate(id)` | Public View | Returns full Certificate struct |
| `verifyCertificate(id, hash)` | Public View | Returns `true` if hash matches |

#### Access Control

Only **approved issuer addresses** (set at contract deployment) can call `issueCertificate` and `storeHash`. Verification is fully public.

#### Deployment

```bash
npx hardhat run scripts/deploy.ts --network mumbai
# → Outputs: CredentialRegistry deployed at 0x...
```

---

### 3.4 OCR Service — `ocr-service/`

A **FastAPI** Python microservice that:
1. Accepts a PDF or image via multipart upload
2. Runs Tesseract OCR to extract text fields
3. Applies OpenCV heuristics to detect pixel-level tamper artifacts
4. Returns a structured JSON response

#### Endpoint

```
POST /analyze
Content-Type: multipart/form-data

Request:
  file: <pdf or image>

Response:
{
  "extracted_fields": {
    "name": "John Doe",
    "degree": "B.Tech Computer Science",
    "institution": "XYZ University",
    "date": "2024-05-15"
  },
  "tamper_score": 0.07,     // 0.0 = pristine, 1.0 = heavily tampered
  "is_tampered": false,     // true if tamper_score > threshold
  "confidence": 0.94
}
```

#### Tamper Detection Heuristics

- **JPEG artifact analysis** — detects re-compression artifacts around text regions
- **Font inconsistency** — flags mismatched pixel densities around characters
- **Noise pattern analysis** — identifies localized noise indicating edits

---

## 4. Data Flow Diagrams

### 4.1 Certificate Issuance

```
University Admin                Backend                  Blockchain
      │                            │                          │
      │── Upload PDF + Metadata ──▶│                          │
      │                            │── SHA-256(PDF) ──────▶  │ hash
      │                            │── issueCertificate() ──▶│
      │                            │◀── txHash ──────────────│
      │                            │── Save to MongoDB        │
      │◀── QR Code + ID ──────────│                          │
```

### 4.2 Certificate Verification

```
Employer                    Backend                 Blockchain          OCR Service
   │                           │                        │                    │
   │── Upload PDF ────────────▶│                        │                    │
   │                           │── SHA-256(PDF)         │                    │
   │                           │── verifyCertificate() ▶│                    │
   │                           │◀── true/false ─────────│                    │
   │                           │── POST /analyze ────────────────────────────▶│
   │                           │◀── tamper_score ────────────────────────────│
   │◀── Verification Report ───│                        │                    │
```

---

## 5. Authentication & Security

| Concern | Approach |
|---|---|
| User authentication | Supabase Auth (Google OAuth + Email/Password) |
| API authorization | JWT Bearer tokens validated in Express middleware |
| Issuer authorization | Smart contract on-chain allowlist (issuer address) |
| Certificate integrity | SHA-256 hash stored immutably on Polygon |
| Private key management | `.env` only — never committed to Git |
| File uploads | `multer` with file type and size limits |

---

## 6. Database Schema (MongoDB)

### `credentials` collection

```json
{
  "_id": "ObjectId",
  "credentialId": "CRED-2024-001",
  "studentName": "John Doe",
  "studentEmail": "john@example.com",
  "degree": "B.Tech Computer Science",
  "institution": "XYZ University",
  "issueDate": "2024-05-15",
  "documentHash": "sha256:abc123...",
  "txHash": "0xabc...",
  "blockNumber": 12345678,
  "qrCodeUrl": "https://credguard.app/verify/CRED-2024-001",
  "status": "confirmed",
  "issuedBy": "userId",
  "createdAt": "ISODate"
}
```

### `users` collection

```json
{
  "_id": "ObjectId",
  "supabaseId": "uuid",
  "email": "admin@university.edu",
  "role": "admin | student | employer",
  "institution": "XYZ University",
  "createdAt": "ISODate"
}
```

---

## 7. Deployment Architecture

```
                    ┌─────────────────────┐
                    │       Vercel          │
                    │   (Next.js Frontend)  │
                    └──────────┬───────────┘
                               │ HTTPS
                    ┌──────────▼───────────┐
                    │       Render          │
                    │   (Express Backend)   │
                    └───┬──────────────┬───┘
                        │              │
             ┌──────────▼──┐    ┌──────▼──────────────┐
             │  MongoDB      │    │   Render / Fly.io    │
             │  Atlas        │    │   (FastAPI OCR)      │
             └──────────────┘    └──────────────────────┘
                                         │
                             ┌───────────▼────────────┐
                             │   Polygon Mumbai /      │
                             │   Polygon Mainnet       │
                             └─────────────────────────┘
```

---

## 8. Environment Variables

### Backend (`backend/.env`)

| Variable | Description |
|---|---|
| `MONGO_URI` | MongoDB connection string |
| `JWT_SECRET` | Secret for signing JWTs |
| `OCR_SERVICE_URL` | Base URL of the FastAPI service |
| `ETH_RPC_URL` | Polygon RPC (Infura/Alchemy) |
| `REGISTRY_CONTRACT_ADDRESS` | Deployed contract address |
| `ISSUER_PRIVATE_KEY` | Testnet-only private key for signing txns |
| `FRONTEND_URL` | CORS allowed origin |

---

## 9. Key Design Decisions

| Decision | Rationale |
|---|---|
| **Hash stored on-chain, not full document** | Keeps gas costs minimal while preserving integrity proof |
| **FastAPI for OCR** | Python ecosystem has the best OCR/CV libraries |
| **Supabase for Auth** | Saves weeks of auth implementation, provides Google OAuth out of the box |
| **Next.js App Router** | Server Components enable faster page loads and SEO |
| **Polygon Mumbai** | Low gas fees, EVM-compatible, good testnet tooling |
| **Monorepo** | Easier local development and cross-service type sharing |
