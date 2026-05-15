# Rubik's Cube Solver

A full-stack application that solves a scrambled Rubik's Cube from photos. Users upload one image per face, and the system uses AI to detect the tile colours and compute an optimal step-by-step solution.

## Repositories

| Repo | Description |
|------|-------------|
| [rubiks-cube-backend](https://github.com/faizanishaq94/rubiks-cube-backend) | Core API — job creation, image processing pipeline, AI tile detection, cube solving |
| [rubiks-cube-auth](https://github.com/faizanishaq94/rubiks-cube-auth) | Authentication service — registration, login, token management via AWS Cognito |
| rubiks-cube-solver-web *(coming soon)* | Next.js web frontend — job submission, status polling, solution display |
| rubiks-cube-solver-mobile *(coming soon)* | React Native + Expo mobile app — iOS and Android |

## Architecture

```
┌──────────────┐     ┌──────────────┐
│  Next.js Web │     │ React Native │
│   Frontend   │     │  Mobile App  │
└──────┬───────┘     └──────┬───────┘
       │                    │
       ▼                    ▼
┌─────────────────────────────────────┐
│           AWS API Gateway           │
│   JWT authorisation, routing,       │
│   throttling                        │
└──────┬──────────────────┬───────────┘
       │                  │
       ▼                  ▼
┌─────────────┐   ┌──────────────────┐
│ Auth Service│   │ Backend Service  │
│  Port 3001  │   │   Port 3002      │
│             │   │                  │
│ AWS Cognito │   │ AWS S3  (images) │
│ PostgreSQL  │   │ AWS SQS (queue)  │
└─────────────┘   │ PostgreSQL       │
                  │ Anthropic API    │
                  └────────┬─────────┘
                           │
                           ▼
                  ┌──────────────────┐
                  │   SQS Worker     │
                  │  Long-polls for  │
                  │  S3 upload events│
                  └──────────────────┘
```

## How It Works

1. **Upload** — User submits one photo per cube face (6 total). The backend generates presigned S3 URLs so images upload directly from the client to S3, bypassing the backend entirely.

2. **Queue** — Each successful S3 upload fires a notification to an SQS queue. The backend's SQS worker long-polls the queue and picks up each event.

3. **Detect** — Once all 6 images are uploaded, the worker sends them to Anthropic Claude, which identifies the colour of every tile on every face.

4. **Solve** — The detected cube state is passed to Kociemba's algorithm, which computes an optimal solution in under 20 moves.

5. **Result** — The solution (move sequence in WCA notation) is stored and the job status is updated. The client polls for completion and displays the result.

## Tech Stack

| Layer | Technology |
|-------|------------|
| Backend runtime | Node.js + TypeScript |
| HTTP framework | Express.js |
| Database | PostgreSQL + Prisma ORM |
| Authentication | AWS Cognito |
| Object storage | AWS S3 + presigned URLs |
| Message queue | AWS SQS |
| AI / tile detection | Anthropic Claude |
| Cube solving | Kociemba's algorithm |
| Web frontend | Next.js + React + Tailwind CSS |
| Mobile frontend | React Native + Expo |
| Validation | Zod |
| Containerisation | Docker (multi-stage builds) |

## Key Technical Decisions

- **Presigned S3 URLs** — Images upload directly from the client to S3 rather than through the backend. This removes the backend as a bottleneck for large file transfers and reduces server load.

- **Asynchronous processing via SQS** — Image processing is decoupled from the HTTP request cycle. The job creation endpoint returns immediately; the heavy work (AI detection + solving) happens in the background. The client polls for completion.

- **Defence in depth on auth** — API Gateway validates JWTs at the infrastructure level. The backend's `requireAuth` middleware re-verifies the signature independently using Cognito's public JWKS keys, so a misconfigured gateway or direct service access doesn't bypass authentication.

- **Microservice separation** — Auth is a standalone service so it can be deployed, scaled, and updated independently of the core job processing logic. Both services share a PostgreSQL database but own their own schema concerns.

- **S3 key structure** — Images are stored at `{userId}/jobs/{jobId}/{face}`, grouping by user at the top level. This allows IAM and bucket policies to be scoped per user and makes cleanup straightforward (delete all objects under a job prefix).

## Services

### rubiks-cube-backend

Handles all job lifecycle — creation, status tracking, and result storage. Runs an Express server alongside a long-polling SQS worker in the same process. The worker parses S3 event notifications to identify which job and face each upload belongs to, then triggers the AI detection and solving pipeline once all 6 images are present.

### rubiks-cube-auth

Wraps AWS Cognito to provide a clean REST API for the full authentication lifecycle — register, confirm email, login, token refresh, logout, forgot/reset password, and get current user. On registration, the service writes the user to its own PostgreSQL table and rolls back the Cognito record if the database write fails, keeping the two systems in sync.

### rubiks-cube-solver-web *(coming soon)*

Next.js web frontend with a dark-mode design system. Features a 6-face drag-and-drop photo uploader, real-time job status polling, and solution display using WCA move notation.

### rubiks-cube-solver-mobile *(coming soon)*

React Native + Expo mobile app targeting iOS and Android. Mirrors the web feature set with native navigation, camera/gallery integration for photo capture, and secure token storage via `expo-secure-store`.
