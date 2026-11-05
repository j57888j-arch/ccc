# WebRaqam automatic delivery design

## Scope

This module allocates an individually tracked, lawfully provided telecom number after Telegram confirms a Stars payment. It delivers the phone number only. It has no SMS inbox, OTP forwarding, Telegram login-code handling, password handling, or 2FA-secret handling.

## Inventory model

Each SIM/number is one `inventory_items` row:

- `service_id`: catalogue service this number belongs to.
- `resource_ciphertext`: Fernet-encrypted phone number.
- `resource_fingerprint`: keyed fingerprint used to prevent duplicate entries.
- `resource_mask`: safe masked value for admin listings.
- `status`: `available`, `reserved`, or `sold`.
- `order_id`: the order holding a reservation.

The aggregate `inventory` row is a count projection. For an individually tracked service, create the service with initial stock `0`, then add each number separately using `/inventory_add SERVICE_ID +998...`.

## Checkout and delivery sequence

1. The customer selects an active service with a positive integer `stars_price`.
2. `create_stars_order` locks the inventory row and reserves one available individual item using PostgreSQL row locking (`FOR UPDATE`, `SKIP LOCKED`).
3. The bot creates a Telegram invoice in `XTR`. The order and payment payload are persisted before sending the invoice.
4. `pre_checkout_query` validates payer, payload, currency, amount, order state, and expiry. It does not deliver the number.
5. Only `successful_payment` is authoritative. The handler verifies the Telegram charge, payload, payer, amount, and `XTR` currency.
6. In the same database transaction, the delivery service changes the reserved item to `sold`, updates inventory counts, marks the order `completed`, records audit/order events, and writes an encrypted message to the transactional outbox.
7. The outbox worker decrypts the prepared notification in memory and sends the number to the buyer. Duplicate payment updates do not create duplicate delivery events.

## Safe fallback

If automatic delivery is disabled, no individual resource is attached, the encryption key is missing, decryption fails, or inventory counts do not reconcile, the payment remains recorded as paid and the order stays in the admin queue. Admins receive a diagnostic message. The system must not silently mark an unpaid order as paid or create a number that is not in inventory.

## Required configuration

- `TELEGRAM_STARS_ENABLED=true`
- `AUTO_DELIVERY_ENABLED=true`
- `PHONE_ENCRYPTION_KEY=<stable Fernet key>`
- `BOT_TOKEN`, `ADMIN_IDS`, PostgreSQL `DATABASE_URL`, and production `REDIS_URL`

Use the same encryption key for the Web Service and Worker. Back it up securely; losing it makes existing inventory values and queued encrypted delivery messages unreadable. Never put it in Git or logs.

## Operational checklist

- Only list numbers you own or are legally authorized to provide, and whose carrier terms allow the intended telecom service.
- Keep each physical SIM active and ensure the actual service/usage rights promised to the buyer can be provided.
- Do not advertise a number as usable if only its digits are being handed over without the corresponding service rights or access.
- Configure `/readyz` in UptimeRobot.
- Back up PostgreSQL before running a new Alembic migration.
- Test a low-value Stars purchase with a test account before opening sales.
- Inspect failed outbox events and paid orders requiring admin review.

## Payment boundary

Virtual numbers and other digital services sold inside a Telegram bot use Telegram Stars (`XTR`) as checkout. A bank-transfer receipt upload must not be offered as an alternative in-bot checkout for these digital services. This project therefore has no card receipt approval flow, CLICK checkout, or Payme checkout enabled.
