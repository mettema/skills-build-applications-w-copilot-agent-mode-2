# OctoFit Tracker architecture

OctoFit Tracker is a three-tier application for a school fitness program. The starter code includes API contracts and intentionally incomplete implementation points so you can use GitHub Copilot agent mode in a test-driven loop.

![Three-tier OctoFit architecture](../.github/images/octofit-architecture.svg)

## Tier boundaries

- **Presentation tier**: React 19, Vite, React Router, Bootstrap, and a shared API client in `octofit-tracker/frontend/src`.
- **Logic tier**: Express and TypeScript routes/services in `octofit-tracker/backend/src` expose `/api/*` behavior on port `8000`.
- **Data tier**: Mongoose models and seed data in `octofit-tracker/backend/src/models` and `src/scripts/seed.ts` persist to MongoDB on port `27017`.

## Ports and environment rules

- MongoDB: `27017`
- Express API: `8000`
- Vite frontend: `5173`
- Only ports `8000`, `5173`, and `27017` should appear in application configuration.
- Service configuration that sets, forwards, or publishes application ports, including .env, Docker/Codespaces, and CI configuration, must use only 8000 (API), 5173 (Vite), and 27017 (MongoDB). This rule does not apply to runtime logs or incidental documentation references; documentation examples should use these ports.

### MongoDB availability

The backend contract tests start an isolated MongoDB instance with
`mongodb-memory-server`; you do not need a separate MongoDB server to run
`npm test --prefix octofit-tracker/backend`.

To run the API or seed data against a persistent database, MongoDB must be
available at `mongodb://localhost:27017` or at the URI configured by
`MONGODB_URI`. In Codespaces, the container startup script starts MongoDB
automatically. Check that it is ready with:

```bash
mongosh --quiet --eval 'db.runCommand({ping:1})'
```

If the check fails in Codespaces, inspect `/tmp/mongod.log` for startup errors
and restart the Codespace so its startup script can try again. In other
environments, start or restart MongoDB using that environment's service manager,
then rerun the readiness check before starting the API or seeding data.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Keep route-level UI in React, use the shared API client for relative /api requests, and show accessible loading, empty, and error states.
- Logic tier decision: Keep request validation and capability behavior in Express routes/services; return deliberate 4xx responses for invalid or unknown records.
- Data tier decision: Use Mongoose models for required fields, references, and basic validation; keep seed data idempotent.
- Activity tracking approach: Validate user/team IDs and existence before creating an activity; store the references, type, minutes, computed points, and occurrence time.
- Team management approach: Require a name, preserve the submitted name and mascot, and support member references when listing and creating teams.
- Leaderboard approach: Aggregate activity points by team, sort highest first, and assign ranks starting at 1.
- Workout suggestions approach: Match the user by email, use their goal or recent activities to choose a suggestion, include a reason, and return 4xx when no user matches.

## Installation

Install backend command: `npm ci --prefix octofit-tracker/backend`
Install frontend command: `npm ci --prefix octofit-tracker/frontend`

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend`
- API command: `npm run test:api --prefix octofit-tracker/backend`
- Frontend command: `npm test --prefix octofit-tracker/frontend`
- Backend build command: `npm run build --prefix octofit-tracker/backend`
- Frontend build command: `npm run build --prefix octofit-tracker/frontend`

Run the full suites once before editing to capture the red baseline, then re-run the matching tier command after each small change.
