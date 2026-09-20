# 01 — Requirements

**Status:** draft v0.1 — output of design phase 1
**Language:** English (public portfolio repository)

Legend: **(A)** = assumption to be confirmed · **(TBD)** = decision deferred to a later phase.

---

## 1. Purpose and scope

A web application for a small bed & breakfast (2 rooms) that handles:

- presentation of the rooms;
- direct bookings with full online payment (PayPal);
- collection of guest documents and identity verification before arrival;
- release of self check-in instructions and access codes;
- contact between guests and host;
- a city blog (post-MVP).

The project is also a secure-by-design reference: design, threat modelling, SAST, DAST / self-pentest and WAF hardening are documented in this repository, step by step.

**Architecture and stack (decided):** monolith, ASP.NET Core with Razor Pages, PostgreSQL, PayPal Orders v2 (server-side), reverse proxy with WAF on a VPS.

---

## 2. Actors

| Actor | Description |
|---|---|
| Visitor | Anonymous user browsing rooms and availability |
| Guest | Visitor who made a booking; no account, no password: access to the booking is via a personal link with a random token |
| Host / Admin | Owner of the B&B, logs in with MFA. One person, possibly one collaborator |
| External systems | PayPal, email provider, Booking / Airbnb calendars (iCal) |

---

## 3. Business rules

### Fixed

| Rule | Value |
|---|---|
| Rooms | 2: one with 5 beds, one with 4 beds |
| Pricing unit | Per room, per night (not per person) |
| Minimum stay | 1 night (e.g. 20th → 21st) |
| Check-in | Self check-in, available 24h, subject to identity verification (see §7) |
| Check-out | By 11:00 |
| Children | Allowed |
| Pets | Allowed; a supplement may apply |
| Payment | Full amount paid online at booking time |
| Booking model | Guest checkout, no registration |

### Configurable from the admin panel (values are placeholders for now)

- nightly rate per room and period (seasons, weekends, special periods);
- discounts;
- pet supplement (applied automatically when a pet is declared; may be 0);
- tourist tax (per person per night, with configurable exemptions).

Each night of a stay is priced by its own date: a stay spanning two periods uses both rates. The price applied to a booking is frozen at booking time.

### Assumptions

- **(A)** One booking can include both rooms, with a single payment.
- **(A)** Currency: EUR.
- **(A)** Site languages: Italian and English.
- **(A)** Dates are put on hold while the guest pays; proposed hold: 15 minutes.

---

## 4. Booking lifecycle

```mermaid
stateDiagram-v2
    [*] --> PendingPayment
    PendingPayment --> Expired
    PendingPayment --> Confirmed
    Confirmed --> DocumentsRequested
    DocumentsRequested --> DocumentsReceived
    DocumentsReceived --> IdentityVerified
    IdentityVerified --> AccessReleased
    AccessReleased --> Completed
    Confirmed --> Cancelled
    DocumentsRequested --> Cancelled
    DocumentsReceived --> Cancelled
    IdentityVerified --> Cancelled
    Cancelled --> Refunded
    Completed --> [*]
    Refunded --> [*]
    Expired --> [*]
```

- `PendingPayment`: dates on hold; the booking is confirmed only after the payment has been captured **and verified server-side** (status, amount, currency, order bound to this booking, not already used).
- `IdentityVerified` is set **only by the host**, after receiving the guests' documents.
- `AccessReleased`: check-in instructions and codes become visible only if the booking is `IdentityVerified` **and** the current time is inside the release window **(A: from 24h before arrival)**.
- Cancellation and refund rules per state are defined in design phase 4.

---

## 5. Functional requirements (user stories)

### 5.1 MVP

**Room showcase**

- **US-01** As a visitor, I want to see the list of rooms with photos, amenities, capacity and an indicative price, so that I can choose.
- **US-02** As a visitor, I want a room detail page with an availability calendar.
- **US-03 (A)** As a visitor, I want the site in Italian and English.

**Booking and payment**

- **US-04** As a guest, I want to choose dates and number of people and see availability and the total price before proceeding.
- **US-05** As a guest, I want to enter my details (name, email, phone, expected arrival time, requests), the number of adults and children, whether I travel with pets, and accept privacy and cancellation terms. The system validates room capacity (5 and 4).
- **US-06** As a guest, I want to pay the full amount with PayPal (card payment without a PayPal account where available).
- **US-07** As a guest, I want a confirmation email with: booking summary; house rules; check-in and check-out information; directions to the property from Naples airport and from Napoli Centrale station; the link to manage my booking. The email contains **no access codes**.
- **US-08** As a guest, I want to view my booking from my personal link and cancel it according to the refund policy.

**Contact**

- **US-09** As a visitor, I want to write to the host through a contact form (protected against spam and abuse).
- **US-10** As a host, I want to receive the message by email and read it in the admin panel.

**Administration**

- **US-11** As a host, I want to log in to the admin panel with MFA.
- **US-12** As a host, I want to see and manage bookings (list, detail, states) and a **centralised calendar** across platforms showing: free, direct booking, occupied via Booking, occupied via Airbnb, manually blocked, pending payment. **(A)** Synchronisation via iCal import/export with Booking and Airbnb. iCal is not real-time: the residual double-booking window is reduced with frequent polling and short holds.
- **US-13** As a host, I want to manage rooms, descriptions and photos.
- **US-14** As a host, I want to block dates manually (maintenance, bookings received elsewhere).
- **US-15** As a host, I want to refund a booking fully or partially, with an audit log of who did what.
- **US-16** As a host, I want to set room prices per period, discounts, the pet supplement and the tourist tax from a dedicated admin page. Every change is written to the audit log.

**Documents, identity verification and access**

- **US-17** As a guest, I want to upload the documents of all people in my group before arrival, using the link received by email. As a host, I want to receive them and mark the verification as confirmed.
  - Upload is **write-only** for the guest (no way to view uploaded files).
  - Files are validated, stored encrypted outside the web root, visible only to the host (MFA, access log) and deleted automatically after a retention period **(TBD, phase 2)**.
- **US-18 (A)** As a guest, I want to receive check-in instructions and access codes only after payment and identity verification, and only inside the release window before arrival. Codes are temporary or rotated at every stay.

**Link recovery**

- **US-19** As a guest who lost my link, I want to receive a new one by entering only my email address. The system always answers with the same message, whether or not a booking exists, and is rate-limited.
- **US-20** As a host, I want to correct the email of a booking, resend links or revoke all links of a booking, with audit log.

### 5.2 Post-MVP

- **US-21** City blog: articles written in Markdown by the host and published from the admin panel.
- **US-22** Discount codes and special rates.
- **US-23** "How to get here" and local guide pages.
- **US-24** Export of revenue reports for accounting.
- **US-25** Newsletter, only with explicit GDPR consent.

Each story will receive acceptance criteria, and its abuse cases will be captured in the threat model (phase 6).

---

## 6. Automated emails

**To the guest**

1. Booking confirmation (US-07): summary, rules, directions, management link.
2. Documents request (US-17): sent after payment, with a personal link to the upload page.
3. Check-in instructions and access codes (US-18): sent only when the booking is verified and inside the release window.
4. **(A)** Reminder if documents are missing N days before arrival (timing TBD).
5. Cancellation / refund confirmation.

**To the host**

- new booking;
- new contact message;
- documents uploaded by a guest;
- reminder for bookings awaiting verification **(A)**.

Sender authentication (SPF, DKIM, DMARC) is a deployment requirement.

---

## 7. Legal and compliance notes (to be verified)

These points influence the design and must be verified with an accountant, a trade association or the competent authorities before go-live. They are not legal advice.

- **GDPR:** data minimisation, privacy notice, cookie policy, retention periods, special care for identity documents.
- **Guest registration:** communication of guests' data to the police through the "Alloggiati Web" portal within 24 hours of arrival.
- **Identity verification and self check-in:** industry sources report that, following a November 2025 ruling of the Consiglio di Stato, self check-in is admitted only with a real-time visual identity check (in person or by video call), and that access with a key box and code alone is not. **To be confirmed.** The design already supports it: the host sets the "identity verified" state, so the verification method can change without changing the flow.
- **Tourist tax:** rates and exemptions to be verified with the municipality; values are editable from the admin panel.
- **Invoices / receipts:** (TBD).

---

## 8. Non-functional requirements (high level)

- Mobile-first, responsive UI; accessibility target WCAG 2.1 AA **(A)**; basic SEO.
- Very simple user experience: no registration, no passwords for guests, a progress bar showing the booking state (*paid → documents → verification → instructions*) with a clear indication of what is missing, photo upload from smartphone with progress and explicit confirmation.
- Minimum personal data collected.
- Single VPS: short downtimes are acceptable; backups stored off the server with **tested** restore.
- Security requirements will be derived from OWASP ASVS (level 1 as baseline, level 2 for payments, admin and documents) in design phase 7.

---

## 9. Out of scope

Guest accounts with passwords, mobile app, live chat, full channel manager, payment methods other than PayPal, multiple properties, housekeeping management.

---

## 10. Open decisions

| # | Decision | Phase |
|---|---|---|
| 1 | Pet supplement: per night or per stay | 4 |
| 2 | Retention periods for guest data and documents | 2 |
| 3 | Cancellation and refund windows; case "paid but identity not verified" | 4 |
| 4 | Date-hold duration (proposed: 15 min) | 3 |
| 5 | Access-release window (proposed: 24h before arrival) | 3 |
| 6 | Reminder timing for missing documents | 3 |
| 7 | Identity verification method (video call?) — pending legal check | 2 |
| 8 | OTA platforms in scope (assumed: Booking, Airbnb) | 3 |
| 9 | One booking with both rooms | 3 |
| 10 | Invoicing / receipts | 2 |
| 11 | Tourist tax rates and exemptions (editable) | 2 |

---

## 11. Traceability

Requirements (this document) → threat model and abuse cases (phase 6) → security requirements from ASVS (phase 7) → tests and pipeline gates (phase 10).
