 # Healthcare Consultation Assistant (SaaS)

 [![Live Website](https://img.shields.io/badge/Live_Website-6c63ff?logo=rocket&logoColor=white&labelColor=5a52d3)](https://projects.kaushikpaul.co.in/healthcare)

 AI-powered assistant for clinicians to turn consultation notes into:
 - Summary of the visit for records
 - Next steps for the doctor
 - Patient-friendly email draft

 The app provides a polished Next.js UI, secure authentication and subscription gating with Clerk, and a FastAPI backend that streams results in real time using OpenRouter models.

 ## Live Demo
 - https://projects.kaushikpaul.co.in/healthcare

 ## Features
 - **Structured consultation workflow**: Patient name, visit date, and free-form notes.
 - **Streaming AI output**: Real-time Server-Sent Events (SSE) rendering into Markdown.
 - **Authentication & subscriptions**: Clerk-powered sign-in and gated access via `premium_subscription` plan.
 - **Modern UI**: Responsive design, date picker, and rich Markdown display.
 - **Two deployment modes**:
   - Main branch: Dockerized, single-container app on AWS App Runner.
   - `vercel-project` branch: Serverless deployment on Vercel (Next.js + Python function).

 ## Architecture Overview
 - **Frontend** (Next.js, Pages Router)
   - Static export via `next.config.ts` with `output: 'export'`.
   - Auth provider in `src/pages/_app.tsx` using `ClerkProvider`.
   - Product page `src/pages/product.tsx` submits to the backend and renders streaming Markdown.
 - **Authentication & Billing** (Clerk)
   - Client UI components for sign-in and subscription (e.g., `Protect`, `PricingTable`, `UserButton`).
   - Protected product route with `<Protect plan="premium_subscription" />`.
 - **Backend API** (FastAPI)
   - Entrypoint: `src/api/server.py`.
   - Endpoint: `POST /api/consultation` streams AI output in 3 sections.
   - Health check: `GET /health` (for AWS).
   - Uses OpenRouter (`OPENROUTER_API_KEY`) and base URL `https://openrouter.ai/api/v1` with model `google/gemini-2.5-flash-lite`.
   - Verifies Clerk JWTs using JWKS URL (`CLERK_JWKS_URL`).
 - **Static serving (AWS mode)**
   - Next.js static build output (`out/`) is copied into the container at `/app/static` and served by FastAPI for all routes.

 ## Tech Stack
 - **Frontend**: Next.js 15 (Pages Router), React 19, Tailwind CSS 4, React Markdown, react-datepicker
 - **Auth & Billing**: Clerk (`@clerk/nextjs` Protect, PricingTable, UserButton)
 - **Backend**: FastAPI, Uvicorn, Pydantic, CORS middleware
 - **AI**: OpenRouter API
 - **Infra**:
   - AWS App Runner (containerized) on main branch
   - Vercel (serverless) on `vercel-project` branch

 ## Prerequisites
 - Node.js 18+ and npm
 - Python 3.12+
 - Accounts/keys:
   - OpenRouter API key (`OPENROUTER_API_KEY`)
   - Clerk account with Publishable Key and JWKS URL

 ## Environment Variables
 These are read by the frontend build or backend at runtime. Create a local `.env` (not committed) and set in your cloud environments as noted below.

 - **Required**
   - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` (client/build-time)
   - `CLERK_JWKS_URL` (backend runtime)
   - `OPENROUTER_API_KEY` (backend runtime)
 - **Optional**
   - `DEBUG` (set to `true` to enable additional logging if you add it)

 Example `.env` (do not commit):
 ```ini
 NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY=pk_live_or_test_...
 CLERK_JWKS_URL=https://...
 OPENROUTER_API_KEY=sk_or_...
 ```

 ## Local Development
 You can run the frontend and backend locally for fast iteration.

 - **Install dependencies**
 ```bash
 npm install
 pip install -r requirements.txt
 ```

 - **Run Next.js (frontend)**
 ```bash
 npm run dev
 # Opens http://localhost:3000
 ```

 - **Run FastAPI (backend)**
 ```bash
 # Option A (recommended): run from the API folder
 cd src/api
 uvicorn server:app --reload --port 8000

 # Option B: module path from repo root (ensure Python can import src/)
 uvicorn src.api.server:app --reload --port 8000
 ```

 - **Using the app locally**
   - Ensure your Clerk publishable key is available to the frontend (via `.env.local` or shell).
   - The frontend expects the backend at the same origin for `/api/consultation`. If running on different ports, update the fetch base URL in `src/pages/product.tsx` accordingly.

 ## Building the Frontend (Static Export)
 `next.config.ts` sets `output: 'export'`, so a static export is produced by:
 ```bash
 npm run build
 # Outputs to ./out
 ```

 ## Docker (used by AWS deployment)
 The `Dockerfile` builds the Next.js static site and assembles a Python runtime:
 - Stage 1 (Node): `npm ci` + `npm run build` → produces `out/`
 - Stage 2 (Python): installs requirements and copies `src/api/server.py` and `out/` → `/app/static`

 - **Build (requires the public Clerk key)**
 ```bash
 docker build \
   --build-arg NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY="$NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY" \
   -t healthcare-consultation-app .
 ```

 - **Run**
 ```bash
 docker run -p 8000:8000 \
   -e CLERK_JWKS_URL="$CLERK_JWKS_URL" \
   -e OPENROUTER_API_KEY="$OPENROUTER_API_KEY" \
   healthcare-consultation-app
 # Open http://localhost:8000
 ```

 ## Deployment

 ### Main branch → AWS App Runner (containerized)
 - Build and push the image to ECR; App Runner pulls `:latest` and runs on port `8000`.
 - Set environment variables in the App Runner service:
   - `CLERK_JWKS_URL`
   - `OPENROUTER_API_KEY`
 - Health check path: `/health`.
 - Scaling: 0.25 vCPU / 0.5 GB works for demos. Keep max instances low to control cost.

 High-level steps:
 1) Build with the Clerk publishable key build-arg and tag for ECR.
 2) Push to ECR.
 3) Create App Runner service from ECR image, set ENV, port 8000, health check `/health`.
 4) Deploy and verify.

 ### `vercel-project` branch → Vercel (serverless)
 - This branch is set up to deploy with Vercel using a Next.js build and a Python serverless function.
 - Configure environment variables in Vercel Project Settings:
   - `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY` (All environments)
   - `CLERK_JWKS_URL` (All environments)
   - `OPENROUTER_API_KEY` (All environments)
 - Deploy via Vercel CLI or Git integration.

 Note: The serverless Python entrypoint and routing are defined in that branch’s configuration for `/api/consultation`.

 ## API Reference
 - **POST `/api/consultation`**
   - Auth: Bearer token (Clerk JWT) in `Authorization: Bearer <token>`
   - Request body (JSON):
     ```json
     {
       "patient_name": "Jane Smith",
       "date_of_visit": "2025-01-15",
       "notes": "Free-text consultation notes..."
     }
     ```
   - Response: `text/event-stream` (SSE). New content arrives as lines; the frontend in `src/pages/product.tsx` buffers and renders Markdown.
 - **GET `/health`**
   - Simple JSON health check: `{ "status": "healthy" }`

 ## Troubleshooting
 - **Authentication required**: Ensure you are signed in via Clerk and a valid JWT is sent by the frontend.
 - **Clerk JWKS errors**: Verify `CLERK_JWKS_URL` is correct and publicly reachable.
 - **OpenRouter errors**: Confirm `OPENROUTER_API_KEY` is set and has quota. Backend uses base URL `https://openrouter.ai/api/v1`.
 - **SSE not streaming**: Check browser console logs; ensure the endpoint is `/api/consultation` and CORS is open (CORS is configured permissively in the backend).
 - **Static assets not served (AWS)**: The container must include `/app/static` (produced by `npm run build`). Rebuild the image.
 - **Health check failing (AWS)**: Verify the app is listening on port `8000` and `/health` returns 200.

## License
This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.

 ## Acknowledgements
 - Clerk for authentication and subscription UI
 - OpenRouter for model routing
 - Next.js and Tailwind for the frontend
 - FastAPI and Uvicorn for the backend

