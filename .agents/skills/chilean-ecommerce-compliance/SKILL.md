---
name: chilean-ecommerce-compliance
description: Chilean legal compliance for B2C e-commerce; consumer protection, tax documents, VAT (Impuesto al Valor Agregado, IVA), and personal data. Use for any storefront, checkout, tax document, marketing, or personal data design or review for a store selling to customers in Chile.
license: MIT
metadata:
  author: rainingdaemons/agents
  version: "1.0"
  source: Generated using as reference Ley 19.496, Ley 825, Resolución Ex. SII N° 53, Ley 19.628, Ley 21.719, Ministerio de Economía Decreto 6 and Ley 830.
---

# Chilean E-commerce Compliance

Use this as the entrypoint when a task is about building, operating or reviewing an online store that sells to consumers in Chile. It is an implementation overlay: it turns the statutes into storefront, checkout, back-office and data-handling requirements, and says which article backs each one. It is **not legal advice**; escalate open interpretation questions to an abogado (consumer/data) or contador (tax).

> **Timing note.** Ley 21.719 enters into force on 01-DEC-2026 (24th month after publication on 13-DEC-2024; Ley 19.628 lapses on 30-NOV-2026). With an MVP launch targeted 8–12 weeks from kick-off, the site will be operating under the new law within its first months, so this checklist treats full Ley 21.719 compliance as a launch requirement, not a later upgrade. The 12-month SME amnesty window (Ley 21.719 transitorio sexto, warnings instead of fines for *empresas de menor tamaño*) is a mitigation, not an excuse to defer.

## Norms covered

| Norm | What it governs for an online store |
|---|---|
| Ley 19.496 (Protección de los Derechos de los Consumidores) | Pre-contract information, price, advertising, promotions, adhesion contracts, retracto, legal guarantee, SERNAC enforcement |
| Decreto 6/2021 Economía (Reglamento de Comercio Electrónico) | Mandatory information on e-commerce platforms; seller vs. operator roles; checkout summary and confirmation |
| DL 825 (Ley de IVA) | 19% IVA, who is a taxpayer, when tax accrues, documents, credit notes, monthly filing |
| DL 830 (Código Tributario) | Inicio de actividades, tax-document authorization, SII audit powers, record keeping, penalties (art. 97), payment-traceability and bank-reporting rules |
| Res. Ex. SII N° 53/2025 | Duty to hand over the printed/virtual boleta or payment voucher in **in-person** sales |
| Ley 19.628 (until 30-NOV-2026) | Current personal-data regime |
| Ley 21.719 (from 01-DEC-2026) | New personal-data regime, rights portal, security, breach notification, fines, Agencia de Protección de Datos Personales |

## Source currency — read before relying on any rule

The attached texts are not all equally current. Treat this table as part of the skill.

| Source | Version in corpus | Caveat |
|---|---|---|
| Ley 19.496 | BCN version of 20-APR-2021 | **Stale.** Ley 21.398 ("Pro Consumidor", D.O. 24-DEC-2021) later: extended the legal guarantee from 3 to **6 months** (from 24-MAR-2022); reworked retracto (art. 3 bis b, new letter c for in-person purchases without direct access to the good); added the pro-consumer interpretation rule (art. 2 ter); and provided that the voluntary warranty need not be exhausted first (art. 20 e). A separate retracto regulation (D.O. 27-AUG-2024, in force 28-FEB-2025) lists goods excluded from retracto. **Re-pull the current text from BCN before finalizing legal copy.** Where this skill quotes 3 months or the 45-day refund period from the attached text, verify. |
| DL 825 | 05-MAR-2026 | Current at corpus date |
| DL 830 | 11-JUL-2025 | Current at corpus date; art. 92 ter threshold set transitorily for 2025–2026 |
| Decreto 6 | Única, 24-MAR-2022 | In force since 24-MAR-2022 |
| Res. Ex. SII 53 | 29-APR-2025 | In force (see Part C) |
| Ley 19.628 | Until 30-NOV-2026 | Replaced in full by Ley 21.719 from 01-DEC-2026 |
| Ley 21.719 | Deferred effect 01-DEC-2026; last modified 05-FEB-2026 (Ley 21.806) | Agencia instructions and regulations may be issued after the corpus; check them |

**Not in the corpus (verify separately):** the retracto regulation; SII resolutions 45/2003, 99/2019, 74/2020, 176/2020, 76/2021 (electronic boleta and payment-voucher mechanics); Ley 20.416 (definition of empresa de menor tamaño); Código Civil rules on capacity of minors; Agencia instructions.

**On "DL 3.063":** DL 830 is the Código Tributario. DL 3.063 is a different norm (Ley de Rentas Municipales: patente comercial municipal) and is not in the attached corpus. Treat the municipal patente as an open item for the contador and do not infer its content from this skill.

## When to Use

- Building or reviewing a storefront, product page, cart, checkout, order confirmation, returns flow or footer/legal pages for Chile
- Designing invoicing: boleta/factura emission, credit notes, monthly IVA, accounting exports
- Adding marketing: emails, SMS/WhatsApp, pixels, discounts, giveaways, influencer seeding
- Adding any feature that collects, stores, shares or profiles customer data
- Running physical sales (pop-ups, events, store) alongside the site
- Selecting vendors (payments, hosting, email, analytics, couriers, invoicing provider)

## How It Works

Run these decision gates in order for every task:

1. **Role.** Is the business a *Vendedor* only, or also an *Operador* of a marketplace (Decreto 6 art. 3)? Does it operate a digital intermediation platform for third-party sellers (DL 825 art. 3 bis)? For data: who is *responsable* and who are *encargados* (Ley 21.719 arts. 2, 15 bis)?
2. **Special-law carve-out.** Another sector law governing the same matter displaces these rules (Ley 19.496 art. 2 bis; Decreto 6 art. 2). For ordinary apparel retail, assume none, but note exceptions.
3. **Consumer flow.** Which Part A requirements apply at this step (listing, price, checkout, delivery, post-sale)?
4. **Tax event.** Does this create a sale, a return, a free distribution, a transfer of goods or a refund? Which document and when (Part B)?
5. **Channel.** Is any part of this in-person (Part C)?
6. **Data.** What personal data, for which purpose, under which legal basis, kept how long, shared with whom, stored where (Part D)?
7. **Evidence.** Can the business prove it afterwards? The burden of proof is on the business for consent (Ley 21.719 art. 12), lawfulness (arts. 3 a, 13), security measures (art. 14 quinquies) and international transfers (art. 28); for tax, on the taxpayer (Código Tributario art. 21).

---

## Part A — Consumer law (Ley 19.496 + Decreto 6)

### A1. Seller identification (Decreto 6 art. 7)
Show before purchase, on the site: name/razón social/nombre de fantasía; RUT; legal address; email, phone or other contact channel; the channel the seller will use to contact the consumer; for legal entities, the legal representative.

### A2. Product information (Decreto 6 art. 9; Ley 19.496 arts. 1 N°3, 32, 33)
Before purchase, in clear Spanish: nature/purpose of the product; place of manufacture/origin; brand, model, sub-model; conventional warranty if offered (term, conditions, how to claim); reference attributes (dimensions, **size**, color, weight, units, material, content); any other relevant characteristic. Claims must be verifiable and non-misleading. "Garantía/garantizado" may be used only if the page states what it consists of and how to enforce it (art. 33). Used, defective or refurbished goods must be disclosed before purchase (art. 14).

### A3. Price and total cost (Ley 19.496 arts. 18, 30, 32; Decreto 6 arts. 3 N°5, 10, 11)
- Display price in CLP, **including IVA and all taxes**.
- Show *Costo total* before contracting: price + taxes + shipping + commissions + any other consumer-borne charge; **shipping shown as a separate line** when included.
- Foreign-currency or indexed prices: say so and show the CLP value at contracting.
- Subscriptions: show total cost per billing period.
- Charging more than the displayed, informed or advertised price is an infraction (art. 18). Price shown at checkout must match the catalog.
- Extra charges for accessories/financing are void in adhesion contracts unless optional, separately listed and accepted case by case (art. 16 b). **No pre-selected add-ons.**

### A4. Stock and availability (Decreto 6 art. 12)
State clearly and visibly when an item is out of stock or unavailable. Once the contract is formed, fulfil it. Made-to-order/pre-order items are exempt from the stock rule but must be labeled as such (drops: say "preventa/por encargo" and the delivery window).

### A5. Delivery (Decreto 6 art. 13)
Before the contract is formed: available delivery/pick-up methods; delivery cost itemized; pick-up date or delivery time range **stating business vs. calendar days**; a mechanism to query delivery status and file complaints. If the operator performs delivery, say so.

### A6. Retracto (Ley 19.496 art. 3 bis; Decreto 6 art. 14)
- Distance purchases: consumer may withdraw without cause within **10 days** from receipt of the goods (services: from contracting and before performance). If the seller failed to send the written confirmation (art. 12 A), the period extends to **90 days** (attached text).
- Display the right prominently **before contract and payment**, plus how to exercise it. If the seller invokes a legal exclusion, that must be shown equally prominently and in advance (Decreto 6 art. 14).
- Refund without withholding costs, as soon as possible and within the statutory period (45 days in attached text; verify) from the withdrawal notice. Return conditions: original packaging, labels, certificates, manuals; not allowed where the item deteriorated by consumer's act.
- **Default for apparel: grant retracto.** Check the 2024 regulation's list of excluded goods before excluding anything, and never rely on a blanket "no returns on sale items".
- Retracto, legal guarantee and voluntary warranty are cumulative, not alternatives.

### A7. Support channel (Decreto 6 art. 15)
Publish how to ask questions, complain, request exchanges/returns.

### A8. Terms and conditions / adhesion contracts (Ley 19.496 arts. 12 A, 16, 17; Decreto 6 arts. 16–17)
- Visible link to T&C before contracting; consumer must be able to **store or print** them; consent is not formed otherwise (art. 12 A). Merely visiting the site binds nobody; silence is not acceptance (art. 3 a; Decreto 6 art. 4).
- Void clauses (art. 16): unilateral change/termination at the seller's sole discretion; shifting administrative errors to the consumer; reversing the burden of proof; absolute liability exclusions; blank spaces; any clause creating a significant imbalance contrary to good faith. If arbitration is designated, inform the consumer of the right to recuse the arbitrator.
- Spanish, legible; consumer must receive a full copy (art. 17).
- Subscriptions/recurring: say when and how to cancel (Decreto 6 art. 17).
- Local-court jurisdiction cannot be waived by contract (art. 50 A).
- Rights are not waivable in advance (art. 4).

### A9. Checkout and confirmation (Ley 19.496 arts. 12 A, 32; Decreto 6 art. 18)
- Before payment, show a **summary**: product(s) and characteristics, *Costo total*, delivery info (art. 13 paras. 1–2).
- Explain the steps to conclude the contract, whether the contract is archived and accessible, a postal or electronic address of the seller, and **technical means to identify and correct input errors** before sending (art. 32).
- After the contract is formed, send a **written confirmation with a complete, legible copy of the contract** through a channel announced in advance (email is fine) (art. 12 A; Decreto 6 art. 18). A marketplace may send it, but the seller remains responsible if it is not sent.

### A10. Advertising, promotions, offers (arts. 1 N°4, 7–8, 28, 28 A, 28 B, 33, 35, 36; Decreto 6 arts. 19–20)
- Objective conditions in advertising become part of the contract (Decreto 6 art. 19). Misleading advertising on price, characteristics, suitability, warranty or environmental claims is an infraction (art. 28).
- **Offers** are temporary reductions vs. the establishment's usual price (art. 1 N°8). A struck-through "precio normal" must be the real usual price. Do not fabricate scarcity or before/after prices.
- Every promotion/offer: state its **bases and duration**; depositing the bases with a notary is not enough (art. 35). Refusal to honor is enforceable by court order.
- Contests/giveaways: publish number/amount of prizes and claim period; publicize results (art. 36).
- Promotional emails: subject line stating the topic, identity of the sender, and a **valid address to request suspension**; further sends are then prohibited. Other channels (SMS, calls, messaging, postal): provide an expedient opt-out (art. 28 B).
- Do not create confusion with competitors' names, marks or signs (art. 28 A).

### A11. After the sale
- **Legal guarantee** (arts. 19–22): for defects/non-conformity the consumer chooses repair, replacement or refund. Period: **6 months** after Ley 21.398 (attached text says 3; verify). Seller, manufacturer or importer may be claimed against. Note: guarantee does not cover "didn't like it / wrong size" — that is retracto or voluntary exchange policy; keep the three flows separate in UI and copy.
- Content shortfall vs. label: replacement/refund (art. 19).
- Issue the credit note and refund through the payment channel in sync (see B2).
- Do not refuse a sale without justification (art. 13).
- If financing is offered directly by the store, art. 37 (credit information, cost caps on collection) applies; installments via card issuer are the issuer's regime. Flag to lawyer.

### A12. Marketplace mode (Decreto 6 art. 8; DL 825 art. 3 bis)
If third parties sell on the platform: the operator must state its role and its contractual obligations before purchase, passes through seller info and conditions, and must send the information the seller supplied. Mark this as a scoping decision up front; it changes tax, consumer and data roles.

### A13. Enforcement exposure
Generic fine up to 300 UTM (art. 24); false/misleading advertising up to 1,500 UTM (2,250 UTM if health, safety or environment); refusal to give SERNAC website records in digital format on request up to 750 UTM (art. 58); infractions prescribe in 2 years (art. 26). Mitigation: reparation before judgment, self-report, cooperation, an SERNAC-approved compliance plan, no prior sanction (art. 24). Collective actions are possible (art. 51). Decreto 6 breaches are sanctioned under art. 24 (Decreto 6 art. 22).

---

## Part B — Tax (DL 825 + Código Tributario DL 830)

### B1. Registration
- Register in the RUT and file *inicio de actividades* **before** starting taxable activity (DL 825 art. 51; Código Tributario arts. 66, 68). Report changes (address, legal representatives, capital, partners, etc.) within 2 months; failure is sanctioned under art. 97 N°1 and extends audit prescription (art. 68).
- False data in the initial declaration or to obtain document authorization is a crime (art. 97 N°23).
- Keep the registered email current: SII notifies by email and notification is valid even if you do not see it (arts. 11, 11 bis).
- Payment-medium providers and digital platforms must demand proof of inicio de actividades from the persons they serve (art. 68, last paragraphs): expect to be asked when contracting a payment gateway.
- *Pequeño contribuyente* simplified regime is **not available to personas jurídicas** (DL 825 art. 35). An SpA cannot use it.

### B2. Documents
- **B2C: boleta electrónica de ventas y servicios; B2B: factura electrónica** (DL 825 art. 53; Código Tributario art. 88). Obligation applies even to exempt sales (art. 52). Documents must be electronic (art. 54); paper only under legal exceptions.
- Payments through electronic means: the comprobante/recibo has the value of a boleta in the form SII determines, and payment and boleta systems must be integrated so that paying generates the document (art. 54).
- **Timing.** Tax accrues on the date of the boleta or on delivery if earlier (art. 9 a); boletas are issued at real or symbolic delivery; services at the moment payment is received (art. 55). Choose and document an issuance rule for ecommerce (on payment vs. on dispatch/delivery) with the contador and confirm it against SII e-boleta instructions (not in corpus).
- **Goods in transit** must travel with guía de despacho, factura or boleta, exhibitable on request (art. 55; Código Tributario art. 97 N°17). Courier flows need this documented.
- **Returns, discounts, retracto, voided sales:** issue a **nota de crédito** (art. 57). Deduct the IVA only within the 6-month window (arts. 21 N°2, 70); request tax refund/annulment within 30 days of the sale being undone (art. 70). Tie credit-note issuance to the refund workflow.
- Keep duplicates of facturas and originals of boletas **6 years** (art. 58). Electronic Registro de Compras y Ventas is built by SII from your documents; reconcile it monthly (art. 59).
- Show IVA separately in invoices and boletas where required (art. 69), while the consumer-facing price is IVA-inclusive (Ley 19.496 art. 30).
- Documents must be issued in the exact amount charged. Over-billing IVA is still owed to the Fisco unless corrected by credit note (art. 22).

### B3. Monthly filing and payment
- IVA 19% (art. 14). Declare and pay by the 12th of the following month (20th for online filers per DTO 1001 referenced in the text) (art. 64). Filing is required **even with no taxable operations** until activities are formally ended (art. 65).
- IVA credit requires valid documents recorded in the books and a link to the business (arts. 23, 25). Credit is denied on false or non-compliant invoices, and payment-method proof can save it (art. 23 N°5). Purchases for the owners' personal use do not qualify.
- Late payment: interest/reajuste (Código Tributario art. 53) plus a 10% fine +2% per month up to 30% for late remittance of withheld/recargo taxes (art. 97 N°11; higher if detected in audit).

### B4. Payment traceability
Code Trib. art. 92 ter: above the threshold (set at 135 UF for 2025 and 2026; check the current value later), purchases/sales must be paid by a traceable means or be backed by a document recording the payer's identity or a factura; cash payments must be noted on the document. Card/transfer online is compliant by design. At pop-ups, define a rule for large cash sales and no splitting of payments.

### B5. Operational traps for a streetwear/drop business
- **Free distributions for promotion** (giveaways, sorteos, influencer seeding, "gifts with purchase" for advertising) are treated as sales for IVA, valued at the assigned or market value (DL 825 art. 8 d; art. 16 b). Exceptions exist for items fitting art. 31 N°3 of the Ley de la Renta. Do not ship free goods without an accounting rule.
- **Exports:** exported goods are exempt (art. 12 D) and exporters can recover IVA (art. 36); require export documentation before treating a sale as exempt.
- **Imports of stock:** imports are taxed with IVA at customs on the customs value plus duties (arts. 8 a, 16 a). Personal-use remote purchases from abroad up to USD 500 have their own rule (art. 4 last para.); do not use it for commercial stock.
- **Foreign SaaS/ads/hosting invoices:** IVA treatment depends on the provider's registration under the simplified regime for non-residents (arts. 8 n, 11 e, 35 A–35 I); confirm with the contador.
- **Marketplace:** if third-party sellers are IVA taxpayers, art. 3 bis does not make the operator taxable for those sales; otherwise the operator is treated as the seller.

### B6. Bank accounts and reporting
Use a dedicated business account for sales. Banks report to SII accounts that receive many deposits from many different people (Código Tributario art. 85 ter: more than 50 credits from 50+ distinct payers in a day/week/month, or 100 from 100 in a semester) and movements above 1,500 UF (art. 85 bis). Collecting customer transfers in a personal account is a red flag in that data, and banks also require the entity's inicio de actividades (art. 89).

### B7. System design implications for a self-built platform
- SII may examine technological accounting systems directly, through access profiles, and may request evidence of correct operation (Código Tributario art. 60 bis); obstruction carries fines (art. 97 N°6). Keep append-only order/document logs, reproducible exports and an owner for technical SII requests.
- The taxpayer proves the truth of declarations (art. 21). Retain order, payment, shipment, credit-note and refund evidence for the audit window (3 years, or 6 if no declaration or maliciously false; art. 200). Align retention with the data-protection schedule (Part E).
- Never edit or delete issued tax documents. Correct via credit/debit notes.

### B8. Penalty map
| Conduct | Provision | Exposure |
|---|---|---|
| Not issuing boleta/factura, or not sending electronic information; fractioning sales | Cód. Trib. art. 97 N°10 | Fine 50%–500% of the operation (min 2 UTM, max 40 UTA) **and closure up to 20 days, including the website or the digital sales channel**; repeat: domain/payment-provider access suspension request and prison for reiteration |
| False/malicious declarations, false documents | art. 97 N°4 | 100–300% of tax defrauded plus imprisonment |
| Late remittance of IVA | art. 97 N°11 | 10% +2% per month, cap 30% |
| Moving goods without guía/factura | art. 97 N°17 | 10%–200% of a UTA; vehicle stopped |
| Not demanding the boleta (consumer) | art. 97 N°19 | Up to 1 UTM per boleta |
| Residual breaches | art. 109 | 1%–100% of a UTA |

### B9. Open item: municipal patente
Operating a business generally requires a municipal patente; this is governed by DL 3.063, which is outside the corpus. Add to the launch checklist for the contador/lawyer.

---

## Part C — Res. Ex. SII N° 53/2025 (in-person sales)

Applies to taxpayers issuing boleta electrónica for sales and services made through **in-person channels** to final consumers (stores, pop-ups, events, showrooms).
- Cash or bank transfer: hand over the **printed or virtual representation of the boleta**.
- Debit, credit, prepaid or other electronic payment: hand over the boleta representation **or** the electronic payment voucher, depending on your document-issuance model.
- "Virtual delivery" = making it available via email, SMS, messaging apps, photo, NFC, QR or equivalent, and the seller must be able to **display** the documents on the spot for SII inspectors (own device, site, app or electronic folder).
- In force: 01-MAY-2025 for those with printing devices; 01-MAR-2026 for others. Both dates have passed.
- Failure to deliver: sanctioned under Cód. Trib. art. 97 N°10; other breaches under art. 109; consumers who refuse or do not demand the document may be sanctioned under art. 97 N°19.
- The resolution does not cover pure online sales. For the web channel, still deliver the boleta electronically (email and customer-account download) as the baseline, and confirm detail against Res. 74/2020 and 176/2020.
- Platform implication: if the system has a POS/event mode, it must emit the boleta at sale, support QR/email/SMS delivery, and store an offline-safe copy.

---

## Part D — Personal data

### D1. Until 30-NOV-2026 (Ley 19.628)
Everything below in D2 is a superset of what is needed today. Minimum current rules: processing only with the data subject's informed consent or legal authorization (art. 4; consent in writing and revocable without retroactive effect); purpose limitation and accuracy (art. 9); sensitive data only with consent/law (art. 10); duty of care and liability (arts. 11, 23); rights of information, rectification, deletion and blocking, free of charge, with a response required in 2 business days before a court action (arts. 12, 16); marketing opt-out (arts. 3, 12); commercial/debt data only under arts. 17–19. Design to D2 from day 1; do not build a separate interim mode.

### D2. From 01-DEC-2026 (Ley 21.719) — launch checklist

1. **Data inventory and roles.** Per processing activity: categories, purpose, legal basis, recipients, location, retention, controller/processor. Many items must be published (art. 14 ter). The business is *responsable* (art. 2 n); vendors acting on its behalf are *encargados* (art. 15 bis).
2. **Legal basis per purpose (arts. 12–13).**
   | Purpose | Basis |
   |---|---|
   | Create account, take and fulfil order, ship, refund | Contract / pre-contract (art. 13 c) |
   | Boletas/facturas, tax records | Legal obligation (art. 13 b; DL 825 art. 58, Cód. Trib. art. 17) |
   | Fraud prevention, security | Legitimate interest (art. 13 d), subject to opposition (art. 8 a) |
   | Newsletter, promos, SMS | **Consent** (art. 12); direct marketing is always opposable (art. 8 b) and Ley 19.496 art. 28 B requires unsubscribe |
   | Behavioral analytics, retargeting pixels, profiling | **Treat as consent-based**; profiling is defined in art. 2 w and monitoring is in scope under art. 1 bis c. The attached text has no cookie-specific rule, so this is a design recommendation: only strictly necessary cookies load without consent |
   | Abandoned-cart reminders | Pre-contract data (art. 14 d: delete/anonymize if not used for a contract); marketing reminders need consent |
3. **Consent UX (art. 12).** Free, informed, specific, prior, unambiguous; no pre-ticked boxes; one purpose per consent; withdrawal as easy as granting, free, any time (no retroactive effect). Consent bundled into a contract where the data are not needed is presumed not free. **Log** who consented, to what, when, which policy version (the business must prove it).
4. **Privacy policy (art. 14 ter).** Publish permanently and accessibly: policy, date and version; controller and legal representative (and DPO if any); contact channel for requests; categories of data and of data subjects; recipients; purposes and legal basis (and which legitimate interests); security measures; rights and the right to complain to the Agencia; international transfers and guarantees; retention period; source of the data; consent withdrawal; automated decisions/profiling and their logic.
5. **Rights portal (arts. 4–11).** Access, rectification, erasure, opposition, portability, blocking. Build: email/form intake; identity authentication per Agencia rules; acknowledgment; response **within 30 calendar days (extendable once by 30)**, in writing, with copies of evidence kept; a reasoned denial that tells the person they have **30 business days to complain to the Agencia**; temporary blocking answered **within 2 business days**; propagate changes to recipients. Rectification, erasure and opposition are free; access free at least quarterly; direct costs only beyond that. Build admin tools: export (structured, machine-readable), anonymize/delete, restrict-processing flag. Rights are inalienable and survive death for heirs unless the person forbade it (art. 4).
6. **Retention and erasure (arts. 3 c, 7, 14 d).** Define a schedule: keep what a legal obligation requires (tax documents, 6 years) in a restricted state; delete or anonymize everything else when the purpose ends. Erasure is not required where processing is needed for a legal obligation or contract or for legal claims (art. 7).
7. **Security (art. 14 quinquies).** Encryption in transit and at rest, pseudonymization where possible, access control and least privilege, resilient systems with tested restore, regular testing of controls, logs. The business must be able to **prove** the measures after an incident. Do not store card numbers/CVV; use the gateway's hosted fields (design recommendation).
8. **Breach response (art. 14 sexies).** Notify the Agencia by the fastest means and without undue delay when there is reasonable risk to data subjects; keep an incident register (nature, effects, categories, approximate number, measures); also notify affected people when the breach involves sensitive data, data of children under 14, or data on economic/financial/commercial obligations. Write the runbook before launch.
9. **Vendors and transfers (arts. 15 bis, 27–28).** For each vendor handling personal data (hosting/cloud, payment gateway, email/SMS, analytics, courier, e-invoicing provider, support tool): a contract with the art. 15 bis content (object, duration, purpose, data types, data-subject categories, rights/duties; no sub-delegation without written authorization; return/deletion at end; breach reporting to the controller), and for any storage or access outside Chile a transfer basis: adequate-country decision by the Agencia, adequate guarantees (model clauses, binding rules, certification), or a narrow exception (e.g., contract necessity, explicit consent for a specific non-habitual transfer). The business must be able to prove it. **Default: block any new vendor until contract and transfer basis are documented.** Check the Agencia's published country list and model clauses.
10. **Minors (art. 16 quáter).** Children (<14) need parental/guardian consent; adolescents (14–17) are treated like adults except that **sensitive data of those under 16** requires parental consent. Processing must follow the child's best interest and progressive autonomy. A youth-oriented streetwear brand should avoid profiling minors and decide on age gating; capacity to contract is a Código Civil question outside the corpus — escalate.
11. **Sensitive, biometric and geolocation data (arts. 16, 16 ter, 16 sexies).** Do not collect sensitive or biometric data (e.g., face scans for try-on) without a documented express-consent design. Geolocation must be disclosed with type, purpose, duration and whether shared with third parties.
12. **Automated decisions (art. 8 bis).** If fraud scoring or similar blocks orders or materially affects a person, provide explanation, human intervention and a way to contest.
13. **Impact assessment (art. 15 ter).** Required for large-scale processing, systematic profiling with legal effects, public-area monitoring, or sensitive data processed under consent exceptions. Document a threshold decision even when the answer is "not required".
14. **Governance (arts. 48–52).** A DPO/delegado is optional; in micro, small and medium companies the owner may act as such. A certified prevention model is a mitigating factor (art. 36 N°5) and is listed in the Registro Nacional de Sanciones y Cumplimiento.

### D3. Sanctions (Ley 21.719 arts. 34–40, 47)
| Level | Examples | Maximum fine |
|---|---|---|
| Leve | Incomplete privacy notice; no contact channel; late/no reply to a request | Written warning or up to 5,000 UTM |
| Grave | No consent/legal basis; changing purpose; impeding rights; minors' data; confidentiality or security breaches; not logging/notifying breaches; unlawful transfers; ignoring an Agencia order | Up to 10,000 UTM |
| Gravísima | Fraudulent processing; sensitive/minor data knowingly misused; deliberately omitting breach notification; breaching an Agencia ruling on a rights claim; omitting a mandatory DPIA | Up to 20,000 UTM |

Repeat offender (2+ sanctions in 30 months): up to 3× the fine; larger companies repeating grave/gravísima: up to 2% / 4% of annual revenue. Prescription 4 years. Civil damages (patrimonial and extra-patrimonial) are claimable after the administrative decision is final (art. 47). Suspension of processing for up to 30 days is possible after repeated gravísimas (art. 38). See the timing note above on the first-12-months SME warning window (transitorio sexto).

---

## Part E — Cross-law interactions (resolve conflicts explicitly)

- **Erasure vs. tax retention.** Honor deletion requests for marketing and account data, but keep the minimum fields needed for tax documents and refund/claim evidence for the legal period, restricted from other uses (Ley 21.719 art. 7; DL 825 art. 58).
- **Marketing opt-out.** One unsubscribe must update both the consent record (Ley 21.719) and the suppression list (Ley 19.496 art. 28 B).
- **Data minimization at checkout.** Boleta does not need the buyer's RUT; collect RUT only for factura or where a legal rule needs it (art. 3 c principle).
- **Retracto/returns.** One workflow: return authorization → inspection → refund through the payment channel → nota de crédito → inventory → record retained. Deadlines: retracto refund period (A6), credit-note window (6 months), data-subject deadlines unaffected.
- **Couriers and invoicing vendors.** Classify as processors or independent controllers; share only the fields required to deliver or to issue the document.
- **SERNAC and SII requests.** Both can demand records, including website-related data in digital format; keep a response owner and log exports. Disclosures to authorities must be limited to what is legally required.
- **Logs and analytics.** No personal data in URLs, client-visible error strings or third-party logs/analytics unless covered by a documented basis and vendor contract.

---

## Guardrails

- Never charge more than the displayed total; never show a consumer price without IVA; never pre-select paid extras.
- Never allow a sale to complete without producing the tax document (or a documented, SII-compliant deferral rule).
- Never present a fake "normal price", fake scarcity or fake countdowns as an offer.
- Never collect marketing consent bundled into terms or by silence; never send promotional messages without an unsubscribe path.
- Never process, share or transfer personal data without a recorded legal basis, retention period and vendor contract.
- Treat new third-party tools (analytics, chat widgets, pixels, fonts/CDNs, payment add-ons) as blocked until their data role and transfer basis are known.
- Never edit or delete issued tax documents; correct with notas de crédito/débito.
- Never use free-shipping/gift/giveaway mechanics without an IVA and accounting rule.
- Label every statement from this skill as "per attached text" when it depends on a stale source (see Source currency) and recommend verification before publishing legal copy.

## Pre-launch go/no-go checklist

**Business setup**
- [ ] Inicio de actividades done with correct giro; registered email monitored (CT 66, 68, 11)
- [ ] Municipal patente confirmed (DL 3.063, out of corpus)
- [ ] Boleta/factura electronic issuance integrated and tested; issuance-timing rule documented (DL 825 arts. 9, 54, 55)
- [ ] Business bank account and payment gateway contracted in the company's name (CT 85 ter, 89)

**Storefront**
- [ ] Seller identity block (D6 art. 7), contact/support (art. 15), T&C link (art. 16)
- [ ] Product pages with all essential attributes incl. size/material/origin/brand (D6 art. 9)
- [ ] IVA-inclusive price and itemized *Costo total* with shipping (LC art. 30; D6 arts. 10–11)
- [ ] Stock/availability and delivery information including business/calendar days (D6 arts. 12–13)
- [ ] Retracto and exclusions, if any, shown prior to payment (D6 art. 14)
- [ ] Legal-guarantee, returns, exchange, voluntary-warranty copy separated and accurate

**Checkout and post-purchase**
- [ ] Pre-payment summary; error-correction step; no silent acceptance (LC art. 12 A, 32; D6 art. 18)
- [ ] Confirmation email with the complete contract copy and the tax document
- [ ] Refund + nota de crédito workflow; retracto window handled; courier documentation

**Marketing**
- [ ] Offer/price-history rules; promo bases and duration (LC arts. 28, 35, 36)
- [ ] Unsubscribe in every message; consent logging (LC art. 28 B; 21.719 art. 12)
- [ ] Giveaway/seeding IVA rule (DL 825 art. 8 d)

**In-person (if any)**
- [ ] POS mode delivers boleta/voucher in print or virtual form and can display it (Res. 53)

**Data (Ley 21.719)**
- [ ] Data inventory, legal-basis table, retention schedule
- [ ] Privacy policy with all art. 14 ter items; cookie/tracker consent gate
- [ ] Rights portal and admin tooling with 30-day / 2-day clocks
- [ ] Security controls and evidence; breach runbook and register
- [ ] Vendor contracts (art. 15 bis) and transfer basis (arts. 27–28) for every vendor
- [ ] Minors decision; automated-decision and DPIA threshold memo
- [ ] Agencia instructions checked for anything issued after the corpus

## Examples

### Example 1: "Add 10% off for newsletter sign-up and free shipping over $X"
- Offer terms (bases, duration, any minimum) shown at sign-up and in the cart (LC art. 35).
- Newsletter checkbox unticked, specific, with logged consent; discount cannot be conditioned on consenting to unrelated processing beyond the newsletter itself (21.719 art. 12, bundled-consent presumption).
- Cart shows the discounted *Costo total*, with shipping as its own line (D6 art. 11); boleta amounts match what was charged.
- First email includes the unsubscribe link (LC art. 28 B).

### Example 2: "Build returns and exchanges for a new drop"
- Three separate flows: **retracto** (10 days from receipt, no cause), **legal guarantee** (defects, 6 months), **voluntary exchange** (size/preference, store policy).
- Retracto and its exclusions displayed before payment; refund without withheld costs within the statutory period.
- Each refund generates a nota de crédito and a record kept 6 years; erasure requests do not delete these.

### Example 3: "Install Meta Pixel and abandoned-cart emails"
- Pixel loads only after consent for marketing/analytics; consent banner offers a real reject choice; log the choice.
- Update privacy policy (recipients, purposes, international transfer to the vendor) and vendor register.
- Abandoned-cart marketing needs consent; transactional order emails do not.
- Make sure no personal data travels in URLs or event payloads beyond what the vendor contract allows.

### Example 4: "Pop-up store with card reader and cash"
- Emit electronic boleta at each sale; deliver print or virtual copy; be able to show the documents on a device (Res. 53).
- Apply traceability rules to large cash sales (CT art. 92 ter).
- Collect customer data at the stand only with an informed, optional, logged opt-in.

## Escalation

Escalate to an abogado or the contador for: retracto exclusions and refund periods under current law; marketplace structures; selling to minors/capacity; installment credit offered by the store; international shipping and customs; municipal patente; textile labeling and product-specific rules; PCI/payment regulation; any Agencia or SERNAC inquiry; any SII notification, audit or penalty notice.
