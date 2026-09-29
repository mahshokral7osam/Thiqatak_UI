# Meeting Requirements — Cleaned & Enhanced
# اجتماع المتطلبات مع صاحب المشروع (عين الثقة / ثقة تك)

> Source: 3-video meeting with the app owner. This document extracts only the **confirmed business requirements** and removes greetings, audio issues, demo navigation, negotiations, duplicates, and filler speech.

---

## 1. Products & Scope

The platform has **three insurance products** and an **operations layer**:

1. **Individual Motor Insurance** (تأمين سيارات الأفراد)
2. **SME Medical Insurance** (التأمين الطبي للمنشآت)
3. **Financed/Leased Vehicle Insurance** (تأمين المركبات المؤجرة) — with dedicated portals for the funder and the lessee

---

## 2. Insurance Companies (شركات التأمين) — Critical Rule

| Decision | Detail |
|---|---|
| **Insurer is reference data only** | Store only the company name and logo/avatar. No back-office portal for insurers on this platform. |
| **All offers come from API** | The platform calls insurer APIs. Offers include price, tax, fees, discounts, deductibles, coverages, add-ons, and eligibility. |
| **No manual offers** | If an insurer's API doesn't respond or declines, that insurer simply doesn't appear. The platform must never fabricate or manually enter an offer. |
| **Immutable snapshots** | Once an offer is returned/selected/purchased, store an immutable copy. The insurer API remains the source of truth. |
| **Ratings/reviews removed** | The owner explicitly said to remove insurer ratings/reviews. They are prototype content, not real data. |

---

## 3. Individual Motor Insurance (تأمين سيارات الأفراد)

### 3.1 Request Initiation
- **Two flows**: New insurance (تأمين جديد) and ownership transfer (نقل ملكية)
- **Customer inputs**: National ID / Iqama number, birth date (Hijri/Gregorian), mobile number, email, city
- **Request summary** updates as data is entered (left panel in UI)
- **Vehicle lookup**: By serial number (رقم تسلسلي) or customs card (بطاقة جمركية)

### 3.2 Ownership Transfer (نقل ملكية)
- This is **NOT** a transfer of an existing policy — it is a **new policy** for a vehicle whose ownership has not yet been transferred
- The buyer needs insurance before the ownership transfer can complete in Absher
- **Required data**: Seller's national ID, vehicle serial number, buyer's national ID, buyer's Hijri birth date, buyer's phone
- The platform must verify seller, buyer, and serial number match the Absher transfer request **before pricing**
- After issuance, track the transfer status. If cancelled/expired → open an exception → resolve with `CancelPolicy`

### 3.3 External Enrichment
- **Elm** (علم): Vehicle data retrieval, customer identity verification, vehicle model codes
- **SPL** (سُبل): National address lookup
- **Najm** (نجم): No-claims discount history (NCD table), accident history. Najm also handles policy registration and links insurance companies with traffic authorities
- The NCD comes from Najm's official table — discount percentage depends on years without claims and coverage type (comprehensive vs TPL)

### 3.4 Offers & Comparison
- Offers arrive within **seconds** via API
- Not all connected insurers must return an offer (some may decline, be offline, or blacklist the customer)
- **Record every API call**: response time in milliseconds, status (quoted/declined/timeout), technical errors — for operational reporting
- Owner specifically said: **response time > 15 seconds = poor quality** (but make this configurable, not hard-coded)
- **Coverage types**: Comprehensive (شامل) and Third-party (ضد الغير)
- For comprehensive: customer chooses deductible amount (500, 1000, 2000, 3000) and sees corresponding pricing
- **Add-ons**: Replacement car, GCC cover, natural disaster coverage — each with external code, name, price, selected flag
- **Comparison**: Customer can add up to 3 offers for comparison
- **Cheapest highlight**: Platform may highlight cheapest, but **customer chooses** — cheapest is NOT auto-selected

### 3.5 Payment
- **Payment gateway**: HyperPay currently; must support switching/adding gateways later
- **Card data stays with the payment gateway** — platform does NOT store card numbers
- **Payment card owner ≠ policy owner** — anyone can pay for someone else's insurance
- **IBAN verification** is a separate step — the IBAN must belong to the policy/request owner (not the payer) because refunds go to the IBAN
- IBAN verification through Elm
- **Persist**: Payment attempts, gateway references, status, failures, retries
- **OTP required** before completing payment

### 3.6 Policy Issuance
- After payment, the selected offer is sent to the insurer API for issuance
- **The insurer** (not ThiqaTech) registers the policy with Najm
- Platform tracks until the issued policy/document is returned from the insurer
- **Policy documents** (PDF, terms, certificate) come from the insurer API
- **Najm registration status** is tracked (pending → uploaded). If pending > 24 hours → Najm delay exception
- Issued policy appears in the customer's account/purchases area

### 3.7 Renewal
- **Renewal reminders**: SMS and email, sent 15 days before expiry (owner discussed 15, 30, 45 days — should be configurable)
- Renewal contains a **direct link** to the platform with the customer's info pre-filled
- Renewal flow is essentially the same as new insurance but with pre-filled data

---

## 4. SME Medical Insurance (التأمين الطبي)

### 4.1 Organization Setup
- **Organization identified by**: Commercial registration number (السجل التجاري) or unified number
- **Data verified through**: GOSI (التأمينات الاجتماعية) for organization and employee data
- **Authorized person** (المفوض): Enters their national ID → OTP sent to the organization owner → verified
- **VAT number** (الرقم الضريبي): Optional, format starts with 3 and ends with 3
- **National address** from SPL

### 4.2 Members (الأعضاء)
- **Employee data pulled from GOSI** (التأمينات الاجتماعية) — employees are NOT manually entered
- **Dependants pulled from Masdr** (مصدر) — spouse, children
- Each dependant is linked to their sponsoring employee
- **Medical classes**: VIP, A, B, C — dependants inherit the employee's class
- Members can be added/removed after issuance → creates an endorsement (ملحق)
- Adding a new employee after issuance: enter national ID → system verifies through GOSI → add as endorsement → pay prorated amount

### 4.3 Medical Disclosure (الإفصاح الطبي)
- After selecting an offer, a **disclosure form** appears (from the insurer)
- Questions like: who has pregnancy, chronic conditions, etc.
- **Answers are frozen once submitted** — corrections require a new disclosure
- Disclosure result affects pricing — premium loading percentage may increase (e.g., 6% loading)
- Underwriting statuses: Pending, Approved, ApprovedWithLoading, Declined

### 4.4 Medical Network (الشبكة الطبية)
- Network of hospitals, clinics, pharmacies comes **from the insurer** per class
- The platform does NOT maintain an independent provider network catalog
- Browsable by class (VIP/A/B/C) and city

### 4.5 Policy Issuance & CHI
- After payment: policy issued, then **CHI (مجلس الضمان الصحي)** registration
- Upload member names to CHI with coverage dates
- Send digital membership cards and numbers to members via SMS
- **Organization dashboard**: active employees count, active dependants count, days remaining, endorsement log

### 4.6 Tax Invoice
- **Standard invoice** (فاتورة) issued to the organization (not simplified like individual)
- Must comply with ZATCA requirements

---

## 5. Financed/Leased Vehicle Insurance (تأمين المركبات المؤجرة)

### 5.1 Funder Portal Model
- Each financing entity (bank, leasing company) = **one tenant** with its own portal
- Portal accessed via **dedicated subdomain** (e.g., `yusr.thiqatak.sa`)
- **Roles**: Account manager (مدير الحساب), Sales employee (موظف مبيعات), Operations employee (موظف عمليات), ThiqaTech support user
- Each tenant has **independent admin panel** with its own settings
- Funder portal and lessee portal can be **independently enabled/suspended/disabled**

### 5.2 Quotation
- Sales user creates a quote for a lessee/vehicle
- **Vehicle selection by model name** (not serial number yet) — because the actual vehicle may not exist yet (bank is still processing the loan)
- Vehicle catalog uses **Elm codes** — the code maps to NIC (national information center) codes
- **Vehicle code mismatch** is a real operational issue — the Elm code used in quoting may not match the NIC code when the actual vehicle arrives
- **Required data**: Lessee identity, vehicle make/model (from catalog), value, contract duration (1-5 years), repair type per year, deductible, NCD years
- **Repair type configuration**: Can be **locked by the funder** (e.g., "agency repair for all clients for 3 years, then workshop") — sales cannot change it. This is controlled per-funder in the admin panel
- **Quote validity**: 60 days (configurable per funder)
- **Quote letter**: Printable, shows per-year pricing, sum insured declining annually (~15%), repair type per year, insurer name, selected price

### 5.3 Price Selection Rule
- **Only the lowest premium before NCD is selectable** — all other offers are stored but cannot be purchased
- This is a funder business rule, not a customer choice

### 5.4 Purchase Flow
- Purchase requires: contract number from the funder, vehicle serial/customs card, vehicle code matching with Elm
- If code doesn't match → **Vehicle Model Mismatch exception** in operations
- After purchase: register with Najm, sync with funder's B2B system
- **Collected vs Paid tracking**: The funder collects a fixed annual insurance amount from the lessee (as part of the loan). The actual insurance cost may differ. Track: `CollectedFromCustomer` and `PaidToInsurer` — the funder (not the platform) manages this

### 5.5 Lessee Portal (منصة المستأجر)
- Shows: financed vehicles, active policy details, insurance amount collected, available services
- **Additional services**: Geographic coverage (تغطية جغرافية), replacement car (سيارة بديلة), extra driver
- Services priced via the insurer API — either fixed price or API-priced per customer
- Lessee pays through the platform's payment gateway
- Lessee can print certificates

### 5.6 Renewal (التجديد)
- **Batch-based renewal** — funders may have thousands of vehicles
- **Workflow**: Upload vehicle list → Funder approves vehicles → Request pricing → Funder approves prices → Purchase → Register with Najm → Send SMS notifications to lessees
- **Bulk operations**: Export/import via Excel for large fleets
- **Re-pricing**: Funder can request re-pricing if a cheaper insurer quote is available
- **Renewal window**: Configurable (45 or 60 days before expiry, default 60 days)
- Pricing validity: 60 days (should not change within this period)

### 5.7 Funder Admin Settings
- **Repair type lock**: Agency/workshop per year, lockable by admin so sales can't change
- **Insurer list**: Enable/disable specific insurers per funder tenant
- **Lessee portal**: Can be enabled/disabled independently
- **User management**: Add users with email-based OTP login
- **Per-funder subdomain and portal link**

---

## 6. Claims (المطالبات)

- Customer selects a vehicle/policy and submits a claim
- **Required data**: Accident date, accident type (collision, theft, etc.), Najm report number
- **Required attachments**: Accident report, damage photos, driving license, vehicle registration
- **Submission blocked** until all required documents are present — cannot save/send incomplete
- Platform forwards the claim to the insurer and tracks status
- Customer service follows up on missing documents and insurer progress
- Customer account shows open claim count

---

## 7. Operations & Admin Dashboard (لوحة العمليات)

### 7.1 Dashboard
- Number of funder tenants, monthly quotations, purchased policies, open exceptions, growth indicators, API health/latency
- All operations must be reflected on the dashboard

### 7.2 Exception Handling (الاستثناءات)
| Exception Type | Description | Resolution |
|---|---|---|
| **PaymentFailed** | Payment didn't complete at gateway | Retry payment on behalf of customer |
| **VehicleModelMismatch** | Elm code doesn't match NIC code on purchase | Show requested vehicle + Elm code + NIC code → **Approve mapping permanently** or **Reject & re-quote** |
| **NajmDelay** | Najm registration pending > 24 hours | Retry upload to insurer |
| **TransferNotCompleted** | Absher ownership transfer cancelled/expired after policy issued | Cancel policy |

### 7.3 Vehicle Catalog Management (قاموس المركبات)
- Operations manages the mapping between Elm codes and NIC codes
- When a mismatch occurs: operations can approve a **permanent code link** (so it auto-maps in the future) or reject and request re-quoting
- New vehicle models may require manual code mapping when first encountered

### 7.4 Customer Service (خدمة العملاء)
- Search by: national ID, mobile number, policy number, or contract number
- **Re-send lessee portal link** to the customer's registered mobile
- **All search and support actions must be audited**
- Customer service access requires OTP verification for security (to prevent employee misuse)

### 7.5 Bulk Renewals (from operations side)
- Operations can handle renewals on behalf of funders who don't have staff
- Same batch workflow but executed by ThiqaTech operations users

---

## 8. Per-Funder Tenant Control

| Control | Detail |
|---|---|
| **Enable/disable funder portal** | Can suspend a funder completely or re-enable after contract renewal |
| **Enable/disable lessee portal** | Independent from funder portal |
| **Enable/disable specific insurers** | Per-funder insurer list |
| **Lock repair type** | Agency/workshop locked per funder — sales can't change |
| **Disable specific features** | Toggle features for specific tenants (e.g., disable certificate printing for lessee) |
| **B2B integration mode** | Some funders use the portal UI, some use B2B API, some use hybrid |

---

## 9. Security & Compliance

- **OTP/2FA** required for login and important actions
- **Email and mobile verification** required
- **CAPTCHA/bot detection** discussed
- **Sensitive data encryption**: National ID, IBAN, phone, email, medical disclosures
- **Card data**: Stays within PCI-compliant payment gateway (HyperPay)
- **National Cybersecurity certification** required (from a certified auditor) — this is a delivery/compliance activity, not an application entity
- **Source code security review**: Required before launch

---

## 10. Technical Decisions

| Decision | Detail |
|---|---|
| **Language/Framework** | .NET (ABP.IO) |
| **Web-first** | Web platform first, mobile app later — but must be responsive (Blazor was discussed for cross-platform) |
| **Modular teams** | Can split work: motor team, medical team, leasing team |
| **External integrations are the hard part** | Owner emphasized: the technical coding is straightforward; the real challenge is getting API contracts signed with Najm, SPL, Elm, GOSI, etc. |
| **Phase 1 approach** | Build complete data model and workflows with mock data first, then integrate external APIs one by one |

---

## 11. Items Still Requiring Confirmation

1. Exact names and contracts of all external API providers
2. Exact quote validity period per product (currently discussed as 10 hours motor, 60 days funder)
3. Exact medical eligibility data source — GOSI for employees, Masdr for dependants (to be confirmed)
4. Whether insurer ratings/reviews exist — **owner says remove them**
5. Exact funder approval/signature rules per tenant
6. Final list of claim document types and statuses from insurers
7. Refund/cancellation flow — which party executes the refund
8. Whether platform-owned promotional coupons exist — **API-provided insurer promotions must NOT be stored as platform-owned coupons**
9. Net insurance report formula: `CollectedFromCustomer - PaidToInsurer` — confirmed as funder-only data
10. Renewal reminder timing: 15 days discussed, should be configurable
