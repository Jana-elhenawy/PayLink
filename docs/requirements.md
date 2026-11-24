# PayLink — Software Requirements Specification (SRS)

| | |
|---|---|
| **Document title** | PayLink — Software Requirements Specification |
| **Version** | 1.1 (revision of 1.0 after review; see Document history) |
| **Status** | Complete requirements baseline for the whole PayLink project (Phase 1 and Phase 2). Submitted with Phase 1. Open instructor questions are listed in Section 11.4. |
| **Preparation date** | 24 November 2026 (version 1.0: 23 November 2026) |
| **Project** | PayLink Year Work Project — Phase 1 (Requirements pack) |
| **Primary source** | *PayLink Year Work Project – Student Brief* (16 pages), referred to as "the Brief" |
| **Prepared by** | PayLink project team (Requirements and Product Lead), with AI assistance (Claude, Anthropic). AI output was reviewed against the Brief; see the prompt log for verdicts. |
| **Repository location** | `docs/requirements.md` (this document); PDF copy submitted to the course page |

**Document history**

| Version | Date | Description |
|---|---|---|
| 1.0 | 23 November 2026 | Baseline for the Phase 1 requirements pack. |
| 1.1 | 24 November 2026 | Review corrections: NFR-07, NFR-09 and NFR-10 clarified; audit and ledger relationship (AC-25.5 to AC-25.9) corrected; refund decisions of US-27 made explicit; all 36 ambiguity records re-reviewed and re-classified; consistency fixes listed in the change log; visual redesign of the PDF. |

## 1. Introduction

### 1.1 Purpose

This document specifies what the PayLink backend must do and how well it must do it. It translates the Brief into testable user stories, fixed business rules, a payment life cycle, measurable quality targets, and a transparent record of every interpretation decision. It is written for the developers who will build PayLink, the testers who will verify it, and the teaching staff who will grade it. It is the input to the Phase 1 design pack and the Phase 2 implementation.

### 1.2 Scope

PayLink is a bill-payment and digital-wallet **backend** for Egypt. This SRS covers:

- customer, agent, support and administrator capabilities exposed by the backend;
- the eight modules named in the Brief (Section 4);
- integration with four billers — ElectroCo, AquaCo, TopUpNet (mandatory) and NetCo (optional) — through mock servers that follow the Brief's contracts;
- the Phase 2 change request for ElectroCo API v2.

**Scope statement.** This SRS describes the **entire** PayLink project, not Phase 1 only. Phase 1 and Phase 2 differ in *when* a deliverable is due (Section 1.8), not in whether a requirement belongs to PayLink. No requirement is removed or weakened because it is implemented in Phase 2.

The user interface is deliberately minimal (a simple web page or command-line client is enough, Brief §3.4). Real payment cards, real SMS providers, real billers and regulatory compliance are outside the scope (Section 13).

### 1.3 Product overview

PayLink lets a customer hold money in an EGP wallet and use it to pay electricity, water, mobile top-up and internet bills. Customers use a mobile application; agents (small shops with a PayLink terminal) can also operate on behalf of customers. PayLink earns a service fee per payment and pays agents a share of that fee as commission.

The system is a money-handling system first. It must never charge twice, never charge for a payment the biller did not receive, never let a balance go negative, and always be able to explain every movement of money. It runs as a modular backend, deployed locally with containers in this project.

### 1.4 Business objectives

| ID | Objective | How it is measured in this SRS |
|---|---|---|
| BO-1 | Be correct with money: no double charges, no charges for undelivered payments, no negative balances | NFR-01; hard cases HC-1, HC-2, HC-4, HC-6 |
| BO-2 | Make adding or changing a biller cheap and safe | NFR-04; HC-8; Phase 2 change request (Section 8.4) |
| BO-3 | Cope with the busiest day of the month | NFR-02 |
| BO-4 | Earn fees and pay commissions exactly as the rules prescribe | FR-16, FR-17; business rules BR-05 to BR-09 |
| BO-5 | Be accountable: every money movement and administrative action can be traced and reconciled with billers | FR-25, FR-26; NFR-05 |
| BO-6 | Keep customers informed with clear outcomes and reasons | FR-18, FR-22, FR-23 |

### 1.5 Definitions and terminology

| Term | Meaning in this document |
|---|---|
| Customer | A registered person who owns a wallet and pays bills. |
| Agent | A registered shop operator who uses a PayLink terminal to act for customers and earns commission. |
| Support employee | A PayLink staff member who investigates payments and manages accounts. |
| Administrator | A PayLink staff member who manages billers, configuration and verification levels. |
| Biller | A company that issues bills and receives payments (utility, mobile operator, internet provider). |
| Bill inquiry | A request to a biller asking how much a customer owes before paying. |
| Wallet | A customer's (or agent's) EGP money account with an available balance and a held balance. |
| Available balance | Money in the wallet that can be spent. |
| Held balance | Money reserved for payments that are not yet confirmed. |
| Ledger | The list of entries recording every money movement. A balance is the sum of its entries and is never overwritten. |
| Hold | Money moved from available to held for a payment that is not yet final. |
| Service fee | The amount PayLink charges the customer per payment. |
| Commission | The share of the service fee PayLink pays to an agent for a payment made at the agent's terminal. |
| Payment reference | The unique identifier PayLink assigns to each payment and sends to the biller. |
| Idempotency key | A unique value sent with a request so that sending it twice has the effect of sending it once. |
| Settlement file | A biller's daily list of payments it actually received. |
| Reconciliation | Comparing PayLink's payment records with the settlement file and flagging differences. |
| Callback | An asynchronous message from a biller (TopUpNet) reporting the final result of an accepted payment. |
| Refund | Returning held or charged money to a wallet. |
| Financial operation | One business operation that moves money, such as a top-up, a hold, a hold release, a payment completion or a refund. One financial operation can produce several ledger entries and has exactly one audit record (AC-25.5). |
| Refund request | A separate record, created for a Completed payment, that support approves or rejects (US-27). The payment itself stays Completed. |
| Basic / Verified account | The two verification levels; they determine the daily payment limit. |
| Final state | A payment state with no outgoing transitions (Completed, Rejected, Refunded). |
| Unknown payment | A payment whose outcome at the biller is not yet known (for example, after a timeout). |
| Mock biller | A small program that follows a biller's contract and can be told to fail on purpose. |
| EGP / piaster | Egyptian pound / one hundredth of a pound. |

### 1.6 References

| Ref | Document |
|---|---|
| [1] | *PayLink Year Work Project – Student Brief*, 16 pages. Primary source of truth. Cited as "Brief §n" using its own section numbers (for example, Brief §2.5 is the payment life cycle). |

The structure of this SRS follows common SRS practice (for example, the section headings of ISO/IEC/IEEE 29148) but no formal conformance is claimed.

### 1.7 Document conventions

**Identifier scheme**

| Prefix | Meaning |
|---|---|
| FR-n | Functional requirement from the Brief (FR-1 to FR-28). Original IDs and priorities are preserved. |
| US-nn | User story written for this SRS. US-01 to US-28 correspond one-to-one to FR-1 to FR-28; US-29 to US-31 are additional stories (Section 5.1). |
| AC-nn.m | Acceptance criterion m of user story nn. |
| BR-nn | Fixed business rule stated in the Brief (Section 6.1). |
| DR-nn | Rule derived by the team to fill a gap in the Brief. Each is an assumption or decision recorded in Section 11 (Section 6.4). |
| NFR-nn | Non-functional requirement. NFR-01 to NFR-06 are the Brief's expected targets; NFR-07 to NFR-10 are team-defined. |
| HC-n | Hard case from Brief §2.7. |
| BI-nn / CR-n | Biller-integration requirement / ElectroCo v2 change-request requirement (Section 8). |
| AMB-nn | Ambiguity, question or decision record (Section 11). |
| RF-n | Open financial decision about refunds of completed payments (US-27, AMB-28). |

**Priorities.** Only three priorities are used: **Must**, **Should** and **Could**. FR priorities are copied from the Brief and are never changed. The Brief does not define the three levels, so this SRS reads them as: *Must* — mandatory for the Phase 2 system, cannot be removed; *Should* — expected and planned, first to defer if time runs out; *Could* — optional, built only if time allows. Stories that do not map one-to-one to an FR state how their priority was derived.

**Money.** All amounts are in EGP with exactly two decimal places and are handled as exact decimals (BR-14). Examples in this document use plain decimal notation such as `352.50`.

**Dates and times.** "Day" means a calendar day in the Africa/Cairo time zone unless stated otherwise (AMB-08).

### 1.8 Delivery stages (Phase 1 and Phase 2)

Every requirement in this SRS applies to the final PayLink system. The table shows when each group of deliverables is due (Brief §4). Marks are those of the Brief.

| Stage | Due | Deliverables | Requirements in this SRS |
|---|---|---|---|
| Phase 1 (30 points) | Wednesday 25 November 2026 | Requirements pack (this SRS as PDF and `docs/requirements.md`); design pack in `docs/design/`; DevOps foundation (protected `main`, Dockerfile and compose file starting an empty application and the database, pipeline that builds, lints and runs at least one test); prompt log | All requirements are *defined* here. NFR-10, Phase 1 part, is *delivered* in Phase 1. |
| Phase 2 (60 points) | Friday 18 December 2026 | Working system with tests, five design patterns, architecture decision records and event-driven evolution of Notifications and Reconciliation, ElectroCo API v2 change, pipeline with local deployment, health check and rollback, deployment comparison, final report, demo video | FR-1 to FR-28 (by priority), US-29 (optional), US-30, NFR-01 to NFR-09, NFR-10 Phase 2 part, the technical requirements of Brief §3.4 (Section 13.2), and the automated test for each hard case |
| Individual discussion (10 points) | In the lab, from Saturday 19 December 2026 | Live change on the team code and questions | Modifiability (NFR-04) and the decisions recorded in Section 11 |

## 2. Problem Summary

### 2.1 The situation

Millions of people in Egypt pay recurring bills — electricity, water, phone credit, internet — every month. Traditionally this means a trip to a utility office or a queue at a shop, handled in cash and within office hours. PayLink exists to replace that effort with a digital wallet. Customers top up once and then pay any supported bill from their phone, or ask a nearby agent to do it for them at a PayLink terminal.

### 2.2 What customers struggle with today

Paying a bill the old way costs time and offers little certainty. Customers must find the right counter, carry cash, and then trust a paper receipt. Each biller has its own process and its own way of showing what is owed. A customer who is unsure whether a payment went through often pays again, and a slow network makes that doubt worse. Customers without bank accounts depend on agents and have no easy way to see their own history. The service must therefore show the amount, the fee and the total *before* paying, and report the outcome clearly *after* paying.

### 2.3 Why holding customer money is hard

A wallet makes PayLink a custodian of other people's money, which carries obligations that a normal web application does not have. A balance has to be explainable at any moment, so it is derived from a ledger of entries instead of being edited in place. Two requests arriving at the same instant must not spend the same money twice. Funds for a payment must be set aside before the biller is contacted and released only when it is certain the biller did not receive the payment. Every mistake is a financial loss for either the customer or PayLink, and possibly a regulatory problem.

### 2.4 Why accurate processing and reconciliation matter

Payments cross a network that can fail at any point. When a biller does not answer, PayLink cannot tell whether the biller never received the request or received it and simply did not reply. Refunding in the first case is correct; refunding in the second means the customer got a free bill payment at PayLink's expense. For this reason the system must have an explicit *Unknown* state and must never refund automatically from it. Reconciliation is the independent second check: each night the biller says what it received, PayLink compares that list with its own records, and every difference becomes work for support instead of a silent loss.

### 2.5 Why integrating several billers is difficult

The four billers behave very differently. One answers instantly in JSON. Another answers slowly in XML with its own field names and error codes and rejects overpayments. A third accepts a request and reports success or failure minutes later through a callback that may arrive twice. A fourth only exchanges a batch file once a day. In addition, billers change over time; ElectroCo will announce a new API version that renames fields, changes the unit of money and requires signed requests. PayLink needs one uniform way of treating payments internally while keeping every biller's peculiarities isolated, so that a biller change or a new biller does not ripple through the rest of the system.

### 2.6 Business value

Building PayLink properly gives the company: fee revenue on every payment; a growing agent network motivated by commission; customer trust, which is the core asset of a wallet business; fewer manual investigations because failures, duplicates and mismatches are handled and reported automatically; faster onboarding of new billers; and the capacity to handle the highest-volume days without incident.

## 3. Stakeholders

| ID | Stakeholder |
|---|---|
| S-01 | Customers |
| S-02 | Agents |
| S-03 | System administrators |
| S-04 | Customer support employees |
| S-05 | Biller companies |
| S-06 | Finance and reconciliation team |
| S-07 | PayLink management |
| S-08 | External service providers (SMS, email, push) |
| S-09 | Development and maintenance team |

### S-01 Customers

- **Description:** Individuals who register, hold a wallet and pay bills, mainly through the mobile app.
- **Responsibilities:** Provide accurate registration and bill details; keep their credentials safe; review the amount, fee and total before confirming.
- **Main goals:** Pay bills quickly without queuing; never be charged twice or for a payment that did not reach the biller; see where their money is.
- **Expectations from the system:** Clear amount, fee and total up front; a single charge per payment even on a slow network; honest status when the outcome is pending; prompt SMS and push messages with clear reasons when something fails.
- **Interactions with PayLink:** Register, top up, run bill inquiries, pay, view balances and history, request refunds of completed payments, receive notifications.

### S-02 Agents

- **Description:** Operators of small shops with a PayLink terminal who serve customers, including those who prefer cash.
- **Responsibilities:** Identify the customer correctly; collect cash for top-ups; act only with the customer's authorisation.
- **Main goals:** Complete customer requests quickly and earn commission.
- **Expectations from the system:** Reliable commission of 40% of the service fee credited to their wallet on completion; clear payment results at the terminal; no commission for payments that do not complete.
- **Interactions with PayLink:** Perform cash top-ups and pay bills on behalf of customers; view their own wallet and commission entries.

### S-03 System administrators

- **Description:** PayLink staff who configure and operate the platform.
- **Responsibilities:** Add, edit and disable billers and services; maintain fees, limits and promotions; upgrade verification levels; review daily totals.
- **Main goals:** Change business parameters without redeploying; keep the catalog accurate; keep an auditable trail of their own actions.
- **Expectations from the system:** Validated configuration changes that apply immediately to new payments; an audit record for every administrative action.
- **Interactions with PayLink:** Administration functions for the catalog, configuration, verification and reporting.

### S-04 Customer support employees

- **Description:** Staff who answer customer questions and investigate payment problems.
- **Responsibilities:** Find payments by reference; interpret payment history; approve or reject refund requests; block and unblock accounts; review reconciliation differences.
- **Main goals:** Resolve a customer's issue from a single view of the payment's full history.
- **Expectations from the system:** Complete, ordered payment histories including biller responses and retries; clear flags for discrepancies and Unknown payments.
- **Interactions with PayLink:** Support functions: payment search, account blocking, refund approval, reconciliation review.

### S-05 Biller companies

- **Description:** ElectroCo (electricity), AquaCo (water), TopUpNet (mobile top-up) and NetCo (internet, optional). Each is simulated by a mock server in this project.
- **Responsibilities:** Answer inquiries and payments according to their contracts; send callbacks or result files; produce daily settlement files.
- **Main goals:** Be paid correctly and quickly, without duplicate or missing payments.
- **Expectations from the system:** Requests that follow each biller's contract, a unique reference on every payment, no duplicate submissions, and a daily comparison of settlement data.
- **Interactions with PayLink:** Receive inquiries, payments and batch files; send responses, callbacks, result files and settlement files.

### S-06 Finance and reconciliation team

- **Description:** Staff who ensure PayLink's records agree with billers' records and that fees and commissions are correct.
- **Responsibilities:** Review daily reconciliation reports; follow up discrepancies; check fee income and commissions.
- **Main goals:** Detect and resolve differences between PayLink and each biller every day.
- **Expectations from the system:** A daily reconciliation report that classifies each difference, and daily financial totals that agree with the ledger.
- **Interactions with PayLink:** Read reconciliation reports and daily totals. The finance role is read-only: settlement files are ingested by the system or uploaded by an administrator (AMB-27, Section 9.3). (This role is a team assumption; see AMB-03.)

### S-07 PayLink management

- **Description:** The company leadership that sponsors the platform.
- **Responsibilities:** Set business objectives and fee policy; accept the system.
- **Main goals:** Protect trust and revenue; scale to peak volumes; add billers cheaply.
- **Expectations from the system:** Correctness with money, extensibility, and readiness for the busiest day of the month.
- **Interactions with PayLink:** Indirect, through reports and the outcomes of the quality targets in Section 9.

### S-08 External service providers (SMS, email, push)

- **Description:** Third-party services that deliver messages to customers.
- **Responsibilities:** Deliver messages and report delivery success or failure.
- **Main goals:** Receive well-formed message requests.
- **Expectations from the system:** Retry only a bounded number of times and tolerate provider outages without affecting payments.
- **Interactions with PayLink:** Receive message requests from the Notifications module. In this project they are simulated.

### S-09 Development and maintenance team

- **Description:** Engineers, testers and DevOps staff who build, test, deploy and evolve PayLink.
- **Responsibilities:** Implement the requirements, keep the pipeline green, add billers and absorb change requests.
- **Main goals:** A modular system where changes stay local, supported by automated tests for every hard case.
- **Expectations from the system:** Clear module boundaries, configuration-driven business rules, and testable requirements.
- **Interactions with PayLink:** Build, test, deploy and maintain the system.

**Note on other parties.** The course instructors and teaching assistants wrote the Brief and grade the work. They are not product stakeholders, but the questions in Section 11 are addressed to them. Regulators are mentioned in the Brief as a risk; the Brief states no regulatory requirements, so none are specified here (Section 13).

## 4. System Scope and Major Modules

### 4.1 Modules

Each module has its own responsibilities, tests and public interface (Brief §3.1). Six modules are core; two are marked "extra" in the Brief.

| ID | Module | Type | Responsibility | Main collaborations |
|---|---|---|---|---|
| M1 | Accounts and Wallet | Core | Customers, agents, verification levels, wallets, ledger entries, holds, top-ups, refunds | Payment Processing (holds, charges, refunds); Fees and Commissions (commission credit); Fraud Checks (verification level, block status) |
| M2 | Biller Catalog | Core | Billers, services, required input fields, minimum and maximum amounts, active or disabled status | Payment Processing and Biller Integration (read catalog data); administrators (maintain it) |
| M3 | Payment Processing | Core | Payment life cycle, idempotency, orchestration of the payment steps | Orchestrates all other modules |
| M4 | Biller Integration | Core | One adapter per biller that hides its protocol; timeout, retry and callback handling | Payment Processing (requests and outcomes); Reconciliation (settlement files); Catalog (biller configuration) |
| M5 | Fees and Commissions | Core | Fee calculation (fixed, percentage, tiered, promotion) and agent commission | Payment Processing (quotes and charges); Accounts and Wallet (income and commission entries) |
| M6 | Notifications | Core | SMS, email and push messages with retry and message templates | Triggered by Payment Processing outcomes |
| M7 | Fraud Checks | Additional ("extra") | Daily-limit, single-payment-limit, velocity and blocked-account checks as an ordered chain of rules | Payment Processing (called before funds are held); Accounts and Wallet (account data) |
| M8 | Reconciliation | Additional ("extra") | Comparison of settlement files with PayLink payments and the difference report | Payment Processing (state updates for Unknown payments); Biller Integration (files) |

"Extra" in the Brief refers to the minimum module count. The requirements those modules implement keep their own priorities: for example, limit enforcement (FR-18) is Must, velocity (FR-19) and the reconciliation report (FR-26) are Should.

**Cross-cutting capabilities.** The Brief does not list audit logging, support operations or administration as modules. This SRS treats audit logging (FR-25) as a cross-cutting capability that every module uses, and allocates support and administration functions (FR-20, FR-21, FR-24, FR-28) to the module that owns the data they act on.

### 4.2 How the modules cooperate in one payment

| Brief §2.2 step | Responsible module(s) |
|---|---|
| 1. Login and limits | Accounts and Wallet; Fraud Checks |
| 2. Bill inquiry | Biller Catalog; Biller Integration |
| 3. Fee and total | Fees and Commissions |
| 4. Fraud checks | Fraud Checks |
| 5. Hold the money | Accounts and Wallet, driven by Payment Processing |
| 6. Call the biller | Biller Integration |
| 7. Confirmation (charge, fee income, commission) | Payment Processing; Accounts and Wallet; Fees and Commissions |
| 8. Receipt | Notifications |
| 9. Night reconciliation | Reconciliation |

### 4.3 System boundary

**Inside the system:** customer and agent accounts, wallets and the ledger; the biller catalog; payment processing and its life cycle; one integration per biller; fee and commission rules; notification generation and retry; fraud checks; reconciliation and reporting; the audit log; administrative and support functions.

**Outside the system (external actors):** customers' mobile app and agents' terminals (simple clients); the biller companies (mock servers following the Brief's contracts); the SMS, email and push providers (simulated); the card gateway used for simulated card top-ups (simulated).

**Phases.** Phase 1 delivers this requirements pack, the design pack and the DevOps foundation. Phase 2 delivers the working system, including the ElectroCo v2 change (Section 13 separates these categories).
## 5. Functional Requirements and User Stories

### 5.1 Overview

This section contains **31 user stories**. US-01 to US-28 map one-to-one to FR-1 to FR-28 of the Brief and keep their priorities. Three further stories make behaviour explicit that the Brief states only in passing: US-29 (the optional NetCo batch capability, part of FR-15), US-30 (the Phase 2 ElectroCo API v2 change request, Brief §2.4) and US-31 (how Unknown payments are resolved, Brief §2.5). Their priorities and their reasons are given in the story.

Stories are grouped by the functional areas A–J. Within each story, every acceptance criterion (AC) is written as Given / When / Then and is intended to become at least one automated test. Where a criterion depends on a team assumption, the assumption is cited (AMB-nn) and listed in Section 11. Rejection and failure reasons are written as logical reason codes (for example `DAILY_LIMIT_EXCEEDED`); Section 6.3 lists them. Business-rule values (BR-nn) come from configuration, never from code.

| Priority | FRs from the Brief | Count |
|---|---|---|
| Must | FR-1, 3, 4, 5, 6, 7, 8, 9, 10, 11, 12, 13, 14, 15, 16, 18, 20, 22, 25 | 19 |
| Should | FR-2, 17, 19, 21, 23, 24, 26 | 7 |
| Could | FR-27, 28 | 2 |

### 5.2 Area A — Account and Wallet Management

#### US-01 — Customer Registration

**User Story ID:** US-01  
**Related Requirement:** FR-1  
**Title:** Customer Registration  
**Priority:** Must  
**Module:** M1 Accounts and Wallet

**User Story:** As a customer, I want to register for a PayLink account so that I can access wallet and bill-payment services.

**Acceptance Criteria:**

- **AC-01.1** **Given** a visitor with no PayLink account, **When** they submit a full name, a valid Egyptian mobile number that is not yet registered, and a credential, **Then** an account is created with verification level *Basic*, an available balance of `0.00` and a held balance of `0.00`, and the customer can log in.
- **AC-01.2** **Given** a mobile number that already belongs to an account, **When** registration is submitted, **Then** it is rejected with `MOBILE_ALREADY_REGISTERED` and no second account is created.
- **AC-01.3** **Given** a missing mandatory field or a malformed mobile number, **When** registration is submitted, **Then** it is rejected with a message naming each invalid field and no account is created.
- **AC-01.4** **Given** a newly registered customer, **When** a payment is checked against limits, **Then** the Basic daily limit (BR-01) applies.
- **AC-01.5** **Given** any registration or login response, **When** it is inspected, **Then** it never contains the customer's credential, and the credential is stored only in non-reversible form.

*Assumptions: AMB-01 (what "basic verification" means), AMB-03 (authentication).*

#### US-02 — Verification Upgrade

**User Story ID:** US-02  
**Related Requirement:** FR-2  
**Title:** Verification Upgrade  
**Priority:** Should  
**Module:** M1 Accounts and Wallet

**User Story:** As an administrator, I want to upgrade a customer from Basic to Verified so that the customer can pay under the higher daily limit.

**Acceptance Criteria:**

- **AC-02.1** **Given** a Basic customer and an authorised administrator, **When** the administrator upgrades the customer and gives a reason, **Then** the customer's level becomes *Verified* and the Verified daily limit (BR-02) applies to every payment checked afterwards.
- **AC-02.2** **Given** a Basic customer who already has `4,000.00` of counted payments today, **When** they are upgraded, **Then** the next payment is checked against `30,000.00` including the existing `4,000.00`.
- **AC-02.3** **Given** a customer who is already Verified, **When** an upgrade is requested, **Then** nothing changes and the response states `ALREADY_VERIFIED`.
- **AC-02.4** **Given** a user without the administrator role, **When** they request an upgrade, **Then** the request is denied and the level is unchanged.
- **AC-02.5** **Given** a successful upgrade, **When** it completes, **Then** an audit record with who, when and why exists (FR-25).

*Assumptions: AMB-02 (who upgrades and on what evidence).*

#### US-03 — Wallet Balance and Transaction History

**User Story ID:** US-03  
**Related Requirement:** FR-3  
**Title:** Wallet Balance and Transaction History  
**Priority:** Must  
**Module:** M1 Accounts and Wallet

**User Story:** As a customer, I want to see my available balance, my held balance and my transaction history so that I know where my money is.

**Acceptance Criteria:**

- **AC-03.1** **Given** a wallet funded with `1,000.00` and one payment of total `352.50` in state FundsHeld, **When** the customer views the wallet, **Then** available is `647.50` and held is `352.50`.
- **AC-03.2** **Given** a wallet with ledger entries, **When** the customer requests the history, **Then** entries are listed newest first with date and time, entry type (for example top-up, hold, hold release, charge, refund), amount, direction, and the payment reference where one exists, in pages of a configurable size.
- **AC-03.3** **Given** a new wallet with no entries, **When** it is viewed, **Then** available and held are `0.00` and the history is empty.
- **AC-03.4** **Given** customer A, **When** A requests the wallet, the history or any ledger entry of customer B, **Then** the request is denied with `ACCESS_DENIED` and the response contains no data about B (NFR-09).
- **AC-03.5** **Given** any balance shown, **When** it is displayed, **Then** it has exactly two decimal places and equals the sum of the corresponding ledger entries (US-05).
- **AC-03.6** **Given** an agent, **When** the agent views a wallet or history, **Then** only the agent's own wallet and its entries (cash top-up debits, commission credits) are available; the balances and history of customers served by the agent are not (NFR-09).

#### US-04 — Wallet Top-up

**User Story ID:** US-04  
**Related Requirement:** FR-4  
**Title:** Wallet Top-up  
**Priority:** Must  
**Module:** M1 Accounts and Wallet

**User Story:** As a customer, I want to top up my wallet with a simulated card or with cash at an agent so that I have funds to pay bills.

**Acceptance Criteria:**

- **AC-04.1** **Given** a valid simulated-card test token and an amount greater than `0.00`, **When** the simulated gateway approves the charge, **Then** the available balance increases by the amount and a top-up ledger entry is created.
- **AC-04.2** **Given** a simulated card that the gateway declines, **When** the top-up is requested, **Then** the balance is unchanged, no credit entry exists, and the customer sees `CARD_DECLINED`.
- **AC-04.3** **Given** an authenticated agent who has received cash from a customer, **When** the agent submits a cash top-up for that customer, **Then** the customer's available balance increases by the amount and the ledger entry records the agent's identity.
- **AC-04.4** **Given** a cash top-up of `500.00`, **When** it completes, **Then** the agent's wallet is debited `500.00` in the same financial operation; if the agent's available balance is lower, the top-up is rejected with `INSUFFICIENT_AGENT_FUNDS` and nothing changes. *(Team decision: AMB-05.)*
- **AC-04.5** **Given** an amount that is zero, negative, or has more than two decimal places, **When** a top-up is requested, **Then** it is rejected with `INVALID_AMOUNT`.
- **AC-04.6** **Given** a top-up request sent twice with the same idempotency key, **When** both arrive, **Then** the wallet is credited once. *(Team decision: AMB-13.)*
- **AC-04.7** **Given** any top-up, **When** it is stored, **Then** no real card number or other real personal data is stored.
- **AC-04.8** **Given** a customer, **When** they submit a card top-up for another customer's wallet, or submit a cash top-up (an agent-only operation), **Then** it is denied with `ACCESS_DENIED` and nothing changes (NFR-09).
- **AC-04.9** **Given** an authenticated agent, **When** the agent tops up their own wallet with a simulated card (AC-04.1), **Then** the agent's available balance increases; this is how an agent funds cash top-ups for customers. *(Team decision: AMB-05.)*

*Assumptions: AMB-04 (simulated card), AMB-05 (agent cash).*

#### US-05 — Ledger-Based Balances

**User Story ID:** US-05  
**Related Requirement:** FR-5  
**Title:** Ledger-Based Balances  
**Priority:** Must  
**Module:** M1 Accounts and Wallet

**User Story:** As a finance team member, I want every balance change recorded as a ledger entry and every balance computed from the ledger so that any balance can be explained and audited.

**Acceptance Criteria:**

- **AC-05.1** **Given** any operation that changes an available or held balance (top-up, hold, hold release, charge, refund, fee, commission), **When** it completes, **Then** at least one ledger entry exists with the wallet or PayLink account, amount, direction, entry type, related reference, financial-operation ID (AC-25.6) and timestamp.
- **AC-05.2** **Given** a wallet at any moment, **When** its balances are read, **Then** available and held each equal the sum of the matching ledger entries, and no operation writes a balance directly. A test recomputes every wallet from its entries and finds zero differences.
- **AC-05.3** **Given** a posted ledger entry, **When** anyone attempts to change or delete it, **Then** the attempt is refused; corrections are made only by new compensating entries.
- **AC-05.4** **Given** a payment of total `352.50` that completes with a fee of `2.50` and an agent commission of `1.00`, **When** the ledger entries of that payment are summed across all wallets and PayLink accounts, **Then** the net is `0.00`. *(Money conservation: AMB-32.)*
- **AC-05.5** **Given** two concurrent debits against one wallet whose available balance covers only one (HC-4), **When** both are processed, **Then** exactly one is applied, the other is rejected with `INSUFFICIENT_FUNDS`, and the available balance is never below `0.00`.
- **AC-05.6** **Given** a failed operation that is rolled back, **When** the ledger is read, **Then** no entry from the failed operation remains.

*Assumptions: AMB-32 (accounting model).*

### 5.3 Area B — Biller Catalog

#### US-06 — Biller and Service Management

**User Story ID:** US-06  
**Related Requirement:** FR-6  
**Title:** Biller and Service Management  
**Priority:** Must  
**Module:** M2 Biller Catalog

**User Story:** As an administrator, I want to add, edit and disable billers and their services so that the catalog always reflects what customers can pay.

**Acceptance Criteria:**

- **AC-06.1** **Given** an administrator, **When** they add a biller service with a name, service category, required input fields (for example "meter number"), and minimum and maximum amounts, **Then** the service is created as *Active* and appears in the customer catalog.
- **AC-06.2** **Given** an existing service, **When** the administrator edits its input fields or amount range, **Then** the change applies to inquiries and payments created afterwards and does not alter existing payments.
- **AC-06.3** **Given** an Active service, **When** the administrator disables it, **Then** it no longer appears in the customer catalog, new inquiries and payments for it are rejected with `BILLER_DISABLED`, and payments already in progress continue to their outcome (AMB-33).
- **AC-06.4** **Given** a disabled service, **When** the administrator re-enables it, **Then** it appears in the catalog again.
- **AC-06.5** **Given** invalid data (minimum greater than maximum, duplicate service name, no input fields), **When** the administrator saves it, **Then** it is rejected with a validation message and nothing is stored.
- **AC-06.6** **Given** a user without the administrator role, **When** they try any catalog change, **Then** it is denied.
- **AC-06.7** **Given** every add, edit or disable, **When** it completes, **Then** an audit record with who, when, why and the before/after values exists.
- **AC-06.8** **Given** a new biller whose integration already exists in the Biller Integration module (HC-8), **When** its catalog entry is created, **Then** it becomes payable with no change to any other module. *(Verified by NFR-04.)*

#### US-07 — View Billers and Required Inputs

**User Story ID:** US-07  
**Related Requirement:** FR-7  
**Title:** View Billers and Required Inputs  
**Priority:** Must  
**Module:** M2 Biller Catalog

**User Story:** As a customer, I want to see the list of available billers and the information each one needs so that I know what to enter before paying.

**Acceptance Criteria:**

- **AC-07.1** **Given** an authenticated customer, **When** they request the catalog, **Then** every Active biller service is returned with its name, service category, required input fields and minimum and maximum amounts.
- **AC-07.2** **Given** a disabled biller service, **When** the catalog is requested, **Then** it is not returned.
- **AC-07.3** **Given** the ElectroCo electricity service and the TopUpNet mobile top-up service, **When** the catalog is requested, **Then** the first lists "meter number" and the second lists "phone number" as required inputs.
- **AC-07.4** **Given** a customer who selected a service that is disabled before they submit, **When** they submit an inquiry, **Then** it is rejected with `BILLER_DISABLED`.
- **AC-07.5** **Given** an unauthenticated request, **When** the catalog is requested, **Then** it is denied.

### 5.4 Area C — Bill Inquiry and Payment Processing

#### US-08 — Bill Inquiry with Fee and Total

**User Story ID:** US-08  
**Related Requirement:** FR-8  
**Title:** Bill Inquiry with Fee and Total  
**Priority:** Must  
**Module:** M3 Payment Processing (with M2, M4, M5)

**User Story:** As a customer, I want to run a bill inquiry and see the amount, the service fee and the total before I pay so that there are no surprises.

**Acceptance Criteria:**

- **AC-08.1** **Given** an Active ElectroCo service and a meter number whose bill is `350.00`, **When** the customer runs an inquiry, **Then** the response shows amount `350.00`, fee `2.50` and total `352.50`, and no hold, ledger entry or payment is created.
- **AC-08.2** **Given** an Active AquaCo service and an account number whose bill is `120.00`, **When** an inquiry is run, **Then** the response shows amount `120.00`, fee `2.50` and total `122.50`.
- **AC-08.3** **Given** a service without a live inquiry (TopUpNet, NetCo), **When** the customer enters a phone or subscriber number and an amount of `100.00`, **Then** the response shows amount `100.00`, the fee from the fee rules and the total (for TopUpNet: `1.50` and `101.50`). *(AMB-18.)*
- **AC-08.4** **Given** an identifier unknown to the biller, **When** an inquiry is run, **Then** the response is `BILL_NOT_FOUND` and no fee is shown.
- **AC-08.5** **Given** a biller that does not answer within the live-biller timeout (BR-11), **When** an inquiry is run, **Then** the response is `BILLER_UNAVAILABLE` and no money or payment record is affected.
- **AC-08.6** **Given** a promotion active for the biller, **When** an inquiry is run, **Then** the fee shown is `0.00` and the total equals the amount.
- **AC-08.7** **Given** a bill amount outside the permitted single-payment range (BR-03, BR-04), **When** an inquiry is run, **Then** the amount is shown with a statement that it cannot be paid and the reason (`AMOUNT_BELOW_MINIMUM` or `AMOUNT_ABOVE_MAXIMUM`).
- **AC-08.8** **Given** a missing required input, **When** an inquiry is run, **Then** it is rejected with `INVALID_INPUT` naming the missing field.

#### US-09 — Pay a Bill from the Wallet

**User Story ID:** US-09  
**Related Requirement:** FR-9  
**Title:** Pay a Bill from the Wallet  
**Priority:** Must  
**Module:** M3 Payment Processing

**User Story:** As a customer, I want to pay a bill from my wallet so that I do not have to queue or carry cash.

**Acceptance Criteria:**

- **AC-09.1** **Given** a Basic customer with available `1,000.00`, no payments today, an ElectroCo bill of `350.00` and a fee of `2.50`, **When** the customer pays with an idempotency key, **Then** the payment passes the checks, `352.50` is held, the request goes to ElectroCo with the payment reference, and on confirmation the payment is *Completed*: held falls by `352.50`, available stays `647.50`, PayLink fee income rises by `2.50`.
- **AC-09.2** **Given** an available balance of `300.00` and a total of `352.50`, **When** the customer pays, **Then** the payment is *Rejected* with `INSUFFICIENT_FUNDS`, no hold is taken and the biller is not called.
- **AC-09.3** **Given** a payment request with no idempotency key, **When** it is submitted, **Then** it is rejected with `IDEMPOTENCY_KEY_REQUIRED` and no payment is created. *(AMB-13.)*
- **AC-09.4** **Given** a missing required input or an amount that differs from the inquired bill amount, **When** the payment is submitted, **Then** it is rejected with `INVALID_INPUT` and no payment is created. *(AMB-18.)*
- **AC-09.5** **Given** two payments from one wallet that arrive at the same moment and a balance that covers only one (HC-4), **When** both are processed, **Then** one proceeds, the other is *Rejected* with `INSUFFICIENT_FUNDS`, and the balance never becomes negative.
- **AC-09.6** **Given** any accepted payment request, **When** a response is returned, **Then** it contains a unique payment reference, which is the reference sent to the biller, and the current payment state.
- **AC-09.7** **Given** a rejected payment, **When** the customer views the response and history, **Then** the rejection reason is stated clearly and no ledger entry exists for it.
- **AC-09.8** **Given** a customer, **When** they submit a payment that names a wallet other than their own, **Then** it is denied with `ACCESS_DENIED`; a payment is always taken from the wallet of the authenticated customer, or of the customer an agent serves under US-17 (NFR-09).

#### US-10 — Idempotent Payment Requests

**User Story ID:** US-10  
**Related Requirement:** FR-10  
**Title:** Idempotent Payment Requests  
**Priority:** Must  
**Module:** M3 Payment Processing

**User Story:** As a customer on a slow network, I want a repeated request to return the original result so that I am never charged twice.

**Acceptance Criteria:**

- **AC-10.1** **Given** a payment request with key K that has been processed, **When** the same key and the same content arrive again (HC-1), **Then** no new payment, hold or biller call is made and the response returns the original payment reference and result.
- **AC-10.2** **Given** the first request with key K is still in progress, **When** the second arrives, **Then** it returns the same payment reference and current state, and still only one payment exists.
- **AC-10.3** **Given** two requests with key K that arrive at the same moment, **When** both are processed, **Then** exactly one payment exists and both responses refer to it.
- **AC-10.4** **Given** the first request with key K was *Rejected*, **When** the same key is replayed, **Then** the same rejection and reason are returned without re-evaluating the rules.
- **AC-10.5** **Given** key K already used for a payment, **When** it is sent with different content (another biller, amount or account), **Then** it is rejected with `IDEMPOTENCY_KEY_REUSED` and the original payment is unaffected.
- **AC-10.6** **Given** a key used within the retention period (configurable; AMB-13), **When** it is replayed, **Then** AC-10.1 to AC-10.5 apply.
- **AC-10.7** **Given** a replay, **When** it is processed, **Then** it causes no ledger entry, no notification and no velocity count (AC-19.5).

*Assumptions: AMB-13 (key scope, retention, mismatching content).*

#### US-11 — Payment Life Cycle and Valid Transitions

**User Story ID:** US-11  
**Related Requirement:** FR-11  
**Title:** Payment Life Cycle and Valid Transitions  
**Priority:** Must  
**Module:** M3 Payment Processing

**User Story:** As PayLink management, I want every payment to follow the defined life cycle and no other so that money and status never disagree.

**Acceptance Criteria:**

- **AC-11.1** **Given** any payment at any moment, **When** its state is read, **Then** it is exactly one of Initiated, FundsHeld, SentToBiller, Completed, Failed, Unknown, Rejected, Refunded.
- **AC-11.2** **Given** a payment and an event listed in Section 7.2 (T1–T9), **When** the event occurs, **Then** the payment moves to the target state and the history records the old state, new state, trigger and time.
- **AC-11.3** **Given** every pair of states, **When** a transition is attempted that is not one of T1–T9 (for example Completed→Failed, FundsHeld→Completed, Unknown→Refunded, Rejected→FundsHeld), **Then** it is refused, the state is unchanged and the attempt is logged. A test covers all 64 state pairs and finds exactly nine permitted.
- **AC-11.4** **Given** a payment in Completed, Rejected or Refunded, **When** any event arrives, **Then** the state does not change.
- **AC-11.5** **Given** a payment in Unknown, **When** any event arrives, **Then** it can reach Refunded only through Failed (T8 then T9).
- **AC-11.6** **Given** two events for the same payment at the same moment (for example a callback and a retry both confirming), **When** both are processed, **Then** exactly one transition is applied and its money effects occur once.

#### US-12 — Fund Holds and Releases

**User Story ID:** US-12  
**Related Requirement:** FR-12  
**Title:** Fund Holds and Releases  
**Priority:** Must  
**Module:** M1 Accounts and Wallet (driven by M3)

**User Story:** As a customer, I want my money reserved before the biller is contacted and returned if the payment fails so that I never lose money on a failed payment.

**Acceptance Criteria:**

- **AC-12.1** **Given** a payment that passes the checks and a wallet that covers the total (amount plus fee), **When** the hold is taken, **Then** available falls and held rises by the total, and the hold is stored before any request is sent to the biller.
- **AC-12.2** **Given** a payment in SentToBiller whose biller refuses it with an error code (HC-3), **When** the refusal arrives, **Then** the payment becomes *Failed*, the whole held total is released to available, the payment becomes *Refunded*, no fee income and no commission are recorded, and the customer receives a message with a clear reason (US-22).
- **AC-12.3** **Given** a payment in Unknown, **When** time passes, **Then** the hold stays in place and is never released automatically (BR-15).
- **AC-12.4** **Given** a payment that completes, **When** the confirmation is processed, **Then** the held amount becomes a final charge and no money returns to available.
- **AC-12.5** **Given** a Failed payment, **When** the release is triggered more than once, **Then** the hold is released exactly once.
- **AC-12.6** **Given** a released hold, **When** the ledger is read, **Then** entries for the hold and the release exist and balance to `0.00` for that payment.

#### US-13 — Timeout, Unknown State and Retries

**User Story ID:** US-13  
**Related Requirement:** FR-13  
**Title:** Timeout, Unknown State and Retries  
**Priority:** Must  
**Module:** M3 Payment Processing (with M4)

**User Story:** As a customer, I want a payment whose biller does not answer to be treated as pending and re-checked so that I am neither charged wrongly nor refunded wrongly.

**Acceptance Criteria:**

- **AC-13.1** **Given** ElectroCo takes 12 seconds to answer (HC-2), **When** the 10-second timeout (BR-11) elapses, **Then** the payment becomes Unknown at the timeout, the hold stays, and the customer receives a response saying the payment is being processed (not that it failed) no later than 12 seconds after the original request (NFR-07).
- **AC-13.2** **Given** a payment that became Unknown, **When** time passes, **Then** the biller is re-queried 1, 5 and 15 minutes after the payment became Unknown, at most three times (BR-12, AMB-21).
- **AC-13.3** **Given** a re-query that returns confirmation or refusal, **When** it is received, **Then** the payment is resolved as described in US-31.
- **AC-13.4** **Given** three re-queries that give no answer, **When** the third has been made, **Then** the payment remains Unknown with funds held, no further automatic query is made, and it waits for reconciliation (AMB-23).
- **AC-13.5** **Given** a re-query that itself times out, **When** it ends, **Then** it counts as one of the three attempts.
- **AC-13.6** **Given** a restart of the application between retries, **When** the system is running again, **Then** every scheduled re-query still takes place.
- **AC-13.7** **Given** a biller response that is neither a definite confirmation nor a definite refusal (connection error, server error, unreadable reply), **When** it is received, **Then** the payment becomes Unknown rather than Failed (AMB-20).
- **AC-13.8** **Given** a payment in Unknown, **When** the customer views the history, **Then** it is shown as being processed.
- **AC-13.9** **Given** a live biller that has not answered, **When** the 10-second timeout elapses, **Then** PayLink stops waiting for that request and returns the response at once; it never holds the customer's request open for the delayed answer. An answer that later arrives on the abandoned request is ignored and logged; the outcome is established only through a re-query, a callback or a settlement file (T7, T8).

*Assumptions: AMB-20, AMB-21, AMB-23.*

#### US-31 — Resolving Unknown Payments

**User Story ID:** US-31  
**Related Requirement:** FR-11, FR-13, FR-26 (Brief §2.5 transitions T7 and T8)  
**Title:** Resolving Unknown Payments  
**Priority:** Must (inherited from FR-11 and FR-13)  
**Module:** M3 Payment Processing (with M4, M8)

**User Story:** As support, I want an Unknown payment to be resolved only on firm evidence from the biller so that no money is wrongly returned or wrongly kept.

**Acceptance Criteria:**

- **AC-31.1** **Given** an Unknown payment, **When** the biller confirms it through a callback, a re-query or a settlement file, **Then** the payment becomes *Completed* once: the hold becomes a final charge, fee income is recorded, any agent commission is credited, and the customer is notified.
- **AC-31.2** **Given** an Unknown payment, **When** the biller refuses it, **Then** the payment becomes *Failed* and then *Refunded* with the hold released, and the customer is notified.
- **AC-31.3** **Given** an Unknown payment whose request was sent before the safety margin that precedes the cut-off of business day D (DR-19), and a complete settlement file for D that has been processed and does not list it, **When** reconciliation runs, **Then** the payment becomes Failed and then Refunded (T8, T9) with reason `NOT_RECEIVED_BY_BILLER`; a payment sent inside the margin is compared with the next day's file instead.
- **AC-31.4** **Given** an Unknown payment with no evidence either way, **When** any amount of time passes, **Then** it stays Unknown and held and is never refunded automatically.
- **AC-31.5** **Given** a confirmation that arrives after the timeout but before reconciliation, **When** it is received, **Then** AC-31.1 applies.
- **AC-31.6** **Given** a confirmation that arrives for a payment already Failed or Refunded, **When** it is received, **Then** no state or money change is made, the event is logged, and the payment is flagged for support review (AMB-26).
- **AC-31.7** **Given** an Unknown payment that is still unresolved after reconciliation for its business day, **When** the report is produced, **Then** it is listed as requiring manual review (AMB-23).
- **AC-31.8** **Given** an Unknown payment flagged for manual review, **When** support requests an additional status re-query, **Then** it has the same effect as an automatic re-query (AC-31.1, AC-31.2), the request is audited with who and why, and silence from the biller still leaves the payment Unknown and held (AMB-23).

*Assumptions: AMB-23, AMB-24, AMB-26, AMB-27.*
### 5.5 Area D — Biller Integrations

#### US-14 — TopUpNet Asynchronous Callbacks

**User Story ID:** US-14  
**Related Requirement:** FR-14  
**Title:** TopUpNet Asynchronous Callbacks  
**Priority:** Must  
**Module:** M4 Biller Integration

**User Story:** As PayLink, I want to accept TopUpNet's callbacks and ignore duplicates so that every mobile top-up completes exactly once.

**Acceptance Criteria:**

- **AC-14.1** **Given** a mobile top-up sent to TopUpNet, **When** TopUpNet replies "accepted", **Then** the payment stays in SentToBiller, the funds stay held, and the customer is told the top-up is being processed.
- **AC-14.2** **Given** a payment awaiting a callback, **When** a "success" callback arrives with its payment reference, **Then** the payment becomes *Completed* once and fee income, commission (if any) and the notification follow.
- **AC-14.3** **Given** the same "success" callback arrives a second time (HC-6), **When** it is processed, **Then** it is ignored: no state change, no ledger entry, no second notification, and the duplicate is logged with the reference and time.
- **AC-14.4** **Given** a payment awaiting a callback, **When** a "failed" callback arrives, **Then** the payment becomes *Failed* and then *Refunded* with the hold released, and the customer is notified with a clear reason.
- **AC-14.5** **Given** a callback with an unknown or malformed payment reference, **When** it arrives, **Then** it has no effect on any payment and is logged.
- **AC-14.6** **Given** a callback that contradicts a final outcome (for example "failed" after Completed), **When** it arrives, **Then** it is ignored, logged and flagged for support review (AMB-26).
- **AC-14.7** **Given** two identical callbacks processed at the same moment, **When** both are handled, **Then** exactly one has an effect.
- **AC-14.8** **Given** no callback within the configured callback wait period, **When** the period ends, **Then** the payment becomes Unknown and follows US-13 (AMB-22).
- **AC-14.9** **Given** a callback that is not authenticated (missing or invalid signature or shared secret), **When** it arrives, **Then** it is rejected, has no effect on any payment, and is logged (AMB-26).

*Assumptions: AMB-22 (missing callback), AMB-26 (callback authenticity and conflicts).*

#### US-15 — Multiple Billers with Different Formats

**User Story ID:** US-15  
**Related Requirement:** FR-15  
**Title:** Multiple Billers with Different Formats  
**Priority:** Must  
**Module:** M4 Biller Integration

**User Story:** As PayLink management, I want customers to pay through at least three billers that use different formats so that PayLink covers electricity, water and mobile top-up.

**Acceptance Criteria:**

- **AC-15.1** **Given** ElectroCo (REST/JSON), **When** a customer inquires by meter number and pays, **Then** the confirmation is received in the same interaction and the payment completes.
- **AC-15.2** **Given** AquaCo (XML, 2–4 second response, own field names and error codes), **When** a customer inquires and pays, **Then** the result is translated to PayLink's common outcomes (confirmed, refused with reason, no answer) and the customer sees a clear PayLink message, never a raw biller code; the raw code is kept in the payment history for support.
- **AC-15.3** **Given** an AquaCo bill of `120.00`, **When** a payment is prepared, **Then** PayLink never sends an amount above the inquired bill amount; if AquaCo nonetheless refuses a payment, US-12 (AC-12.2) applies.
- **AC-15.4** **Given** TopUpNet (asynchronous), **When** a top-up is paid, **Then** US-14 applies.
- **AC-15.5** **Given** one biller that is completely down (every call times out or errors), **When** customers pay the other two billers, **Then** those payments complete normally (NFR-03).
- **AC-15.6** **Given** a new biller to be added (HC-8), **When** it is implemented, **Then** only a new integration and configuration are added; existing modules need no change (NFR-04).
- **AC-15.7** **Given** a biller error code that PayLink has no mapping for, **When** it is received, **Then** the payment is treated as Unknown, not Failed (AMB-20).

#### US-29 — NetCo Batch Payments (Optional)

**User Story ID:** US-29  
**Related Requirement:** FR-15 (optional NetCo part, Brief §2.4 and §3.2)  
**Title:** NetCo Batch Payments (Optional)  
**Priority:** Could  
**Module:** M4 Biller Integration

**User Story:** As a customer, I want to pay my internet bill through NetCo so that all four bill types are available. This is an optional capability; Phase 2 acceptance does not depend on it.

**Acceptance Criteria:**

- **AC-29.1** **Given** an Active NetCo service, **When** a customer enters a subscriber number and an amount, **Then** no live inquiry is made, the fee follows the tiered rule (BR-07) and the payment, once the checks pass, holds the total and waits in FundsHeld until the daily batch.
- **AC-29.2** **Given** payments waiting for the daily batch, **When** the batch runs once per day, **Then** one CSV file listing each of them is produced and uploaded, each listed payment moves to SentToBiller, and no payment appears in two batches. *(AMB-36.)*
- **AC-29.3** **Given** the next morning's result file, **When** it is processed, **Then** each confirmed payment becomes Completed and each refused payment becomes Failed and then Refunded with the hold released.
- **AC-29.4** **Given** a payment that is in SentToBiller but absent from the result file, **When** the file is processed, **Then** it stays in SentToBiller and is listed in the next reconciliation report; it is not refunded.
- **AC-29.5** **Given** the live-biller timeout (BR-11), **When** a NetCo payment waits for its result file, **Then** the timeout does not apply, because NetCo is not a live biller.

#### US-30 — ElectroCo API v2 Change Request (Phase 2)

**User Story ID:** US-30  
**Related Requirement:** Brief §2.4 change request; NFR-04  
**Title:** ElectroCo API v2 Change Request  
**Priority:** Must (mandatory Phase 2 deliverable, Brief §4; no FR number exists)  
**Module:** M4 Biller Integration

**User Story:** As the development team, I want to absorb ElectroCo's new API version by changing only the ElectroCo integration so that the rest of PayLink is unaffected.

**Acceptance Criteria:**

- **AC-30.1** **Given** an ElectroCo mock that implements API v2 (renamed fields, amounts in piasters, signed requests), **When** a customer pays an electricity bill of `350.00`, **Then** the request carries `35000` piasters and the payment completes, while the customer still sees `350.00` EGP.
- **AC-30.2** **Given** the v2 mock, **When** a request is sent without a valid signature, **Then** the mock refuses it; every PayLink request to ElectroCo carries a valid signature.
- **AC-30.3** **Given** unit conversion, **When** amounts are converted between EGP and piasters, **Then** the conversion is exact (`0.01` EGP is `1` piaster) with no floating-point arithmetic.
- **AC-30.4** **Given** the completed change, **When** the changed files are listed, **Then** they belong only to the ElectroCo integration (and its configuration and tests); no file of another module changed. The Phase 2 report lists these files.
- **AC-30.5** **Given** the change is applied, **When** the hard-case tests (HC-1 to HC-8) run, **Then** they all still pass.

### 5.6 Area E — Fees and Commissions

#### US-16 — Fee Calculation

**User Story ID:** US-16  
**Related Requirement:** FR-16  
**Title:** Fee Calculation  
**Priority:** Must  
**Module:** M5 Fees and Commissions

**User Story:** As PayLink, I want to charge the correct service fee for each biller type, including promotions, so that revenue and customer pricing follow the published rules.

**Acceptance Criteria:**

- **AC-16.1** **Given** an electricity or water payment of any amount, **When** the fee is calculated, **Then** it is `2.50` (BR-05).
- **AC-16.2** **Given** mobile top-up amounts, **When** the fee is calculated, **Then** `100.00` gives `1.50`; `50.00` gives `1.00` because `0.75` is below the minimum; `33.33` gives `1.00`; `10,000.00` gives `150.00` (BR-06, rounding per AMB-14).
- **AC-16.3** **Given** internet amounts, **When** the fee is calculated, **Then** `150.00` and `200.00` give `2.00`, and `200.01` gives `4.00` (BR-07).
- **AC-16.4** **Given** a promotion configured for one biller with a start and end date, **When** a payment for that biller is made within the period, **Then** the fee is `0.00`; for other billers, or outside the period, the normal fee applies (BR-08).
- **AC-16.5** **Given** a promotion that has been configured for a biller, **When** another promotion with an overlapping period is saved, **Then** it is rejected (AMB-16).
- **AC-16.6** **Given** any fee calculation, **When** it is performed, **Then** it uses exact decimal arithmetic, the result has two decimal places, and a test with values such as `0.1 + 0.2` shows no rounding error (BR-14).
- **AC-16.7** **Given** a payment that is Rejected or Failed, **When** the ledger is read, **Then** no fee income exists for it; fee income is recorded only at completion.
- **AC-16.8** **Given** the fee rules, **When** the rule values are read, **Then** they come from configuration (BR-05 to BR-08).

*Assumptions: AMB-14, AMB-15, AMB-16, AMB-17.*

#### US-17 — Agent Payments and Commission

**User Story ID:** US-17  
**Related Requirement:** FR-17  
**Title:** Agent Payments and Commission  
**Priority:** Should  
**Module:** M5 Fees and Commissions (with M1, M3)

**User Story:** As an agent, I want to pay a bill on behalf of a customer at my terminal and earn commission so that serving customers is worthwhile.

**Acceptance Criteria:**

- **AC-17.1** **Given** an authenticated agent linked to a terminal, a customer identified by registered mobile number who has authorised the payment (AC-17.8), and a wallet that covers the total, **When** the agent submits the payment, **Then** it follows the normal life cycle against the customer's wallet and limits, and the payment record stores the agent and the terminal. *(Team decision: AMB-06.)*
- **AC-17.2** **Given** an agent payment with a fee of `2.50`, **When** it completes, **Then** `1.00` (40% of the fee, BR-09) is credited to the agent's wallet by a commission ledger entry.
- **AC-17.3** **Given** an agent payment whose fee is `0.00` because of a promotion, **When** it completes, **Then** the commission is `0.00` and no credit entry is created.
- **AC-17.4** **Given** an agent payment that is Rejected, Failed or Unknown, **When** its state is read, **Then** no commission has been credited; **When** an Unknown payment later becomes Completed, **Then** the commission is credited once at that moment.
- **AC-17.5** **Given** a payment made directly by the customer, **When** it completes, **Then** no commission is created.
- **AC-17.6** **Given** a fee of `1.07`, **When** commission is calculated, **Then** it is `0.43` (rounding per AMB-14).
- **AC-17.7** **Given** a user who is not an agent, **When** they use the agent channel, **Then** the request is denied.
- **AC-17.8** **Given** an agent who has started a payment for a customer, **When** PayLink sends a single-use one-time code to the customer's registered mobile through the Notifications module and the agent submits the payment with that code before it expires, **Then** the payment is authorised; a missing, wrong, expired or already used code is rejected with `AGENT_AUTHORISATION_FAILED` and no payment is created. *(Team decision: AMB-06.)*
- **AC-17.9** **Given** an agent, **When** the agent requests data, **Then** only the agent's own wallet, own commission entries and the payments the agent submitted are visible; a customer's balance or history is never shown to the agent (NFR-09).

*Assumptions: AMB-06, AMB-14.*

### 5.7 Area F — Fraud and Security

#### US-18 — Single-Payment and Daily Limits

**User Story ID:** US-18  
**Related Requirement:** FR-18  
**Title:** Single-Payment and Daily Limits  
**Priority:** Must  
**Module:** M7 Fraud Checks

**User Story:** As PayLink, I want payments above the single-payment or daily limit rejected with a clear reason so that exposure to fraud and error is bounded.

**Acceptance Criteria:**

- **AC-18.1** **Given** a payment amount of `10,000.01`, **When** it is checked, **Then** it is *Rejected* with `AMOUNT_ABOVE_MAXIMUM`; an amount of `10,000.00` passes this rule (BR-04).
- **AC-18.2** **Given** a payment amount of `4.99`, **When** it is checked, **Then** it is *Rejected* with `AMOUNT_BELOW_MINIMUM`; `5.00` passes this rule (BR-03).
- **AC-18.3** **Given** a Basic customer with `4,800.00` of counted payments today, **When** they pay `300.00`, **Then** it is *Rejected* with `DAILY_LIMIT_EXCEEDED`; a payment of `200.00` is accepted because the limit is "at most" `5,000.00` (BR-01, AMB-10).
- **AC-18.4** **Given** a Verified customer, **When** their counted payments plus the new amount would exceed `30,000.00`, **Then** it is *Rejected* with `DAILY_LIMIT_EXCEEDED` (BR-02).
- **AC-18.5** **Given** earlier payments in states Failed, Refunded or Rejected, **When** the daily total is computed, **Then** they are not counted; payments in FundsHeld, SentToBiller, Unknown and Completed are counted (AMB-08).
- **AC-18.6** **Given** a payment that violates more than one rule, **When** it is checked, **Then** the reason of the first failing rule in the fixed order is returned (AMB-12).
- **AC-18.7** **Given** a rejection by these rules, **When** it happens, **Then** the payment is recorded as Rejected with the reason code and a human-readable message, no hold is taken, no biller is called and no ledger entry is created.
- **AC-18.8** **Given** a new calendar day in Africa/Cairo, **When** a payment is checked, **Then** the previous day's payments are not counted (AMB-08).
- **AC-18.9** **Given** an administrator changes a limit (US-24), **When** the next payment is checked, **Then** the new value is used.

*Assumptions: AMB-07, AMB-08, AMB-10, AMB-12.*

#### US-19 — Velocity Check and Temporary Block

**User Story ID:** US-19  
**Related Requirement:** FR-19  
**Title:** Velocity Check and Temporary Block  
**Priority:** Should  
**Module:** M7 Fraud Checks

**User Story:** As PayLink, I want an account that makes too many payments too quickly to be blocked for a short time so that automated abuse is stopped.

**Acceptance Criteria:**

- **AC-19.1** **Given** an account that has made five payment requests within 60 seconds, **When** a sixth arrives inside the same 60 seconds, **Then** the sixth is *Rejected* with `VELOCITY_EXCEEDED` and the account is blocked for 10 minutes (BR-10).
- **AC-19.2** **Given** an account blocked by this rule, **When** any payment is attempted during the 10 minutes, **Then** it is *Rejected* with `ACCOUNT_BLOCKED`, the message states the block is temporary and when it ends, and the block is not extended.
- **AC-19.3** **Given** a velocity block that is 10 minutes old, **When** a payment is attempted, **Then** the block is lifted automatically and the payment is processed normally.
- **AC-19.4** **Given** six payments spread over more than 60 seconds so that no 60-second window holds more than five, **When** they are made, **Then** no block occurs.
- **AC-19.5** **Given** a replayed request with a known idempotency key (US-10), **When** it arrives, **Then** it does not count towards the five.
- **AC-19.6** **Given** payments for the same customer account made by the customer and by an agent, **When** the window is evaluated, **Then** they are counted together.
- **AC-19.7** **Given** a velocity block, **When** support unblocks the account (US-21), **Then** payments are accepted again immediately.
- **AC-19.8** **Given** a sixth request within 60 seconds that also violates an earlier rule in the fixed order (for example the daily limit), **When** it is checked, **Then** it is rejected with the earlier rule's reason (AMB-12) and the velocity count still includes it, so the 10-minute block starts (AMB-11).

*Assumptions: AMB-11, AMB-12.*

### 5.8 Area G — Notifications

#### US-22 — Completion and Failure Notifications

**User Story ID:** US-22  
**Related Requirement:** FR-22  
**Title:** Completion and Failure Notifications  
**Priority:** Must  
**Module:** M6 Notifications

**User Story:** As a customer, I want an SMS and a push message when my payment completes or fails so that I know the result without opening the app.

**Acceptance Criteria:**

- **AC-22.1** **Given** a payment that becomes Completed, **When** the outcome is recorded, **Then** an SMS and a push message are sent containing the payment reference, biller, amount, fee, total and time.
- **AC-22.2** **Given** a payment that becomes Failed (HC-3), **When** the outcome is recorded, **Then** an SMS and a push message are sent stating a clear reason and that the held money has been returned.
- **AC-22.3** **Given** a payment in Unknown, **When** its state is Unknown, **Then** no completion or failure message is sent; a message is sent when it is resolved (AMB-29).
- **AC-22.4** **Given** the SMS provider is down when a payment completes (HC-5), **When** the notification fails, **Then** the payment remains Completed, the ledger is unchanged, and US-23 governs the retries.
- **AC-22.5** **Given** a duplicate callback or a replayed request, **When** it is ignored, **Then** no additional notification is sent.
- **AC-22.6** **Given** any message, **When** it is produced, **Then** its text comes from a message template and contains no credential.
- **AC-22.7** **Given** a payment that is Rejected, **When** the customer receives the response, **Then** the reason is part of the response and no SMS is required (AMB-29).

*Assumptions: AMB-29.*

#### US-23 — Notification Retries and Undelivered Status

**User Story ID:** US-23  
**Related Requirement:** FR-23  
**Title:** Notification Retries and Undelivered Status  
**Priority:** Should  
**Module:** M6 Notifications

**User Story:** As PayLink, I want failed notifications retried a limited number of times and then marked undelivered so that temporary provider outages do not lose messages and support can see what failed.

**Acceptance Criteria:**

- **AC-23.1** **Given** a notification whose send fails, **When** retries are made, **Then** it is retried up to three times after the first attempt (four attempts in total), each attempt recorded with time and result (BR-13, AMB-29).
- **AC-23.2** **Given** a notification whose fourth attempt also fails (HC-5), **When** the last attempt ends, **Then** its status becomes *Undelivered*, it is visible to support, and the payment state and ledger are unaffected.
- **AC-23.3** **Given** a retry that succeeds, **When** it succeeds, **Then** the status becomes *Delivered* and no further attempts are made.
- **AC-23.4** **Given** an SMS that fails and a push message that succeeds for the same payment, **When** their statuses are read, **Then** each channel has its own status and retry count.
- **AC-23.5** **Given** scheduled retries, **When** the application restarts, **Then** the retries still take place.
- **AC-23.6** **Given** the retry interval, **When** retries are scheduled, **Then** the interval is read from configuration.

*Assumptions: AMB-29.*

### 5.9 Area H — Support and Administration

#### US-20 — Payment Search and Full History for Support

**User Story ID:** US-20  
**Related Requirement:** FR-20  
**Title:** Payment Search and Full History for Support  
**Priority:** Must  
**Module:** M3 Payment Processing

**User Story:** As a support employee, I want to find a payment by its reference and see its full history so that I can answer customer questions.

**Acceptance Criteria:**

- **AC-20.1** **Given** an existing payment reference, **When** support searches for it, **Then** the result shows customer, biller, amount, fee, total, current state, agent and terminal if any, timestamps, and the complete ordered list of state transitions with triggers.
- **AC-20.2** **Given** the same payment, **When** support opens its history, **Then** it also shows the linked ledger entries, the biller responses and codes, each retry attempt and outcome, callbacks received (including ignored duplicates) and notification statuses.
- **AC-20.3** **Given** a reference that does not exist, **When** support searches, **Then** the response is `PAYMENT_NOT_FOUND`.
- **AC-20.4** **Given** a user whose role is not support (customer, agent, administrator or finance), **When** they try the support search, **Then** it is denied with `ACCESS_DENIED`.
- **AC-20.5** **Given** a Rejected payment, **When** support opens it, **Then** the rejection reason is shown.

#### US-21 — Block and Unblock Accounts

**User Story ID:** US-21  
**Related Requirement:** FR-21  
**Title:** Block and Unblock Accounts  
**Priority:** Should  
**Module:** M1 Accounts and Wallet

**User Story:** As a support employee, I want to block and unblock accounts so that I can stop suspicious activity and restore service when it is cleared.

**Acceptance Criteria:**

- **AC-21.1** **Given** an active account and a stated reason, **When** support blocks it, **Then** new payments are *Rejected* with `ACCOUNT_BLOCKED`, and payments already in progress continue to their outcome (AMB-12).
- **AC-21.2** **Given** a blocked account and a stated reason, **When** support unblocks it, **Then** new payments are accepted again and any active velocity block is cleared.
- **AC-21.3** **Given** a block or unblock request without a reason, **When** it is submitted, **Then** it is rejected.
- **AC-21.4** **Given** an account that is already blocked, **When** support blocks it again, **Then** nothing changes and the response says so.
- **AC-21.5** **Given** a user without the support role, **When** they try to block or unblock, **Then** it is denied.
- **AC-21.6** **Given** each block or unblock, **When** it completes, **Then** an audit record with who, when and why exists.

#### US-24 — Runtime Configuration of Fees and Limits

**User Story ID:** US-24  
**Related Requirement:** FR-24  
**Title:** Runtime Configuration of Fees and Limits  
**Priority:** Should  
**Module:** M5 Fees and Commissions and M7 Fraud Checks (configuration)

**User Story:** As an administrator, I want to edit fee and limit values without redeploying so that business changes do not need a software release.

**Acceptance Criteria:**

- **AC-24.1** **Given** an administrator, **When** they change a fee value, a limit value or a promotion period and save, **Then** payments initiated after the save use the new value without a restart or redeployment.
- **AC-24.2** **Given** a value that is invalid (negative amount, percentage outside 0–100, minimum above maximum, Verified daily limit below Basic daily limit, promotion end before start), **When** it is saved, **Then** it is rejected and the old value remains.
- **AC-24.3** **Given** a payment already in progress, **When** a value changes, **Then** the amounts already calculated for that payment do not change (AMB-31).
- **AC-24.4** **Given** each change, **When** it is saved, **Then** an audit record with who, when, why, old value and new value exists.
- **AC-24.5** **Given** a user without the administrator role, **When** they try to change a value, **Then** it is denied.
- **AC-24.6** **Given** the business rules of Section 6, **When** the configuration is read, **Then** at least the fees, the single and daily limits, and the promotion are editable at runtime; the other rule values (timeouts, retries, velocity) are configuration but need not be editable at runtime.

*Assumptions: AMB-31.*

#### US-25 — Audit Log

**User Story ID:** US-25  
**Related Requirement:** FR-25  
**Title:** Audit Log  
**Priority:** Must  
**Module:** Cross-cutting (all modules)

**User Story:** As a finance team member, I want every money movement and administrative action written to an audit log so that I can establish who did what, when and why.

**Acceptance Criteria:**

- **AC-25.1** **Given** any money movement (top-up, hold, release, charge, fee, commission, refund), **When** it completes, **Then** an audit record exists with who (user, agent or named system component), when, why (the reason or triggering event and the payment reference) and the amount and wallet involved.
- **AC-25.2** **Given** an administrative action (catalog change, configuration change, verification upgrade, block or unblock, refund decision), **When** it completes, **Then** an audit record exists with who, when, why and the before and after values.
- **AC-25.3** **Given** an audit record, **When** anyone tries to change or delete it through the application, **Then** the attempt is refused.
- **AC-25.4** **Given** an automatic action such as a retry, a callback or reconciliation, **When** it moves money, **Then** the "who" identifies the system component, not a person.
- **AC-25.5** **Given** any financial operation (Section 1.5), **When** it completes, **Then** exactly one audit record exists for that operation, identified by a financial-operation ID. One operation can produce several ledger entries (for example a payment completion produces the charge, biller, fee-income and commission entries), so the number of ledger entries is **not** required to equal the number of audit records (NFR-05).
- **AC-25.6** **Given** any ledger entry, **When** it is read, **Then** it carries the ID of its financial operation and the reference of that operation's audit record, and the audit record lists the ledger entries it covers.
- **AC-25.7** **Given** a test run over all ledger entries and all audit records, **When** they are cross-checked, **Then** every financial operation has an audit record, every ledger entry has a corresponding audit record, and no audit record states that money moved without matching ledger entries: zero orphans in either direction (100%, NFR-05).
- **AC-25.8** **Given** a financial operation, **When** it is committed, **Then** its ledger entries, balance effects and audit record are committed in the same atomic transaction, so that either all of them exist or none exists (AMB-30).
- **AC-25.9** **Given** an operation that fails and is rolled back, **When** the ledger and the audit log are read, **Then** neither contains an entry stating that the money moved; an attempt that moved no money is visible only as a failure in the payment history.

*Assumptions: AMB-30.*

### 5.10 Area I — Reconciliation and Reporting

#### US-26 — Settlement Reconciliation Report

**User Story ID:** US-26  
**Related Requirement:** FR-26  
**Title:** Settlement Reconciliation Report  
**Priority:** Should  
**Module:** M8 Reconciliation

**User Story:** As a finance team member, I want a daily reconciliation report built from the biller's settlement file so that every difference between PayLink and the biller is found.

**Acceptance Criteria:**

- **AC-26.1** **Given** a settlement file for business day D, **When** it is processed, **Then** every record is compared with PayLink's payments by payment reference, amount and status, and a report lists matched records and every difference.
- **AC-26.2** **Given** a payment that PayLink holds as Failed (or Failed then Refunded) but the file lists as received (HC-7), **When** the report is produced, **Then** the difference is flagged `STATUS_MISMATCH` for support review, and no payment state or ledger entry is changed automatically.
- **AC-26.3** **Given** a payment PayLink holds as Completed whose request was sent before the safety margin that precedes the cut-off of business day D (DR-19), **When** the report is produced and the file for D is complete, **Then** a payment absent from the file is flagged `MISSING_IN_SETTLEMENT`; a payment sent inside the margin is compared with the next day's file instead.
- **AC-26.4** **Given** a file record whose reference matches no PayLink payment, **When** the report is produced, **Then** it is flagged `UNKNOWN_IN_PAYLINK`.
- **AC-26.5** **Given** a record whose amount differs from PayLink's, **When** the report is produced, **Then** it is flagged `AMOUNT_MISMATCH` with both amounts.
- **AC-26.6** **Given** Unknown payments, **When** the file is processed, **Then** a listed payment is resolved per US-31 and an unlisted payment is handled per AC-31.3.
- **AC-26.7** **Given** a malformed file, a file that fails its completeness check (the record count in its trailer line), or malformed rows, **When** it is processed, **Then** the problems are reported and valid rows are still processed; no file is silently dropped, and an incomplete file is never used to conclude that a payment was not received (AC-31.3).
- **AC-26.8** **Given** the same file processed twice, **When** the second run finishes, **Then** the report is identical and no money or state effect is repeated.
- **AC-26.9** **Given** a report, **When** it is produced, **Then** it contains counts and totals by category and is stored and available to support, finance and administrators.
- **AC-26.10** **Given** a file record whose payment PayLink holds in Initiated, FundsHeld, SentToBiller or Rejected, **When** the report is produced, **Then** the difference is flagged `STATUS_MISMATCH` with PayLink's state, and no payment state or ledger entry is changed automatically.

*Assumptions: AMB-25, AMB-27.*

#### US-28 — Daily Totals

**User Story ID:** US-28  
**Related Requirement:** FR-28  
**Title:** Daily Totals  
**Priority:** Could  
**Module:** M8 Reconciliation (reporting)

**User Story:** As an administrator, I want to see daily totals so that I can follow volume, fee income, commissions and failures per biller.

**Acceptance Criteria:**

- **AC-28.1** **Given** a business day D, **When** an administrator requests totals, **Then** the response shows the number and total amount of Completed payments, total fees earned, total commissions paid, and the number of Failed payments per biller.
- **AC-28.2** **Given** a day without payments, **When** totals are requested, **Then** every figure is `0` or `0.00`.
- **AC-28.3** **Given** a day's totals, **When** they are compared with the ledger, **Then** fees earned equal the sum of fee-income entries (net of any fee-income reversal by an approved refund, US-27) and commissions paid equal the sum of commission entries (net of reversals) for that day.
- **AC-28.4** **Given** a user whose role is neither administrator nor finance, **When** they request totals, **Then** it is denied with `ACCESS_DENIED` (Section 9.3).
- **AC-28.5** **Given** a day boundary, **When** totals are computed, **Then** the Africa/Cairo calendar day is used (AMB-08).

### 5.11 Area J — Refund Management

#### US-27 — Refund of a Completed Payment

**User Story ID:** US-27  
**Related Requirement:** FR-27  
**Title:** Refund of a Completed Payment  
**Priority:** Could  
**Module:** M1 Accounts and Wallet (with M3)

**User Story:** As a customer, I want to request a refund of a completed payment and have support decide it so that mistaken payments can be corrected.

**Acceptance Criteria:**

- **AC-27.1** **Given** a Completed payment of the requesting customer, **When** they request a refund with a reason, **Then** a refund request in status *Pending* is created, the payment remains Completed and no money moves.
- **AC-27.2** **Given** a Pending request, **When** support approves it with a reason, **Then** the request becomes *Approved*, the customer's wallet is credited by compensating ledger entries, the customer is notified, and the decision is audited as one financial operation (US-25). The amount credited, the funding source, the treatment of fee income and the treatment of agent commission follow the refund policy RF-1 to RF-5 below; the working defaults apply until the instructors confirm them. *(AMB-28, Requires Instructor Confirmation.)*
- **AC-27.3** **Given** a Pending request, **When** support rejects it with a reason, **Then** the request becomes *Rejected*, no money moves and the customer is notified.
- **AC-27.4** **Given** a payment that is not Completed (Failed, Unknown, Rejected, Refunded), **When** a refund is requested, **Then** it is rejected with `REFUND_NOT_ELIGIBLE`; payments that failed are returned automatically through US-12.
- **AC-27.5** **Given** a payment that already has a Pending or Approved request, **When** another is requested, **Then** it is rejected, so a payment is never refunded twice.
- **AC-27.6** **Given** customer A, **When** A requests or views a refund for B's payment or B's refund requests, **Then** the request is denied with `ACCESS_DENIED` and no data about B is returned (NFR-09).
- **AC-27.7** **Given** this feature, **When** the payment state machine is examined, **Then** it is unchanged: the payment stays Completed and the refund is recorded separately (AMB-28).
- **AC-27.8** **Given** an approved refund, **When** its compensating entries are summed across all wallets and PayLink accounts, **Then** the net is `0.00` (AC-05.4), whichever funding option RF-2 is configured.
- **AC-27.9** **Given** two approvals of the same request at the same moment, or one approval replayed, **When** they are processed, **Then** the customer's wallet is credited once.
- **AC-27.10** **Given** an approved refund that reverses an agent commission, **When** the agent's available balance is lower than the commission to reverse, **Then** the wallet is debited only up to its available balance (never below `0.00`, NFR-01) and the remainder is recorded as an amount owed by the agent in a separate PayLink account (RF-4).
- **AC-27.11** **Given** a Completed payment that has an Approved refund, **When** the daily total is computed, **Then** the payment is still counted: it remains Completed (DR-01) and a refund does not restore the customer's daily limit (RF-6, DR-17).
- **AC-27.12** **Given** a customer, **When** they list refund requests, **Then** they see only their own requests with status and decision reason; support sees all requests and can list those that are Pending; other roles are denied (Section 9.3).
- **AC-27.13** **Given** a Completed payment with an Approved refund, **When** reconciliation processes a settlement file that lists the payment, **Then** it matches normally; the refund creates no difference, because the payment is still Completed (AC-26.1).
- **AC-27.14** **Given** a Rejected refund request, **When** the customer requests a refund of the same payment again, **Then** a new request is accepted; only a Pending or Approved request blocks a new one (AC-27.5).

**Open financial decisions (AMB-28).** The Brief says that a customer can request a refund of a completed payment and that support approves it (FR-27, Could). It does not say what is refunded, who funds it first, or who bears the loss. The table lists each decision, the options and the **working default** the design uses so that the feature is testable. A working default is **not** a confirmed business policy. RF-1 to RF-5 require instructor confirmation; the values are configuration, so a confirmed answer changes configuration and the affected criteria, not the payment state machine.

| ID | Decision | Options | Working default | Status |
|---|---|---|---|---|
| RF-1 | What the customer receives | (a) the bill amount only; (b) the bill amount plus the service fee, that is, the whole amount charged | (b) the whole amount charged, because the Brief's glossary describes a refund as returning "held or charged money" | Requires Instructor Confirmation |
| RF-2 | Who funds the refund first | (a) PayLink credits the customer at once and later recovers the money from the biller; (b) PayLink credits the customer only after the biller has returned the money | (a) PayLink funds first; the amount is booked as *recoverable from the biller* so the customer does not wait | Requires Instructor Confirmation |
| RF-3 | Fee income | (a) PayLink reverses its fee income; (b) PayLink keeps the fee | Follows RF-1: reversed when the fee is refunded, kept when only the bill amount is refunded | Requires Instructor Confirmation |
| RF-4 | Agent commission | (a) the agent's commission is reversed; (b) the agent keeps it | Follows RF-3: reversed when the fee is reversed. The agent wallet never goes below 0.00 (AC-27.10); any shortfall is booked as owed by the agent | Requires Instructor Confirmation |
| RF-5 | Money that cannot be recovered from the biller | (a) PayLink absorbs the loss; (b) the loss is shared with the agent | (a) PayLink absorbs the loss, booked to a refund-loss account; the customer's credit is never taken back | Requires Instructor Confirmation |
| RF-6 | Time window and daily limit | Whether refunds expire after a number of days; whether a refund restores the daily limit | No time window is enforced (the Brief states none); a refund does not restore the daily limit (DR-17) | Team Decision |

*Assumptions: AMB-28.*
## 6. Business Rules

### 6.1 Fixed business rules from the Brief

These values are fixed by the Brief (§2.2, §2.5, §2.6) so that every team solves the same problem. They are **configuration, not code**: the implementation must read each value from configuration and must not hardcode it.

| ID | Rule | Value | Brief source |
|---|---|---|---|
| BR-01 | Daily payment limit, Basic account | 5,000.00 EGP (a customer may pay *at most* this per day) | §2.2, §2.6 |
| BR-02 | Daily payment limit, Verified account | 30,000.00 EGP | §2.6 |
| BR-03 | Minimum single payment | 5.00 EGP | §2.6 |
| BR-04 | Maximum single payment | 10,000.00 EGP | §2.6 |
| BR-05 | Service fee for electricity and water | Fixed 2.50 EGP | §2.6 |
| BR-06 | Service fee for mobile top-up | 1.5% of the amount, minimum 1.00 EGP | §2.6 |
| BR-07 | Service fee for internet | Tiered: 2.00 EGP for amounts up to 200.00 EGP; 4.00 EGP above 200.00 EGP | §2.6 |
| BR-08 | Promotion | During a promotion period the fee is 0.00 EGP for one chosen biller; start and end dates are configurable | §2.6 |
| BR-09 | Agent commission | 40% of the service fee, credited to the agent's wallet when the payment completes | §2.6 |
| BR-10 | Velocity rule | More than 5 payments in 60 seconds from one account blocks the account for 10 minutes | §2.6 |
| BR-11 | Live-biller timeout | 10 seconds; after that the payment becomes Unknown | §2.6 |
| BR-12 | Unknown-payment retries | Re-queried up to 3 times, after 1, 5 and 15 minutes, then it waits for reconciliation | §2.6 |
| BR-13 | Notification retry | A failed SMS or email is retried 3 times, then marked undelivered | §2.6 |
| BR-14 | Rounding and precision | All money uses exact decimal types, never floating point | §2.6 |
| BR-15 | Unknown payments | Never refunded automatically; money is returned only once the system is sure the biller did not receive the payment | §2.5 |
| BR-16 | Final states | Completed, Rejected and Refunded are final | §2.5 |
| BR-17 | Amount held | The hold equals the bill amount plus the service fee (example: `352.50` for a `350.00` bill) | §2.2 |
| BR-18 | When fees and commission are booked | Fee income and commission are recorded when the payment completes, not before | §2.2, §2.6 |

### 6.2 Configuration parameters

| Parameter | Default | Source | Runtime-editable (US-24) |
|---|---|---|---|
| Daily limit, Basic / Verified | 5,000.00 / 30,000.00 EGP | BR-01, BR-02 | Yes |
| Minimum / maximum single payment | 5.00 / 10,000.00 EGP | BR-03, BR-04 | Yes |
| Fee parameters per biller type | See BR-05 to BR-07 | BR-05 to BR-07 | Yes |
| Promotion (biller, start date, end date) | None | BR-08 | Yes |
| Agent commission rate | 40% | BR-09 | Not required |
| Velocity: payment count, window, block duration | 5, 60 seconds, 10 minutes | BR-10 | Not required |
| Live-biller timeout | 10 seconds | BR-11 | Not required |
| Unknown re-query schedule | 1, 5, 15 minutes | BR-12 | Not required |
| Notification retry count | 3 | BR-13 | Not required |
| Notification retry interval | *Not set by the Brief; to be chosen in the design* | AMB-29 | Not required |
| Idempotency key retention | 24 hours (team proposal) | AMB-13 | Not required |
| Callback wait period (TopUpNet) | 30 minutes (team proposal) | AMB-22 | Not required |
| Agent authorisation code validity | 5 minutes (team proposal) | AMB-06 | Not required |
| Settlement safety margin before the cut-off | 15 minutes (team proposal) | AMB-27 | Not required |
| Refund policy (RF-1, RF-2) | Whole amount charged; PayLink funds first (working default, not confirmed) | AMB-28 | Not required |
| Business-day time zone | Africa/Cairo (team proposal) | AMB-08 | Not required |

"Not required" means the value must live in configuration but the system need not allow editing it while running (FR-24 asks only for fee and limit values).

### 6.3 Reason codes

Logical identifiers used in the acceptance criteria. The customer-facing wording is decided in the design; every code needs a clear, human-readable message.

| Code | Used for | Meaning |
|---|---|---|
| `ACCOUNT_BLOCKED` | Rejected payment | The account is blocked manually or temporarily by the velocity rule |
| `AMOUNT_BELOW_MINIMUM` | Rejected payment / inquiry | Amount is below the minimum single payment |
| `AMOUNT_ABOVE_MAXIMUM` | Rejected payment / inquiry | Amount is above the maximum single payment |
| `DAILY_LIMIT_EXCEEDED` | Rejected payment | The payment would take the day's total above the daily limit |
| `VELOCITY_EXCEEDED` | Rejected payment | More than 5 payments in 60 seconds; triggers the 10-minute block |
| `INSUFFICIENT_FUNDS` | Rejected payment | Available balance does not cover the total |
| `BILLER_DISABLED` | Rejected payment / inquiry | The biller service is disabled |
| `INVALID_INPUT` | Rejected request | A required input is missing or invalid |
| `IDEMPOTENCY_KEY_REQUIRED` | Rejected request | No idempotency key was supplied |
| `IDEMPOTENCY_KEY_REUSED` | Rejected request | The key was already used with different content |
| `BILL_NOT_FOUND` | Inquiry | The biller does not know the identifier |
| `BILLER_UNAVAILABLE` | Inquiry | The biller did not answer in time |
| `MOBILE_ALREADY_REGISTERED` | Registration | The mobile number belongs to an account |
| `CARD_DECLINED` | Top-up | The simulated gateway declined the card |
| `INSUFFICIENT_AGENT_FUNDS` | Cash top-up | The agent's wallet cannot fund the top-up (team decision, AMB-05) |
| `AGENT_AUTHORISATION_FAILED` | Agent payment | The customer's one-time authorisation code is missing, wrong, expired or already used (AMB-06) |
| `ACCESS_DENIED` | Any protected request | The caller is authenticated but not permitted to perform the operation or see the object (NFR-09) |
| `AUTHENTICATION_REQUIRED` | Any protected request | No valid credential, session or token was supplied (NFR-09) |
| `INVALID_AMOUNT` | Top-up | The amount is not a positive two-decimal amount |
| `ALREADY_VERIFIED` | Upgrade | The customer is already Verified |
| `PAYMENT_NOT_FOUND` | Support search | No payment has this reference |
| `REFUND_NOT_ELIGIBLE` | Refund request | The payment is not Completed or already has a refund request |
| `BILLER_REFUSED` | Failed payment | The biller refused the payment; the biller's error code is translated to a clear message and kept in the history |
| `NOT_RECEIVED_BY_BILLER` | Failed payment | Reconciliation established that the biller has no such payment |
| `STATUS_MISMATCH`, `MISSING_IN_SETTLEMENT`, `UNKNOWN_IN_PAYLINK`, `AMOUNT_MISMATCH` | Reconciliation report | Difference categories (AMB-27) |

### 6.4 Rules derived by the team

The Brief leaves some details open. The following rules fill the gaps. **They are not in the Brief.** Each is a team decision or assumption, recorded with its reasoning in Section 11 and flagged there if it needs stakeholder confirmation.

| ID | Derived rule | Record |
|---|---|---|
| DR-01 | Daily-limit totals count payments in FundsHeld, SentToBiller, Unknown and Completed; they exclude Rejected, Failed and Refunded. | AMB-08 |
| DR-02 | Single-payment and daily limits apply to the bill amount, not to the fee. | AMB-07 |
| DR-03 | A day is a calendar day in the Africa/Cairo time zone. | AMB-08 |
| DR-04 | Fraud rules run in this order and the first failure decides the reason: blocked account, single-payment limits, daily limit, velocity. | AMB-12 |
| DR-05 | The velocity count includes every non-replayed payment request that reaches the fraud checks, over a sliding 60-second window; the sixth request is rejected and starts the block. | AMB-11 |
| DR-06 | Fees and commissions are rounded half-up to two decimals; the 1.00 EGP minimum for mobile top-up is applied after rounding. | AMB-14 |
| DR-07 | The fee is calculated when the payment is initiated and stored with the payment. | AMB-17 |
| DR-08 | An idempotency key is mandatory, scoped to the authenticated user who submits the request (the customer for customer requests, the agent for agent-submitted requests), remembered for at least 24 hours (configurable), and rejected if reused with different content. | AMB-13 |
| DR-09 | A biller answer that is not a definite confirmation or a definite refusal makes the payment Unknown. | AMB-20 |
| DR-10 | Re-query times are measured from the moment the payment became Unknown. | AMB-21 |
| DR-11 | Failed is followed immediately by the hold release (Refunded) in the same operation; both transitions are recorded. | AMB-25 |
| DR-12 | The effective amount range of a service is the overlap of the global range (BR-03, BR-04) and the service's own range. | AMB-19 |
| DR-13 | A notification has one first attempt plus up to three retries, tracked per channel. | AMB-29 |
| DR-14 | A refund of a completed payment is a separate refund record; the payment stays Completed. Amount, funding, fee and commission treatment are open decisions RF-1 to RF-5 with working defaults. | AMB-28 |
| DR-15 | Reconciliation flags four difference categories and never changes a payment or the ledger on its own for a mismatch; it resolves Unknown payments only as BR-15 allows. | AMB-25, AMB-27 |
| DR-16 | Disabling a biller does not interrupt payments already in progress. | AMB-33 |
| DR-17 | A refund of a Completed payment does not restore the daily limit; the payment keeps counting because it stays Completed. | AMB-08, AMB-28 |
| DR-18 | Every callback is authenticated; a callback that fails authentication has no effect and is logged. | AMB-26 |
| DR-19 | A settlement file has a business-day cut-off and a record-count trailer. A payment sent within the safety margin before the cut-off is compared with the next day's file, and an incomplete file is never used to conclude that a payment was not received. | AMB-27 |
| DR-20 | An agent payment is authorised by a single-use one-time code sent to the customer's registered mobile. | AMB-06 |
| DR-21 | Support may request one additional status re-query for an Unknown payment flagged for manual review; it has the same effect as an automatic re-query. | AMB-23 |
| DR-22 | Access is governed by the role and object-ownership rules of Section 9.3. | AMB-03 |

## 7. Payment Lifecycle and State Management

This section reproduces the life cycle of Brief §2.5. The state machine is not changed in any way; the clarifications in Section 7.4 only describe behaviour inside the permitted transitions.

### 7.1 States

| State | Meaning | Money position | Final? |
|---|---|---|---|
| Initiated | Payment created and awaiting checks | No money moved | No |
| FundsHeld | Checks passed; total reserved in the held balance | Total held | No |
| SentToBiller | Request sent to the biller; awaiting the outcome | Total held | No |
| Completed | Biller confirmed | Hold converted to final charge; fee income and any commission recorded | **Yes** |
| Failed | Biller refused, or it is established that it did not receive the payment | Hold awaiting release (see DR-11) | No (leads to Refunded) |
| Unknown | No answer from the biller in time | Total held; no refund | No |
| Rejected | Fraud check failed or funds were insufficient | No money moved | **Yes** |
| Refunded | Hold released back to the wallet after a failure | Total returned to available | **Yes** |

### 7.2 Permitted transitions

| ID | From | Event (trigger) | To | Expected system behaviour |
|---|---|---|---|---|
| T1 | Initiated | Fraud checks pass and the wallet has enough funds | FundsHeld | Move the total (amount plus fee) from available to held; record ledger entries; persist the hold before contacting the biller |
| T2 | Initiated | A fraud check fails or funds are insufficient | Rejected | Record the reason code; no money moves; the biller is not called; the customer receives the reason |
| T3 | FundsHeld | Request sent to the biller | SentToBiller | Send the request with the unique payment reference; start the timeout for live billers (for NetCo, the request is the uploaded batch file) |
| T4 | SentToBiller | Biller confirms | Completed | Turn the hold into a final charge; credit PayLink fee income; credit commission if an agent was involved; notify the customer |
| T5 | SentToBiller | Biller refuses | Failed | Record the translated reason and the biller's code; trigger the hold release (T9) |
| T6 | SentToBiller | No answer within the timeout (10 seconds for live billers, BR-11; for TopUpNet, no callback within the callback wait period, AMB-22) | Unknown | Keep the hold; record the transition at the timeout; return the "being processed" response without waiting for the biller, no later than 12 seconds after the original request (NFR-07); schedule the re-queries (BR-12) |
| T7 | Unknown | Biller later confirms (callback, retry or settlement file) | Completed | Same effects as T4 |
| T8 | Unknown | Biller later refuses, or reconciliation finds no payment (only from a complete settlement file and outside the safety margin, AC-31.3) | Failed | Same effects as T5 |
| T9 | Failed | Hold released to the wallet | Refunded | Move the held total back to available; no fee income and no commission; notify the customer |

No other transition is allowed (AC-11.3).

### 7.3 Constraints that must always hold

1. **Completed, Rejected and Refunded are final.** No event changes them (BR-16).
2. **An Unknown payment is never refunded automatically.** It can reach Refunded only through Failed (T8 then T9), and only on firm evidence (BR-15).
3. **Money is returned only when the system is sure the biller did not receive the payment.** A timeout, a connection error or an unreadable reply is not such evidence.
4. **Every payment is in exactly one valid state at any time**, and every change is recorded in its history with the trigger and time.
5. **Funds are held before the biller is called** (T1 precedes T3).
6. **Each money effect happens once**, even when the same event arrives twice or two events race (AC-11.6).
7. **Customer response is prompt.** The customer response for a timed-out live-biller request is returned within 12 seconds of the original request and never waits for the delayed biller answer (NFR-07, AC-13.9).

### 7.4 Clarifications inside the permitted transitions

- *Failed is momentary.* T5 or T8 is immediately followed by T9 in the same operation, and both transitions appear in the history (DR-11, AMB-25).
- *Non-definitive answers.* A connection error, server error or unreadable reply is handled as "no answer" (T6), because the biller may have received the payment (DR-09, AMB-20).
- *Timeout responsiveness.* The live-biller timeout is 10 seconds. At the timeout the payment moves to Unknown (T6) and the customer is told it is being processed within 12 seconds of the original request. An answer that arrives later on the abandoned request is ignored; the outcome is established only through a re-query, a callback or a settlement file (AC-13.9).
- *TopUpNet.* After the "accepted" reply the payment stays in SentToBiller until the callback arrives (T4 or T5), or until the callback wait period ends (T6).
- *NetCo (optional).* A payment waits in FundsHeld until the daily batch is uploaded (T3); the 10-second timeout does not apply.
- *Rejected* means that no money was ever held. A payment that fails after the hold is Failed, not Rejected.

## 8. Biller Integration Requirements

### 8.1 Comparison of billers

| Biller | Service | Protocol | Integration behaviour |
|---|---|---|---|
| ElectroCo | Electricity | REST/JSON | Immediate confirmation (under 1 second); inquiry by meter number |
| AquaCo | Water | SOAP-style XML over HTTP | 2–4 second response; different field names and different error codes; rejects payments above the bill amount |
| TopUpNet | Mobile top-up | REST/JSON, asynchronous | Replies "accepted", then calls PayLink's callback URL minutes later with "success" or "failed" |
| NetCo | Internet | CSV batch | No live inquiry; PayLink uploads a file of payments once per day and receives a result file the next morning; **optional** |

| Aspect | ElectroCo | AquaCo | TopUpNet | NetCo (optional) |
|---|---|---|---|---|
| Live inquiry | Yes, by meter number | Yes (the bill amount it returns limits the payment) | No bill to inquire; customer enters amount (AMB-18) | No |
| Result arrives | In the same interaction | In the same interaction, after 2–4 seconds | By callback, minutes later | In a result file the next morning |
| 10-second timeout applies | Yes | Yes | To the first reply; then the callback wait period (AMB-22) | No |
| Service fee | Fixed 2.50 | Fixed 2.50 | 1.5%, minimum 1.00 | Tiered 2.00 / 4.00 |

### 8.2 Integration requirements

| ID | Requirement | Related |
|---|---|---|
| BI-01 | PayLink shall use every biller through one common way of making an inquiry (where the biller supports it), a payment and a status re-query, regardless of protocol; all protocol, format and error-code details shall stay inside that biller's integration. | FR-15, NFR-04 |
| BI-02 | Every request to a biller shall carry the unique payment reference so that the biller and PayLink can match records and detect duplicates. | FR-10, FR-26 |
| BI-03 | For live billers, a request without an answer within 10 seconds shall make the payment Unknown, and the customer shall receive the "being processed" response within 12 seconds of the original request. | BR-11, FR-13, NFR-07 |
| BI-04 | Each biller's error codes shall be mapped to PayLink outcomes (confirmed, refused with a clear reason, no answer). A code with no mapping shall be treated as no answer. | DR-09, AMB-20 |
| BI-05 | ElectroCo: PayLink shall support inquiry by meter number and immediate confirmation. | FR-8, FR-15 |
| BI-06 | AquaCo: PayLink shall exchange XML, accept the 2–4 second response time as normal, and never send an amount above the inquired bill amount. | FR-15 |
| BI-07 | TopUpNet: PayLink shall expose a callback endpoint, treat "accepted" as not final, authenticate every callback (AMB-26), process each callback at most once and ignore and log duplicates. | FR-14 |
| BI-08 | NetCo (optional): PayLink shall write one batch file per day, process the result file and never include a payment in two batches. | US-29 |
| BI-09 | A failure of one biller shall not prevent payments to any other biller. | NFR-03 |
| BI-10 | The mock billers shall be configurable to fail on purpose: delay, refuse with an error code, not answer, send a callback twice, and produce settlement files with chosen differences. This is required to test the hard cases. | HC-1 to HC-8 |

### 8.3 Failure scenarios per biller

| Biller | Scenario | Expected outcome |
|---|---|---|
| ElectroCo | Answers after 12 seconds | Unknown at the 10-second timeout; customer told within 12 seconds; money stays held (HC-2, NFR-07) |
| ElectroCo | Refuses the payment with a clear error | Failed, then Refunded |
| ElectroCo | Server error or unreadable reply | Unknown (DR-09) |
| ElectroCo | Complete outage | ElectroCo payments become Unknown; other billers unaffected (NFR-03) |
| AquaCo | Refuses with an error code | Failed, hold released, SMS with a clear reason (HC-3) |
| AquaCo | Refuses because the amount exceeds the bill | Failed; PayLink should prevent this case (BI-06) |
| AquaCo | Slow but valid answer after 3 seconds | Normal processing |
| AquaCo | Error code with no mapping | Unknown (BI-04) |
| TopUpNet | "Accepted", then no callback within the wait period | Unknown, then the re-query rules (AMB-22) |
| TopUpNet | Same "success" callback twice | Completed once; second ignored and logged (HC-6) |
| TopUpNet | Callback "failed" | Failed, then Refunded |
| TopUpNet | Callback with an unknown reference, or contradicting a final state | Ignored, logged and flagged (AMB-26) |
| NetCo | Batch upload fails | Payments stay in FundsHeld and go into the next batch (AMB-36) |
| NetCo | Payment missing from the result file | Stays in SentToBiller and appears in the reconciliation report |

### 8.4 Phase 2 change request: ElectroCo API v2

In Phase 2, ElectroCo announces a new API version (Brief §2.4). This is a **required Phase 2 implementation requirement**, not an out-of-scope item.

| ID | Requirement |
|---|---|
| CR-1 | API field names change; the mapping to PayLink's internal terms shall be in the ElectroCo integration only. |
| CR-2 | Amounts change from pounds to piasters; the conversion shall be exact, at the ElectroCo boundary only, and customers shall still see EGP with two decimals. |
| CR-3 | Every request shall be signed; signing and its secret handling shall be inside the ElectroCo integration and its configuration. |
| CR-4 | Only the ElectroCo integration (with its configuration and tests) may change; no other module may need modification. The report shall list the changed files. |

These requirements are covered by US-30 and NFR-04.

## 9. Non-Functional Requirements

### 9.1 Quality targets

NFR-01 to NFR-06 use the exact quality targets of Brief §3.3. NFR-07 to NFR-10 are **additional, team-defined** targets, justified in Section 9.2. Priorities: the Brief requires at least six measured qualities and the team selected all six expected ones, so they are *Must*. NFR-09 (access control) and NFR-10 (deployability) are also *Must*: PayLink holds customers' money, and the Brief's "Done when" statements make a clean, reproducible deployment mandatory for both phases. NFR-07 and NFR-08 are *Should*.

| ID | Quality attribute | Requirement | Measurement / acceptance target | Origin | Priority |
|---|---|---|---|---|---|
| NFR-01 | Correctness | Prevent duplicate charges and negative wallet balances | Zero double charges and zero negative balances in duplicate and concurrency tests | Brief §3.3 | Must |
| NFR-02 | Performance | Handle payment throughput | 100 payments per second sustained for 1 minute against mock billers | Brief §3.3 | Must |
| NFR-03 | Resilience | Isolate biller failures | An outage of one biller must not stop payments to other billers | Brief §3.3 | Must |
| NFR-04 | Modifiability | Support adding new billers | Adding a biller must only require changes to the Biller Integration module and configuration | Brief §3.3 | Must |
| NFR-05 | Auditability | Track all financial operations | 100% of money movements must have an audit record containing who, when and why | Brief §3.3 | Must |
| NFR-06 | Testability | Ensure domain logic quality | At least 70% line coverage on domain logic, measured in CI | Brief §3.3 | Must |
| NFR-07 | Responsiveness under biller timeout | Tell the customer promptly when a live biller does not answer | The live-biller timeout is 10 seconds (BR-11); at the timeout the payment becomes Unknown (T6). In the 12-second ElectroCo test (HC-2), 100% of 20 runs return the "being processed" response within 12 seconds of the original request (the 10-second timeout plus at most 2 seconds of PayLink processing), and the application never waits for the delayed biller answer | Team-defined | Should |
| NFR-08 | Recoverability | Lose no payment when the application restarts | After a forced restart during 50 in-flight payments, 100% continue to a permitted state, with zero duplicate charges and all scheduled re-queries and notification retries still performed | Team-defined | Should |
| NFR-09 | Access control | Enforce role-based access and data isolation | In the automated access-control suite 100% of cross-customer, wrong-role and unauthenticated attempts are denied, with zero unauthorised successes and zero leaked data; 100% of own-object requests by the correct role succeed; 100% of endpoints declare an access rule (criteria NFR-09.1 to NFR-09.8, Section 9.3) | Team-defined | Must |
| NFR-10 | Deployability | Start the system reproducibly at each delivery stage | **Phase 1:** from a clean clone, `docker compose up` starts the empty application and the database; on every push the pipeline builds, lints and runs at least one test, and is green on the `phase-1` tag. **Phase 2:** `docker compose up` starts the working application, the database and the mock billers and the health check reports healthy; on `main` the pipeline deploys the full stack locally, runs the health check and a rollback step is documented; the pipeline is green on the `final` tag. Both parts are project requirements; they belong to different delivery stages (Section 1.8) | Brief §3.4, §4 ("Done when") | Must |

### 9.2 Notes and justification

- **NFR-01 and NFR-03** depend on the concurrency, idempotency and isolation behaviour in US-05, US-09, US-10 and US-15 (HC-1, HC-4, HC-6).
- **NFR-02** is measured with a load test against the mock billers; the mock billers must therefore answer without artificial delay in that test. The test must use enough funded accounts that the velocity rule (BR-10) and the daily limits (BR-01, BR-02) are not triggered by the test itself.
- **NFR-04** is verified through HC-8 and the ElectroCo v2 change (US-30): the list of changed files must show no module other than Biller Integration and configuration.
- **NFR-05** is verified by the audit and ledger cross-checks of AC-25.5 to AC-25.9: every financial operation has an audit record, every ledger entry links to its operation and audit record, and there are no orphans in either direction. The counts of ledger entries and audit records are deliberately **not** compared one to one, because one financial operation can create several ledger entries.
- **NFR-06** is measured by the pipeline's coverage report limited to the domain logic. Brief §3.3 asks for at least 70% and Brief §4 for coverage above 70%; the Phase 2 pipeline must therefore report above 70%.
- **NFR-07** is justified by BR-11 and HC-2: the customer must not wait indefinitely when a biller is silent. The 10-second timeout comes from the Brief; the 2-second margin up to the 12-second response is a team choice that covers PayLink's own work (checks, hold, ledger writes, response).
- **NFR-08** is justified by Section 2.4: in a money system a restart must not leave a payment stranded or duplicated.
- **NFR-09** is justified by the staff, agent and customer functions (US-02, US-03, US-06, US-17, US-21, US-24, US-27, US-28) and by the risk of unauthorised money operations (AMB-03). Its acceptance criteria and the role matrix are in Section 9.3.
- **NFR-10** is derived from the Brief's Phase 1 and Phase 2 "Done when" statements (Brief §4) and the technical requirements for Docker and CI/CD (Brief §3.4). Its two parts are verified at different stages (Section 12.2).

No other quality attribute is claimed without a measurable target; words such as "fast" or "secure" are deliberately not used as requirements.

### 9.3 Access control (NFR-09)

**Roles.** Customer, agent, support employee, administrator and a read-only finance role (AMB-03). The role and the identity of the caller come from the authenticated credential, never from request parameters. A customer owns exactly one wallet and the payments, ledger entries, top-ups, notifications and refund requests that belong to it (AMB-09).

**Rules.**

1. A customer can read and act only on objects they own; an object owned by another customer is never returned or changed.
2. An agent can perform agent operations (cash top-up for a customer, payment for a customer after the customer's authorisation) and see only the agent's own wallet, commission entries and submitted payments.
3. Support employees, administrators and the finance role can perform only the operations in the matrix below.
4. A request without a valid credential is denied with `AUTHENTICATION_REQUIRED`; a request by an authenticated caller who lacks the role or ownership is denied with `ACCESS_DENIED`. Denial changes nothing and reveals nothing about the object.

**Role and permission matrix.** "Own" means only objects owned by the caller; "Yes" means any object the operation applies to; a dash means denied.

| Operation | Story | Customer | Agent | Support | Administrator | Finance |
|---|---|---|---|---|---|---|
| View wallet, balances and history | US-03 | Own | Own agent wallet | — | — | — |
| Top up own wallet by simulated card | US-04 | Own | Own agent wallet | — | — | — |
| Cash top-up for a customer | US-04 | — | Yes | — | — | — |
| View the biller catalog | US-07 | Yes | Yes | — | — | — |
| Add, edit, disable billers and services | US-06 | — | — | — | Yes | — |
| Bill inquiry and payment | US-08, US-09 | Own wallet | For a customer, with authorisation (US-17) | — | — | — |
| Search a payment by reference, full history | US-20 | — | — | Yes | — | — |
| Block and unblock an account | US-21 | — | — | Yes | — | — |
| Upgrade a verification level | US-02 | — | — | — | Yes | — |
| Edit fees, limits and promotion | US-24 | — | — | — | Yes | — |
| Request a refund of a completed payment | US-27 | Own payments | — | — | — | — |
| View refund requests | US-27 | Own | — | Yes | — | — |
| Approve or reject a refund request | US-27 | — | — | Yes | — | — |
| Request an extra status re-query of a flagged Unknown payment | US-31 | — | — | Yes | — | — |
| Read reconciliation reports | US-26 | — | — | Yes | Yes | Yes (read-only) |
| Upload a settlement file manually | US-26 | — | — | — | Yes | — |
| View daily totals | US-28 | — | — | — | Yes | Yes (read-only) |
| Read audit records | US-25 | — | — | — | Yes (read-only) | Yes (read-only) |

Registration and login are public. The TopUpNet callback endpoint is not role-based; it is protected by the callback authentication of AMB-26 (AC-14.9). Audit records can never be changed through the application (AC-25.3), whatever the role.

**Acceptance criteria and verification.**

| ID | Criterion | Measure | Verification |
|---|---|---|---|
| NFR-09.1 | **Customer-to-customer isolation.** Customer A cannot read or change any wallet, balance, ledger entry, payment, top-up, notification or refund request of customer B | 100% of cross-customer requests are denied with `ACCESS_DENIED`; 0 responses contain data of B; the denial for an object that exists and one that does not is indistinguishable | Automated test: two customers with data; every customer-scoped endpoint called by A with B's identifiers |
| NFR-09.2 | **Own-object access.** Each customer can access their own wallets, transactions and refund requests | 100% of own-object requests succeed (no over-blocking) | Positive test for every customer-scoped endpoint |
| NFR-09.3 | **Agent scope.** An agent can perform only agent operations and see only the agent's own data (AC-03.6, AC-17.9) | 100% of non-agent operations and of customer-data requests by an agent are denied | Negative tests with an agent credential against every endpoint of the matrix |
| NFR-09.4 | **Staff scope.** Support, administrator and finance users can perform only the operations granted in the matrix | 100% of operations outside a role's column are denied; finance and audit access is read-only | Role by operation test generated from the matrix: every cell is asserted |
| NFR-09.5 | **Authentication.** Every protected endpoint rejects missing, invalid and expired credentials | 100% return `AUTHENTICATION_REQUIRED`; no state change | Automated test over all protected endpoints |
| NFR-09.6 | **Coverage.** Every endpoint declares its access rule | 100% of endpoints in the OpenAPI description declare a role and ownership rule; a new endpoint without one fails the pipeline | CI check that enumerates the OpenAPI endpoints |
| NFR-09.7 | **Server-side enforcement.** Role and customer identity come only from the credential | 100% of requests that tamper with a role, customer or wallet identifier in the body or query are denied or ignored | Tamper tests |
| NFR-09.8 | **Traceability of denials.** Denied attempts on staff and agent operations are recorded | 100% of such denials are logged with caller, operation, time and reason code | Log assertion in the negative tests |

## 10. Hard Cases and Exception Handling

The eight hard cases of Brief §2.7 follow. Each must have at least one automated test. Expected outcomes are copied from the Brief; detail is added only where needed to test them.

### HC-1 — The same payment is sent twice

- **Scenario:** The mobile app sends the same payment twice because the network was slow.
- **Expected behaviour:** One payment, one charge; the second request returns the first result.
- **Relevant requirements:** FR-9, FR-10, US-10, NFR-01.
- **State and financial impact:** One payment; one hold; one biller call; the wallet is debited once.
- **Automated test:** Submit the same request (same idempotency key and content) twice in sequence and then concurrently against the mock ElectroCo. Assert one payment, one biller call, one set of ledger entries, and identical responses.

### HC-2 — ElectroCo takes 12 seconds to answer

- **Scenario:** ElectroCo answers after 12 seconds.
- **Expected behaviour:** The payment becomes Unknown; the customer is told it is being processed; the money stays held.
- **Relevant requirements:** FR-12, FR-13, BR-11, BR-15, NFR-07, US-13.
- **State and financial impact:** SentToBiller → Unknown (T6) at the 10-second timeout; the customer receives the "being processed" response within 12 seconds of the original request (NFR-07); held balance unchanged; no refund; re-queries scheduled at 1, 5 and 15 minutes.
- **Automated test:** Configure the mock ElectroCo to delay 12 seconds. Assert that the payment is Unknown at the 10-second timeout, that the response saying the payment is being processed arrives within 12 seconds of the original request without waiting for the mock's delayed answer, that held still equals the total, that no release entry exists, that the mock's late answer on the abandoned request changes nothing, and that the first re-query is scheduled.

### HC-3 — AquaCo refuses with an error code

- **Scenario:** AquaCo refuses the payment with an error code.
- **Expected behaviour:** The payment becomes Failed, the hold is released and the customer gets an SMS with a clear reason.
- **Relevant requirements:** FR-12, FR-15, FR-22, US-12, US-15.
- **State and financial impact:** SentToBiller → Failed (T5) → Refunded (T9); available restored by the full total; no fee income and no commission.
- **Automated test:** Configure the mock AquaCo to refuse. Assert both transitions, restored balances, a notification whose text states a clear reason, and the biller's raw code in the history.

### HC-4 — Two payments at the same moment, balance covers one

- **Scenario:** Two payments from the same wallet arrive together and the balance covers only one.
- **Expected behaviour:** One succeeds, the other is Rejected, and the balance never goes negative.
- **Relevant requirements:** FR-5, FR-9, FR-12, US-05, US-09, NFR-01.
- **State and financial impact:** One payment proceeds from Initiated to FundsHeld; the other goes Initiated → Rejected with `INSUFFICIENT_FUNDS` (T2); no entry makes the balance negative.
- **Automated test:** With a wallet of `400.00` and two payments of total `352.50`, run both in parallel, repeated at least 100 times. Assert exactly one hold and one Rejected per run, available never below `0.00`, and ledger sums consistent.

### HC-5 — The SMS provider is down

- **Scenario:** The SMS provider is down when a payment completes.
- **Expected behaviour:** The payment still completes; the SMS is retried and finally marked undelivered.
- **Relevant requirements:** FR-22, FR-23, BR-13, US-22, US-23.
- **State and financial impact:** The payment is Completed and the ledger is normal; the SMS has four attempts (first plus three retries) and ends as Undelivered; the push message is tracked separately.
- **Automated test:** Make the SMS provider stub fail every call. Assert Completed, four SMS attempts, final status Undelivered, and unaffected push status.

### HC-6 — TopUpNet calls the callback twice

- **Scenario:** TopUpNet sends the "success" callback twice.
- **Expected behaviour:** The payment completes once; the second callback is ignored and logged.
- **Relevant requirements:** FR-14, US-14, NFR-01.
- **State and financial impact:** SentToBiller → Completed once (T4); one set of ledger entries; one notification.
- **Automated test:** Make a payment, let the mock TopUpNet reply "accepted", then deliver the identical "success" callback twice in sequence and concurrently. Assert one transition, one ledger set, one notification and a logged duplicate.

### HC-7 — The settlement file lists a payment PayLink has as Failed

- **Scenario:** The nightly settlement file contains a payment that PayLink recorded as Failed.
- **Expected behaviour:** The difference is flagged in the reconciliation report for support to review.
- **Relevant requirements:** FR-26, US-26, DR-15.
- **State and financial impact:** No automatic change to the payment or the ledger. The report shows `STATUS_MISMATCH`; if the customer was already refunded, PayLink is exposed to a loss that support must resolve.
- **Automated test:** Create a payment that ended Failed then Refunded, process a settlement file that lists its reference, and assert the `STATUS_MISMATCH` entry, no state change and no ledger change.

### HC-8 — A new biller must be added

- **Scenario:** A new biller has to be supported.
- **Expected behaviour:** Only a new integration and configuration are added; existing modules are unchanged.
- **Relevant requirements:** FR-6, FR-15, US-06, US-15, NFR-04.
- **State and financial impact:** None on existing payments or data.
- **Automated test:** Add a stub biller by adding one integration and its configuration only. Run the unchanged test suite and a payment to the new biller. Assert the suite passes and a change check shows no modified file outside the Biller Integration module and configuration.
## 11. Assumptions, Questions, Ambiguities, and Decisions

### 11.1 How to read this section

The questions below were **identified by the AI assistant** (Claude) while analysing the Brief, and then checked by the team. They are **not** questions that real stakeholders or instructors have answered: no instructor or stakeholder has been consulted, and no answer is claimed. For every item, the Question is what was identified, and the Proposed decision is what the team adopts.

The Brief asks teams to list the ambiguities they found and the decisions they made. This SRS records each one in a fixed team format: (1) question or ambiguity, (2) why clarification is needed, (3) whether the Brief answers it, (4) proposed decision, (5) reasoning, (6) impact on implementation or testing, (7) status. Version 1.1 adds (8) a **review note** stating whether the ambiguity is genuinely unresolved, whether the team can decide it, and what changed from version 1.0.

The status is exactly one of the following.

- **Confirmed by Brief:** the Brief explicitly specifies the answer. It is followed as written.
- **Team Decision:** the Brief does not settle the point; the team decided it to fill a gap, with the reasoning recorded. It is **not** an instructor decision.
- **Requires Instructor Confirmation:** a material issue that the team cannot settle from the Brief and should confirm externally. A working default is used until then.

### 11.2 Register of ambiguities and decisions

#### AMB-01 — Basic identity verification at registration

- **Question:** FR-1 says a customer "has a basic verification level". What does basic verification consist of, and how is identity checked?
- **Why clarification is needed:** It decides what registration collects and what tests assert.
- **In the Brief?** No. The Brief defines only the levels *basic* and *verified* and their limits.
- **Proposed decision:** Basic means a full name, a unique Egyptian mobile number and a credential. No identity document is checked and no one-time code is required in this project.
- **Reasoning:** It is the smallest design that gives every customer a level and a unique identity, and the Brief says to avoid real personal data.
- **Impact:** US-01 criteria; registration validation tests; no document-upload feature.
- **Review (v1.1):** Genuinely unresolved: the Brief names the Basic level but not how identity is checked. The team can decide it from the Brief's two levels and its ban on real personal data. Consistent with US-01 and BR-01; no other requirement is affected. No external confirmation needed.
- **Status:** Team Decision

#### AMB-02 — Verification upgrade: who, how, on what evidence

- **Question:** FR-2 says a customer "can be upgraded". Who upgrades, on what evidence, and can a customer be downgraded?
- **Why clarification is needed:** It defines the actor, the audit trail and the permission tests.
- **In the Brief?** No.
- **Proposed decision:** An administrator performs the upgrade after a simulated document check and records a reason. Downgrade is not specified and is not supported.
- **Reasoning:** FR-2 is *Should* and gives no flow; an administrative action is simple and auditable (FR-25).
- **Impact:** US-02; role check; audit record for the upgrade.
- **Review (v1.1):** Genuinely unresolved: FR-2 (Should) says only that a customer "can be upgraded". An administrator-only upgrade is consistent with the Brief's administrator functions (FR-6, FR-24) and with the audit rule (FR-25). Affects US-02 and the permission matrix (Section 9.3). No external confirmation needed.
- **Status:** Team Decision

#### AMB-03 — Authentication mechanism and roles

- **Question:** What authentication is expected, and which roles exist?
- **Why clarification is needed:** Every story with a role restriction and NFR-09 depend on it.
- **In the Brief?** Partly. Section 2.2 says "login" and the Brief names customers, agents, support employees and administrators, but no mechanism.
- **Proposed decision:** Credential-based login that yields a session or token; no multi-factor authentication. Roles: customer, agent, support employee, administrator, and a read-only finance role for reports. Which role may perform which operation, and which objects a role may see, is defined in the permission matrix of Section 9.3.
- **Reasoning:** It is the simplest mechanism that supports role-based access control. The finance role comes from the stakeholder analysis (S-06).
- **Impact:** Role-based denial tests (NFR-09, Section 9.3); every staff story has a "denied" criterion; AC-28.4, AC-20.4 and S-06 aligned with the matrix.
- **Review (v1.1):** Genuinely unresolved, but the team can decide it. v1.1 adds the role and permission matrix (Section 9.3) and aligns AC-28.4 and S-06 with the read-only finance role (v1.0 let only administrators read totals although S-06 lets finance read them).
- **Status:** Team Decision

#### AMB-04 — Simulated card top-up

- **Question:** How is a simulated card top-up represented, and are there top-up limits or a maximum wallet balance?
- **Why clarification is needed:** It defines the top-up test data and whether limits apply.
- **In the Brief?** No. FR-4 only says "simulated card or agent cash".
- **Proposed decision:** A simulated gateway accepts test tokens (no real card numbers or personal data are stored) and returns approved or declined. The Brief sets no top-up limit and no balance cap, so none is enforced.
- **Reasoning:** The Brief forbids real personal data (§5) and sets no top-up rule; inventing a limit would add an undocumented business rule.
- **Impact:** US-04; a stub gateway for tests; `CARD_DECLINED` path.
- **Review (v1.1):** Genuinely unresolved, low risk. The Brief forbids real personal data and sets no top-up rule, so adding a limit would invent a business rule. Agents can fund their own wallet with the same simulated card (AMB-05). No external confirmation needed.
- **Status:** Team Decision

#### AMB-05 — Funding of an agent cash top-up

- **Question:** When an agent takes cash and tops up a customer, where does the money come from in PayLink's books?
- **Why clarification is needed:** A credit with no matching debit breaks the conservation of money (US-05).
- **In the Brief?** No. FR-4 mentions "agent cash" only.
- **Proposed decision:** The customer's wallet is credited and the agent's wallet is debited by the same amount in one financial operation, so the agent works with an electronic float and keeps the cash. The agent's identity is recorded. An agent funds their own wallet with the same simulated-card top-up as a customer (AC-04.9) or through test data. The agent's available balance never goes below 0.00.
- **Reasoning:** It keeps every movement balanced (AC-05.4) using only entities the Brief names: the agent already has a wallet because commission is credited to it. The alternative (cash handled outside the system with a PayLink-owned account funding the credit) is equally possible; keeping the counter-account in the ledger design makes a later change local.
- **Impact:** AC-04.3, AC-04.4, AC-04.9; agent wallets need a balance in test data; the cash top-up is audited as one financial operation (US-25).
- **Review (v1.1):** Changed from Requires Stakeholder Clarification. The Brief is silent, but the answer follows from FR-5 (every balance change is a ledger entry) and BR-09 (agents have wallets): debiting the agent keeps the ledger balanced without inventing a PayLink cash account the Brief never mentions. v1.1 also records how an agent funds the wallet (AC-04.9), a gap in v1.0. The risk of a wrong choice is limited to the counter-account of one operation and does not touch the payment state machine. If the instructors prefer another model, only AC-04.4 and `INSUFFICIENT_AGENT_FUNDS` change.
- **Status:** Team Decision

#### AMB-06 — Agent payments: identification, authorisation and whose funds

- **Question:** FR-17 says an agent pays "on behalf of a customer". How is the customer identified at the terminal, how does the customer authorise, and is the payment taken from the customer's wallet, the agent's wallet or cash?
- **Why clarification is needed:** It decides whose limits, balance and fraud checks apply and who is liable.
- **In the Brief?** Partly. FR-17 says an agent pays "on behalf of a customer", and Section 2.1 and §2.6 say the agent earns commission on "a payment made at the agent's terminal". The Brief does not say whose funds are used or how the customer authorises.
- **Proposed decision:** The agent is authenticated and linked to a terminal; the customer is identified by registered mobile number and authorises the payment with a single-use one-time code sent to that mobile; the payment is taken from the customer's wallet, uses the customer's limits and counters, and records the agent and terminal.
- **Reasoning:** "On behalf of a customer" makes the customer the payer. A one-time code is the simplest testable proof of authorisation that reuses the Notifications module, and the same payment path and fraud rules apply; commission is defined against the fee of a completed payment.
- **Impact:** US-17 (AC-17.1, AC-17.8, AC-17.9); the payment record carries agent and terminal; velocity counts the customer account (AC-19.6); new reason code `AGENT_AUTHORISATION_FAILED`; one more notification type.
- **Review (v1.1):** Changed from Requires Stakeholder Clarification. "On behalf of a customer" (FR-17) and "a payment made at the agent's terminal" (§2.6) indicate that the payment belongs to the customer, so the customer's wallet and limits apply. What the Brief does not give is a way to prove the customer's authorisation; "has authorised" in v1.0 was not testable. v1.1 defines a one-time code sent to the customer's registered mobile (AC-17.8, DR-20). Affects US-17, US-19 (velocity counts the customer account), the reason codes and NFR-09.
- **Status:** Team Decision

#### AMB-07 — Do limits apply to the bill amount or the total?

- **Question:** Do the minimum, maximum and daily limits apply to the bill amount or to amount plus fee?
- **Why clarification is needed:** It changes which payments are accepted at the edges.
- **In the Brief?** No. The Brief says "payment" and "single payment".
- **Proposed decision:** Limits apply to the bill amount. The fee is excluded.
- **Reasoning:** The bill amount is what is paid to the biller; the fee is a PayLink charge on top of it.
- **Impact:** AC-18.1 to AC-18.4 boundary tests.
- **Review (v1.1):** Genuinely unresolved. The fee is a PayLink charge, and the Brief's limits speak of "payment" and "single payment". Consistent with BR-17 (hold = amount plus fee) and with AC-18.x boundaries. No external confirmation needed.
- **Status:** Team Decision

#### AMB-08 — How the daily limit is calculated

- **Question:** Which payment states count towards the daily limit, and what is a "day"?
- **Why clarification is needed:** Failed, Unknown and Rejected payments could otherwise block a customer wrongly or allow overspending.
- **In the Brief?** No.
- **Proposed decision:** Payments in FundsHeld, SentToBiller, Unknown and Completed count; Rejected, Failed and Refunded do not. A day is a calendar day in Africa/Cairo (configurable). A Completed payment that later receives an approved refund (US-27) still counts, because it stays Completed.
- **Reasoning:** Counted states are those where money is, or may be, committed to the biller. Egypt is the operating country.
- **Impact:** AC-18.5, AC-18.8, AC-27.11; test data for each state; day-boundary tests.
- **Review (v1.1):** Genuinely unresolved. Counted states are those where money is or may be committed to the biller. v1.1 adds the refund interaction: a Completed payment that later receives an approved refund still counts, because it stays Completed (DR-17, AC-27.11).
- **Status:** Team Decision

#### AMB-09 — Payment reference and wallet cardinality

- **Question:** What is the payment reference, and does a customer or agent have one wallet?
- **Why clarification is needed:** The reference is the key for idempotency, billers and settlement files.
- **In the Brief?** Partly. "A unique reference" is sent to the biller (§2.2) but its format is not given; one wallet per customer is implied but not stated.
- **Proposed decision:** PayLink generates a unique, opaque reference per payment. Each customer and each agent has exactly one EGP wallet.
- **Reasoning:** One wallet is sufficient for every requirement and avoids multi-currency or multi-wallet rules.
- **Impact:** US-09 (AC-09.6); matching in US-26; wallet model.
- **Review (v1.1):** Genuinely unresolved; one wallet per customer is the smallest model and is used consistently by US-03, NFR-09 and the refund rules. No external confirmation needed.
- **Status:** Team Decision

#### AMB-10 — Is the daily limit inclusive?

- **Question:** Is a payment that brings the day's total to exactly 5,000.00 allowed?
- **Why clarification is needed:** Boundary tests need an exact answer.
- **In the Brief?** Yes: Section 2.2 says basic accounts "may pay at most 5,000 EGP per day".
- **Proposed decision:** A total of exactly the limit is allowed; one cent above is rejected. The same reading applies to the Verified limit and to the single-payment range (5.00 to 10,000.00 inclusive).
- **Reasoning:** "At most" is inclusive.
- **Impact:** AC-18.1 to AC-18.4.
- **Review (v1.1):** The Brief answers the question: Section 2.2 says basic accounts "may pay at most 5,000 EGP per day", so exactly 5,000.00 is allowed. Reading the Verified limit and the single-payment range (5.00 to 10,000.00) as inclusive too is a team inference by analogy, not explicit Brief text.
- **Status:** Confirmed by Brief

#### AMB-11 — What the velocity rule counts

- **Question:** Does the velocity rule count only accepted payments or all attempts, which payment triggers the block, and are replays counted?
- **Why clarification is needed:** It decides when exactly an account is blocked.
- **In the Brief?** Partly. Rule: "more than 5 payments in 60 seconds … is blocked for 10 minutes". Section 2.2 mentions "number of payments in the last minute".
- **Proposed decision:** Every payment request that reaches the fraud checks counts, excluding idempotent replays, over a sliding 60-second window. The sixth request is rejected with `VELOCITY_EXCEEDED` and starts a 10-minute block; a sixth request that breaks an earlier rule is rejected with that rule's reason but still starts the block. Agent and customer payments for the same account count together.
- **Reasoning:** Counting requests catches rapid-fire abuse even when payments are rejected; excluding replays avoids punishing network retries.
- **Impact:** AC-19.1 to AC-19.8; time-window tests with a controllable clock.
- **Review (v1.1):** Genuinely unresolved ("payments" versus requests). v1.1 closes an edge case: a sixth request that also breaks an earlier rule still counts and still starts the block (AC-19.8). Consistent with DR-04 and DR-05.
- **Status:** Team Decision

#### AMB-12 — Order of fraud rules and what "blocked" means

- **Question:** In which order are the fraud rules applied, what can a blocked account still do, and how do manual blocks and velocity blocks differ?
- **Why clarification is needed:** A payment can break several rules at once, and the reason shown must be predictable.
- **In the Brief?** Partly. Section 2.2 lists the checks and §3.1 describes "a chain of rules", but gives no order.
- **Proposed decision:** Order: blocked account, single-payment limits, daily limit, velocity; the first failing rule decides the reason. A blocked account (manual or velocity) cannot start payments; it can still log in and view its wallet. Both block types return `ACCOUNT_BLOCKED`; a velocity block expires after 10 minutes and a manual block ends only by unblocking. In-flight payments continue.
- **Reasoning:** Cheap status checks first, then amount rules, then rate rules; blocking only new payments protects customers' existing money.
- **Impact:** AC-18.6, AC-19.2, AC-21.1; chain-ordering tests.
- **Review (v1.1):** Genuinely unresolved (the Brief lists the checks but no order). The funds check follows the fraud chain (T1, T2), which is consistent with DR-04 and AC-09.2. No external confirmation needed.
- **Status:** Team Decision

#### AMB-13 — Idempotency: scope, retention and mismatching requests

- **Question:** Is the key mandatory, what is its scope, how long is it kept, and what if the same key arrives with different content?
- **Why clarification is needed:** Duplicate protection fails without precise key rules (NFR-01).
- **In the Brief?** Partly. FR-10 and HC-1 say a repeated key returns the original result.
- **Proposed decision:** The key is mandatory for payments; it is scoped to the authenticated user who submits the request (the customer, or the agent for agent-submitted requests); it is kept at least 24 hours (configurable); the same key with different content is rejected with `IDEMPOTENCY_KEY_REUSED`; a replay of a rejected payment returns the same rejection. The same mechanism is applied to top-ups.
- **Reasoning:** It satisfies FR-10 exactly and closes obvious loopholes. The Brief names payments only; extending it to top-ups protects money movements consistently.
- **Impact:** US-10, AC-04.6, AC-09.3; tests for replay, concurrency and mismatch.
- **Review (v1.1):** Genuinely unresolved beyond "a repeated key returns the original result". v1.1 fixes the key scope: "scoped to the customer account" in v1.0 was ambiguous for agent-submitted requests, so the scope is now the authenticated user who submits the request (DR-08).
- **Status:** Team Decision

#### AMB-14 — Rounding of fees and commissions

- **Question:** How are fees and commissions rounded, and when is the 1.00 EGP minimum applied?
- **Why clarification is needed:** A 1.5% fee often has more than two decimals; totals must agree to the piaster.
- **In the Brief?** No. Only "exact decimal types, never floating point" (BR-14).
- **Proposed decision:** Half-up rounding to two decimals; for mobile top-ups the minimum of 1.00 is applied after rounding; commission is 40% of the stored fee, rounded half-up; PayLink keeps the remainder of the fee. Example: fee `1.07` gives commission `0.43`.
- **Reasoning:** Half-up is the common commercial convention and is deterministic.
- **Impact:** AC-16.2, AC-17.6; boundary tests such as `33.33`.
- **Review (v1.1):** Genuinely unresolved (the Brief requires exact decimals only). Checked with AC-16.2 (33.33 gives 1.00) and AC-17.6 (fee 1.07 gives commission 0.43). Consistent with BR-14. No external confirmation needed.
- **Status:** Team Decision

#### AMB-15 — Internet fee tier boundary

- **Question:** Which fee applies to exactly 200.00 EGP?
- **Why clarification is needed:** The tier boundary needs one answer.
- **In the Brief?** Yes: "2 EGP up to 200 EGP, 4 EGP above".
- **Proposed decision:** `200.00` costs 2.00; `200.01` costs 4.00.
- **Reasoning:** "Up to" includes the boundary.
- **Impact:** AC-16.3.
- **Review (v1.1):** The Brief answers the question: "2 EGP up to 200 EGP, 4 EGP above", so exactly 200.00 costs 2.00.
- **Status:** Confirmed by Brief

#### AMB-16 — Promotion rules and overlaps

- **Question:** What happens when a promotion overlaps another promotion or another fee rule, are the start and end dates inclusive, and what is the commission when the fee is zero?
- **Why clarification is needed:** Without it the same payment could get two different fees.
- **In the Brief?** Partly. "During a promotion period the fee is 0 for one chosen biller (configurable start and end dates)". Commission is 40% of the fee (BR-09).
- **Proposed decision:** At most one promotion exists at a time, for one biller; saving an overlapping promotion is rejected. During the period the promotion overrides the normal fee for that biller only. Start and end dates are inclusive whole days in Africa/Cairo. Commission on a zero fee is `0.00`.
- **Reasoning:** "One chosen biller" implies a single active promotion; commission follows arithmetically from BR-09.
- **Impact:** AC-16.4, AC-16.5, AC-17.3; configuration validation (AC-24.2).
- **Review (v1.1):** Genuinely unresolved (the Brief says "one chosen biller" but not overlaps or date inclusivity). Consistent with BR-08, BR-09 and AC-24.2. No external confirmation needed.
- **Status:** Team Decision

#### AMB-17 — When is the fee fixed?

- **Question:** Is the fee calculated at inquiry or at payment, and what if the fee rules change in between?
- **Why clarification is needed:** The customer must not pay a total different from the one shown without being told.
- **In the Brief?** No. Section 2.2 shows the fee at inquiry and adds it to the hold at payment.
- **Proposed decision:** The inquiry shows the fee under the current rules. The fee is calculated again when the payment is initiated, stored with the payment, and returned in the payment response. A configuration change affects only payments initiated afterwards.
- **Reasoning:** It keeps one source of truth per payment. No quote-validity rule is invented.
- **Impact:** AC-24.3; the payment record stores fee and total; tests that change configuration between inquiry and payment.
- **Review (v1.1):** Genuinely unresolved. The decision keeps one source of truth per payment and is consistent with AC-24.3; the fee is stored with the payment (DR-07). No quote-validity rule is invented.
- **Status:** Team Decision

#### AMB-18 — Inquiry for billers without a live inquiry, and partial payments

- **Question:** TopUpNet and NetCo have no bill to inquire about. Also, may a customer pay less than the full inquired bill?
- **Why clarification is needed:** FR-8 requires "amount, fee and total" for every payment.
- **In the Brief?** Partly. The contract gives inquiry for ElectroCo; NetCo has "no live inquiry"; AquaCo "rejects payments above the bill amount".
- **Proposed decision:** For TopUpNet and NetCo the customer enters the amount and PayLink shows amount, fee and total. For billers with inquiry, the amount paid equals the inquired bill amount; partial payments are not specified and are not supported.
- **Reasoning:** It is the smallest behaviour consistent with FR-8 and the contracts.
- **Impact:** AC-08.3, AC-09.4; mock biller bills.
- **Review (v1.1):** Genuinely unresolved. Smallest behaviour consistent with FR-8 and the biller contracts; affects AC-08.3 and AC-09.4. No external confirmation needed.
- **Status:** Team Decision
#### AMB-19 — Global amount limits versus biller-specific amount limits

- **Question:** The catalog stores minimum and maximum amounts per service, and the business rules set global limits. Which applies?
- **Why clarification is needed:** The two could conflict.
- **In the Brief?** Partly. Both exist (§3.1, §2.6) but their relationship is not stated.
- **Proposed decision:** The effective range is the overlap of the two. A violation of the global range returns `AMOUNT_BELOW_MINIMUM` or `AMOUNT_ABOVE_MAXIMUM`; the same codes are used for the service range.
- **Reasoning:** Both are real constraints, and the stricter must win.
- **Impact:** AC-06.5, AC-08.7, AC-18.1/18.2; catalog validation.
- **Review (v1.1):** Genuinely unresolved (two sets of limits). The stricter range wins; consistent with DR-12. No external confirmation needed.
- **Status:** Team Decision

#### AMB-20 — Biller answers that are neither confirmation nor refusal

- **Question:** How is a connection error, server error, unreadable reply or unmapped error code treated? The state table has no transition from FundsHeld to Failed.
- **Why clarification is needed:** Treating "maybe received" as "not received" would refund wrongly (BR-15).
- **In the Brief?** Partly. "No answer within the timeout" gives Unknown; a refusal gives Failed.
- **Proposed decision:** Only a definite refusal, using a documented refusal code of the biller, makes the payment Failed. Everything else is handled as no answer: the payment becomes Unknown and follows the re-query rules.
- **Reasoning:** It follows the Brief's principle that money returns only when PayLink is sure.
- **Impact:** AC-13.7, AC-15.7, BI-04; mock billers must produce each error type.
- **Review (v1.1):** Genuinely unresolved in detail, but follows directly from BR-15: only a definite refusal returns money. Consistent with T6, AC-13.7 and AC-15.7. No external confirmation needed.
- **Status:** Team Decision

#### AMB-21 — Retry timing and what a re-query is

- **Question:** Are the 1, 5 and 15 minutes measured from the moment the payment became Unknown or from the previous retry, and what does a re-query do for each biller?
- **Why clarification is needed:** The schedule must be testable.
- **In the Brief?** Partly. "Re-queried up to 3 times (after 1, 5 and 15 minutes)". The mock contracts are not given in detail.
- **Proposed decision:** The three times are offsets from the moment the payment became Unknown. A re-query asks the biller for the status of the payment reference; the mock billers will offer this operation. A re-query that times out still counts.
- **Reasoning:** Fixed offsets are simplest to schedule and test.
- **Impact:** AC-13.2, AC-13.5; a status operation in each live mock biller; clock-controlled tests.
- **Review (v1.1):** Genuinely unresolved (offsets versus intervals). Fixed offsets from the moment the payment became Unknown are testable and consistent with BR-12 and DR-10. The mock billers need a status operation. No external confirmation needed.
- **Status:** Team Decision

#### AMB-22 — TopUpNet callback that never arrives

- **Question:** After "accepted", how long does PayLink wait for the callback before treating the payment as Unknown?
- **Why clarification is needed:** The Brief gives a 10-second timeout for live billers and "minutes" for the callback but nothing for a missing callback.
- **In the Brief?** No.
- **Proposed decision:** A configurable callback wait period (proposal: 30 minutes). When it ends the payment becomes Unknown (T6) and follows BR-12; the mock TopUpNet offers a status operation for the re-queries (AMB-21).
- **Reasoning:** Without it, a payment could stay in SentToBiller forever. The value is a team proposal, not a Brief value, and is kept in configuration.
- **Impact:** AC-14.8; configuration value; a mock that never calls back; T6 wording in Section 7.2.
- **Review (v1.1):** Changed from Requires Stakeholder Clarification. The Brief says only that the callback arrives "minutes later". The wait period is a configuration value with a safe failure mode: when it ends the payment becomes Unknown, which never triggers a refund, and a late callback still completes the payment (T7). A badly chosen value therefore cannot lose money. It uses the existing transition T6, so the state machine is unchanged.
- **Status:** Team Decision

#### AMB-23 — Unknown after all retries and after reconciliation

- **Question:** What happens to a payment that is still Unknown after the three retries, and after reconciliation if the settlement file gives no answer?
- **Why clarification is needed:** Customer money stays held; the end of that situation must be defined.
- **In the Brief?** Partly. After the retries it "waits for reconciliation"; reconciliation that "finds no payment" leads to Failed. Nothing covers the case where reconciliation cannot decide.
- **Proposed decision:** It stays Unknown and held and is never refunded automatically. Reconciliation "cannot decide" when the settlement file for the relevant day is missing, malformed or incomplete, or when the payment was sent inside the safety margin (AMB-27). Such a payment is listed in the reconciliation report as requiring manual review. Support may request an additional status re-query (AC-31.8) and the answer is applied by the normal rules (T7 or T8). There is no automatic time-based refund and no manual state change that is not backed by biller evidence.
- **Reasoning:** It obeys BR-15 and uses only the evidence channels of the Brief for T7 and T8 (callback, re-query, settlement file). A time limit would risk refunding paid bills.
- **Impact:** AC-13.4, AC-31.4, AC-31.7, AC-31.8; DR-21; no automatic expiry job.
- **Review (v1.1):** Changed from Requires Stakeholder Clarification. The rule follows BR-15 and the Brief's own T8 trigger ("reconciliation finds no payment"): nothing is refunded without evidence. The real gap in v1.0 was a dead end, a payment "listed for manual review" with no way to work it. v1.1 defines when reconciliation cannot decide and lets support request an additional status re-query (AC-31.8, DR-21), using only the evidence channels the Brief lists.
- **Status:** Team Decision

#### AMB-24 — Biller confirms after the timeout but before reconciliation

- **Question:** What if the biller confirms a payment later than the timeout and before the nightly reconciliation?
- **Why clarification is needed:** Customers could be refunded for a paid bill if this is mishandled.
- **In the Brief?** Yes. Transition Unknown → Completed "Biller later confirms (callback, retry or settlement file)".
- **Proposed decision:** The late confirmation completes the payment once (T7) with the same effects as T4, including the notification and any commission.
- **Reasoning:** It is the Brief's rule.
- **Impact:** AC-31.1, AC-31.5.
- **Review (v1.1):** The Brief answers the question: transition Unknown to Completed on "Biller later confirms (callback, retry or settlement file)". Implemented as T7 and AC-31.1, AC-31.5.
- **Status:** Confirmed by Brief

#### AMB-25 — Failed versus Refunded, and what "has as Failed" means

- **Question:** The table has Failed → Refunded when the hold is released. Is Failed observable, and in HC-7 what is a payment "PayLink has as Failed" if it is refunded straight away?
- **Why clarification is needed:** Reconciliation compares against PayLink's recorded outcome.
- **In the Brief?** Partly. HC-3 says the payment "becomes Failed" and "the hold is released"; HC-7 refers to a payment PayLink "has as Failed".
- **Proposed decision:** T9 follows T5 or T8 in the same operation; both transitions are stored. For reconciliation, any payment whose recorded outcome is failure (state Failed or Refunded) counts as "Failed". A mismatch is only flagged; no automatic correction is made.
- **Reasoning:** It keeps the state machine unchanged and makes HC-7 testable.
- **Impact:** AC-12.2, AC-26.2; history shows both transitions.
- **Review (v1.1):** Genuinely unresolved ("has as Failed"). Keeps the state machine unchanged. v1.1 adds AC-26.10 for listed payments in other states (Initiated, FundsHeld, SentToBiller, Rejected), which v1.0 did not cover.
- **Status:** Team Decision

#### AMB-26 — Callback authenticity, conflicting callbacks and late events

- **Question:** How is a callback authenticated, and what if a callback or confirmation conflicts with a final state (for example "failed" after Completed, or a confirmation after Failed)?
- **Why clarification is needed:** A forged or contradictory event must not move money.
- **In the Brief?** Partly. Duplicates are ignored (FR-14); nothing is said about authenticity or contradiction.
- **Proposed decision:** A callback is authenticated (for example with a per-biller shared secret or signature configured in the integration) and matched by payment reference. A callback that fails authentication is rejected, logged and has no effect. An unknown reference has no effect and is logged. A callback or confirmation that contradicts a final or already refunded state is ignored, logged and flagged for support.
- **Reasoning:** The state machine forbids such transitions, and financial anomalies need a human.
- **Impact:** AC-14.5, AC-14.6, AC-14.9, AC-31.6; DR-18; BI-07; the mock TopUpNet must sign callbacks and be able to send a contradictory callback.
- **Review (v1.1):** Genuinely unresolved, and v1.0 left a financial gap: it called callback authentication "desirable but not specified", which would let a forged callback complete a payment or fail it and trigger a refund. v1.1 makes authentication a requirement (AC-14.9, DR-18, BI-07); the mechanism remains a design choice inside the TopUpNet integration.
- **Status:** Team Decision

#### AMB-27 — Settlement file format, matching and discrepancy types

- **Question:** What is the settlement file format, how are records matched, what is the cut-off time, and which differences are reported?
- **Why clarification is needed:** Reconciliation cannot be built or tested without a defined file.
- **In the Brief?** No. It says only "the biller's daily list of payments it received" and that reconciliation "flags differences".
- **Proposed decision:** One CSV file per biller and business day, named `<biller>_<YYYY-MM-DD>.csv`. A header row names the columns `payment_reference`, `amount_egp`, `status` (RECEIVED) and `received_at`; the last line is a trailer giving the number of data rows so that an incomplete file can be detected. Records are matched by payment reference. The business day is the Africa/Cairo calendar day of `received_at`; the cut-off is its midnight boundary. A payment sent within a safety margin before the cut-off (configurable, proposal 15 minutes) is compared with the next day's file. An incomplete or malformed file is never used to conclude that a payment was not received. Four difference categories: `STATUS_MISMATCH` (also used for a listed payment PayLink holds in Initiated, FundsHeld, SentToBiller or Rejected), `MISSING_IN_SETTLEMENT`, `UNKNOWN_IN_PAYLINK`, `AMOUNT_MISMATCH`. The mock billers produce files in this format; for ElectroCo v2 the file format is assumed unchanged.
- **Reasoning:** The simplest format that makes HC-7 and the resolution of Unknown payments testable, with the protections needed so that a payment the biller received is never refunded (BR-15).
- **Impact:** US-26 (AC-26.1 to AC-26.10), US-31 (AC-31.3), DR-19; a file parser per biller; a mock settlement generator with day-boundary cases. If the instructors supply their own mock billers or files, only the parser changes.
- **Review (v1.1):** Changed from Requires Stakeholder Clarification. The Brief says that the instructors give the biller contracts and that teams may write the mock servers (§2.4); the settlement file is part of those mocks, so its format is the team's to define. v1.1 also fixes a financial risk in v1.0: with a plain midnight cut-off, a payment received by the biller just after midnight would be missing from day D's file and an Unknown payment could be refunded although the biller received it (BR-15). v1.1 adds a record-count trailer, a safety margin before the cut-off, and the rule that an incomplete file never proves absence (DR-19).
- **Status:** Team Decision

#### AMB-28 — Refund of a completed payment

- **Question:** FR-27 lets a customer request a refund of a *completed* payment, but Completed is final and Refunded is reachable only from Failed. How is such a refund approved and processed, for what amount, and who funds it?
- **Why clarification is needed:** It conflicts with the state machine that must not change, and it affects money.
- **In the Brief?** Partly. FR-27: "A customer can request a refund of a completed payment; support approves it" (*Could*). The glossary defines a refund as returning money "after a failed or cancelled payment".
- **Proposed decision:** The payment stays Completed. A separate refund record (Pending, Approved, Rejected) is created and support approves or rejects it. Until the instructors answer, the design uses the working defaults of RF-1 to RF-5 (US-27): the customer receives the whole amount charged; PayLink funds the refund first and books the amount as recoverable from the biller; PayLink's fee income is reversed; the agent's commission is reversed without taking the agent wallet below zero; money that cannot be recovered is a PayLink loss. These defaults are **not** a confirmed policy and are configuration. Only a Completed payment with no Pending or Approved request is eligible; partial refunds are not supported.
- **Reasoning:** It honours FR-27 without changing the payment state machine (FR-11) and keeps the ledger balanced (AC-27.8). Each open financial decision is documented instead of being silently decided.
- **Impact:** US-27 (AC-27.1 to AC-27.14); a refund record; ledger accounts for amounts recoverable from the biller, refund losses and amounts owed by an agent (AMB-32); US-28 totals net of reversals (AC-28.3); DR-14, DR-17. FR-27 is Could, so the risk is limited.
- **Review (v1.1):** Remains Requires Instructor Confirmation. Genuinely unresolved: the Brief defines a refund only after a failed or cancelled payment (glossary) and gives no amount, funding or loss policy for a completed payment. The choices decide who loses money and which ledger entries exist, and cannot be derived from the Brief. The state-machine question is resolved by the team (the payment stays Completed). The daily-limit interaction is a team decision (DR-17). The open decisions are RF-1 to RF-5 in US-27.
- **Status:** Requires Instructor Confirmation

#### AMB-29 — Notification channels, events, retries and templates

- **Question:** Which channels and events are required, how many attempts are made, what is the retry interval, and what is sent for Unknown or Rejected payments?
- **Why clarification is needed:** FR-22 names SMS and push; the module and the retry rule also mention email.
- **In the Brief?** Partly. FR-22: SMS and push on completion and failure. BR-13: SMS or email retried 3 times, then undelivered. FR-23: "failed notifications are retried 3 times".
- **Proposed decision:** SMS and push are required; email is optional. All channels use one first attempt plus three retries, tracked per channel. The interval is configuration. No message is sent for Unknown; a message follows when it resolves. A rejected payment is explained in the response; no SMS is required. Texts come from templates; language is not specified.
- **Reasoning:** "Retried 3 times" is read as three retries after the first attempt. Applying it to all channels is the conservative reading.
- **Impact:** US-22, US-23; HC-5 expects four SMS attempts.
- **Review (v1.1):** Genuinely unresolved. "Retried 3 times" is read as three retries after the first attempt (four attempts); the retry count is configuration, so a different reading from the instructors would change one value and the expected count in HC-5. Not material enough to block the design.
- **Status:** Team Decision

#### AMB-30 — Audit log content, immutability and coverage

- **Question:** What exactly do "who, when and why" mean, and is the log protected from change?
- **Why clarification is needed:** NFR-05 requires 100% coverage.
- **In the Brief?** Partly. "Every money movement and admin action is written to an audit log" with "who, when and why".
- **Proposed decision:** Who is a user, an agent or a named system component. When is the timestamp. Why is a reason or triggering event with the payment reference; administrators must enter a reason. The audit grain is the financial operation: each financial operation has exactly one audit record, each ledger entry carries the ID of its operation and the reference of its audit record, and ledger and audit counts are not required to match one to one. Records are append-only through the application and are committed in the same transaction as the operation.
- **Reasoning:** It makes the three words testable. One operation can create several ledger entries, so a count comparison would wrongly fail; orphan checks in both directions are the correct test.
- **Impact:** US-25 (AC-25.5 to AC-25.9), NFR-05; audit written in the same transaction as the movement.
- **Review (v1.1):** Genuinely unresolved. v1.1 corrects the audit grain: v1.0 expected the number of ledger entries to equal the number of audit records, which is wrong because one financial operation creates several entries (AC-05.4). The audit record now belongs to the financial operation and each ledger entry links to it (AC-25.5 to AC-25.9).
- **Status:** Team Decision

#### AMB-31 — Scope of runtime configuration

- **Question:** Which values are editable without redeployment, and what happens to in-flight payments?
- **Why clarification is needed:** FR-24 names "fee and limit values" only.
- **In the Brief?** Partly. The Brief says to put all fixed numbers in configuration.
- **Proposed decision:** Fees, single and daily limits, and the promotion are editable at runtime with validation and audit; other values live in configuration but need not be editable at runtime. Changes affect payments initiated after the change.
- **Reasoning:** It meets FR-24 exactly and does not over-promise.
- **Impact:** US-24; validation rules for configuration.
- **Review (v1.1):** Genuinely unresolved beyond FR-24. Meets FR-24 exactly and does not over-promise; consistent with AC-24.3 and DR-07. No external confirmation needed.
- **Status:** Team Decision

#### AMB-32 — Ledger accounting model

- **Question:** Besides customer wallets, which accounts does the ledger need (for example PayLink fee income and amounts owed to billers), and how are commissions booked?
- **Why clarification is needed:** "Credits its own fee income" and "records the commission" (§2.2) imply PayLink accounts, which the Brief does not define.
- **In the Brief?** Partly.
- **Proposed decision:** Besides wallets there are PayLink accounts for fee income and for the amounts owed to each biller. At completion the customer's held balance is reduced by the total; the biller account receives the amount; fee income receives the fee. A commission is a separate entry crediting the agent wallet and reducing fee income. The entries of every payment net to zero. Refunds of completed payments (US-27) may also use accounts for amounts recoverable from the biller, refund losses and amounts owed by an agent after a commission reversal.
- **Reasoning:** It makes balances explainable and gives a testable conservation rule (AC-05.4).
- **Impact:** US-05, US-28 (totals); the exact chart of accounts belongs to the design.
- **Review (v1.1):** Genuinely unresolved ("fee income" and "commission" imply PayLink accounts). v1.1 notes that refunds of completed payments need three more accounts (recoverable from biller, refund loss, owed by agent) so that AC-27.8 can hold.
- **Status:** Team Decision

#### AMB-33 — Disabled billers and payments in progress

- **Question:** What happens to in-flight payments when a biller is disabled?
- **Why clarification is needed:** Interrupting them would strand held money.
- **In the Brief?** No. FR-6 says only that billers can be disabled.
- **Proposed decision:** Disabling blocks new inquiries and payments only; payments already in progress run to their outcome.
- **Reasoning:** It protects customer money.
- **Impact:** AC-06.3, AC-07.4.
- **Review (v1.1):** Genuinely unresolved. Protects customer money; consistent with DR-16 and AC-06.3. No external confirmation needed.
- **Status:** Team Decision

#### AMB-34 — Cancelled payments

- **Question:** The glossary mentions refunds after a "cancelled" payment. Can a customer cancel a payment?
- **Why clarification is needed:** The state machine has no cancellation transition.
- **In the Brief?** No. No cancellation event exists in §2.5.
- **Proposed decision:** Customer cancellation is not supported. Money returns only through Failed → Refunded.
- **Reasoning:** Adding it would require a state-machine change, which is forbidden.
- **Impact:** Out of scope (Section 13.4); no cancel operation in the API.
- **Review (v1.1):** Genuinely unresolved, but strongly constrained: FR-11 allows no transition other than those of Section 2.5, and none is a cancellation, so cancellation cannot exist without changing the state machine. Recorded as a team decision because the Brief does not say "no cancellation" in words.
- **Status:** Team Decision

#### AMB-35 — Priority of FR-15 and the optional NetCo biller

- **Question:** FR-15 is *Must* but says NetCo is optional. How is its priority recorded?
- **Why clarification is needed:** A Must item must not be downgraded, yet part of it is optional.
- **In the Brief?** Yes. "At least 3 billers … (ElectroCo, AquaCo, TopUpNet); NetCo batch is optional."
- **Proposed decision:** FR-15 stays *Must* for the three named billers (US-15). The optional NetCo part is a separate story, US-29, with priority *Could*.
- **Reasoning:** It preserves the Brief's priority and marks the optional capability clearly.
- **Impact:** US-15, US-29; NetCo is not needed for acceptance.
- **Review (v1.1):** Changed from Confirmed by Brief. The Brief is explicit that FR-15 is Must and that the NetCo batch is optional, but recording the optional part as a separate Could story (US-29) is the team's modelling choice, so it is not marked as confirmed.
- **Status:** Team Decision

#### AMB-36 — NetCo batch behaviour (optional)

- **Question:** When is a NetCo payment "sent", what if the upload fails, and what if the result file omits a payment?
- **Why clarification is needed:** The state table has no "queued" state.
- **In the Brief?** Partly. NetCo uploads a file once per day and receives a result file the next morning.
- **Proposed decision:** A payment waits in FundsHeld until the batch file is uploaded (T3). If the upload fails, it goes into the next batch. A payment missing from the result file stays in SentToBiller and appears in the reconciliation report. The 10-second timeout does not apply.
- **Reasoning:** It uses only permitted transitions.
- **Impact:** US-29; only relevant if NetCo is built.
- **Review (v1.1):** Genuinely unresolved (optional feature). Uses only permitted transitions; consistent with US-29 and AC-26.10. No external confirmation needed.
- **Status:** Team Decision

### 11.3 Summary of decisions

| Status | Count | Records |
|---|---|---|
| Confirmed by Brief | 3 | AMB-10, AMB-15, AMB-24 |
| Team Decision | 32 | AMB-01 to AMB-09, AMB-11 to AMB-14, AMB-16 to AMB-23, AMB-25 to AMB-27, AMB-29 to AMB-36 |
| Requires Instructor Confirmation | 1 | AMB-28 |
| **Total** | **36** | Each record contains one proposed decision |

### 11.4 Questions to confirm with the instructors or teaching assistants

None of the following has been answered. Until it is, the working default applies. All of them belong to AMB-28 (refund of a completed payment, FR-27, Could); every other record is a Team Decision or is confirmed by the Brief.

| Ref | Question to confirm | Working default |
|---|---|---|
| RF-1 (AMB-28) | When a completed payment is refunded, does the customer receive the bill amount only, or the bill amount plus the service fee? | The whole amount charged |
| RF-2 (AMB-28) | Does PayLink fund the refund first and recover the money from the biller, or does the customer wait until the biller returns the money? | PayLink funds first; the amount is booked as recoverable from the biller |
| RF-3 (AMB-28) | Is PayLink's fee income reversed? | Reversed when the fee is refunded |
| RF-4 (AMB-28) | Is the agent's commission reversed, and what happens if the agent has already spent it? | Reversed, never below zero; any shortfall is owed by the agent |
| RF-5 (AMB-28) | If PayLink cannot recover the money from the biller, who bears the loss? | PayLink absorbs the loss |

## 12. Requirement Traceability Matrix

### 12.1 Functional requirements to user stories

| FR | Requirement (short) | Priority | User story | Module | Acceptance criteria and validation method |
|---|---|---|---|---|---|
| FR-1 | Customer registers with basic level | Must | US-01 | M1 | AC-01.1 to 01.5; API integration tests with negative cases |
| FR-2 | Upgrade to verified raises limits | Should | US-02 | M1 | AC-02.1 to 02.5; integration test with limit check before and after |
| FR-3 | View balance (available, held) and history | Must | US-03 | M1 | AC-03.1 to 03.6; API tests against known ledger data; cross-customer test (NFR-09) |
| FR-4 | Top up (simulated card or agent cash) | Must | US-04 | M1 | AC-04.1 to 04.9; stub gateway tests; duplicate request test; role test |
| FR-5 | Ledger entry for each change; balances computed | Must | US-05 | M1 | AC-05.1 to 05.6; recomputation test; concurrency test (HC-4) |
| FR-6 | Admin adds, edits, disables billers and services | Must | US-06 | M2 | AC-06.1 to 06.8; API tests; role test; HC-8 test |
| FR-7 | Customer sees billers and input fields | Must | US-07 | M2 | AC-07.1 to 07.5; API tests |
| FR-8 | Bill inquiry shows amount, fee, total | Must | US-08 | M3 (M2, M4, M5) | AC-08.1 to 08.8; mock biller tests; fee unit tests |
| FR-9 | Pay a bill from the wallet | Must | US-09 | M3 | AC-09.1 to 09.8; end-to-end test with mock ElectroCo |
| FR-10 | Same idempotency key returns original result | Must | US-10 | M3 | AC-10.1 to 10.7; HC-1 test; concurrency test |
| FR-11 | Life cycle with no other transitions | Must | US-11 | M3 | AC-11.1 to 11.6; unit test over all 64 state pairs |
| FR-12 | Funds held before biller call; released on failure | Must | US-12 | M1 (M3) | AC-12.1 to 12.6; HC-3 test; ledger assertions |
| FR-13 | Timeout becomes Unknown; re-query rules | Must | US-13 | M3 (M4) | AC-13.1 to 13.9; HC-2 test with response timing; clock-controlled retry test; restart test |
| FR-14 | TopUpNet callbacks accepted; duplicates ignored | Must | US-14 | M4 | AC-14.1 to 14.9; HC-6 test; callback authentication test |
| FR-15 | At least 3 billers with different formats | Must | US-15 | M4 | AC-15.1 to 15.7; contract tests per mock biller; outage test (NFR-03) |
| FR-16 | Fees per section 2.6 including promotion | Must | US-16 | M5 | AC-16.1 to 16.8; parameterised unit tests with decimal values |
| FR-17 | Agent pays for customer and earns commission | Should | US-17 | M5 (M1, M3) | AC-17.1 to 17.9; integration test with agent role and authorisation code |
| FR-18 | Reject above single or daily limit with reason | Must | US-18 | M7 | AC-18.1 to 18.9; boundary-value unit tests |
| FR-19 | More than 5 payments in 60 s blocks 10 min | Should | US-19 | M7 | AC-19.1 to 19.8; time-controlled tests |
| FR-20 | Support searches payment by reference, full history | Must | US-20 | M3 | AC-20.1 to 20.5; API tests; role test |
| FR-21 | Support blocks and unblocks accounts | Should | US-21 | M1 | AC-21.1 to 21.6; API tests; audit check |
| FR-22 | SMS and push on completion and failure | Must | US-22 | M6 | AC-22.1 to 22.7; notification stub tests; HC-3, HC-5 |
| FR-23 | Failed notifications retried 3 times, then undelivered | Should | US-23 | M6 | AC-23.1 to 23.6; HC-5 test |
| FR-24 | Admin edits fees and limits without redeploying | Should | US-24 | M5, M7 | AC-24.1 to 24.6; integration test changing values at runtime |
| FR-25 | Audit log of money movements and admin actions | Must | US-25 | Cross-cutting | AC-25.1 to 25.9; ledger and audit cross-check test (NFR-05) |
| FR-26 | Daily reconciliation report from settlement file | Should | US-26 | M8 | AC-26.1 to 26.10; HC-7 test; file-processing tests |
| FR-27 | Customer requests refund of completed payment; support approves | Could | US-27 | M1 (M3) | AC-27.1 to 27.14; workflow tests; ledger conservation test |
| FR-28 | Admin views daily totals | Could | US-28 | M8 | AC-28.1 to 28.5; comparison with ledger sums; role test (administrator, finance) |
| FR-15 (optional part) | NetCo batch | Could | US-29 | M4 | AC-29.1 to 29.5; batch file tests (optional) |
| Brief §2.4 | ElectroCo API v2 change request (Phase 2) | Must | US-30 | M4 | AC-30.1 to 30.5; mock v2 tests; changed-file report |
| FR-11, FR-13 (T7, T8) | Resolution of Unknown payments | Must | US-31 | M3 (M4, M8) | AC-31.1 to 31.8; callback, retry, support re-query and reconciliation tests |

### 12.2 Non-functional requirements to verification methods

| NFR | Quality attribute | Verification method | Evidence | Stage |
|---|---|---|---|---|
| NFR-01 | Correctness | Duplicate-request tests (HC-1, HC-6) and concurrency tests (HC-4); recomputation of every balance from the ledger | Test report with zero double charges and zero negative balances | Phase 2 |
| NFR-02 | Performance | Load test of 100 payments per second for 1 minute against the mock billers, with enough funded accounts that velocity and daily limits are not triggered | Load-test report with throughput and error counts | Phase 2 |
| NFR-03 | Resilience | Integration test taking one mock biller down while payments to the other billers continue | Test report showing the other billers' payments completing | Phase 2 |
| NFR-04 | Modifiability | HC-8 test; ElectroCo v2 change (US-30); review of the change list | List of changed files limited to Biller Integration and configuration | Phase 2 |
| NFR-05 | Auditability | Ledger and audit cross-check (AC-25.5 to AC-25.9): every financial operation has an audit record, every ledger entry links to its operation and audit record, no orphans, atomic commit, no false record after a rollback | Zero orphans in either direction, plus who, when and why in each record | Phase 2 |
| NFR-06 | Testability | Coverage report on domain logic from the CI pipeline | Coverage above 70% shown in the pipeline (Brief §3.3: at least 70%; Brief §4: above 70%) | Phase 2 |
| NFR-07 | Responsiveness under timeout | 20 runs of the 12-second mock ElectroCo scenario; response time measured from the original request; state checked at the 10-second timeout | All responses within 12 seconds; state Unknown at the timeout | Phase 2 |
| NFR-08 | Recoverability | Forced restart during 50 in-flight payments, then a state and ledger check | All payments continue; zero duplicates | Phase 2 |
| NFR-09 | Access control | Automated access-control suite: role by operation matrix (Section 9.3), cross-customer tests, unauthenticated tests, tamper tests, OpenAPI access-rule check | Zero unauthorised successes, zero leaked data, 100% of endpoints with a declared rule | Phase 2 |
| NFR-10 | Deployability | **Phase 1:** clean-clone `docker compose up` of the empty application and database; pipeline builds, lints and runs a test. **Phase 2:** clean-clone `docker compose up` of application, database and mock billers; health check; local deployment on `main`; documented rollback | Phase 1: green pipeline on the `phase-1` tag. Phase 2: healthy status and green pipeline on the `final` tag | Phase 1 and Phase 2 |

### 12.3 Hard cases to user stories

| Hard case | Stories | Related NFR |
|---|---|---|
| HC-1 duplicate request | US-09, US-10 | NFR-01 |
| HC-2 12-second ElectroCo | US-13 | NFR-07 |
| HC-3 AquaCo refusal | US-12, US-15, US-22 | NFR-01 |
| HC-4 concurrent payments | US-05, US-09 | NFR-01 |
| HC-5 SMS provider down | US-22, US-23 | None specific |
| HC-6 duplicate callback | US-14 | NFR-01 |
| HC-7 settlement lists a Failed payment | US-26 | NFR-05 |
| HC-8 new biller | US-06, US-15, US-30 | NFR-04 |

### 12.4 Business rules to user stories

| Rule | Stories | Rule | Stories |
|---|---|---|---|
| BR-01 | US-01, US-02, US-18 | BR-10 | US-19, US-21 |
| BR-02 | US-02, US-18 | BR-11 | US-08, US-13, US-29 |
| BR-03, BR-04 | US-08, US-18 | BR-12 | US-13, US-31 |
| BR-05 to BR-07 | US-08, US-16, US-29 | BR-13 | US-23 |
| BR-08 | US-08, US-16, US-24 | BR-14 | US-16, US-30 |
| BR-09 | US-17, US-27 | BR-15 | US-12, US-13, US-31 |
| BR-16 | US-11 | BR-17 | US-09, US-12 |
| BR-18 | US-09, US-16, US-17, US-31 | | |

### 12.5 Coverage summary

| Measure | Result |
|---|---|
| Original functional requirements covered | 28 of 28 (FR-1 to FR-28) |
| User stories | 31 (28 mapped one-to-one, 3 additional) |
| Acceptance criteria (Given / When / Then) | 224, every story has at least one |
| Stories by priority | Must 21, Should 7, Could 3 |
| Original FRs by priority (unchanged from the Brief) | Must 19, Should 7, Could 2 |
| Non-functional requirements | 10 (6 from the Brief, 4 team-defined); Must 8, Should 2 |
| Hard cases documented | 8 of 8 |
| Ambiguity and decision records | 36: Confirmed by Brief 3, Team Decision 32, Requires Instructor Confirmation 1 |

## 13. Out of Scope

This section separates four categories so that Phase 2 work is not mistaken for out-of-scope work.

### 13.1 Phase 1 deliverables (due Wednesday 25 November)

- **Requirements pack:** this SRS as PDF and `docs/requirements.md`.
- **Design pack** (`docs/design/`, source files): use-case diagram, domain class diagram, sequence diagrams for "pay a bill" and "refund a failed payment", payment state diagram, and ER diagram, in PlantUML or Mermaid. *This SRS contains no diagrams; they belong to the design pack.*
- **DevOps foundation:** repository with a protected main branch and pull-request rules; a Dockerfile and compose file that start an empty application and the database; a pipeline that builds, lints and runs at least one test on every push.
- **Prompt log:** at least five entries, two of them showing an improved prompt after a wrong first answer.

### 13.2 Phase 2 implementation requirements (due Friday 18 December)

These are **in scope**, but implemented after this Phase 1 pack:

- the working system for the six core modules, with the database, at least three mock billers, and OpenAPI documentation;
- tests with TDD evidence, more than 70% coverage on domain logic, and one automated test for each hard case;
- five design patterns, architecture decision records, and the evolution of Notifications and Reconciliation to an event-driven or service-based style;
- the **ElectroCo API v2 change request** (Section 8.4, US-30);
- the pipeline with local deployment, health check and rollback; the deployment comparison; the final report and demo video; the individual discussion;
- the technical requirements of Brief §3.4 (a relational database of at least 10 tables with migrations in the repository, at least 20 REST endpoints documented with OpenAPI, one asynchronous flow, design patterns, architecture, Docker, CI/CD). They constrain the design and implementation of the final system; they are carried by the stage table in Section 1.8, by NFR-04, NFR-06, NFR-09 and NFR-10, and are not restated as functional requirements.

### 13.3 Optional capabilities

- **NetCo batch processing** (US-29, *Could*).
- **Email as a notification channel** (named in the module description and the retry rule, but not in FR-22).

### 13.4 Not specified by the Brief and not planned

The following are not in the Brief. They are excluded unless stakeholders confirm otherwise:

- real card, bank or payment-gateway integration; real SMS, email or push providers; real biller connections;
- document-based identity verification, multi-factor authentication, anti-money-laundering or other regulatory reporting;
- wallet-to-wallet transfers, cash-out or withdrawals, multiple currencies, and a maximum wallet balance;
- partial bill payments, customer cancellation of payments (AMB-34), scheduled or recurring payments, and chargebacks;
- biller onboarding or commercial contract management;
- tax on fees and customer-facing invoices;
- a rich mobile or web user interface (a simple page or command-line client is enough, and marks are for the backend);
- the commercial and operational process by which PayLink recovers refunded money from a biller (the ledger records the amount recoverable, AMB-28).
