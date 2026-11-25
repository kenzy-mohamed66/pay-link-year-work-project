# PayLink Requirements Pack (Phase 1)

> User stories 10 to 41 were written by the team (Members 2, 3 and 4). Sections 5 and 6.2 are AI-assisted drafts: the owners must review them before the `phase-1` tag. The full story tables (pre/post-conditions, scenarios) are in the SRS Word document.

## 1. Problem Summary

PayLink is an Egyptian company that lets millions of people pay monthly utility bills (electricity, water, mobile, internet) from a digital wallet, through a mobile app or through agents' terminals. The company earns a service fee on every payment and pays agents a commission. Because PayLink holds customers' money, any mistake, such as a double charge or a payment charged but never received by the biller, costs trust and money and may trigger regulatory trouble.

The challenge is to build a new backend that is:

- **Correct with money:** no double charges, no negative balances, and every movement recorded in a ledger and an audit log.
- **Resilient to failure:** every payment step can fail (biller timeout, refusal, duplicate request, SMS outage). The system must handle each case through a strict payment life cycle (Initiated, FundsHeld, SentToBiller, Completed / Failed / Unknown, Refunded) and never refund a payment that may still have reached the biller.
- **Easy to extend:** billers use different protocols (REST/JSON, SOAP-style XML, asynchronous callbacks, daily batch files). Adding or changing a biller must affect only its integration module.
- **Ready for peak load:** it must sustain a high payment volume on the busiest days of the month.

## 2. Stakeholders

| Stakeholder | Role | Main interest / concern |
|---|---|---|
| Customer | Pays bills from the wallet | Fast and safe payments; clear fee and total before paying; refund on failure; receipt notification |
| Agent | Shop owner serving customers | Smooth terminal experience; correct commission (40% of the fee) when a payment completes |
| Biller (ElectroCo, AquaCo, TopUpNet, NetCo) | Issues bills and receives payments | Being paid correctly and only once; a clean integration contract; settlement records that match |
| PayLink management | Business owner | Fee revenue, low losses, customer trust, easy onboarding of new billers |
| Admin | Operates the platform | Manage billers and services; edit fees and limits without redeploying; daily totals; audit trail |
| Support employee | Handles customer issues | Search a payment by reference and see its history; block/unblock accounts; review reconciliation differences; approve refunds |
| Regulator / compliance | External authority | Auditability of every money movement; fraud controls; limits per verification level |
| Development team | Builds and maintains the system | Clear requirements, testable design, modular code, working CI/CD |
| Operations / DevOps | Deploys and runs the system | Reliable pipeline, health checks, rollback |
| External providers (SMS, email, push) | Deliver notifications | Retries when they are down; their outage must not block payments |
| Course instructor / TA | Evaluates the project | Correctness, honest AI usage, justified design decisions |

## 3. Functional Requirements and Priorities

| ID | Requirement | Priority | Owner |
|---|---|---|---|
| FR-1 | A customer can register and has a basic verification level | Must | Member 1 |
| FR-2 | A customer can be upgraded to verified, which raises limits | Should | Member 1 |
| FR-3 | A customer can view wallet balance (available and held) and transaction history | Must | Member 1 |
| FR-4 | A customer can top up the wallet (simulated card or agent cash) | Must | Member 1 |
| FR-5 | Every balance change creates a ledger entry; balances are computed from the ledger | Must | Member 1 |
| FR-6 | An admin can add, edit and disable billers and services | Must | Member 1 |
| FR-7 | A customer can see the list of billers and the input fields each requires | Must | Member 1 |
| FR-8 | A customer can run a bill inquiry and see the amount, fee and total before paying | Must | Member 2 |
| FR-9 | A customer can pay a bill from the wallet | Must | Member 2 |
| FR-10 | A repeated request with the same idempotency key returns the original result | Must | Member 2 |
| FR-11 | A payment follows the life cycle in section 2.5 of the brief and no other transitions | Must | Member 2 |
| FR-12 | Funds are held before the biller is called and released if the payment fails | Must | Member 2 |
| FR-13 | A payment that times out becomes Unknown and is re-queried by the retry rules | Must | Member 2 |
| FR-14 | The system accepts asynchronous callbacks from TopUpNet and ignores duplicates | Must | Member 2 |
| FR-15 | The system supports at least 3 billers with different formats (ElectroCo, AquaCo, TopUpNet); NetCo batch is optional | Must | Member 3 |
| FR-16 | Fees follow the rules in section 2.6 of the brief, including the promotion period | Must | Member 3 |
| FR-17 | An agent can pay on behalf of a customer and earns commission | Should | Member 3 |
| FR-18 | Payments above the single-payment limit or the daily limit are rejected with a clear reason | Must | Member 3 |
| FR-19 | More than 5 payments in 60 seconds blocks the account for 10 minutes | Should | Member 3 |
| FR-20 | A support employee can search a payment by reference and see its full history | Must | Member 3 |
| FR-21 | A support employee can block and unblock an account | Should | Member 3 |
| FR-22 | A customer receives an SMS and a push message on completion and on failure | Must | Member 4 |
| FR-23 | Failed notifications are retried 3 times then marked undelivered | Should | Member 4 |
| FR-24 | An admin can edit fee and limit values without redeploying | Should | Member 4 |
| FR-25 | Every money movement and admin action is written to an audit log | Must | Member 4 |
| FR-26 | The system produces a daily reconciliation report from a settlement file | Should | Member 4 |
| FR-27 | A customer can request a refund of a completed payment; support approves it | Could | Member 4 |
| FR-28 | An admin can view daily totals: payments, fees earned, commissions paid, failures per biller | Could | Member 4 |

## 4. User Stories

### 4.1 FR-1 to FR-7 (Member 1)

#### US-1 Register (FR-1, Must)
**As a** new customer, **I want to** register with my phone number and name, **so that** I get an account and a wallet.

- **Given** a phone number not registered before, **when** I submit valid registration data, **then** an account is created with verification level *Basic* and a wallet with 0.00 EGP available and 0.00 EGP held.
- **Given** a phone number already registered, **when** I try to register again, **then** the system rejects it with a clear "already registered" error and creates nothing.
- **Given** missing or invalid data (e.g. a malformed phone number), **when** I submit, **then** the system returns a validation error listing the invalid fields.

#### US-2 Upgrade to Verified (FR-2, Should)
**As a** customer, **I want to** be upgraded to *Verified*, **so that** my daily limit rises from 5,000 to 30,000 EGP.

- **Given** a Basic customer whose verification is approved, **when** the upgrade is applied, **then** the level becomes *Verified* and the new limit applies to the next payment.
- **Given** a Basic customer whose verification is not approved, **when** they pay above 5,000 EGP in one day, **then** the payment is rejected.
- **Given** any upgrade, **when** it happens, **then** an audit log entry records who, when and why.

#### US-3 View balance (FR-3, Must)
**As a** customer, **I want to** see my available and held balance, **so that** I know what I can spend.

- **Given** a wallet with 1,000 EGP and a 352.50 EGP hold, **when** I open my wallet, **then** I see available = 647.50 and held = 352.50.
- **Given** a balance, **when** it is displayed, **then** it equals the sum of the ledger entries, never a stored number.

#### US-4 View transaction history (FR-3, Must)
**As a** customer, **I want to** see my transaction history, **so that** I can track my spending.

- **Given** I have past transactions, **when** I request my history, **then** I see them newest first with date, type, amount and status.
- **Given** I have no transactions, **when** I request my history, **then** I get an empty list, not an error.
- **Given** another customer's wallet, **when** I try to read it, **then** access is denied.

#### US-5 Top up by card (FR-4, Must)
**As a** customer, **I want to** top up my wallet with a (simulated) card, **so that** I have money to pay bills.

- **Given** a valid simulated card and amount, **when** I top up, **then** the available balance increases by exactly that amount and a ledger entry is created.
- **Given** a declined card, **when** I top up, **then** the balance does not change and I see the reason.
- **Given** an amount of zero or negative, **when** I top up, **then** it is rejected.

#### US-6 Top up by agent cash (FR-4, Must)
**As an** agent, **I want to** top up a customer's wallet after taking cash, **so that** customers without cards can use PayLink.

- **Given** a valid agent and customer, **when** the agent submits a cash top-up, **then** the customer's wallet is credited and the entry records the agent's ID.
- **Given** a blocked or unknown customer account, **when** the agent submits a top-up, **then** it is rejected.

#### US-7 Ledger for every balance change (FR-5, Must)
**As a** support or audit officer, **I want** every balance change recorded as a ledger entry, **so that** all money movements are traceable.

- **Given** any top-up, hold, charge, release, refund, fee or commission, **when** it happens, **then** a ledger entry is created with amount, type, reference and timestamp.
- **Given** a wallet, **when** its balance is computed, **then** it equals the sum of its entries, and no code overwrites a balance directly.
- **Given** a ledger entry, **when** someone tries to edit or delete it, **then** the system refuses (corrections are new entries).
- **Given** two simultaneous debits and a balance that covers only one, **when** both run, **then** one succeeds and the balance never goes negative.

#### US-8 Manage billers and services (FR-6, Must)
**As an** admin, **I want to** add, edit and disable billers and services, **so that** the catalog stays current.

- **Given** valid biller data, **when** I add a biller, **then** it appears in the catalog with its services, required fields and min/max amounts.
- **Given** an active biller, **when** I disable it, **then** customers can no longer start payments to it, and payments already in progress continue.
- **Given** any change, **when** it is saved, **then** an audit entry records the admin and the change.
- **Given** a non-admin user, **when** they try to change the catalog, **then** access is denied.

#### US-9 Browse billers and required fields (FR-7, Must)
**As a** customer, **I want to** see the active billers and the input each requires, **so that** I know what to enter.

- **Given** active billers, **when** I open the catalog, **then** I see each biller, its services and its required fields (e.g. meter number for electricity, phone number for top-up).
- **Given** a disabled biller, **when** I open the catalog, **then** it is not shown (or is shown as unavailable).

### 4.2 FR-8 to FR-14 (Member 2)


#### US-10 Bill inquiry with amount, fee and total (FR-8, Must)
**As a customer**  
**I would like to enter my bill identifier and see the bill amount, the PayLink fee and the total**  
**So that I know exactly what I will pay before confirming.**

- **Given** active billers and a valid meter number with a bill of 350.00 EGP, **when** I run a bill inquiry on ElectroCo, **then** I see amount = 350.00, fee = 2.50 and total = 352.50 EGP.
- **Given** a biller that has a fee promotion running today, **when** I run a bill inquiry, **then** the fee shown is 0.00 and the total equals the bill amount.
- **Given** an unknown or invalid meter number, **when** I run a bill inquiry, **then** I get a clear "bill not found" error and no payment is created.

#### US-11 Bill inquiry failure handling (FR-8, Must)
**As a customer**  
**I would like a clear message when the inquiry cannot be completed**  
**So that I can retry or choose another biller.**

- **Given** a biller that does not answer within 10 seconds, **when** I run a bill inquiry, **then** I see "biller temporarily unavailable" and nothing is charged.
- **Given** a disabled biller, **when** I run a bill inquiry, **then** the request is rejected with "biller unavailable".
- **Given** NetCo (batch biller, no live inquiry), **when** I try an inquiry, **then** I am told that the amount must be entered manually and that it is processed in the daily batch.

#### US-12 Pay a bill from the wallet (FR-9, Must)
**As a customer**  
**I would like to pay a bill from my wallet balance**  
**So that I do not have to queue or pay cash.**

- **Given** available balance 1,000 EGP and a bill total of 352.50 EGP, **when** I confirm the payment and the biller confirms, **then** the payment becomes Completed, the hold becomes a final charge, PayLink fee income is credited, and I get a receipt notification.
- **Given** a payment initiated by an agent, **when** the payment completes, **then** the agent wallet receives 40% of the fee as commission.
- **Given** a blocked account, **when** I try to pay, **then** the payment is Rejected with the reason "account blocked".

#### US-13 Payment rejected for insufficient funds or limits (FR-9, Must)
**As a customer**  
**I would like to be told immediately when I cannot pay**  
**So that I can top up or reduce the amount.**

- **Given** available balance 100 EGP and a total of 352.50 EGP, **when** I pay, **then** the payment is Rejected with "insufficient funds" and the balance stays 100.00.
- **Given** two simultaneous payments and a balance that covers only one, **when** both run, **then** one is FundsHeld and the other is Rejected, and the balance never goes negative.
- **Given** an amount below 5 EGP or above 10,000 EGP, **when** I pay, **then** the payment is Rejected with the limit that was violated.

#### US-14 Idempotent payment requests (FR-10, Must)
**As a customer**  
**I would like a repeated request to never charge me twice**  
**So that a slow network does not cost me money.**

- **Given** a payment already processed with key K, **when** the app sends the same request with key K again, **then** no new payment or ledger entry is created, and the original result is returned.
- **Given** the first request with key K is still being processed, **when** the same key arrives again at the same moment, **then** only one payment is created, and the second request returns the current status.
- **Given** a request without an idempotency key, **when** I pay, **then** the request is rejected with a validation error.

#### US-15 Idempotency key reuse with different data (FR-10, Must)
**As PayLink**  
**I would like to detect a key reused with a different request body**  
**So that a client bug cannot hide a different payment.**

- **Given** key K was used for 350.00 EGP to ElectroCo, **when** a request with key K asks for 500.00 EGP, **then** it is rejected with a conflict error, and the original payment is untouched.
- **Given** a stored idempotency key, **when** its retention period has not expired, **then** it keeps returning the original result.

#### US-16 Payment life cycle follows allowed transitions only (FR-11, Must)
**As a support or audit officer**  
**I would like every payment to be in exactly one state and move only along the allowed transitions**  
**So that the payment history is trustworthy.**

- **Given** a payment in Initiated, **when** fraud checks pass and funds are available, **then** it moves to FundsHeld.
- **Given** a payment in Initiated, **when** a fraud check fails or funds are insufficient, **then** it moves to Rejected.
- **Given** a payment in SentToBiller, **when** the biller confirms, refuses or times out, **then** it moves to Completed, Failed or Unknown respectively.
- **Given** a payment in Failed, **when** the hold is released to the wallet, **then** it moves to Refunded.

#### US-17 Invalid transitions are refused (FR-11, Must)
**As a developer or auditor**  
**I would like illegal state changes to be refused**  
**So that a bug can never, for example, refund a Completed payment.**

- **Given** a payment in Completed, Rejected or Refunded (final states), **when** any state change is requested, **then** it is refused.
- **Given** a payment in FundsHeld, **when** a change straight to Completed is requested, **then** it is refused because it must pass through SentToBiller.
- **Given** a refused transition, **when** it happens, **then** an audit entry records the payment, the attempted transition and the time.

#### US-18 Funds are held before the biller is called (FR-12, Must)
**As a customer**  
**I would like my money reserved before PayLink contacts the biller**  
**So that the payment cannot fail later for lack of funds.**

- **Given** available = 1,000.00 and total = 352.50, **when** the payment reaches FundsHeld, **then** available = 647.50, held = 352.50, and the biller has not been called yet.
- **Given** the hold could not be created, **when** the payment is processed, **then** the biller is never called.

#### US-19 Hold is released when the payment fails (FR-12, Must)
**As a customer**  
**I would like my held money back if the biller refuses**  
**So that I do not pay for a bill that was not paid.**

- **Given** AquaCo refuses with an error code, **when** the payment becomes Failed, **then** the hold is released, the payment becomes Refunded and I get an SMS with a clear reason.
- **Given** a payment that was released once, **when** release is triggered again, **then** no second refund happens (the ledger has exactly one release entry).

#### US-20 Payment becomes Unknown on timeout (FR-13, Must)
**As a customer**  
**I would like to be told my payment is being processed when the biller is slow**  
**So that I am not wrongly charged or refunded.**

- **Given** ElectroCo takes 12 seconds to answer, **when** the 10-second timeout is reached, **then** the payment becomes Unknown, I see "your payment is being processed" and the money stays held.
- **Given** a payment in Unknown, **when** I view my wallet, **then** the amount is shown as held, not spent and not refunded.

#### US-21 Unknown payments are re-queried by the retry rules (FR-13, Must)
**As PayLink**  
**I would like to re-query Unknown payments automatically**  
**So that most of them are resolved without manual work.**

- **Given** an Unknown payment, **when** 1, 5 and 15 minutes have passed, **then** the biller is re-queried at each of these times (at most 3 retries).
- **Given** a re-query returns "paid", **when** the answer arrives, **then** the payment becomes Completed.
- **Given** a re-query returns "not paid", **when** the answer arrives, **then** the payment becomes Failed and the hold is released.
- **Given** all 3 retries gave no answer, **when** the retries are exhausted, **then** the payment stays Unknown and is marked "awaiting reconciliation".

#### US-22 Unknown payments are never auto-refunded (FR-13, Must)
**As management**  
**I would like money in Unknown payments to never be returned automatically**  
**So that PayLink never refunds a payment the biller actually received.**

- **Given** a payment in Unknown, **when** any refund is attempted before a confirmed biller refusal or reconciliation result, **then** the refund is refused.
- **Given** the settlement file does not contain the payment, **when** reconciliation runs, **then** the payment becomes Failed and the normal release/refund path applies.

#### US-23 Receive asynchronous callbacks from TopUpNet (FR-14, Must)
**As PayLink**  
**I would like to accept TopUpNet's callback**  
**So that top-up payments are completed or failed minutes after they were accepted.**

- **Given** a payment accepted by TopUpNet, **when** the callback says "success", **then** the payment becomes Completed, the fee is credited, and the customer is notified.
- **Given** a payment accepted by TopUpNet, **when** the callback says "failed", **then** the payment becomes Failed and the hold is released.
- **Given** a callback with an unknown payment reference, **when** it arrives, **then** it is rejected and logged, and no payment is changed.

#### US-24 Duplicate callbacks are ignored (FR-14, Must)
**As PayLink**  
**I would like duplicate callbacks to be ignored**  
**So that a payment completes only once.**

- **Given** TopUpNet calls the callback twice with "success", **when** the second callback arrives, **then** the payment stays Completed once, no new ledger entry is created and the duplicate is logged.
- **Given** a callback "failed" after the payment is already Completed, **when** it arrives, **then** it is ignored, logged and flagged for support review.

#### US-25 Callback authenticity check (FR-14, Must)
**As PayLink**  
**I would like to verify that a callback really comes from TopUpNet**  
**So that nobody can complete a payment by faking a request.**

- **Given** a callback without a valid shared secret or signature, **when** it arrives, **then** it is rejected with 401 and nothing changes.
- **Given** a malformed callback body, **when** it arrives, **then** it is rejected with a validation error and logged.

### 4.3 FR-15 to FR-21 (Member 3)


#### US-26 Support multiple biller integrations (FR-15, Must)
**As a system administrator**  
**I want the system to support multiple billers using different integration formats**  
**So that customers can pay bills from different providers without changing the core payment system.**

- **Given** a new biller needs to be added, **when** its integration adapter is implemented and configured, **then** the new biller can be supported without modifying the core payment processing logic.
- **Given** a biller uses a different communication protocol, **when** the system sends a payment request, **then** the corresponding adapter handles the request and response correctly.

#### US-27 Apply fees and promotional rules (FR-16, Must)
**As a customer**  
**I want the system to calculate the correct service fee according to the configured fee rules**  
**So that I know the total amount I will pay before confirming the transaction.**

- **Given** a customer requests a payment, **when** the system calculates its service fee, **then** the fee follows the configured fixed, percentage, tiered, or promotional rules.
- **Given** a promotional period is active, **when** a customer makes a payment during that period, **then** the promotional fee rules are applied.
- **Given** a promotional period has expired, **when** a customer makes a payment, **then** the normal fee rules are applied.
- **Given** a payment fee has been calculated, **when** the customer reviews the payment before confirmation, **then** the service fee and total amount are displayed clearly.

#### US-28 Agent payment and commission (FR-17, Should)
**As an agent**  
**I want to make bill payments on behalf of customers and receive a commission**  
**So that I can serve customers through my shop and earn from completed payments.**

- **Given** an active agent and a valid customer wallet, **when** the agent requests a bill payment on behalf of the customer, **then** the system processes the payment using the customer's wallet.
- **Given** an agent-assisted payment is completed successfully, **when** the system calculates the agent's commission, **then** the agent receives 40% of the service fee.
- **Given** an agent-assisted payment fails, **when** the payment is marked as Failed, **then** no commission is awarded to the agent.
- **Given** an agent performs a payment, **when** the transaction is recorded, **then** the system records the agent's identity and the payment reference.

#### US-29 Enforce payment limits (FR-18, Must)
**As a customer**  
**I want the system to check single-payment and daily payment limits**  
**So that payments exceeding the allowed limits are rejected and my account remains protected.**

- **Given** a customer requests a payment, **when** the payment amount exceeds the allowed single-payment limit, **then** the system rejects the payment and displays a clear reason.
- **Given** a customer has already reached the daily payment limit, **when** the customer requests another payment, **then** the system rejects the payment and informs the customer that the daily limit has been reached.
- **Given** a customer requests a payment within the allowed limits, **when** the system validates the payment, **then** the payment is allowed to continue processing.
- **Given** different verification levels have different payment limits, **when** a payment is requested, **then** the system applies the limits configured for that customer's verification level.

#### US-30 Block accounts after excessive payment attempts (FR-19, Should)
**As a system administrator**  
**I want the system to detect excessive payment activity and temporarily block suspicious accounts**  
**So that the system can reduce fraud and protect customers' wallets.**

- **Given** a customer has made 5 payment attempts within 60 seconds, **when** the customer makes another payment attempt within the same 60-second period, **then** the system blocks the account for 10 minutes.
- **Given** an account is temporarily blocked, **when** the customer attempts to make another payment, **then** the system rejects the request and informs the customer that the account is temporarily blocked.
- **Given** an account has been blocked for 10 minutes, **when** the blocking period expires, **then** the system automatically unblocks the account.
- **Given** a customer makes 5 or fewer payment attempts within 60 seconds, **when** the system checks the payment activity, **then** the account is not blocked by this rule.

#### US-31 Search payment and view full history (FR-20, Must)
**As a support employee**  
**I want to search for a payment using its reference and view its complete history**  
**So that I can investigate customer complaints and track every payment step.**

- **Given** a support employee has a valid payment reference, **when** the employee searches for that reference, **then** the system displays the matching payment details.
- **Given** a payment is found, **when** the support employee opens its history, **then** the system displays all recorded payment status changes and related events with their timestamps.
- **Given** a support employee enters an invalid or non-existing reference, **when** the search is performed, **then** the system displays a clear message that no payment was found.
- **Given** a payment has passed through multiple stages, **when** its history is displayed, **then** the employee can trace its complete life cycle, including holds, biller requests, completion, failure, and refunds when applicable.

#### US-32 Block and unblock customer accounts (FR-21, Should)
**As a support employee**  
**I want to block and unblock customer accounts**  
**So that I can prevent suspicious or unauthorized account activity and restore access when appropriate.**

- **Given** a customer has an active account, **when** an authorized support employee blocks the account, **then** the system changes the account status to Blocked.
- **Given** a customer account is blocked, **when** the customer attempts to perform a payment, **then** the system rejects the payment request.
- **Given** a customer account is blocked, **when** an authorized support employee unblocks the account, **then** the system changes the account status back to Active.
- **Given** a support employee blocks or unblocks an account, **when** the action is completed, **then** the system records the employee identity, account reference, action type, and timestamp in the audit log.

### 4.4 FR-22 to FR-28 (Member 4)


#### US-33 Completion and failure notifications (FR-22, Must)
**As a customer**  
**I like to be able to receive an SMS and a push message when my payment completes or fails**  
**So that I know what happened to my money.**

- **Given** a payment that becomes Completed, **when** the system finishes the payment, **then** I receive an SMS and a push message with the amount, biller and payment reference.
- **Given** AquaCo refuses a payment and it becomes Failed, **when** the hold is released, **then** I receive an SMS and a push message with a clear reason for the failure.
- **Given** the SMS provider is down when a payment completes, **when** the SMS cannot be sent, **then** the payment stays Completed and the push message is still sent.

#### US-34 Notification retry (FR-23, Should)
**As PayLink**  
**I like to be able to retry SMS and email messages that failed**  
**So that a short provider outage does not make customers miss their receipt.**

- **Given** an SMS that fails on the first attempt, **when** the system retries, **then** it retries up to 3 times and stops at the first success.
- **Given** all 3 retries failed, **when** the last retry fails, **then** the notification is marked "undelivered" and every attempt is recorded with its time.
- **Given** the SMS provider stays down, **when** customers pay other bills, **then** those payments are processed normally and are not delayed by the retries.

#### US-35 Edit fee and limit values without redeploying (FR-24, Should)
**As an admin**  
**I like to be able to edit fee values and payment limits**  
**So that business rules can change without redeploying the system.**

- **Given** I am logged in as admin, **when** I change the basic daily limit from 5,000 to 4,000 EGP, **then** the next payment check uses 4,000 EGP without restarting the system.
- **Given** I change the electricity fee from 2.50 to 3.00 EGP, **when** a customer runs a new bill inquiry, **then** the fee shown is 3.00, and payments already in progress keep their original fee.
- **Given** a negative or non-numeric value, **when** I save it, **then** it is rejected with a validation error and the old value stays.
- **Given** a non-admin user, **when** they try to change a fee or limit, **then** access is denied.
- **Given** any change, **when** it is saved, **then** an audit entry records the admin, the old value, the new value and the time.

#### US-36 Audit log for every money movement (FR-25, Must)
**As an audit officer**  
**I like to be able to see an audit record for every money movement**  
**So that every change in a balance can be traced to who, when and why.**

- **Given** any ledger entry is created, **when** it is saved, **then** an audit record exists with the actor, timestamp, reason, amount and payment reference.
- **Given** an agent-assisted payment completes, **when** the fee and the commission are credited, **then** each of them has its own audit record.
- **Given** an audit record, **when** someone tries to edit or delete it, **then** the system refuses.
- **Given** all ledger entries and all audit records, **when** the audit test compares them, **then** every ledger entry has a matching audit record and no field is missing.

#### US-37 Audit log for admin and support actions (FR-25, Must)
**As an audit officer**  
**I like to be able to see an audit record for every admin and support action**  
**So that changes to billers, fees, limits and accounts can be reviewed.**

- **Given** an admin disables a biller, **when** the change is saved, **then** an audit record shows the admin, the biller, the action and the time.
- **Given** a support employee blocks an account, **when** the action is completed, **then** an audit record shows the employee, the account and the time.
- **Given** a non-admin user tries to change the catalog, **when** the request is refused, **then** the attempt is logged and nothing is changed.
- **Given** an audit officer searches by user or date, **when** the search runs, **then** the matching records are shown newest first.

#### US-38 Daily reconciliation report (FR-26, Should)
**As a support employee**  
**I like to be able to get a daily report that compares PayLink payments with the biller's settlement file**  
**So that missing or different payments are found and reviewed.**

- **Given** the settlement file lists a payment that PayLink has as Completed with the same amount, **when** reconciliation runs, **then** the item is marked Matched.
- **Given** the settlement file lists a payment that PayLink has as Failed, **when** reconciliation runs, **then** the difference is flagged in the report for support to review and the payment is not changed automatically.
- **Given** the settlement file lists a payment that PayLink does not have, **when** reconciliation runs, **then** it is flagged as "missing in PayLink".
- **Given** an Unknown payment that the settlement file lists as paid, **when** reconciliation runs, **then** the payment becomes Completed.
- **Given** a malformed settlement file, **when** it is uploaded, **then** it is rejected with an error and no report is produced.

#### US-39 Request a refund of a completed payment (FR-27, Could)
**As a customer**  
**I like to be able to request a refund of a completed payment**  
**So that I can get my money back after a mistake.**

- **Given** I have a Completed payment, **when** I submit a refund request with a reason, **then** the request is created as Pending and I am told that support will review it.
- **Given** a payment that is not Completed (for example Failed or Unknown), **when** I request a refund, **then** the request is rejected with a clear reason.
- **Given** another customer's payment, **when** I request a refund for it, **then** access is denied.
- **Given** a payment that already has a Pending or approved refund request, **when** I submit another request, **then** it is rejected as a duplicate.

#### US-40 Support approves or rejects a refund request (FR-27, Could)
**As a support employee**  
**I like to be able to approve or reject refund requests**  
**So that only valid refunds return money to customers.**

- **Given** a Pending refund request, **when** a support employee approves it, **then** the wallet is credited with the refund amount through a new ledger entry and an audit record is written.
- **Given** a Pending refund request, **when** a support employee rejects it with a reason, **then** no money moves and the customer is notified with the reason.
- **Given** a request that was already approved, **when** it is approved again, **then** the second approval is refused and the ledger has exactly one refund entry.
- **Given** a user who is not a support employee, **when** they try to approve a request, **then** access is denied.

#### US-41 Admin daily totals (FR-28, Could)
**As an admin**  
**I like to be able to see daily totals of payments, fees earned, commissions paid and failures per biller**  
**So that I can follow how the business is doing.**

- **Given** payments were made on a day, **when** I open the daily totals for that date, **then** I see the number of payments, the total amount, the fees earned and the commissions paid.
- **Given** some payments failed on that day, **when** I open the daily totals, **then** I see the number of failures per biller.
- **Given** a day with no payments, **when** I open the daily totals, **then** I see zeros and not an error.
- **Given** the totals for a day, **when** they are compared with the ledger, **then** they match the ledger entries of that day.
- **Given** a non-admin user, **when** they open the daily totals, **then** access is denied.

> Total: 41 user stories (US-1 to US-41), above the minimum of 25.

## 5. Non-Functional Requirements

| ID | Quality | Requirement | How we measure it |
|---|---|---|---|
| NFR-1 | Correctness | Zero double charges and zero negative balances | Concurrency and duplicate-request tests (hard cases 1 and 4) all pass |
| NFR-2 | Performance | The system sustains 100 payments per second for 1 minute against the mock billers | Load test report |
| NFR-3 | Resilience | An outage of one biller does not stop payments to the other billers | Test that stops one mock biller while paying through the others |
| NFR-4 | Modifiability | Adding a biller changes only the Biller Integration module and configuration | List of changed files for the ElectroCo v2 change and a new biller |
| NFR-5 | Auditability | 100% of money movements have an audit record with who, when and why | Test that compares ledger entries with audit records |
| NFR-6 | Testability | At least 70% line coverage on domain logic | Coverage report from the CI pipeline |
| NFR-7 | Accuracy | All money uses exact decimal types, never floating point | Code review and rounding tests |
| NFR-8 | Maintainability | The pipeline builds, lints and tests on every push in under 10 minutes | CI run time |

## 6. Ambiguities and Decisions

### 6.1 FR-1 to FR-7 (Member 1)

| Question / ambiguity | Decision |
|---|---|
| How does a customer "verify" in FR-2? No process is given. | Simulated: an admin or support user approves it. |
| What identifies a customer at registration? | A unique phone number. |
| Can a top-up be reversed? | Out of scope; only payments are refunded. |
| Does the daily limit reset at midnight or after a rolling 24 hours? | Calendar day in Africa/Cairo time (configurable). |
| Does a hold count toward the daily limit? | Yes, from the moment the funds are held. |

### 6.2 FR-8 to FR-28 (Member 3)

| Question / ambiguity | Decision |
|---|---|
| What happens to a payment in Unknown? | It is never refunded automatically. It is re-queried after 1, 5 and 15 minutes, then waits for the settlement file. |
| Is the service fee returned when a payment fails? | Yes. The whole held total (amount plus fee) is released to the wallet. |
| How long is an idempotency key valid and what is its scope? | Per customer, for 24 hours (configurable). |
| What if the same key is sent with a different body? | Rejected with a conflict error. |
| What if a callback arrives for an unknown reference? | It is ignored and logged. |
| Which wallet gets the agent's commission? | The agent's own wallet, when the payment completes. |
| Is the commission paid if the payment fails or is refunded? | No. |
| Which biller gets the promotion? | One biller chosen in configuration, with start and end dates. |
| How do we know a TopUpNet callback is authentic? | A shared secret or signature is checked (assumption, to confirm with the TA). |
| Is NetCo (daily batch CSV) implemented? | Optional. Not in Phase 2 unless time allows. |
| Does a blocked account keep its balance and history? | Yes. It cannot start new payments but can be viewed by support. |
| Who approves a refund of a completed payment (FR-27)? | A support employee. |
