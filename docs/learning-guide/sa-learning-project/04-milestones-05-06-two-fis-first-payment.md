# M5 + M6 – Simulate Two Financial Institutions and Execute Your First Successful Payment

> **Phase 2 · Lab and first payment** · Time box: 4–6 h · Prerequisite: [M3/M4](03-milestones-03-04-setup-and-run.md) (switch running, provisioning and P2P test passed) · Next: [T1 – FSPIOP](05-topic-01-fspiop.md)
> Reference reading: [04 §4](../04-business-use-cases.md#4-real-world-business-scenarios), [13 §2](../13-customization-and-extension.md#2-add-a-new-financial-institution-register-an-fsp)

---

## Why this matters to a solution architect

Onboarding a bank to a scheme is a sequence of **business decisions expressed as API calls**: participant name, currencies, risk limit, callback endpoints, contacts, prefunding, published identifiers. Doing it yourself gives you the exact onboarding checklist you will hand to a bank and a scheme operator.

## Learning objectives

1. Inspect how an FSP exists inside the hub: participant, accounts per currency, limit, endpoints, position.
2. Explain how the Testing Toolkit (TTK) *plays* an FSP, and the limits of that simulation.
3. Onboard two institutions of your own, **Bank A** (`banka`) and **Wallet B** (`walletb`).
4. Execute a P2P payment from Bank A to Wallet B, and prove its effect on the ledger.
5. Capture the identifiers you will trace in [M7](11-milestone-07-trace-transaction.md).

---

## Core concepts

### 1. What "an FSP" is inside the hub

| Object | Created by | Meaning |
| --- | --- | --- |
| `participant` | `POST /participants {name, currency}` | The legal participant (`banka`) |
| `participantCurrency` rows | Same call, per currency | One **POSITION** and one **SETTLEMENT** ledger account per currency |
| `participantLimit` | `POST /participants/{name}/initialPositionAndLimits` | `NET_DEBIT_CAP` value for the currency |
| `participantPosition` | Same call (`initialPosition`) | Running position (`value`, `reservedValue`) |
| `participantEndpoint` rows | `POST /participants/{name}/endpoints` | Callback URLs per resource type, plus notification emails |
| Settlement balance | `POST /participants/{name}/accounts/{id}` `recordFundsIn` | Prefunding recorded on the SETTLEMENT account |
| Published identifiers | FSPIOP `POST /participants/{Type}/{ID}` (by the FSP itself) | Directory entries in the oracle |

### 2. How the harness simulates institutions

| Simulation style | Institutions | How it works | Realism |
| --- | --- | --- | --- |
| **TTK as FSP** (standard profile) | `testingtoolkitdfsp` (payer), `payeefsp` (payee) | Their callback URLs point to the TTK (`http://mojaloop-testing-toolkit:4040`). The TTK's **callback rules** auto-answer as the payee. | Protocol-level only; no core banking |
| **SDK connector + TTK backend** (golden-path profile) | `payerfsp`, `payeefsp`, `testfsp1..4` | Callbacks go to each FSP's SDK Scheme Adapter, which calls the TTK as its "core banking backend" | Realistic integration architecture |

In this milestone you use the first style for your own two institutions. You will use the second style when you build an integration in [M9](14-milestone-09-build-fsp-integration.md).

---

## Lab M5 – Part A: inspect the two default institutions

Run from the repository root (PowerShell or Git Bash):

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/payeefsp/endpoints
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/testingtoolkitdfsp/accounts
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/testingtoolkitdfsp/limits
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/testingtoolkitdfsp/positions
```

The same picture from the database (tables verified against central-ledger v19.4.4 migrations):

```bash
docker exec mysql mysql -uroot -D central_ledger -e "SELECT p.name, pc.participantCurrencyId, pc.currencyId, lat.name AS account, pp.value AS position, pp.reservedValue FROM participant p JOIN participantCurrency pc ON pc.participantId=p.participantId JOIN ledgerAccountType lat ON lat.ledgerAccountTypeId=pc.ledgerAccountTypeId LEFT JOIN participantPosition pp ON pp.participantCurrencyId=pc.participantCurrencyId ORDER BY p.name, pc.currencyId, lat.name;"
```

**Record in your journal:**
- Which callback URL `payeefsp` has, and why that means the TTK answers on its behalf.
- The NDC of `testingtoolkitdfsp`, and its SETTLEMENT balance after funds-in.
- That each participant has a POSITION and a SETTLEMENT account **per currency** (`XXX`, `XTS`).
- The `Hub` participant's accounts (`HUB_MULTILATERAL_SETTLEMENT`, `HUB_RECONCILIATION`).

## Lab M5 – Part B: onboard Bank A and Wallet B

You will reuse the official onboarding collections with different inputs, through the TTK web UI. No repository file is changed. Menu names below match recent TTK UI versions and may differ slightly in yours.

1. Open <http://localhost:9660>.
2. **Load the environment.** In the TTK UI's environment manager (within Test Runner), import `docker/ml-testing-toolkit/test-cases/environments/default-env.json` from your disk. Then edit these input values in the UI (not on disk):

| Input | Value | Why |
| --- | --- | --- |
| `payerfsp` | `banka` | `participant_sender.json` creates `{$inputs.payerfsp}` |
| `payeefsp` | `walletb` | `participant_receiver.json` creates `{$inputs.payeefsp}` |
| `fromFspId` | `banka` | Payer identity in the P2P test |
| `toFspId` | `walletb` | Payee FSP used for ALS registration and quote destination |
| `toIdValue` | `27713809999` (any unused MSISDN) | Wallet B's customer |

Keep `CALLBACK_ENDPOINT_BASE_URL` = `http://mojaloop-testing-toolkit:4040`, so the TTK simulates both of your institutions.

3. **Import and run onboarding.** In the Test Runner, import the files `collections/provisioning/participant_sender.json` and `participant_receiver.json` and run them. Expect `201` responses (or the tolerated "already exists" codes on re-runs). The sender collection also performs **funds-in** for Bank A.
4. **Verify:**

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/banka
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/walletb/endpoints
```

**Deliverable D4 (part 1):** an onboarding sheet listing, for each institution, every API call made, in business terms: "registered participant", "set NDC 1,000,000 XXX", "registered N callback endpoints (list the types)", "funded settlement account", …

#### Alternative: the same onboarding from the command line (this is the method that was verified)

If the UI's environment editor gives you trouble, run the same collections with the TTK command-line runner and a **copy** of the environment file. Run from the repository root, PowerShell shown:

1. Create `..\my-lab\lab-ttk\banka-walletb-env.json`: a copy of `docker/ml-testing-toolkit/test-cases/environments/default-env.json` in which you change only the five inputs from the table above.
2. Run the two onboarding collections against it:

```powershell
docker compose --profile all-services --profile ttk-provisioning --profile ttk-tests run --rm --no-deps -v "$((Resolve-Path ..\my-lab\lab-ttk).Path):/opt/app/lab" ttk-provisioning sh -c "npm run cli -- -u http://mojaloop-testing-toolkit:5050 -l 2 -i collections/provisioning/participant_sender.json,collections/provisioning/participant_receiver.json -e lab/banka-walletb-env.json --report-format html --save-report true --report-target file://reports/lab-m5-onboarding.html"
```

Verified result: **72/72 assertions passed**. Both institutions got POSITION and SETTLEMENT accounts in `XXX` and `XTS`, an NDC of 1,000,000 with `alarmPercentage` 10, and 28 callback endpoints. Bank A was funded with 5,000,000 in each currency. (Git Bash: prefix with `MSYS_NO_PATHCONV=1` and use a `C:/…` style source path.)

> Note: the FX provider created by the standard onboarding is named **`perffxp`**. You'll see it in `GET /participants` next to `Hub`, `payeefsp` and `testingtoolkitdfsp`.

## Lab M6 – Execute your first successful payment (Bank A → Wallet B)

### Step 1 – The baseline payment (already done)

The `ttk-tests` container already executed `testingtoolkitdfsp → payeefsp` successfully (31/31 on a warm stack; see the cold-start note in [M3/M4](03-milestones-03-04-setup-and-run.md)). Open `reports/ttk-func-tests-report.html` and read each request and **callback**. That is your first successful payment.

### Step 2 – Your own payment, and an important lesson

Import `collections/tests/p2p.json` in the Test Runner and **duplicate** the test case. Before running it, understand this:

> When the TTK fakes a payee's `PUT /parties` callback, it fills `party.partyIdInfo.fspId` from its own config value **`FSPID`, which is `payeefsp`** ([spec_files/api_definitions/fspiop_2.0/callback_map.json](../../../docker/ml-testing-toolkit/spec_files/api_definitions/fspiop_2.0/callback_map.json) `bodyOverride`, and `FSPID` in [user_config.json](../../../docker/ml-testing-toolkit/spec_files/user_config.json)). The **header** `FSPIOP-Source` is correctly `walletb`. `p2p.json` takes the payee FSP for the quote and transfer **from the callback body**. Run unchanged, your "Wallet B" payment would quietly go to `payeefsp`.

In your duplicated test case, edit the quote and transfer requests: replace `{$prev.3.callback.body.party.partyIdInfo.fspId}` with `{$inputs.toFspId}` (in the quote's `payee.partyIdInfo.fspId` and the transfer's `payeeFsp`). Then run it.

**Expected:**
- ALS registration of `27713809999` → `walletb` (202 + callback)
- Party lookup (202 + callback; header source `walletb`)
- Quote (202 + callback with `transferAmount`, `ilpPacket`, `condition`)
- Transfer (202 + callback `transferState: COMMITTED`)

**This gotcha was reproduced during verification.** Running the *unchanged* `p2p.json` with the Bank A / Wallet B inputs gave **31/31 assertions passed**. Yet the ledger showed `RESERVED: banka +100` and `COMMITTED: payeefsp −100`, and Wallet B's position stayed at 0. With the two references replaced by `{$inputs.toFspId}` (a copied file, run with `-i lab/p2p-walletb.json`), the result was again 31/31, and this time `banka +100`, `walletb −100`.

> **Architect's lesson from the gotcha:** a passing test suite can still move money to the wrong institution. A payer FSP must route by the party information the *switch* delivered, and should check that the `FSPIOP-Source` of the `PUT /parties` matches the `fspId` in the body. Mismatches are a classic integration defect. Write the check into your integration test plan, and always assert ledger effects, not just callbacks.

### Step 3 – Prove the ledger effect

Capture the `transferId` from the TTK request details (the transfer body), then:

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/banka/positions
```

```bash
docker exec central-ledger wget -qO- http://127.0.0.1:3001/participants/walletb/positions
```

```bash
docker exec mysql mysql -uroot -D central_ledger -e "SELECT transferStateId, reason, createdDate FROM transferStateChange WHERE transferId='<transferId>' ORDER BY transferStateChangeId;"
```

**Expected (verified):** Bank A's position rises by the amount (100 with the default `amount` input) and Wallet B's falls by the same amount. The state history is `RECEIVED_PREPARE → RESERVED → RECEIVED_FULFIL → COMMITTED`, all within the same second on a warm stack.

### Step 4 – See it visually

Open the mobile simulator (<http://localhost:9660/mobilesimulator>) and send a payment. Watch the live sequence diagram and map each arrow to the M2 sequence.

---

## Architect's lens

- **The onboarding checklist** you just executed is the skeleton of a bank's technical onboarding: participant and currencies → NDC → endpoints (HTTPS, reachable from the hub) → contacts → prefunding → alias registration → certification test.
- **Decisions the bank must make before onboarding:** currencies; NDC level and prefunding source; callback hosting (DMZ); which aliases to publish; operational contacts for NDC alerts.
- **Simulation limits:** the TTK proves protocol behaviour, not core-banking behaviour (holds, postings, failures). That comes in M9/M10.

---

## Checkpoint

<details><summary>1. How many ledger accounts does a new participant get for one currency, and of which types?</summary>
Two: a POSITION account and a SETTLEMENT account (participantCurrency rows with different ledgerAccountTypeId).
</details>

<details><summary>2. Why does the TTK answer for walletb?</summary>
Because walletb's callback endpoints, registered during onboarding, point to the TTK URL, so the switch delivers walletb's requests to the TTK, whose callback rules reply as the payee.
</details>

<details><summary>3. After a committed transfer of 100 from banka to walletb, what are the position changes?</summary>
banka +100 (reserved at prepare, then committed); walletb −100 at commit.
</details>

<details><summary>4. What check would have caught the fspId mismatch in a real integration?</summary>
Validating that the PUT /parties header FSPIOP-Source equals party.partyIdInfo.fspId, and asserting the payee FSP of the quote and transfer in integration tests.
</details>

---

## Deliverable D4 – Onboarding sheet and first-payment evidence

1. The onboarding sheet (Part B).
2. Evidence of your Bank A → Wallet B payment: IDs, the callbacks' key fields, positions before and after, and the state history.

Next: [T1 – FSPIOP protocol](05-topic-01-fspiop.md).
