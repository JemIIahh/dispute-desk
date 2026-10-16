# Decision policy

Every decision is made by deterministic code. The LLM analyst only reads text and reports what it found. All numbers below are **configurable policy assumptions**, not real win rates.

## 1. Triage: when the agent stays out

Airwallex handles some disputes itself. The agent leaves these alone and shows why:

- small pre-chargebacks, which Airwallex accepts automatically;
- disputes on payments that were already fully refunded;
- pre-arbitration;
- eligible Visa 10.4 chargebacks that Airwallex challenges automatically under Compelling Evidence 3.0.

The agent acts only on disputes whose status is "requires response", and never after the deadline has passed.

## 2. Evidence checklists by reason code

| Reason | What the agent looks for |
|---|---|
| 10.4 "I didn't authorise this" | signed approval receipt, passed step-up, IP or device match with earlier orders, shipping address match, at least two earlier undisputed orders aged 120 to 365 days |
| 13.1 "It never arrived" | carrier proof of delivery, shipping address, promised delivery date, terms shown at purchase |
| 13.6 "Credit not processed" | refund proof, the refund policy shown at purchase, customer communication |

How complete the checklist is maps to a likelihood tier. Earlier orders that were themselves disputed never count as good history, and unknown devices or addresses never count as a match.

## 3. Cost rule

Fighting a dispute is only worth it when the expected recovery beats what it costs:

- **At chargeback stage** the fee is already paid, so challenge only if `likelihood × amount > labour cost`.
- **At pre-chargeback stage**, challenge only if `likelihood × amount > (chargeback fee − resolution fee) + labour cost`.
- **For 13.6**, judge the merits first: if a credit is owed, accept. The cost rule only breaks ties.

## 4. Refusal rules: never challenge when

- the receipt is missing, or its signature does not verify;
- the receipt's terms differ from the order (items, quantity, merchant or total);
- the receipt belongs to a different payment;
- the receipt was approved after the order was created;
- the analyst flags an injection attempt or a contradiction in the evidence;
- the evidence is still incomplete at the cutoff and the cost rule fails.

## 5. Hold for a human

"Escalate" is not something a merchant can send to a card network, so the agent's third option is **hold for a human**. A held case is re-evaluated whenever new evidence arrives. Because evidence can be submitted only once, the agent decides at the deadline minus a safety margin, using whatever evidence exists by then.

## 6. Human approval

- Every challenge needs a human approval, and so does any accept above a threshold.
- The approval is bound to a hash of the case, the action, the amount, the evidence pack's hash and the deadline. If any of these change after the human has seen them, the approval is void and they are asked again.
- If the deadline passes while a case is waiting for approval, it closes as expired and the loss is booked, so nothing is ever sent late.

## 7. Sending

- **One terminal action per dispute:** accept or challenge, never both, never twice. The agent never accepts and refunds the same dispute.
- **Idempotent requests:** each action uses a deterministic request ID that is stored before sending. After an unclear failure, the agent reads the dispute's state before retrying.
- **The ledger** books chargeback fees, recoveries, losses and accepted losses, so the net effect of the agent's decisions is visible.

## 8. Analyst contract

- **Input:** the case facts, plus untrusted documents (customer emails, carrier notices) clearly delimited as data.
- **Output:** schema-checked JSON with claims (field, value, exact quote, source), flags (injection, contradiction, terms changed and others, each with a quote) and a short summary.
- **Never:** amounts, decisions or actions. A claim without an exact source quote is rejected. Instructions found in documents are never followed. The analyst has no tools.
