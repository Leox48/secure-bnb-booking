# Secure B&B Booking

> A real-world Bed & Breakfast booking platform, built as a hands-on exercise in secure software development — from architecture to deployment.

**Status:** 🚧 Work in progress

---

## About This Project

This is not a toy CRUD app. It's a booking and guest-management platform being built for an actual Bed & Breakfast, used deliberately as a personal project to practice the **builder side** of application security — designing, coding, and hardening a real system end-to-end, rather than only assessing one as a penetration tester.

The goal is to apply secure-by-design principles from the first commit: threat modeling the guest/host data flow, validating input at every boundary, handling identity documents and payment data with the right safeguards, and progressively layering in static analysis and a WAF as the project matures.

---

## Core Functionality

- 🛏️ Room browsing and availability calendar (2 rooms, minimum 1-night stay)
- 📅 Booking flow with per-room, per-night pricing
- 💳 Full-amount online payment via PayPal
- 🪪 Guest identity verification workflow — document upload via a unique link, host review, verified status before check-in instructions are released
- 🔑 Self check-in support, with check-in/check-out details sent automatically
- 🗓️ Host admin panel: pricing, availability, booking and verification status
- 🐾 Support for children and pets (with surcharge)

---

## Tech Stack

- **Backend/Frontend:** ASP.NET Core, Razor Pages (monolithic architecture)
- **Payments:** PayPal Checkout integration
- **Security tooling:** static analysis via CodeQL; WAF planned ahead of go-live

---

## Repository Structure

```
secure-bnb-booking/
│
├── src/       ← Application source code
├── tests/     ← Automated tests
├── docs/      ← Design notes, threat model, data flow docs
└── deploy/    ← Deployment configuration
```

---

## Security Approach

This project is used as a practical secure-SDLC exercise, with a focus on:

- **Data minimization & handling of sensitive data** — identity documents and personal data are collected only where needed for the verification workflow, with access restricted to the host role
- **Secure payment handling** — no card data touches the application directly; checkout is delegated to PayPal
- **Static analysis (SAST)** — CodeQL integrated into the development workflow to catch vulnerability classes early
- **Defense in depth for go-live** — a WAF is planned as an additional control layer before the application accepts real public traffic

No real guest, payment, or identity document data is ever committed to this repository. Any example data in `docs/` or `tests/` is synthetic.

---

## Roadmap

- [x] Core booking and pricing model
- [x] PayPal checkout integration
- [ ] Guest document upload & verification workflow
- [ ] Host admin panel (calendar, pricing, verification status)
- [ ] CodeQL SAST integrated in CI
- [ ] WAF deployment ahead of go-live
- [ ] Public launch

---

## Author

**Leonardo Sole** — Penetration Tester, exploring the application security lifecycle beyond offensive testing.

[LinkedIn](https://www.linkedin.com/in/leonardo-sole48/)
