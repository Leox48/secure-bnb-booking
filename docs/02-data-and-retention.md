# 02 — Data classification and retention

**Status:** draft v0.1 — output of design phase 2
**Language:** English (public portfolio repository)

Related: [01 — Requirements](01-requirements.md)

---

## 1. Principles

1. **Minimisation:** every field exists because a feature needs it.
2. **Retention is configuration:** every retention period is a named parameter, not a constant in the code.
3. **Deletion is automatic where possible** and always leaves an audit entry (without the deleted content).
4. **Documents are ephemeral:** identity-document photos live only as long as the host needs them.
5. **No personal data in logs:** logs contain identifiers, never names, emails, document data or tokens.
6. **Secrets are never stored in clear:** access tokens are stored as hashes; API secrets live outside the repository.

---

## 2. Classification levels

| Level | Meaning | Examples |
|---|---|---|
| **Restricted** | Severe harm if disclosed | Identity-document photos, registration data (date/place of birth, document number), admin credentials and MFA secrets, PayPal API secrets, access codes |
| **Confidential** | Personal or business data | Guest name, email, phone, booking details, contact messages, exemption declarations, audit logs, iCal feed URLs (they act as secrets) |
| **Internal** | Operational configuration | Rates, seasons, discounts, tourist-tax parameters |
| **Public** | Intended for visitors | Room content, photos, blog, house rules, CIN and business information |

---

## 3. Data inventory

| Data | Class | Collected in | Purpose |
|---|---|---|---|
| Guest name, email, phone | Confidential | US-05 | Contract, communication |
| Arrival time, requests, pet declaration | Confidential | US-05 | Service, supplement |
| Number of guests, guests under the exemption age, resident flag | Confidential | US-05 | Tourist-tax computation |
| Payment reference, amounts, refund records | Confidential | US-06, US-15 | Payment reconciliation, accounting |
| Registration data per person (name, sex, date and place of birth, citizenship, document type and number) | Restricted | US-17 | Guest registration by the host |
| Photo of identity document | Restricted | US-17 | Visual identity verification by the host |
| Booking access token | Restricted | US-07 | Guest access to own booking (stored as hash) |
| Access codes / check-in instructions | Restricted | US-18 | Self check-in |
| Contact form messages | Confidential | US-09 | Reply to the visitor |
| Audit log (admin actions, refunds, price changes, document access) | Confidential | US-11..US-20 | Accountability |
| Application / security logs | Internal | Runtime | Operations, incident analysis |

---

## 4. Retention parameters (defaults)

| Data | Default retention | Parameter | Removed by |
|---|---|---|---|
| Document photos | Until "Registration submitted"; safety net at arrival + 48h | `Retention:DocumentsSafetyNetHours = 48` | Host action or background job |
| Registration data per person | Same as document photos | same | Host action or background job |
| Registration receipt reference | Set at go-live | `Retention:RegistrationReceiptMonths` | Background job |
| Payment and fiscal records of a booking | Set at go-live | `Retention:FiscalRecordsMonths` | Background job |
| Tourist-tax exemption declarations | Set at go-live | `Retention:ExemptionRecordsMonths` | Background job |
| Guest contact data (name, email, phone) | Anonymised 24 months after check-out | `Retention:GuestContactMonths = 24` | Background job (anonymisation) |
| Contact form messages | 12 months | `Retention:MessagesMonths = 12` | Background job |
| Booking access tokens | Expire 30 days after check-out | `Retention:TokenGraceDays = 30` | Expiry check |
| Application logs | 90 days | `Retention:AppLogDays = 90` | Log rotation |
| Audit log | 24 months | `Retention:AuditMonths = 24` | Background job |
| Database backups | 30 days | `Backup:RetentionDays = 30` | Backup rotation |

**Anonymisation** replaces name, email and phone with irreversible placeholders and keeps only the fields needed for statistics and accounting.

---

## 5. Deletion mechanics

- A daily background service applies the retention parameters and writes an audit entry per run (counts, not content).
- The "Registration submitted" action (US-17) deletes photos and registration data immediately, in the same transaction that records the timestamp and the optional receipt reference.
- Files are stored on an encrypted volume; deletion removes the file and its metadata.
- **Backups:** the documents volume is **excluded from backups**, so deleted documents do not survive in them. After a database restore, the retention jobs run **before** the application is brought back online, so data deleted since the backup is not resurrected.
- Automated tests cover the retention jobs (a record past its retention is deleted or anonymised; a record within it is untouched).

---

## 6. Access to data

| Actor | Access |
|---|---|
| Guest (personal link) | Own booking only; can **write** documents and registration data, cannot read them back |
| Host (MFA) | All bookings; documents visible only in the verification view; every document access is logged |
| Application service | Database and file volume through least-privilege credentials |
| Third parties | See §7 |

---

## 7. Data recipients and flows

| Recipient | Data | Notes |
|---|---|---|
| PayPal | Payment data | Card data never touches the application |
| Hosting provider (VPS) | All application data | EU region; disk encryption |
| Email provider | Guest email, message content | Emails never contain access codes outside the release window, never contain document data |
| Cloudflare (DNS / CDN / DDoS) | Traffic metadata | Only if adopted at deployment |
| Guest-registration portal | Registration data | Manual submission by the host, outside the application |

- No third-party analytics or trackers; only technical cookies.
- The PayPal SDK is loaded **only** on the payment page.
- The recipients above are listed in the privacy notice.

---

## 8. Privacy documentation to produce

Produced in later phases and kept in `docs/privacy/`:

- privacy notice and cookie notice;
- record of processing activities (a simple table derived from §3);
- personal-data-breach procedure: detect, assess, contain, notify, document.

---

## 9. Consequences for the design

- US-17 and US-27 (host workflow) are built around the ephemeral-documents rule.
- Data model (phase 3): separate tables for booking, guest contact, registration data and document files, so each can be deleted independently.
- Threat model (phase 6): document upload and storage, token handling, retention jobs (failure to delete, restore resurrecting data) and audit-log integrity are explicit targets.
