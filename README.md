# Campus Service Request & Issue Management System (Backend)

A complete REST API for the CS3542 Full Stack Development-II lab assessment. It uses **Node.js, Express, MongoDB/Mongoose, JWT and bcryptjs**. No frontend is included, as the assessment says a frontend is not required.

## 1. Requirements
- Node.js 18+
- MongoDB running locally, or a MongoDB Atlas connection string
- Postman or Thunder Client for API testing

## 2. Setup
1. Extract the ZIP and open this folder in VS Code.
2. Open a terminal in this folder and run:
   ```bash
   npm install
   ```
3. Copy `.env.example` to a new file named `.env`.
4. Update `MONGODB_URI` and `JWT_SECRET` in `.env`. Keep `.env` private.
5. Start MongoDB if using a local database.
6. Create the admin account:
   ```bash
   npm run seed:admin
   ```
   The initial credentials are the values in `.env` (`admin@campus.edu` / `Admin@12345` by default). Change them before use.
7. Start the API:
   ```bash
   npm run dev
   ```
8. Open `http://localhost:5000/`. A JSON welcome response confirms that the API is running.

## 3. Authentication
Register a student using `POST /api/auth/register`, then login using `POST /api/auth/login`. Copy the returned token. For protected endpoints, add this header in Postman:

`Authorization: Bearer YOUR_JWT_TOKEN`

Public registration always creates a **student**. Admin accounts are created with the seed script so a user cannot grant themselves admin access by sending `role: "admin"`.

## 4. API endpoints

| Method | Endpoint | Access | Purpose |
|---|---|---|---|
| POST | `/api/auth/register` | Public | Register student |
| POST | `/api/auth/login` | Public | Login and receive JWT |
| POST | `/api/requests` | Student | Create service request |
| GET | `/api/requests` | Student/Admin | Student sees own requests; admin sees all |
| GET | `/api/requests/:id` | Owner/Admin | View request details |
| PUT | `/api/requests/:id` | Owner (Pending only) | Update title, description or category |
| DELETE | `/api/requests/:id` | Owner/Admin | Student may delete own Pending request; admin may delete any |
| PUT | `/api/requests/:id/status` | Admin | Advance workflow and optionally add response |
| PUT | `/api/requests/:id/assign` | Admin | Assign to an admin/staff account |\n| PATCH | `/api/requests/:id/priority` | Admin | Set Low, Medium or High priority |
| GET | `/api/admin/dashboard` | Admin | Dashboard statistics |

### Query parameters for admin request listing
- `status`: Pending, Under Review, Assigned, In Progress, Resolved, Closed, Rejected
- `priority`: Low, Medium, High
- `category`: Computer/IT, Classroom Equipment, Internet, Hostel Maintenance, Electrical, Library, Other
- `search`: text search across title and description
- `page`, `limit`: pagination (maximum limit 100)

Example: `GET /api/requests?status=Pending&priority=High&search=internet`

## 5. Request workflow
Allowed transitions are deliberately restricted:
- Pending → Under Review → Assigned → In Progress → Resolved → Closed
- Pending → Rejected
- Under Review → Rejected

A request cannot jump directly from Pending to Closed. Use the status endpoint for normal transitions. The assign endpoint assigns a staff/admin account and sets status to Assigned; it accepts requests currently Under Review or Assigned.

Example status body:
```json
{
  "status": "Under Review",
  "adminResponse": "Your request has been received and is being reviewed."
}
```

Example assignment body:
```json
{
  "assignedTo": "ADMIN_USER_OBJECT_ID"
}
```

## 6. Example request bodies

### Register student
`POST /api/auth/register`
```json
{
  "name": "Aarav Sharma",
  "email": "aarav@example.com",
  "password": "Student@123",
  "department": "Computer Science"
}
```

### Login
`POST /api/auth/login`
```json
{
  "email": "aarav@example.com",
  "password": "Student@123"
}
```

### Create request
`POST /api/requests`
```json
{
  "title": "Computer in Lab 204 is not working",
  "description": "The desktop does not start when the power button is pressed. This is affecting our practical session.",
  "category": "Computer/IT"
}
```

### Update Pending request
`PUT /api/requests/REQUEST_ID`
```json
{
  "title": "Computer in Lab 204 is not starting",
  "description": "The desktop still does not start after checking the power cable.",
  "category": "Computer/IT"
}
```

### Change status as admin
`PUT /api/requests/REQUEST_ID/status`
```json
{
  "status": "Under Review",
  "adminResponse": "The issue has been received by the administration."
}
```

### Set priority
Priority is set by admin. The assessment requires admin priority management; this API provides it through the dedicated endpoint:
`PATCH /api/requests/REQUEST_ID/priority`
```json
{ "priority": "High" }
```

### Dashboard
`GET /api/admin/dashboard` returns total requests, Pending, In Progress, Resolved, High Priority, counts by status and counts by category.

## 7. Suggested Postman testing order
1. Register a student and save the returned JWT.
2. Login as admin using the seeded credentials and save that JWT.
3. As student, create a request; copy its `_id`.
4. As student, list requests and verify only their own request is returned.
5. As a different student, try opening the copied ID; expect **403**.
6. As admin, list all requests and test status/priority/category/search filters.
7. As admin, change status Pending → Under Review.
8. Try Pending → Closed on a fresh request; expect **400** (invalid workflow jump).
9. Assign the request to the admin account using its MongoDB user `_id`.
10. Move it through Assigned → In Progress → Resolved → Closed.
11. As admin, call dashboard and verify the counts.
12. Try a protected endpoint without a token (expect **401**) and an admin endpoint with a student token (expect **403**).

## 8. Error responses
- `400`: invalid input or invalid workflow transition
- `401`: missing, invalid or expired JWT
- `403`: wrong role or non-owner access
- `404`: route or resource not found
- `409`: duplicate email

## 9. Project structure
```text
campus-service-management/
├── server.js
├── package.json
├── .env.example
├── README.md
└── src/
    ├── app.js
    ├── config/db.js
    ├── models/User.js
    ├── models/ServiceRequest.js
    ├── controllers/
    ├── middleware/
    ├── routes/
    ├── scripts/seedAdmin.js
    └── utils/
```

## 10. Important notes for your lab viva
- A **User** can create many **ServiceRequest** documents. `createdBy` and `assignedTo` are MongoDB ObjectId references to the User collection.
- Passwords are hashed with bcryptjs; plain-text passwords are never stored.
- JWT middleware verifies the token and loads the user. Role middleware protects admin-only routes.
- The workflow is enforced on the server, not trusted to the client.
- Dashboard statistics use MongoDB aggregation for grouped status and category counts.

This is a backend implementation. You still need to install dependencies, configure MongoDB, run it locally and test the endpoints. Read through the code so you can explain it during your assessment.
