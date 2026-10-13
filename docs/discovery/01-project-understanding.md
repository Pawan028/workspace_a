# 01 — Project Understanding (pre-meeting)

> Source: `CLN_Power_Mart_Deck_v2.pptx` (26 slides, dated Oct 2026), plus our own regulatory/market research (see References).
> Status: **Discovery. Nothing here is agreed scope.** Anything marked _[CONFIRM]_ must be answered by the client before we commit to it.
> Prepared: 2026-10-13

---

## 1. What the client is building (business, in one paragraph)

CLN Power Mart is the new retail and finance arm of CLN Energy Ltd (Noida, BSE SME-listed lithium battery maker). It sells CLN's own LFP e-rickshaw batteries (Rs 75,000) to e-rickshaw owners in Uttar Pradesh and Bihar. Owners pay Rs 15,000 down. The other Rs 60,000 is a 12/15/18-month loan, sourced by a Lending Service Provider (LSP) and lent by NBFC co-lending partners. Sales happen through company-owned company-operated (COCO) hubs, with 20 hubs (2 staff each) planned by Mar 2027, and through 200 dealers (10 per hub) paid on incentive slabs. The deck's claimed edge over today's dealer-led model:

- OEM knows the asset.
- Battery telemetry over IoT.
- Daily QR repayments.
- Repossessed batteries get refurbished and redeployed.
- Service happens at the hub.

## 2. Key numbers from the deck (they drive capacity and the data model)

| Item | Value | Slide |
|---|---|---|
| Battery price | Rs 75,000 | 2, 11 |
| Down payment / loan | Rs 15,000 (20%) / Rs 60,000 (80%) | 11 |
| Tenures | 12 / 15 / 18 months, principal Rs 5,000 / 4,000 / 3,333 a month. Interest and fees TBC | 11 |
| Hubs | 4 new a month (2 UP + 2 Bihar), Nov 2026 to Mar 2027, 20 total | 15 |
| Dealers | 10 per hub within 3 months, 200 total (100 per state) | 16 |
| UP target | 3,000 batteries to Mar 2027 (Rs 22.5 Cr). Jan–Mar at 880 a month | 17 |
| Bihar target | Mirrors UP, **placeholder** | 19, 20 |
| Cities | 40 per state by Dec 2026 in 3 phases. Phase 1 is **Oct 2026** | 18, 20 |
| Dealer incentive | 1–4 a month: Rs 4,000; 5–8: Rs 4,500; 9+: Rs 5,000 per battery | 16 |
| People | 57 by Mar 2027 (2 leadership, 4 HO, 40 hub, 11 UP field). Bihar field team TBD | 23 |
| KPIs | Sales/hub, sales/dealer, active dealers, hubs live, approval rate, login→disbursal days, collection efficiency, overdue share | 24 |

**What this means for engineering:** volume is small. The plan is about 6,000 batteries and about 6,000 loans by Mar 2027, under 300 internal and partner users, and 6,000 IoT devices. **Scale is not the hard problem.** The hard problems are correctness across multiple parties, the regulatory perimeter, partner integrations, field usability, and the timeline (see doc 02).

## 3. Actors and the surfaces each one needs

| Actor | Who | What they need from us | Likely surface |
|---|---|---|---|
| E-rickshaw owner (customer) | Daily earner, Rs 20–40k a month. Low-to-mid literacy. Hindi, plus regional dialects. Low-end Android or shared phone | Understand the offer, find a hub, track the loan and payments, raise service requests | Public website (Hindi-first), WhatsApp, SMS, IVR. **Not an app download** _[CONFIRM]_ |
| Reference partner | Technicians, showroom owners, individuals | Refer leads, see status, get paid a referral fee _[CONFIRM: is there a payout?]_ | WhatsApp or a lightweight web form |
| Dealer | Local shop | Onboarding (KYC, GST, security deposit), stock, sales entry, loan-file help, incentive statement | Dealer portal (mobile web) |
| Hub manager / executive | 40 staff | Leads, demo, loan-file prep, fitment, stock, service, dealer support | Ops app (laptop at hub, plus mobile) |
| FOS / ASM / Zonal head | Field sales | Dealer acquisition, visits, targets, (collections? _[CONFIRM]_) | Mobile, offline-tolerant |
| Head office (CRM, Ops, HR, Back office), President | 6 | Dashboards (slide 24), approvals, incentive payouts, reconciliation, masters | Admin console |
| LSP | Partner | Loan file, documents, status | API, SFTP, or their own app _[CONFIRM]_ |
| NBFC(s) | Lender, system of record for the loan | Sanction, disbursal, KFS, repayment schedule, collections, DPD | API or file integration _[CONFIRM]_ |
| CLN Energy (OEM) | Parent | Supply, warranty, service, take-back, refurbish | ERP integration _[CONFIRM: SAP/Tally/other?]_ |
| IoT/BMS platform | Vendor or in-house | Telemetry, location, health, immobilisation (see legal caveat) | API or webhooks |
| Investors / public | Prospective | Brand, model, credibility | Public website |

## 4. What "the web" means — our proposed scope breakdown

The client says "website". The deck actually describes a **platform**. We will lay out the layers and let the client pick the commitment for each phase.

1. **Public website**: brand, offer, hub locator, lead capture, dealer and partner sign-up, about and investor pages. Hindi and English. SEO for "e-rickshaw lithium battery + city".
2. **Power Mart Ops platform** (the core): leads and CRM, customer and loan-file orchestration, battery serial lifecycle and inventory, hub and dealer management, incentive engine, service and warranty tickets, KPI dashboards.
3. **Integration layer**: LSP, NBFC (status, disbursal, repayment and DPD feed), payments (dealer deposits only; see doc 04 §3), IoT/BMS, CLN ERP and GST invoicing, WhatsApp, SMS (DLT), maps.
4. **Later (roadmap, slides 13 and 25)**:
   - Battery-health analytics and refurbish/resale value.
   - Alternate-data underwriting support for the NBFC.
   - Multi-state, multi-lender, and multi-product (inverter batteries, vehicle finance).

**Our core architectural position:** Power Mart is the system of record for **leads, customers (shared), batteries, channel, sales, and service**. The NBFC is the system of record for **the loan and every rupee of repayment**. We never hold, pool, or pass through borrower money. Doc 02 explains why this is regulatory, not a preference.

> **Update (2026-10-13, after the client's first meeting):** each product has its own financier, and financiers can change. Power Mart acts purely as the **merchant**. Financiers (NBFCs or platforms) do all KYC and lending work. Power Mart receives only sale proceeds (down payment, disbursal as seller), never EMIs. See [05-discussion-log](05-discussion-log.md) and [architecture/02-financier-integration](../architecture/02-financier-integration.md).

## 5. Things the deck states that conflict with current regulation or need re-design

These are the points we must bring to the table. Each is backed in doc 02 and in the References.

1. **"Remote on/off" battery control for collections (slides 7, 13).** RBI's recovery-conduct amendments were issued 6 Aug 2026 and take effect **1 Jan 2027**. They bar lenders and their partners (including digital lending partners and other third parties) from using technology to restrict a borrower's device as a recovery tool. Secondary sources describe the only exception as phones, tablets, and laptops financed by the loan, with a graduated process. Coverage of EV batteries is **unclear**, and the safer reading is that immobilising a livelihood asset is not covered by the exception. On top of that, cutting power to a moving vehicle is a **physical safety hazard**. Our position: we build telemetry; the immobilisation capability, if it exists at all, stays disabled behind a legal sign-off from the NBFC and a hard safety interlock. No commitment until counsel opines.
2. **"Mobile data" as an underwriting input (slide 13).** The RBI Digital Lending Directions 2025 prohibit DLAs and LSPs from accessing phone files, media, contacts, call logs, and telephony functions. Camera, mic, and location are one-time, with consent, for KYC. Any "mobile data" signal must come through consented channels such as Account Aggregator or bureau, via the NBFC.
3. **Daily QR EMI (slides 6, 13, 14).** Repayments must flow **directly into the RE's (NBFC's) account** with no pass-through via LSP or partner accounts. The dynamic QR must be issued against the NBFC's collection account and VPA, not Power Mart's. The NBFC's loan management system (LMS) must support a **daily** repayment schedule _[CONFIRM]_.
4. **"Field team plus NBFC collections" (slide 7) vs "Power Mart sells; NBFC runs collections" (slide 14).** If Power Mart staff collect or follow up on overdues, they act as recovery agents under the same Jan 2027 framework: certification, calling hours, recording retention. This needs a clear decision.
5. **"Finance-linked income (TBC)" (slide 9).** If Power Mart earns a sourcing fee, it is effectively an LSP, with the agreement, due diligence, and data obligations that come with that. If it gives any first-loss or buyback guarantee, the Default Loss Guarantee rules apply. Either way the platform design changes.
6. **"Battery goes back to the OEM" on non-payment (slides 9, 14).** The NBFC holds the hypothecation, not the OEM. We need the legal chain (NBFC repossession → transfer to OEM → refurbish → redeploy) before we model the asset state machine.
7. **EMI display on the website (slide 11).** The deck shows principal only, "per day". A public EMI calculator that omits interest and fees and has no lender KFS risks being misleading. The site may show only what the NBFC approves.

## 6. Observations on the deck's own numbers (raise politely; they affect dashboards and targets)

- **UP ramp vs channel capacity.** Using the plan's own peak productivity (10 a month per hub, 5 a month per dealer), UP capacity in **Mar 2027** is 10 hubs × 10 + 100 dealers × 5 = **600 a month**. In Jan it is at most 6 × 10 + 100 × 5 = 560. The plan needs **880 a month from Jan**. Either the target or the channel assumption has to move, and the KPI dashboard will show the gap from week one.
- **Dealers before hubs.** UP dealer onboarding starts **Oct 2026** (10 dealers) but hubs start **Nov 2026**. The system must allow dealers attached to head office or a zone, not only to a hub.
- **Bihar** figures and field team are placeholders, so the system must be config-driven per state, not hard-coded.
- **"BSE-listed".** CLN Energy is listed on **BSE SME** (Jan 2025). Public-site copy should be precise.
- **Market data.** Down To Earth's VAHAN analysis puts UP alone at more than 2 lakh e-rickshaws in FY25-26 and Bihar at more than 60,000, against the deck's about 2.1 lakh for UP and Bihar combined. WRI puts the lead-acid share at about 90%, against the deck's 71%. These come from different sources and years. Any figure on the public site needs one cited source.
- **Incentive slabs.** Is the slab **retrospective** (all batteries at the top slab reached) or **marginal**? On what event is a sale counted: fitment, disbursal, or after N EMIs? Is there a clawback on early default? This defines the incentive engine.

## 7. What we are confident about vs what is unknown

**Known enough to start designing:**
- Personas.
- Battery-centric domain model.
- Channel structure.
- The KPI set.
- The regulatory perimeter.
- Hosting in India.
- Hindi-first public UX.

**Unknown and blocking** (see the meeting agenda, doc 03):
- Who the LSP and NBFC are, and how mature their APIs are.
- Whether Power Mart touches KYC data at all.
- The IoT vendor and its data access.
- CLN's ERP.
- The legal entity (a CLN division or a separate company?).
- Budget, timeline commitment, and ownership of code and cloud accounts.

## References
- RBI Digital Lending Directions 2025 (summaries): [IndiaLaw](https://www.indialaw.in/blog/civil/rbi-notifies-digital-lending-directions/), [Mondaq](https://www.mondaq.com/india/financial-services/1632398/digital-lending-in-india-a-regulatory-reset-under-the-reserve-bank-of-india-digital-lending-directions-2025), [SS Rana](https://ssrana.in/articles/rbi-introduced-digital-lending-direction-2025/), [Legal500](https://www.legal500.com/developments/thinked-leadership/the-rbis-digital-lending-directions-2025-a-unified-code-for-a-fragmented-sector/)
- RBI Co-Lending Arrangements Directions 2025 (effective 1 Jan 2026): [RBI](https://www.rbi.org.in/Scripts/BS_ViewMasDirections.aspx?id=12952), [AZB](https://www.azbpartners.com/bank/rbi-issues-comprehensive-directions-on-co-lending-arrangements/), [SCC Online](https://www.scconline.com/blog/post/2025/08/08/rbi-co-lending-directions-2025-joint-lending-framework-banks-nbfcs/)
- RBI recovery and device-locking amendments (6 Aug 2026, effective 1 Jan 2027): [Business Standard](https://www.business-standard.com/finance/news/rbi-extends-recovery-norms-deadline-eases-certification-timeline-126080602037_1.html), [Inc42](https://inc42.com/buzz/rbi-bars-lenders-from-locking-borrowers-devices-except-for-gadget-loans/), [Vinod Kothari on the May 2026 draft](https://vinodkothari.com/2026/05/remote-device-locking-rbi-proposes-highly-guarded-path/), [Corporate Professionals](https://www.corporateprofessionals.com/articles/when-the-recovery-call-comes-rbi-rewrites-the-rules-of-engagement-for-nbfcs/)
- DPDP Rules 2025 (notified Nov 2025, core obligations about May 2027): [Deccan Herald](https://www.deccanherald.com/amp/story/india%2Fcentre-notifies-dpdp-rules-implementation-planned-in-phases-spread-over-12-18-months-3798465), [TCSA](https://www.tcsa.in/resources/dpdp-rules-2025-implementation-roadmap)
- Battery Pack Aadhaar (BPAN), MoRTH draft (Dec 2025 / Jan 2026, not yet final): [PV Magazine](https://www.pv-magazine-india.com/2026/01/05/india-introduces-battery-aadhaar-system-to-track-ev-batteries-across-lifecycle), [Energetica](https://www.energetica-india.net/news/govt-issues-draft-battery-pack-aadhaar-guidelines-for-ev-traceability-and-sustainability)
- CLN Energy listing: [5paisa](https://www.5paisa.com/news/cln-energy-ipo-listing-today), [Samco](https://www.samco.in/knowledge-center/articles/cln-energy-ipo-listing-a-tepid-market-debut/)
- Market: [Down To Earth (VAHAN FY25-26)](https://www.downtoearth.org.in/energy/india-on-e-wheels-e-rickshaws-powering-electric-mobility-growth-in-several-states-finds-dte-analysis), [WRI India](https://wri-india.org/research/powering-indian-e-rickshaws-toward-alternate-battery-technologies), [JMK Research](https://jmkresearch.com/mandatory-registration-of-all-electric-rickshaws-across-states-in-india/)

> All regulatory points above come from secondary legal commentary. They must be confirmed against the RBI, MeitY, and MoRTH primary texts, and by the NBFC's compliance team, before any design is final.
