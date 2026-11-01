# Assessment Platform Backend

Express API, BullMQ worker and mail cron for an online assessment platform with timed quiz and coding sections.

![Node.js](https://img.shields.io/badge/Node.js-339933?style=for-the-badge&logo=nodedotjs&logoColor=white)
![Express](https://img.shields.io/badge/Express-000000?style=for-the-badge&logo=express&logoColor=white)
![MongoDB](https://img.shields.io/badge/MongoDB-47A248?style=for-the-badge&logo=mongodb&logoColor=white)
![Redis](https://img.shields.io/badge/Redis-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![BullMQ](https://img.shields.io/badge/BullMQ-DC382D?style=for-the-badge&logo=redis&logoColor=white)
![Docker](https://img.shields.io/badge/Docker-2496ED?style=for-the-badge&logo=docker&logoColor=white)
![SendGrid](https://img.shields.io/badge/SendGrid-1A82E2?style=for-the-badge&logo=twilio&logoColor=white)

## About

This is the backend for a skills assessment platform. A candidate registers with a skill level, is mapped to an assessment, and works through its sections (multiple-choice quizzes and coding problems). Code is run against test cases through a Judge0-compatible execution API, submitted sections are graded asynchronously by a BullMQ worker, and a cron process emails candidates once their attempt has been evaluated.

## Features

- **Registration:** `POST /api/v1/user/register` creates a user and assigns an assessment based on `skillLevel` (easy, medium, hard)
- **Assessment snapshots:** when a solution is created, the assessment's question pools and problem pools are snapshotted onto the candidate's solution, with Redis caching of pools
- **Section flow:** start, submit section by section, auto-advance to the next section, and flag unfair means (UFM)
- **Server time sync** endpoint for client-side timers
- **Code execution:** run code against the first two test cases (`/eval`), submit against all test cases (`/submit`) and poll results by token (`/fetch`). Supports C, C++, Java, Python, JavaScript, Go, Rust, PHP and C#
- **Async grading:** section submissions are queued on `assesmentQueue` (BullMQ, 3 attempts with exponential backoff). The worker scores single-answer MCQs, multi-answer MCQs, text answers and coding sections
- **Notification cron:** every 10 seconds, finds evaluated solutions that have not been notified and sends a thank-you email through SendGrid
- Security and ops middleware: Helmet, CORS allowlist, rate limiting, Morgan request logging

> Note: the `isAuthenticated` and `isAllowed` middlewares are currently stubs (marked TODO) that let every request through.

## Tech Stack

- **Runtime:** Node.js (ES modules), Express 5
- **Database:** MongoDB with Mongoose
- **Queue and cache:** Redis, ioredis, BullMQ
- **Scheduling:** node-cron
- **Email:** SendGrid (`@sendgrid/mail`), Nodemailer
- **Other:** Axios, Helmet, express-rate-limit, Morgan, jsonwebtoken
- **Deployment:** Docker images for the API, worker and cron

## Project Structure

```
.
├── index.js              # API entry point
├── config/               # Mongo, Redis, CORS, rate limiter, Morgan
├── constants/            # Queue names
├── controllers/          # Assessment start / solution / submit logic
├── middlewares/          # Auth stubs
├── models/               # User, Assesment, QuestionPool, Problem, Solution
├── routes/               # assesment, code-execution, user, test routers
├── queue/                # BullMQ queue setup
├── services/             # queueService (add jobs)
├── workers/              # worker.js + evaluator.js (grading)
├── cron/                 # Evaluated-user email cron and templates
├── Dockerfile            # API image
├── worker.Dockerfile     # Worker image
├── cron.Dockerfile       # Cron image
└── docker-compose.yml
```

More detail lives in [ARCHITECTURE.md](ARCHITECTURE.md), [SETUP.md](SETUP.md) and [db.md](db.md).

## Getting Started

Prerequisites: Node.js 18+, Yarn, and reachable MongoDB and Redis instances.

```bash
yarn install
cp .env.example .env   # fill in your values

yarn dev               # API with nodemon (index.js)
yarn worker            # grading worker, in a second terminal
node cron/index.js     # email cron, in a third terminal
```

With Docker, `docker-compose up --build` builds and starts the `backend` (port 3000), `worker` and `cron` services using `.env`. The compose file does not start MongoDB or Redis, so point `MONGODB_URI` and `REDIS_HOST` at your own instances.

### Environment variables

| Name | Purpose |
| ---- | ------- |
| `PORT` | API port |
| `ALLOWED_ORIGINS` | CORS origin |
| `NODE_ENV` | `DEV` or `PROD` (controls logging and CSP) |
| `MONGODB_URI` | MongoDB connection string |
| `MONGODB_DB_NAME` | Database name |
| `REDIS_HOST`, `REDIS_PORT`, `REDIS_PASSWORD` | Redis connection |
| `EXECUTION_API` | Code execution API base URL |
| `CODE_EXECUTION_API` | Code execution API base URL used by the worker |
| `SENDGRID_KEY` | SendGrid API key for result emails |

## API Endpoints

All routes are prefixed with `/api/v1`.

| Method | Path | Description |
| ------ | ---- | ----------- |
| `GET` | `/health` | Health check |
| `POST` | `/user/register` | Register a candidate and assign an assessment by skill level |
| `POST` | `/assesments/solution` | Create or fetch the candidate's solution (assessment snapshot) |
| `POST` | `/assesments/start-assesment` | Start an assessment |
| `PUT` | `/assesments/submit-section` | Submit a section and queue it for grading |
| `POST` | `/assesments/submit-assesment` | Submit the whole assessment |
| `POST` | `/assesments/mark-ufm` | Flag unfair means |
| `GET` | `/assesments/server-sync` | Server time for client timers |
| `GET` | `/assesments/users/assesments/:userId` | List a user's assessments |
| `GET` | `/assesments/solution/:solutionId` | Get a solution by id |
| `POST` | `/code-execution/eval` | Run code against sample test cases |
| `POST` | `/code-execution/submit` | Submit code against all test cases |
| `POST` | `/code-execution/fetch` | Fetch execution results by token |
| `POST` | `/test/add-job` | Add a test job to the queue |

## Author

**Aryan Bhandari**: [GitHub](https://github.com/aryanbha123)
