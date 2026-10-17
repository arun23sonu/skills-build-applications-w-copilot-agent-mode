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
- The browser calls relative `/api` paths. Vite proxies them to `http://localhost:8000`, so the unauthenticated API port stays private in Codespaces. `VITE_API_BASE_URL` can override the base URL.

## API contract

| Capability | Method and path | Expected behavior |
| --- | --- | --- |
| Activity tracking | `GET /api/activities`, `POST /api/activities` | List and create valid activity records. POST accepts `userId`, `teamId`, `type`, and `minutes`; `userId` and `teamId` must be well-formed IDs for an existing user and team; otherwise respond with a deliberate 4xx (never a 500 from a cast error). A created activity stores the submitted `user`, `team`, `type`, and `minutes` plus computed `points` and `occurredAt`. |
| Teams | `GET /api/teams`, `POST /api/teams` | List and create teams with members. Created and listed teams keep the submitted `name` and `mascot`. Reject a team without a name with `400`. |
| Leaderboard | `GET /api/leaderboard` | Return teams ordered by the sum of their activity points, highest first. Each entry has `teamName`, `totalPoints`, and a 1-based `rank`. |
| Workout suggestions | `GET /api/workouts/suggestions/:userEmail` | Recommend workouts from recent activity patterns for the matching seeded user's email address. The first suggestion's `focus` matches that user's goal or recent activity, with a `reason` explaining why. An email with no matching user returns a deliberate 4xx (for example, `404`). |

## Implementation decisions

- Presentation tier decision: Build the UI as route-oriented React views that fetch from a single API client, keep state local to each screen, and render explicit loading, empty, and error states so the experience matches the contract without coupling to database internals.
- Logic tier decision: Implement small Express endpoints and a shared service layer in `octofit-tracker/backend/src` that validate input, convert IDs safely, and return consistent JSON payloads with 4xx responses for invalid requests and 2xx/201 for valid writes.
- Data tier decision: Model the domain around `User`, `Team`, `Activity`, and `Workout` in Mongoose, use ObjectId references for relationships, and keep the seed script idempotent so rerunning it does not duplicate or corrupt seeded fixtures.
- Activity tracking approach: Accept `userId`, `teamId`, `type`, and `minutes`, validate both references before writing, and compute a non-zero `points` value plus an ISO timestamp. Reject malformed IDs or missing owners with a deliberate 4xx instead of allowing a Mongo cast error to leak as a 500.
- Team management approach: Require `name` and `mascot` when creating teams, keep `members` as an array of user IDs, and ensure listed teams preserve the submitted payload while exposing members for later leaderboard and suggestion logic.
- Leaderboard approach: Aggregate team points from activity records, sort by total points descending, and assign a 1-based rank per team so the API matches the expected ranking contract.
- Workout suggestions approach: Match the request email to a seeded user, prefer the user’s fitness goal or recent activity focus, return the best matching workout(s) with a human-readable `reason`, and return a 4xx for unknown emails instead of falling back to an empty success response.

## Test plan

- Data-tier command: `npm run test:data --prefix octofit-tracker/backend` — fail here means the Mongoose schemas or the seed script do not satisfy the domain contract, such as missing validation, non-idempotent seeding, or broken relationships.
- API command: `npm run test:api --prefix octofit-tracker/backend` — fail here means the Express logic is returning the wrong status codes, wrong payload shapes, or incorrect business rules for team creation, leaderboard ordering, or workout suggestions.
- Frontend command: `npm test --prefix octofit-tracker/frontend` — fail here means the UI is not rendering the API data correctly, not honoring loading/empty/error states, or not requesting the right workout URL for the current user.
- Backend build command: `npm run build --prefix octofit-tracker/backend` — fail here means the TypeScript server or shared backend modules do not compile cleanly after the logic tier work.
- Frontend build command: `npm run build --prefix octofit-tracker/frontend` — fail here means the presentation tier has type or bundling issues that will prevent the app from building even if tests pass in isolation.
