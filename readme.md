# REST API Backend Boilerplate

A reusable Express and MongoDB backend for projects that need user accounts. Start with registration, email verification, cookie-based sessions, password recovery, and role-based access. Add your own resources to the same API. For a storefront, the boilerplate also includes products, reviews, and orders.

It also includes a front-end for account verification and password reset.

The backend lives in `server/`. You can exercise every endpoint with curl or Postman independently of the React authentication client in `front-end/`.


## External Services & Integrations
** **
- **MongoDB** – Database for storing and managing application data.
- **Resend** – Email delivery service for transactional emails.

## Use it for your project

The backend runs independently from server/. The optional React app in front-end/ demonstrates the account flows. You can also connect your own client or work directly in Postman.

To add a resource such as a task or booking, define its Mongoose schema in server/models/, put its request handlers in server/controllers/, and map HTTP methods to those handlers in server/routes/. Mount the router in server/app.js under /api/v1. Reuse authenticateUser for signed-in routes and authorizePermissions('admin') after it for admin actions.

The commerce routers are mounted by default. For a project that only needs accounts, remove the product, review, and order router imports and their three mounts from server/app.js. The auth and user routes work independently.


## Run locally

1. Start MongoDB locally or obtain a MongoDB connection URI. Use Node.js 20 or later with npm. The server manifest declares Node 14, but the installed Resend email client requires Node 20 or later.
2. From the project root, run `cd server` and `npm install`. Use `npm install` .
3. From the project root, run `cd front-end` and `npm install`. This is optional. The front-end is there if needed
4. Create `server/.env` with the following settings. Generate a secret with `openssl rand -hex 32` and supply your own Resend API key for verification and password-reset emails.

   ```dotenv
   MONGO_URL=mongodb://127.0.0.1:27017/rest_api_boilerplate
   JWT_SECRET=replace-with-generated-secret
   RESEND_API_KEY=replace-with-your-resend-api-key
   PORT=5001
   ```

4. Run `npm start` from `server/`. Once the server connects to MongoDB, try the public catalog from a second terminal.

```sh
API=http://localhost:5001/api/v1
curl "$API/products"
```

On an empty database, the response is:

```json
{ "products": [], "count": 0 }
```

## Register, verify, and sign in

All paths below are relative to `/api/v1`. For JSON requests, set `Content-Type: application/json`. Use an inbox you can access wherever the examples show `ada@example.com`. The mail helper uses `onboarding@resend.dev`, so choose a recipient your Resend account can deliver to.

**1. Register** with `POST /auth/register`. The name must be 3–50 characters and the password at least 6 characters. The first account in an empty database becomes the admin and can create products.

```json
{
  "name": "Ada Okafor",
  "email": "ada@example.com",
  "password": "Example-pass-42"
}
```

**2. Verify** with `POST /auth/verify-email`. Copy the emailed link's `token` query parameter into the `verificationToken` field. This works without starting the React client.

```json
{
  "email": "ada@example.com",
  "verificationToken": "paste-token-from-verification-link"
}
```

**3. Sign in** with `POST /auth/login`.

```json
{
  "email": "ada@example.com",
  "password": "Example-pass-42"
}
```

Login sets signed HTTP-only cookies named `accessToken` and `refreshToken`. Leave cookies enabled in Postman so it saves them after sign-in and sends them on later requests to the API. With curl, `-c cookies.txt` saves the cookies and `-b cookies.txt` reads them for the next request.

```sh
curl -c cookies.txt -H 'Content-Type: application/json' \
  -d '{"email":"ada@example.com","password":"Example-pass-42"}' "$API/auth/login"
curl -b cookies.txt -c cookies.txt "$API/users/showMe"
```

Both calls return the current user in this shape:

```json
{ "user": { "name": "Ada Okafor", "userId": "665000000000000000000001", "role": "admin" } }
```

**4. Request a password reset** with `POST /auth/forgot-password`. Enter the email address you registered with.

```json
{
  "email": "ada@example.com"
}
```

**5. Reset your password** with `POST /auth/reset-password`. Copy the `email` and `token` query parameters from the emailed link and enter your new password in `password`. The link points to `http://localhost:3000`, but you can submit this request directly to the API. The token expires after 10 minutes.

```json
{
  "email": "ada@example.com",
  "token": "paste-token-from-reset-link",
  "password": "New-example-pass-43"
}
```

Sign in with the new password to confirm the change. The reset endpoint returns the plain text `reset password` .

## Create a product, review it, and place an order

**Create a product** with an admin session and `POST /products`. Allowed categories are `office`, `kitchen`, and `bedroom`. Allowed companies are `ikea`, `liddy`, and `marcos`.

```json
{
  "name": "Oak desk",
  "description": "A compact writing desk with one drawer.",
  "price": 15000,
  "category": "office",
  "company": "ikea",
  "inventory": 8
}
```

The server returns HTTP 201 with the new document under `product`. Copy its `_id` and replace the sample product ID in both requests below.

**Add a review** with `POST /reviews` while signed in. Each user can review a product once. Ratings range from 1 to 5.

```json
{
  "product": "665000000000000000000002",
  "rating": 4,
  "title": "Good fit for a small office",
  "comment": "The drawer holds a notebook and the desktop feels sturdy."
}
```

**Place an order** with `POST /orders` while signed in. Use a positive integer quantity within available stock. Supply numeric, nonzero values for both fees. The server looks up product prices and calculates the total. Two desks at 15000 each, plus the tax and shipping below, total 32000.

```json
{
  "items": [{ "product": "665000000000000000000002", "amount": 2 }],
  "tax": 1500,
  "shippingFee": 500
}
```

The HTTP 201 response contains `order` and `clientSecret`. Payment processing is simulated. To mark a pending order paid, send the following body to `PATCH /orders/:id`, substituting the returned `order._id` in the path. While the server process stays running, it cancels orders still pending after 20 minutes and restores their inventory.

```json
{ "paymentIntentId": "pi_demo_001" }
```

## Endpoint reference

| Method | Path | Access | Purpose or request body |
| --- | --- | --- | --- |
| POST | `/auth/register` | Public | Register with name, email, and password |
| POST | `/auth/verify-email` | Public | Verify with email and verificationToken |
| POST | `/auth/login` | Public | Sign in with email and password |
| DELETE | `/auth/logout` | Signed in | End the session and clear cookies |
| POST | `/auth/forgot-password` | Public | `{"email":"ada@example.com"}` |
| POST | `/auth/reset-password` | Public | `{"email":"ada@example.com","token":"paste-reset-token","password":"New-example-pass-43"}` |
| GET | `/users/showMe` | Signed in | Read the current session user |
| PATCH | `/users/updateUser` | Signed in | `{"name":"Ada Okafor","email":"ada@example.com"}` |
| PATCH | `/users/updateUserPassword` | Signed in | `{"oldPassword":"Example-pass-42","newPassword":"New-example-pass-43"}` |
| GET | `/users` | Admin | List accounts with the user role |
| GET | `/users/:id` | Self or admin | Read a user |

### Optional e-commerce API

| Method | Path | Access | Purpose or request body |
| --- | --- | --- | --- |
| GET | `/products`, `/products/:id` | Public | List products or fetch one with its reviews |
| POST | `/products` | Admin | Create a product |
| PATCH | `/products/:id` | Admin | Update fields, e.g. `{"inventory":12}` |
| DELETE | `/products/:id` | Admin | Delete a product |
| POST | `/products/uploadImage` | Admin | Send multipart form field `image` with an image up to 1 MiB. Use the returned `image` path on a product. |
| GET | `/products/:id/reviews` | Public | List reviews for one product |
| GET | `/reviews`, `/reviews/:id` | Public | List reviews or fetch one |
| POST | `/reviews` | Signed in | Create a review |
| PATCH | `/reviews/:id` | Author or admin | Send all three fields, e.g. `{"rating":5,"title":"Still sturdy","comment":"Holding up well."}` |
| DELETE | `/reviews/:id` | Author or admin | Delete a review |
| POST | `/orders` | Signed in | Place an order |
| GET | `/orders/showAllMyOrders` | Signed in | List your orders |
| GET | `/orders` | Admin | List all orders |
| GET | `/orders/:id` | Owner or admin | Read an order |
| PATCH | `/orders/:id` | Owner or admin | Mark a pending demo order paid |
