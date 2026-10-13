# 03 — Client Meeting: Agenda, Our Positions, Decision Questions

> **Update (2026-10-13):** the client is non-technical (decision D1). Use **Part 0** below in the room. Parts 1–5 are our **internal** reference. We answer the technical and regulatory questions ourselves, or ask the financiers directly, never the client.

## Part 0 — Simple questions for the client (use these in the meeting)
1. Which products will you sell, and which financier handles each one today?
2. How does each financier work with you now: their app, their website, or paper forms? Can you give us a contact person at each?
3. Does the customer pay the Rs 15,000 to you, to the dealer, or to the financier?
4. When the loan is approved, does the financier send the money to you or to the dealer?
5. Dealer incentives: when does a sale "count" (on fitment, or when the financier pays)? If a customer stops paying early, does the dealer lose the incentive?
6. If someone refers a customer (a technician or showroom), do they get paid? How much, and when?
7. Who in your team approves new dealers, and who checks the monthly incentive before it's paid?
8. Do you already have: a website domain, a logo and brand colours, a WhatsApp Business number, a payment gateway account (Cashfree, Razorpay, etc.)?
9. Who on your side will see the daily dashboard, and what do they most want to see each morning?
10. Which date does the first hub open, and which city?

**Show, don't tell:** walk through the clickable Hindi prototype (hub sale flow + "add a financier" screen) before asking these.

---

> Purpose: leave the meeting with **decisions and named owners**, not general impressions. Every question below changes what we build or when.
> Format: we open with our understanding (5 min), then present our positions (15 min), then work through the decision questions (40 min) and agree next steps (10 min).

---

## Part 1 — Our understanding (play back to the client)

> "Power Mart sells CLN's LFP e-rickshaw battery at Rs 75,000. The customer pays Rs 15,000 down, and the rest is an NBFC co-lent loan sourced by an LSP. You sell through 20 COCO hubs and 200 dealers in UP and Bihar by March 2027. Your edge comes from OEM asset knowledge, IoT data, daily QR repayment, and the ability to refurbish and redeploy batteries. You need a digital backbone that runs this channel, ties every battery to its sale, loan and service history, and gives management and lenders a single view of growth and portfolio quality."

## Part 2 — Positions we put on the table

1. **The lender owns the loan and the money. We own the battery, the channel, and the customer relationship.** The platform never holds borrower funds (RBI DL Directions 2025: repayments go directly to the RE's account).
2. **The battery serial number is the backbone of the system.** Everything is scanned, nothing is typed, and every handover is recorded.
3. **Remote battery immobilisation is a legal and safety decision, not a feature request.** We will not build it until there is a written opinion and NBFC sign-off. The RBI device-lock rules take effect 1 Jan 2027, and stopping a moving vehicle is unsafe. Telemetry, alerts, and location are fine with consent.
4. **"Mobile data" underwriting, as the deck describes it, is not permitted** for LSPs and DLAs. We can support consented Account Aggregator and bureau data via the NBFC.
5. **Phased delivery.** The first hubs open in about 3 weeks. We propose Release 0 for launch readiness, with full workflows following. Details in doc 04.
6. **Compliance by design.** Data is hosted in India, DPDP consent comes first, Aadhaar is never stored in full, and partner-ready security evidence is available (VAPT report, policies) because the NBFC will run due diligence on us.
7. **Hindi-first, low-bandwidth, WhatsApp-native** for customers and dealers.

## Part 3 — Decision questions (grouped, with why each matters)

### 3.1 Business model and legal structure
1. Is Power Mart a separate legal entity or a division of CLN Energy Ltd? *This drives GST registration, invoicing entity, contracts, and the DPDP data fiduciary.*
2. Will Power Mart receive any fee from the LSP or NBFC? Will it give any guarantee, buyback, or first-loss cover? *This determines whether we are an LSP and whether DLG rules apply.*
3. Is the LSP a separate company? Who is it? Does it have a DLA listed with the RBI? *If the LSP owns KYC capture, our compliance surface shrinks a lot.*
4. Who is the NBFC (or NBFCs)? Is the co-lending agreement signed? When is their sandbox available?
5. On default, what is the legal chain for the battery? Hypothecation holder → repossession → transfer to OEM → refurbish → resale? Who signs off each step?
6. Collections: NBFC only (slide 14), or will Power Mart field staff also follow up (slide 7)?
7. Remote on/off: has the NBFC's compliance team or counsel opined on this? Does the IoT hardware even support it today?

### 3.2 Partners and systems
8. LSP and NBFC integration maturity: REST API, SFTP files, or manual? Which events do they send (login, sanction, KFS issued, disbursal, repayment, DPD)?
9. Does the NBFC LMS support a **daily** repayment schedule and **dynamic UPI QR** per loan? Who is the payment partner?
10. IoT/BMS: is it in-house or a vendor? What data, how often, and through what API? Is a SIM or module already in every pack? Is consent captured at sale?
11. CLN ERP: which system (SAP, Tally, other)? Is it the master for stock, invoices, and warranty? Does it have APIs?
12. Existing tools already in use (CRM, Excel trackers, WhatsApp groups)? Is there data to migrate?

### 3.3 Product scope and priorities
13. Public website: is the purpose lead generation, credibility for investors and lenders, or both? Do you have a domain, brand guidelines, and content owners?
14. For hub launch in Nov 2026, which **three things** must work? (Our suggestion: lead capture and tracking, the battery serial and sales register, and dealer onboarding.)
15. Customer channel: WhatsApp, an app, or both? Do you have a WhatsApp Business account and SMS DLT entity registration?
16. Reference partners: is there a referral payout? How much, and when is it earned?
17. Incentive slabs: are they retrospective or marginal? What event counts as a sale? Is there a clawback on early default? How is payout made (credit note or bank transfer)?
18. Dealer security deposit: what amount, how is it collected, and what are the refund and forfeiture rules?
19. Service and warranty: what is the SLA? Does the hub do diagnostics and swap, or send the battery to the OEM? Are loaner batteries part of the model?
20. Which KPIs from slide 24 need **daily** visibility, and who looks at them?

### 3.4 Data, security, compliance
21. Who is the grievance officer and data protection contact?
22. Does the NBFC have a vendor security questionnaire or VAPT requirement for its LSP and partners? Can we see it now?
23. Do you have a hosting preference or existing cloud accounts (AWS, Azure, GCP)? Who owns the accounts? (We recommend the client owns them and we operate them.)
24. Retention: how long must customer and loan-file data be kept after loan closure? (This must be confirmed with the NBFC.)

### 3.5 Commercials and engagement
25. Budget range and the commercial model (fixed-scope phases or a monthly team)?
26. Ownership of IP and code, repository hosting, and handover expectations.
27. Who is our single decision-maker and day-to-day product owner? What is the weekly demo cadence?
28. Run costs after go-live: who pays for cloud, WhatsApp, SMS, maps, and KYC APIs?

## Part 4 — Points to raise carefully (consistency checks on the deck)
- At the plan's own peak productivity, UP channel capacity is about 600 a month by Mar 2027. The plan needs 880 a month from Jan (see doc 01 §6).
- UP dealers start in Oct, but hubs start in Nov. Who supports those dealers in October?
- Bihar targets and field team are placeholders. When will they be final?
- Website copy should say "BSE SME-listed" and cite one source per market statistic.
- An EMI display that shows principal only, without interest or fees, cannot go on the public site until the NBFC's KFS is available.

## Part 5 — What we will deliver after the meeting (within 5 working days)
1. A scope and phase plan with a fixed Release 0 date.
2. The system-of-record matrix and integration plan per partner.
3. A domain model (entities and state machines) for client sign-off.
4. A clickable prototype of the hub sales flow and dealer onboarding (Hindi).
5. A security and compliance pack draft for NBFC due diligence.
