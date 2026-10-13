# ADR-0003: Payment gateway with hosted checkout only

- Status: **Accepted (pattern)**. **Vendor: Proposed.** The client picks between Cashfree and Razorpay.
- Date: 2026-10-13
- Related: discussion log D13–D15, [engineering/06](../engineering/06-adversarial-security.md)

## Context
Power Mart collects its own money: down payments (when the scheme says Power Mart collects them) and dealer security deposits. Later it pays out incentives and referral fees. It never collects loan repayments; those go to the financier only.

## Decision
- Use an RBI-authorised payment aggregator's **hosted checkout and payment links only**. Card and UPI data never touch our servers or pages. This keeps our PCI DSS scope to the lightest self-assessment level.
- **Shortlist:** **Cashfree** (strong payouts and split settlement, useful for dealer incentive payouts) and **Razorpay** (fastest integration, excellent docs).
- Before contracting, verify the vendor is on the RBI's list of authorised payment aggregators. Compare MDR, settlement (T+1), and refund and payout APIs. The client's banking relationship may decide.
- Integration rules:
  - Amounts are created server-side from the DO or config.
  - The redirect result is never trusted.
  - Success requires a signed webhook **and** a server-side re-fetch from the PG API.
  - Idempotent processing.
  - A daily settlement reconciliation.
  - Refunds are maker-checker.
- Wrapped behind a `PaymentGateway` port, so switching vendors is an adapter change.

## Consequences
- Minimal compliance burden and no card data risk.
- A dependency on vendor uptime. Payment links can be re-sent, and down payments can fall back to bank transfer with manual verification (maker-checker).
