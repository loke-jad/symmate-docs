# Confidential Symmate Architecture

**Status:** High-level design, ready to integrate.
**Owner:** Justadestination LLC
**Relates to:** Symmates, parlourcraft, Webcraft, Convocation
**Last reviewed:** 2026-06-20

---

## 1. Purpose

Define how a customer-facing Symmate represents a small business **without ever holding data that business or its customers don't want exposed.** The agent is treated as untrusted by design; confidentiality is an architectural property, not a behavioural promise.

This doc is the privacy/trust layer for the parlourcraft → Symmates path. It does not change the product surface — it defines what the product is allowed to know.

---

## 2. Core principle (the load-bearing invariant)

> **The LLM is removed from the trusted set. The trusted component is the deterministic substitution layer.**

Everything else follows from this. We do not ask the model to *decide* what is safe to reveal — models are not security boundaries, because instructions and attacks arrive through the same channel (text). Instead, the front agent **never receives the real value in the first place.** You cannot leak what you do not hold.

The property holds **if and only if**:
1. The substitution function is correct, and
2. The real secret never enters the model's context.

Those two conditions are the entire security argument. Protect them; everything downstream is detail.

---

## 3. Threat model

### Defended by construction
- **Prompt injection / social engineering of the front agent.** The agent has only substituted values to give. Output is worthless to an attacker.
- **Compromise of the agent's context, logs, or transcript store.** Contains substituted values only — low intrinsic value.
- **Interception of the customer ↔ agent channel.** Carries substituted values; real values were encrypted client-side before leaving the browser.

### Explicitly NOT defended (named, not hidden)
- **Free-text PII volunteered in prose** ("book it under my partner, Jane Okafor"). Caught only by entity-matching, which is probabilistic. See §6.
- **Compromise of the substitution layer itself.** It is the trusted component; if it is subverted, the property fails. This is where defensive effort belongs.
- **Compromise of the back-of-house vault / index** (Phase 2). The concentration target. Key custody is the control. See §7.
- **Silent back-of-house failure after the agent has confirmed.** An integrity problem, not a confidentiality one. See §5.

A claim of "bullet-proof" is only meaningful against this defined model. Outside it, no claim is made.

---

## 4. Architecture — route by data class

Not one mechanism. Data is routed by class, and each class takes the cheapest correct path.

### 4a. Payment data → straight to the processor
- Card data flows **browser → payment processor (Stripe hosted fields / Elements)**. It never touches Justadestination infra.
- We receive a processor token back. Lightest PCI tier.
- **The substitution layer is not used for cards.** Direct-pipe is strictly better here — there is nothing to substitute because we never see it.

### 4b. Non-payment PII → client-side substitution
- Real values (name, contact, preferences) are substituted **in the browser**, before transmission.
- The agent sees only opaque handles / substituted values.
- Real values travel **encrypted** to back-of-house, never through the agent.

### 4c. Back-of-house (sealed)
- Receives encrypted real values, performs the real transaction, emails the receipt **directly** — bypassing the agent entirely.
- The front Symmate "goes through the motions" but is not connected to the transaction path.

```
                 ┌─────────────────────────────────────────────┐
  Customer ──────┤ BROWSER                                       │
   (real data)   │  • card → processor (hosted fields)           │
                 │  • PII  → substitution → encrypt              │
                 └───────────┬───────────────────┬───────────────┘
                             │ substituted        │ encrypted real
                             ▼                     ▼
                   ┌──────────────────┐   ┌────────────────────┐
                   │ FRONT SYMMATE    │   │ BACK OF HOUSE       │
                   │ (untrusted)      │   │ (sealed)            │
                   │ sees fakes only  │   │ • real transaction  │
                   │ no real data     │   │ • receipt by email  │
                   └────────┬─────────┘   └──────────┬──────────┘
                            │   confirmation gated    │
                            └─────────◄───────────────┘
                                  (real outcome only)
```

---

## 5. Confirmations

The agent may present a seamless "you're booked" — **only after a real callback from back-of-house confirms settlement.** It may look, to the customer, exactly as if the agent did the work.

What it must never do: narrate success on faith. The UX can be theatre; the **success signal must be real**, or a customer walks away confidently misinformed about a transaction that silently failed.

---

## 6. The free-text gap (honest limit)

Substitution catches **structured fields** deterministically (`name on booking: ___`). It does **not** catch real data buried in prose, because that requires NER/entity-matching, which has false negatives. A miss is not a near-miss — it is the real value sitting in the agent's context as cleartext.

**Fix is to shrink the surface, not trust the matcher:**
- Force PII into **structured fields** wherever the flow allows. This is the actual mitigation.
- For PII that must be free text: treat confidentiality as **best-effort, not provable.**
- **Bound the blast radius:** memory is scoped **per-customer/per-session**, so a matcher miss leaks one session — never the index. Scoping is what makes the residual gap survivable.

> THE MATCHER IS A MODEL; A MODEL IS NOT A SECURITY BOUNDARY. Structured-field capture is the boundary. NER is a convenience on top of it.

---

## 7. Concentration risk & the index (Phase 2 only)

Scattered encrypted fields are low-value to steal. A clean, entity-resolved **index** of every customer bond is the highest-value single target in the whole design. Confidentiality-at-the-agent (solved) and concentration-at-the-vault (separate problem) are different concerns.

- **Index lives on the customer's container** — per-business-client isolation *(ASSUMPTION — confirm, see §10)*. This **partitions** the concentration risk; it does not remove it. Inside one container the index is still the prize.
- **Read-control:** define who/what can query the entity-resolved index, under what auth.
- **Key custody:** three-key split *(provisional — inferred Shamir 2-of-3; NOT yet validated against the crypto doc)*. No single share leak opens the vault.
- **Worst case in the entire design is the vault key, not the agent.** No amount of front-end substitution matters if the index key is one compromise from handing over the resolved set.

---

## 8. Component glossary

| Name | What it is | Trust |
|---|---|---|
| **Symmate (front)** | Customer-facing agent instance. Goes through the motions; sees only substituted values. | **Untrusted by design** |
| **Substitution layer** | Deterministic, client-side. Swaps real values for handles before transmission. | **Trusted** (the one thing that must be correct) |
| **Back-of-house** | Sealed transaction path. Decrypts, transacts, emails receipt. Not reachable from the agent. | Trusted |
| **Wyrmpi** *(spelling — confirm)* | Ingest/extraction-and-indexing pass fired at `snap2personality` creation. Operates **only on customer-provided data, on our infra** (confirmed metaphor, not autonomous reach). Encrypts and indexes the bond's information. | Trusted; Phase 2 |
| **snap2personality** | The creation event that triggers Wyrmpi ingest. | — |
| **Token memory** | Stable mapping (`NAME_07 → real`) the agent **never resolves**. Enables cross-session personalization without identity exposure. | Vault-side; agent-opaque |

---

## 9. Phasing — mapped to revenue

The architecture splits cleanly along the line between *what closes one paying bar* and *what scales to many.* Do not build Phase 2 to win Phase 1.

### Phase 1 — Sellable now (closes one client)
Delivers the **full sellable privacy story**: *"We never hold your card. Our agent literally cannot leak your data."*
- Stripe hosted fields (payment never touches infra).
- Forward-only Symmate: takes name / time / request as **structured fields**, forwards.
- Client-side substitution for those fields.
- Back-of-house transacts + emails receipt.
- Confirmation gated on back-of-house callback.
- **No Wyrmpi. No vault. No three-key split. No cross-session memory.**

> You get the entire v1 confidentiality property from Stripe + a forward-only agent. The worm and the split vault are not on the path to one bar owner saying yes.

### Phase 2 — Scale (multi-customer, persistence)
Solves **concentrated PII across many customers** — a problem that exists only *after* Phase 1 sells.
- Wyrmpi ingest at `snap2personality`.
- Per-container encrypted, entity-resolved index.
- Three-key split vault *(pending crypto-doc review)*.
- Cross-session token memory (`NAME_07 → real`, agent-opaque).

### Cross-cutting (both phases)
- LLM untrusted; substitution layer trusted.
- PII forced into structured fields; free text is best-effort + scoped.
- Per-container isolation bounds blast radius.

---

## 10. Open decisions

1. **Container scope** — per-business-client (assumed) or per-end-user? Changes the isolation claim in §7.
2. **Free-text scrubbing** — does the substitution layer attempt NER on prose, or do we constrain inputs to structured fields and accept the residual? (§6)
3. **Three-key split** — validate against the crypto doc. Params, ceremony, recovery, blast radius on single-share leak. (§7)
4. **Wyrmpi write-auth** — what authenticates Wyrmpi's write into the bond vault, so a compromised ingest cannot poison or overwrite other bonds? (Phase 2)
5. **Red-team target** — point the Symmates antagonist simulator at the **substitution layer** (the trusted component), not the agent. The agent failing the test proves nothing; the layer surviving it is the claim.

---

## 11. One-paragraph integration summary

The Confidential Symmate Architecture slots into the Symmates/parlourcraft path as its trust layer. Phase 1 is the minimum that ships a paying client and already carries the complete privacy pitch using only Stripe hosted fields and a forward-only, untrusted agent. Phase 2 — Wyrmpi ingest, per-container entity-resolved index, three-key vault, cross-session token memory — is deferred until the concentration problem actually exists, i.e. after the first bar converts. The durable principle across both phases: the model is never trusted; the deterministic substitution layer is.
