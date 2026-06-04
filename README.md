# pesapal-node-boilerplate

A clean, framework-agnostic Pesapal v3 integration for Node.js projects.

Works with Express, Next.js, or any Node.js framework. The core service layer has zero framework dependencies — plug it into whatever stack you're building on.

---

## What this covers

- Authentication (bearer token generation and refresh)
- Order submission and payment redirect
- IPN (Instant Payment Notification) registration and handling
- Transaction status check
- Refunds
- Recurring payments
- Sandbox and production environments

---

## Requirements

- Node.js v18 or higher
- A Pesapal merchant account — [register here](https://www.pesapal.com)
- Pesapal API credentials (Consumer Key and Consumer Secret)
- A publicly accessible URL for IPN callbacks (use [ngrok](https://ngrok.com) for local development)

---

## Getting started

### 1. Clone the repository

```bash
git clone https://github.com/K-Irungu/pesapal-node-boilerplate.git
cd pesapal-node-boilerplate
```

### 2. Install dependencies

```bash
npm install
```

### 3. Set up environment variables

```bash
cp .env.example .env
```

Open `.env` and fill in your credentials:

```env
PESAPAL_CONSUMER_KEY=your_consumer_key
PESAPAL_CONSUMER_SECRET=your_consumer_secret
PESAPAL_ENV=sandbox                          # sandbox | production
PESAPAL_IPN_URL=https://yourdomain.com/api/ipn
```

---

## Project structure

```
pesapal-node-boilerplate/
├── src/
│   ├── config/
│   │   └── pesapal.config.js       # Environment setup and base URLs
│   ├── services/
│   │   └── pesapal.service.js      # Core Pesapal logic — framework agnostic
├── examples/
│   ├── express/
│   │   ├── app.js                  # Express app setup
│   │   └── routes.js               # Express route handlers
│   └── nextjs/
│       └── pages/api/
│           ├── pay.js              # Next.js payment route
│           └── ipn.js              # Next.js IPN handler
├── .env.example
├── package.json
└── README.md
```

---

## How it works

### Authentication

Pesapal requires a bearer token on every request. The service handles token generation and attaches it automatically via an Axios interceptor — you never manage tokens manually.

```javascript
import { getAuthToken } from './src/services/pesapal.service.js'

const token = await getAuthToken()
```

---

### Submit an order

Call `submitOrder()` with the order details. It returns a redirect URL — send the user there to complete payment on Pesapal's hosted page.

```javascript
import { submitOrder } from './src/services/pesapal.service.js'

const { redirectUrl, orderTrackingId } = await submitOrder({
  amount: 1500,
  currency: 'KES',
  description: 'Order #1234',
  callbackUrl: 'https://yourdomain.com/payment/callback',
  billingAddress: {
    firstName: 'John',
    lastName: 'Doe',
    emailAddress: 'john@example.com',
    phoneNumber: '0712345678'
  }
})

// Redirect the user to redirectUrl
// Store orderTrackingId to verify payment later
```

---

### Register your IPN URL

Pesapal needs to know where to send payment notifications. Register your IPN URL once — typically on server startup.

```javascript
import { registerIPN } from './src/services/pesapal.service.js'

const { ipnId } = await registerIPN({
  url: process.env.PESAPAL_IPN_URL,
  ipnNotificationType: 'GET'
})
```

---

### Handle IPN callbacks

When a payment is completed, Pesapal hits your IPN URL with the `orderTrackingId`. Verify the transaction status and update your system accordingly.

```javascript
import { getTransactionStatus } from './src/services/pesapal.service.js'

const { status, amount, currency } = await getTransactionStatus(orderTrackingId)

if (status === 'COMPLETED') {
  // Update your database, send confirmation email, etc.
}
```

---

### Check transaction status

You can also check the status of any transaction on demand — useful for order pages and admin dashboards.

```javascript
import { getTransactionStatus } from './src/services/pesapal.service.js'

const transaction = await getTransactionStatus(orderTrackingId)

console.log(transaction.status)    // COMPLETED | FAILED | PENDING | INVALID
```

---

### Process a refund

```javascript
import { processRefund } from './src/services/pesapal.service.js'

const refund = await processRefund({
  orderTrackingId: 'your-order-tracking-id',
  amount: 1500,
  username: 'admin@yourdomain.com',
  remarks: 'Customer requested refund'
})
```

---

### Recurring payments

```javascript
import { createRecurringOrder } from './src/services/pesapal.service.js'

const { redirectUrl } = await createRecurringOrder({
  amount: 2000,
  currency: 'KES',
  description: 'Monthly subscription',
  accountNumber: 'SUB-001',
  billingAddress: {
    firstName: 'Jane',
    lastName: 'Doe',
    emailAddress: 'jane@example.com',
    phoneNumber: '0712345678'
  },
  subscriptionDetails: {
    startDate: '2024-01-01',
    endDate: '2025-01-01',
    frequency: 'MONTHLY'
  }
})
```

---

## IPN flow

```
User completes payment on Pesapal
        ↓
Pesapal sends GET/POST to your IPN URL
        ↓
Your server receives orderTrackingId and orderMerchantReference
        ↓
Call getTransactionStatus(orderTrackingId)
        ↓
Pesapal returns payment status
        ↓
Update your database / trigger business logic
```

---

## Sandbox vs production

The boilerplate reads `PESAPAL_ENV` from your `.env` file and points to the correct base URL automatically.

| Environment | Base URL |
|-------------|----------|
| Sandbox | https://cybqa.pesapal.com/pesapalv3 |
| Production | https://pay.pesapal.com/v3 |

To go live, change `PESAPAL_ENV=sandbox` to `PESAPAL_ENV=production` in your `.env` and swap in your production credentials.

---

## Common errors

**`Consumer key/secret invalid`**
Double check your credentials in `.env`. Make sure you're using sandbox credentials against the sandbox URL and production credentials against the production URL — mixing them is the most common mistake.

**`IPN URL not reachable`**
Pesapal needs to reach your IPN URL from the internet. For local development, use [ngrok](https://ngrok.com) to expose your local server.

**`Token expired`**
Pesapal tokens expire after 5 minutes. The service handles refresh automatically via the Axios interceptor — if you're seeing this error, check that the interceptor is set up correctly.

**`Order already exists`**
Each order needs a unique `merchantReference`. If you're resubmitting an order, generate a new reference.

---

## Framework examples

See the `/examples` folder for complete working implementations:

- [`/examples/express`](./examples/express) — Express.js
- [`/examples/nextjs`](./examples/nextjs) — Next.js API routes

---

## Contributing

Found a bug or want to add something? Open an issue or submit a pull request. All contributions welcome.

---

## License

MIT