# Building a Small Serverless Workout Tracker on AWS

I wanted to build the smallest, cheapest, simpliest complete serverless application possible. My goal is to use this as a base for more applications like it. I built a static frontend, authenticated API, Lambda business logic, and DynamoDB storage. The app is dead simple: create a workout, see previous workouts, and update a selected workout with today's date. The object is simply to see what workouts I haven't done in a while, so I know what I should focus on.

## The Problem

Narrow scope and simple design:

- Serve a frontend without running a web server.
- Protect the app with sign-in.
- Expose a few API routes.
- Run backend code only when requests arrive.
- Store a small amount of structured data.
- Keep operating cost low for a single-user (me).

The result is a useful pattern: static frontend plus authenticated API plus event-driven backend.

## The Solution

The application uses these AWS services:

- **S3** stores the frontend files: `index.html`, `app.js`, `styles.css`, and `config.js`.
- **CloudFront** serves those files over HTTPS.
- **Cognito** provides the hosted sign-in page.
- **API Gateway** exposes the backend routes and validates Cognito tokens.
- **Lambda** handles list, create, and update operations.
- **DynamoDB** stores workout records.

The frontend is plain HTML, CSS, and JavaScript. There is no React, build step, package install, or app server. The backend is a single Python Lambda function using `boto3`, which is available in the AWS Lambda Python runtime.

## How It Works

### Architecture

![Fitness Tracker AWS Architecture](/images/architecture-diagram-aws-icons.svg)

### Frontend Flow

When the user opens the CloudFront URL, the browser loads the static app. No data is exposed until the user signs in.

Clicking sign in starts Cognito Hosted UI login using authorization code with PKCE. After login, Cognito redirects back to the CloudFront URL with an authorization code. The frontend exchanges that code for tokens and stores the token in browser storage.

After that, API requests include:

```text
Authorization: Bearer <id_token>
```

The app then calls `GET /workouts` and renders the workout list.

### Backend Flow

The Lambda function handles three routes:

```text
GET   /workouts
POST  /workouts
PATCH /workouts/{workoutId}
```

The function looks at the HTTP method and path from the API Gateway proxy event, then dispatches to the right operation.

The DynamoDB table uses:

```text
Partition key: pk
Sort key: workoutId
```

Because this is a single-user app, all records use:

```text
pk = USER#single
```

Each workout has its own generated `workoutId`, so duplicate workout names are allowed (I do the same kettlebell and stretch routine 8 times a month).

Example item:

```json
{
  "pk": "USER#single",
  "workoutId": "generated-uuid",
  "workout": "Bench Press",
  "workoutDate": "2026-08-20",
  "weight": 335
}
```

## Key Design Choices

- **Plain frontend files instead of a frontend framework:** This keeps the teaching surface small. The app can be uploaded directly to S3.
- **CloudFront in front of S3:** Phone browsers need a reliable HTTPS URL, and Cognito redirect URLs must match exactly.
- **Cognito SPA app client:** A browser app cannot safely store a client secret, so the frontend uses authorization code with PKCE.
- **API Gateway Cognito authorizer:** Static files can be public, but workout data is protected at the API boundary.
- **Single Lambda for all routes:** For a three-route app, one function is simpler to deploy and reason about.
- **DynamoDB with generated IDs:** Duplicate workout names work naturally because identity is based on `workoutId`, not workout text.


## What’s Next

This project is intentionally small, but the same architecture can grow in useful directions:

- Support multiple users
- Update routing so an unauthenticated user is routed to the login page
- More complex use of HTML/CSS/JS static files for more complex frontend actions
- More complex Lambda functions
- More AWS services leveraged (maybe an AI app that uses AWS AI services)
- Add basic logging and alarms.
- Split Lambda handlers if the API grows beyond a few routes.

The core pattern stays the same: put static assets behind CloudFront, authenticate users with Cognito, protect API Gateway with a Cognito authorizer, run backend logic in Lambda, and store application state in DynamoDB.