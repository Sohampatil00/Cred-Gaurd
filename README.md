<div align="center">

<h1>🔐 CredGuard</h1>
<h3>Secure Academic Credential Verification using Blockchain + AI</h3>

![Next.js](https://img.shields.io/badge/Next.js-15-black?logo=next.js)
![Node.js](https://img.shields.io/badge/Node.js-Express-green?logo=node.js)
![Solidity](https://img.shields.io/badge/Solidity-Smart_Contract-363636?logo=ethereum)
![FastAPI](https://img.shields.io/badge/FastAPI-OCR_Service-009688?logo=fastapi)
![MongoDB](https://img.shields.io/badge/MongoDB-Database-4EA94B?logo=mongodb)
![Supabase](https://img.shields.io/badge/Supabase-Auth-3ECF8E?logo=supabase)

</div>

---

## 📌 What is CredGuard?

CredGuard is a **full-stack academic credential verification platform** that eliminates certificate fraud by combining **blockchain immutability**, **AI-powered OCR tamper detection**, and an immersive **3D cyberpunk UI**.

- 🏛️ **Universities** issue digital certificates — hashed and stored on-chain via a Solidity smart contract.
- 🎓 **Students** access a wallet-like dashboard to view and share QR-coded verification links.
- 🏢 **Employers/Institutions** instantly verify uploaded certificates against on-chain hashes with AI tamper analysis.

---

## 🧱 Architecture Overview

```
┌──────────────────────────────────────────────────────────────────┐
│                        CredGuard System                          │
│                                                                  │
│  ┌─────────────┐    ┌─────────────┐    ┌──────────────────────┐ │
│  │  Next.js    │───▶│  Express    │───▶│  MongoDB             │ │
│  │  Frontend   │    │  Backend    │    │  (Credentials, Users)│ │
│  │  (Port 3000)│    │  (Port 4000)│    └──────────────────────┘ │
│  └─────────────┘    │             │                              │
│                     │             │───▶  Polygon/Hardhat         │
│                     │             │     (CredentialRegistry.sol) │
│                     │             │                              │
│                     │             │───▶  FastAPI OCR             │
│                     │             │     (Port 8000)              │
│                     └─────────────┘                              │
└──────────────────────────────────────────────────────────────────┘
```

---

## 🌟 Key Features

| Feature | Description |
|---|---|
| 🔗 **On-Chain Verification** | Certificate hashes stored in `CredentialRegistry` Solidity contract |
| 🤖 **AI Tamper Detection** | Tesseract OCR + OpenCV heuristics via FastAPI microservice |
| 📱 **QR Code Sharing** | Students share scan-to-verify QR links |
| 🔐 **Supabase Auth** | Google OAuth + email/password via Supabase |
| 🌐 **3D Animated UI** | React Three Fiber, Framer Motion, and GSAP animations |
| 📊 **Admin Dashboard** | University admins can issue and manage credentials |
| 🔎 **Explorer Page** | Public on-chain credential lookup |

---

## 🗂️ Project Structure

```
Cred-Gaurd/
├── frontend/           # Next.js 15 App Router UI
│   └── src/app/
│       ├── page.tsx        # Landing page (3D animated)
│       ├── issue/          # Admin — issue credentials
│       ├── verify/         # Employer — verify certificate
│       ├── student/        # Student dashboard
│       ├── admin/          # Admin dashboard
│       └── explorer/       # Public blockchain explorer
│
├── backend/            # Node.js + Express REST API
│   └── src/
│       ├── controllers/    # Route handlers
│       ├── models/         # MongoDB schemas
│       ├── routes/         # API routes
│       ├── services/       # Blockchain & OCR integration
│       └── middleware/     # Auth, error handling
│
├── blockchain/         # Hardhat project
│   ├── contracts/
│   │   └── CredentialRegistry.sol
│   └── scripts/
│       └── deploy.ts
│
├── ocr-service/        # FastAPI + Tesseract OCR
│   └── app/
│       └── main.py     # POST /analyze endpoint
│
└── docs/
    └── architecture.md
```

---

## 🛠️ Tech Stack

| Layer | Technology |
|---|---|
| **Frontend** | Next.js 15 (App Router), React, TypeScript, TailwindCSS |
| **3D / Animation** | React Three Fiber, Framer Motion, GSAP |
| **Backend** | Node.js, Express, TypeScript |
| **Database** | MongoDB, Supabase |
| **Auth** | Supabase Auth (Google OAuth + Email) |
| **Blockchain** | Solidity, Hardhat, Ethers.js (Polygon Mumbai) |
| **AI / OCR** | FastAPI, Tesseract OCR, OpenCV |
| **QR Codes** | qrcode npm package |

---

## ⚙️ Getting Started

### Prerequisites

- Node.js v18+
- Python 3.9+
- Tesseract OCR (`sudo apt install tesseract-ocr`)
- MongoDB instance (local or Atlas)
- Supabase project (for auth)
- MetaMask + Polygon Mumbai testnet (for blockchain)

---

### 1. Clone & Install Root Dependencies

```bash
git clone https://github.com/Sohampatil00/Cred-Gaurd.git
cd Cred-Gaurd
npm install
```

---

### 2. Configure Backend Environment

```bash
cd backend
cp .env.example .env
```

Edit `.env`:

```env
MONGO_URI=mongodb://localhost:27017/cred-guard
JWT_SECRET=your-jwt-secret
OCR_SERVICE_URL=http://localhost:8000
ETH_RPC_URL=https://polygon-mumbai.infura.io/v3/YOUR_INFURA_KEY
REGISTRY_CONTRACT_ADDRESS=0xYourDeployedContractAddress
FRONTEND_URL=http://localhost:3000
ISSUER_PRIVATE_KEY=0xYourTestnetPrivateKey
```

---

### 3. Deploy Smart Contract (Polygon Mumbai Testnet)

```bash
cd blockchain
npm install
npx hardhat compile
npx hardhat run scripts/deploy.ts --network mumbai
```

> Copy the deployed contract address into `backend/.env` as `REGISTRY_CONTRACT_ADDRESS`.

---

### 4. Start OCR Service

```bash
cd ocr-service
python -m venv .venv
source .venv/bin/activate          # Windows: .venv\Scripts\activate
pip install -r requirements.txt
uvicorn app.main:app --reload --host 0.0.0.0 --port 8000
```

---

### 5. Run All Services (Dev)

From the project root, run frontend + backend together:

```bash
npm run dev:all
```

Or individually:

```bash
npm run dev:frontend    # http://localhost:3000
npm run dev:backend     # http://localhost:4000/api
```

---

## 🔑 Smart Contract — `CredentialRegistry`

Located at `blockchain/contracts/CredentialRegistry.sol`

| Function | Description |
|---|---|
| `issueCertificate(...)` | Stores a new certificate hash + metadata on-chain (issuer only) |
| `storeHash(...)` | Updates the stored PDF hash (for re-issuance) |
| `getCertificate(id)` | Reads the full certificate struct from the chain |
| `verifyCertificate(id, hash)` | Returns `true` if the provided hash matches the stored hash |

---

## 📡 API Endpoints (Backend)

| Method | Route | Description |
|---|---|---|
| `POST` | `/api/auth/login` | User login |
| `POST` | `/api/credentials/issue` | Issue a new credential |
| `GET` | `/api/credentials/:id` | Get credential by ID |
| `POST` | `/api/credentials/verify` | Verify a certificate |
| `POST` | `/api/ocr/analyze` | Run OCR tamper analysis |

---

## 🤖 OCR Service

The FastAPI microservice runs on `http://localhost:8000` and exposes:

```
POST /analyze
Content-Type: multipart/form-data
Body: { file: <pdf_or_image> }

Response: {
  "extracted_fields": { ... },
  "tamper_score": 0.12,
  "is_tampered": false
}
```

---

## 🚀 Deployment

- **Frontend**: Vercel (`vercel deploy` from `frontend/`)
- **Backend**: Render / Railway / AWS EC2
- **OCR Service**: Render (Docker) or Fly.io
- **Blockchain**: Polygon Mumbai (testnet) or Polygon Mainnet

---

## 📜 License

This project is built as a **B.Tech Mini-Project** for academic demonstration purposes.

---

<div align="center">
Made with ❤️ using Blockchain + AI + Next.js
</div>
