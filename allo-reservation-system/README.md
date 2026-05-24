This is a [Next.js](https://nextjs.org) project bootstrapped with [`create-next-app`](https://nextjs.org/docs/app/api-reference/cli/create-next-app).

## Getting Started

First, run the development server:

```bash
npm run dev
# or
yarn dev
# or
pnpm dev
# or
bun dev
```

Open [http://localhost:3000](http://localhost:3000) with your browser to see the result.

You can start editing the page by modifying `app/page.tsx`. The page auto-updates as you edit the file.

This project uses [`next/font`](https://nextjs.org/docs/app/building-your-application/optimizing/fonts) to automatically optimize and load [Geist](https://vercel.com/font), a new font family for Vercel.

## Learn More

To learn more about Next.js, take a look at the following resources:

- [Next.js Documentation](https://nextjs.org/docs) - learn about Next.js features and API.
- [Learn Next.js](https://nextjs.org/learn) - an interactive Next.js tutorial.

You can check out [the Next.js GitHub repository](https://github.com/vercel/next.js) - your feedback and contributions are welcome!

## Deploy on Vercel

The easiest way to deploy your Next.js app is to use the [Vercel Platform](https://vercel.com/new?utm_medium=default-template&filter=next.js&utm_source=create-next-app&utm_campaign=create-next-app-readme) from the creators of Next.js.

Check out our [Next.js deployment documentation](https://nextjs.org/docs/app/building-your-application/deploying) for more details.
Allo Inventory Reservation System
A Next.js application for managing inventory reservations across multiple warehouses — built as part of the Allo Engineering take-home exercise.
Live Demo
**https://allo-inventory.vercel.app/
**
The app is seeded with sample products and warehouses — no setup needed to explore the full flow.


The Problem
When a customer proceeds to checkout, payment can take several minutes (3DS flows, UPI confirmations, wallet redirects). During that window:

Decrementing stock at cart time causes phantom depletion — ~80% of carts are abandoned, so available stock looks lower than it really is.
Decrementing stock at payment time allows two customers to pay for the same physical unit, leading to refunds, bad UX, and manual ops cleanup.

The solution: a short-lived reservation that temporarily holds stock when a customer enters checkout. If payment succeeds, the reservation is confirmed and stock is permanently decremented. If payment fails or the timer expires, the hold is released back to available inventory.

Tech Stack
LayerChoiceFrameworkNext.js 14 (App Router)LanguageTypeScript (end-to-end)DatabasePostgreSQL via Supabase + Prisma ORMLockingRedis via Upstash for distributed locksValidationZod (shared between API and forms)UITailwind CSS + shadcn/uiHostingVercel

Running Locally
1. Clone and install
bashgit clone https://github.com/allo-reservation-system/allo-reservation-system.git
cd allo-reservation-system
npm install
2. Set up environment variables
Create a .env file in the root of the project:
envDATABASE_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?sslmode=require"
DIRECT_URL="postgresql://USER:PASSWORD@HOST:PORT/DATABASE?sslmode=require"
REDIS_URL="redis://default:PASSWORD@HOST:PORT"
3. Run database migrations
bashnpx prisma migrate dev
4. Seed the database
bashnpx prisma db seed
5. Start the dev server
bashnpm run dev
Visit http://localhost:3000.

API Reference
MethodPathDescriptionGET/api/productsList all products with available stock per warehouseGET/api/warehousesList all warehousesPOST/api/reservationsReserve units — returns 409 if stock is insufficientPOST/api/reservations/:id/confirmConfirm reservation (payment succeeded) — returns 410 if expiredPOST/api/reservations/:id/releaseRelease reservation early (payment failed or user cancelled)

Concurrency Guarantee
The POST /api/reservations endpoint uses a PostgreSQL row-level lock (SELECT ... FOR UPDATE) inside a transaction to ensure exactly one request succeeds when two arrive simultaneously for the last available unit:
sqlBEGIN;
  SELECT reserved_units, total_units
  FROM stock
  WHERE product_id = $1 AND warehouse_id = $2
  FOR UPDATE;

  -- If total_units - reserved_units < requested_quantity → ROLLBACK → 409

  UPDATE stock SET reserved_units = reserved_units + $qty ...;
  INSERT INTO reservations (...) VALUES (...);
COMMIT;
Redis is used as an additional distributed lock layer to reduce contention at the DB level for high-traffic SKUs.

Data Model
Product
  id, name, description, imageUrl

Warehouse
  id, name, location

Stock
  productId (FK), warehouseId (FK)
  totalUnits, reservedUnits
  availableUnits = totalUnits - reservedUnits

Reservation
  id, productId, warehouseId
  quantity, status (pending | confirmed | released)
  expiresAt, createdAt, updatedAt

Reservation Expiry
Reservations are expired via a Vercel Cron Job that runs every minute:
json{
  "crons": [
    {
      "path": "/api/cron/expire-reservations",
      "schedule": "* * * * *"
    }
  ]
}
The handler finds all pending reservations where expiresAt < NOW(), updates their status to released, and returns the held units back to available stock. Expired stock is reclaimed within ~1 minute regardless of traffic.

Bonus: Idempotency
The POST /api/reservations and POST /api/reservations/:id/confirm endpoints support an Idempotency-Key header.
How it works: on each request the server checks Redis for a cached response under idempotency:{key}. If found, it returns the cached response immediately with no side effects repeated. If not found, it executes the operation, stores the response in Redis with a 24-hour TTL, and returns it.
POST /api/reservations
Idempotency-Key: client-generated-uuid-here
This prevents duplicate reservations when clients retry after a network timeout.

Project Structure
src/
├── app/
│   ├── page.tsx                        # Product listing page
│   ├── reservation/[id]/
│   │   └── page.tsx                    # Checkout page with live countdown
│   └── api/
│       ├── products/route.ts
│       ├── warehouses/route.ts
│       ├── reservations/
│       │   ├── route.ts                # POST /api/reservations
│       │   └── [id]/
│       │       ├── confirm/route.ts
│       │       └── release/route.ts
│       └── cron/
│           └── expire-reservations/route.ts
├── lib/
│   ├── prisma.ts
│   ├── redis.ts
│   └── lock.ts
├── schemas/
│   └── reservation.ts
prisma/
├── schema.prisma
└── seed.ts

Trade-offs & What I'd Do Differently
What I focused on: correctness of the reservation lock (the hardest and most important part), a clean data model where availableUnits = totalUnits - reservedUnits is trivially queryable, and clear user-facing error states — 409 (no stock) and 410 (expired) are both surfaced in the UI.
Trade-offs made: expiry runs on a ~1-minute cron rather than a real-time queue. For production, BullMQ with delayed jobs would give second-level precision without polling. No authentication is included — in production, reservations would be scoped to a user session to prevent abuse.
With more time I'd add end-to-end concurrency tests (spinning up simultaneous requests and asserting exactly one 200 and N-1 409s), a webhook/event system to notify downstream fulfillment on confirmation, and an admin dashboard for ops to view and manage live reservations.
