# Core application meeting notes

This document removes duplicated speech-model output, greetings, audio problems, screen-sharing instructions, negotiations, and unrelated conversation. It records only the application behavior discussed in the three videos.

## Confirmed product areas

1. Individual motor insurance.
2. SME medical insurance.
3. Finance/leasing motor insurance, including portals for the financing entity and its customers.
4. Operational services around issued policies: renewals, endorsements, additional services, claims, payments, and customer support.

## Critical data-ownership rule

- An insurance company is a display/reference record only: company name and avatar/logo. A technical ID and external provider code are still required internally for mapping.
- Insurance companies do not have a back-office portal in this platform.
- The platform requests offers through an external API.
- All offer content comes from that API: eligibility, price, tax and fee breakdown, discounts, deductible choices, coverages, additional benefits, and availability.
- If a company does not respond, is unavailable, or considers the customer ineligible, no offer from that company is shown. The platform must not manufacture or manually enter a replacement offer.
- The platform should keep an immutable snapshot of a returned offer when it is displayed/selected/purchased, but the external API remains the source of truth.

## 1. Individual motor insurance

### Request initiation

- The user selects individual motor quotation.
- The request supports a normal new-insurance journey and insurance needed before completing vehicle ownership transfer. The latter is still a new policy; it is not a transfer of an existing policy.
- Initial customer inputs include identity/Iqama number, birth date, mobile number, email, and city.
- The request summary updates as data is entered.

### External enrichment

- Customer and vehicle information is retrieved from external government/data services rather than retyped wherever possible.
- Vehicle/identity data is obtained through Elm-related integration.
- National address data is obtained from the relevant external address service.
- Najm supplies accident/claim history and the no-claims discount context.
- The platform sends the enriched risk/request payload to the external quotation API.

### Offer retrieval and display

- API responses should normally arrive within seconds.
- Not every connected insurer must return an offer.
- Record every request attempt, returned/not-returned status, response time, and technical error for operational reporting.
- A response taking more than approximately 15 seconds was discussed as an operational quality concern; it should be configurable rather than hard-coded as a business rule.
- The offer card may display the insurer name/avatar plus API-provided information such as price, VAT/fees, discounts, deductible options, coverage features, and promotional labels.
- The offer screen supports comparison of multiple offers.
- The platform may highlight the cheapest offer, but the customer chooses the offer; the cheapest offer is not automatically selected.
- For comprehensive insurance, the customer can choose among API-provided deductible amounts and see the corresponding pricing/coverage result.
- Ratings/review counts are not confirmed as insurer master data. They should be removed unless a separate reviewed source is defined.

### Selection and payment

- The user selects an offer and accepts the required declarations/terms.
- Payment-card data belongs to the payment gateway and must not be stored by the platform.
- The implementation should support replacing or adding a payment gateway later.
- IBAN verification is a separate pre-payment/refund-related step. The refund IBAN must belong to the policy/request owner; it is not the same as the payment card owner.
- Persist payment attempts, gateway references, status, failures, and retries.

### Issuance and post-purchase

- After successful payment, the selected offer is sent to the insurer/API for policy issuance.
- The insurance company—not Thiqa Tech—performs the insurer-side policy registration/integration with Najm.
- The platform acts as broker/intermediary and tracks the request until the issued policy/document is returned.
- The issued policy appears in the customer's purchases/policies area.
- Renewal reminders should be sent by mobile/SMS and email before expiry; 15 days was discussed.

## 2. SME medical insurance

### Organization and member data

- An organization account contains users with roles/permissions.
- Employee/member data should be retrieved and verified through external integrations where available; arbitrary manual creation is not the normal flow.
- Adding an employee or dependent after issuance creates an endorsement attached to the original policy.
- Employee and dependent eligibility must be verified through the relevant external source, including social-insurance/authorized data sources.

### Quotation, policy, and renewal

- Medical quotation uses organization/member information and selected medical class/benefits.
- Medical classes discussed include VIP/A/B/C (exact code list must come from the external source/configuration).
- The medical provider network—hospitals, clinics, pharmacies, and other providers—comes from the insurer/external source for the selected class. It should not be maintained as an independent platform-owned master catalog unless a separate synchronization requirement is approved.
- Policy renewal repeats member retrieval/verification, allows a new start date, and then proceeds through quotation, payment, and issuance.
- Endorsements may add/remove employees or dependents and may require additional payment or refund calculation.

## 3. Finance/leasing insurance

### Portal model

- Each financing entity has a dedicated tenant/portal and its own users.
- Roles discussed include account manager, sales employee, and Thiqa Tech support/operations users.
- Screens and actions are controlled by role permissions.
- A financing tenant and its customer portal can be enabled, suspended, or disabled independently.

### Quotation process

- A finance user creates a quotation for a lessee/vehicle.
- Required data includes lessee identity, vehicle data/class, value, contract duration, repair method by year, deductible, and relevant financing data.
- Vehicle information may be incomplete early in the financing process; manual/model-based selection can be required before the final vehicle card/data becomes available.
- Quotation offers are obtained through the external API.
- A quotation can remain valid for a stated period; 60 days was discussed for the finance quotation, but the actual expiry should be stored from the API/contract rule.
- The user can return to a previous offer or request repricing when allowed.
- The selected price is not necessarily the cheapest price.

### Approval and issuance

- The quotation may require approval/signature by the financing entity/authorized party.
- Approved vehicles/offers are sent for purchase and issuance.
- Bulk renewal supports selecting policies/vehicles approaching expiry, requesting prices, obtaining financing-entity approval, and issuing renewed policies.
- Renewal preparation windows of 45 or 60 days were discussed and should be configurable.

### Lessee/customer services

- The customer portal shows the customer's financed vehicles, active policy, collected insurance amount, and available services.
- Optional services such as geographic coverage or replacement-car coverage are priced/validated through the external insurer/API flow.
- The customer can pay for the selected additional service and receive its certificate/document.

## 4. Claims

- A customer selects an insured vehicle/policy and submits a claim.
- Captured data includes accident date, accident type, Najm/report number, and required attachments.
- Attachments discussed include the accident report, damage photos, and driving licence; the required list should be configurable by claim type/API response.
- Submission must be blocked until required documents are present.
- The platform forwards the claim to the insurance company and tracks its status on behalf of the customer.
- Customer service/operations can communicate missing-document requirements and follow up with the insurer.

## 5. Operations and administration

- The Thiqa Tech operations dashboard should show financing entities, monthly quotations, purchased policies, open exceptions, growth/operational indicators, and API health.
- Exception handling includes failed payments, API timeouts/no response, vehicle-model/code mismatches, failed issuance, and repricing requests.
- Vehicle-code mismatch handling must show the requested vehicle, the external code received from Elm, and the mapped/internal code. Operations can approve a mapping or reject it and request repricing.
- Customer service can search by identity, mobile, policy, or contract and resend a customer-portal link.
- All searches and support actions must be audited.
- The platform needs configurable roles and permissions for internal users and tenant users.
- Every external API call should have correlation ID, request/response timestamps, latency, outcome, and sanitized payload/error logging.

## 6. Security and compliance discussed

- OTP/two-factor verification is required for important identity/login flows.
- Email and mobile verification are required.
- Bot/human verification such as CAPTCHA was discussed.
- Sensitive personal and financial data must be encrypted and audited.
- Card details remain within the PCI-compliant payment gateway.
- Source-code/security review and national cybersecurity requirements were discussed as delivery/compliance activities, not application business entities.

## Items requiring confirmation

- Exact names and contracts of all external APIs.
- Exact quote validity for each product; it should not be globally hard-coded.
- Exact medical eligibility data source for employees versus dependents.
- Whether insurer ratings/reviews are real data or only prototype content.
- Exact finance approval/signature rules by tenant.
- Final list of claim document types and statuses returned by insurers.
- Refund/cancellation calculation and which party executes the refund.
- Whether locally managed promotional coupons exist; API-provided insurer promotions must not be stored as platform-owned coupons.
