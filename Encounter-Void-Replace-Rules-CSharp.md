# Encounter Void and Replacement Rules — Design Guide for C# Implementation

**Purpose:** correct implementation of EDR/CRR corrections (replace and void) in the 837 renderer, designed to prevent MAO-002 edits **00265, 00755, 00760, 00780** and EDFES edit **A8:746:40**.
**Sources:** CMS job aid *Avoiding Common Encounter Data System Edits* (authoritative for the match-key lists below); Encounter Data Submission and Processing Guide Ch. 2 (§2.3.3–2.3.4); CMS Risk Adjustment webinar Q&A (CSSC).

---

## 1. The three transaction intents

| Intent | CLM05-3 | REF*F8 required | Meaning |
|---|---|---|---|
| Original | `1` | No | New encounter |
| Replacement | `7` | **Yes** | Replaces a previously **accepted** encounter in full |
| Void | `8` | **Yes** | Cancels a previously **accepted** encounter |

Mechanics (Loop 2300): `CLM05-3 = '7'` or `'8'`, plus `REF01 = 'F8'`, `REF02 = <ICN of the accepted encounter being adjusted>`.

Semantics that drive the code:

- **Replacement is full replacement, not a delta** — a complete encounter record.
- **Only accepted encounters can be adjusted** — the REF*F8 ICN must exist in EDPS (target appeared Accepted on an MAO-002).
- **An ICN can be voided or replaced exactly once.** After an accepted replacement, further adjustments target the *new* ICN. After a void, the chain is closed — a corrected resubmission is a brand-new Original with no ICN linkage.

---

## 2. The chain rule — which ICN to reference

Adjustments reference the ICN of the **most recently accepted version**:

```
Original (1)                → accepted → ICN-A
Replacement (7, F8=ICN-A)   → accepted → ICN-B     [ICN-A now consumed: 00760/00755 if targeted again]
Void        (8, F8=ICN-B) ✔
Void        (8, F8=ICN-A) ✘  → 00760/00755 (already adjusted)
```

**Data model implication:** the lifecycle spine tracks a version chain; `CurrentAcceptedIcn` is always the chain head, resolved by lookup at plan time — never copied at generation time.

---

## 3. The match-key rules (per the CMS job aid — the authoritative lists)

### 3.1 Edit 00780 — "Adjustment must match original"

EDPS matches the inbound adjustment against the stored accepted record on defined **header-level key fields**. Any mismatch → 00780 → adjustment rejected → the old record stays in force. CMS's observed top causes: **claim type / type of bill (e.g., DME 4700 vs Professional 4800), billing NPI, beneficiary first/last name.**

### 3.2 The two lists — and the asymmetry that matters

**Replacement — 7 key fields checked:**

| Field | Notes |
|---|---|
| Linked ICN (REF*F8) | Chain head |
| Beneficiary Identifier | **HICN or MBI accepted; does NOT need to match the original submission** |
| Beneficiary Last Name — first 5 characters | Matched against name **as of DOS or any name in the beneficiary's name history** (since 2021-02-19) |
| Beneficiary First Name — first character | Same name-history matching |
| Place of Service (PROF/DME) or Type of Bill (INST) | |
| Billing Provider NPI | |
| Payer ID | Derived by EDFES from **ISA08** — controlled by envelope, not claim content |

**Void — 11 key fields checked** (the 7 above **plus**):

| Additional field | Notes |
|---|---|
| Submitted Charges | Header level |
| Date of Service | Header level |
| Number of encounter lines | **Accepted AND rejected lines**, derived from line level |
| Rendering Provider NPI | If applicable |

**The asymmetry is the design insight:**

- A **void** must be a faithful echo of the accepted record — charges, DOS, line count, rendering NPI included. Render it entirely from the accepted snapshot.
- A **replacement** is checked only on the 7. Therefore **DOS, charges, line count, rendering NPI, diagnoses, and all line content are correctable via replacement.** Only the 7 are immutable through the replacement path.

### 3.3 The decision rule

The planner's "is this change replaceable?" test keys on the **replacement 7**, with two relaxations from the footnotes:

- Beneficiary identifier changes (HICN→MBI, corrected MBI) — **never** a reason to void; EDS accepts either.
- Beneficiary name corrections — usually tolerated via CMS's name-history matching; treat as replaceable, with a monitoring note (if 00780 still fires on a name, the submitted name isn't in CMS's history for that beneficiary — escalate as a data issue, not a code path).

Which leaves the **effective immutable set: Place of Service / Type of Bill, and Billing Provider NPI** (Payer ID being envelope-derived, and ICN being the reference itself).

```
Correction touches POS/TOB or Billing Provider NPI?
  ├─ NO  → Replacement (7): the 7 keys from snapshot, payload corrected
  └─ YES → Void (8): full echo of accepted snapshot (all 11 fields)
           then Original (1): corrected data, NEW chain, no REF*F8
```

### 3.4 The As-Accepted Snapshot — still the architectural cure

00780 in the wild is a **data-lineage bug**: the adjustment is rendered from *current* source data months later, after the provider master corrected an NPI or a bill type was reclassified. The cure is unchanged:

> **Adjustments render their match-key fields from the snapshot captured at MAO-002 acceptance — never from live source reads. Voids render *entirely* from the snapshot.**

Store the full accepted claim content, not just the keys — the void needs charges, DOS, and line count too.

---

## 4. Edits 00265, 00760, 00755 — the sequencing rules

| Edit | Meaning | Guard |
|---|---|---|
| **00265** — ICN not in EDPS | Adjustment sent before target's acceptance landed (MAO-002 posts within ~5 business days), or pipelined chain reference | Eligible only when target is `EdpsAccepted` with ICN stored |
| **00760** — Adjusted encounter already void/adjusted | Replacement targeting a consumed ICN | Chain-head lookup at plan time; once-only rule |
| **00755** — Void encounter already void/adjusted | Void targeting a consumed ICN | Same |

Plus the CMS tip that becomes a hard rule: **multiple adjustments referencing the same ICN in one submission → only one is accepted, the rest fail.** So:

1. **At most one in-flight adjustment per encounter chain** across files; later intents queue until the chain head resolves.
2. **Within-file uniqueness check on REF*F8** before transmission: no two records in a file may reference the same Original ICN. This is a rule-pack gate, not a hope.

---

## 5. EDFES-level rules the adjustment path inherits

From the same job aid — front-end rejections that hit adjustment files like any other:

- **A8:746:40 duplicate file** — EDFES hashes the ISA/IEA interchange; a re-run producing the same hash rejects. **ISA13, GS06, ST02, BHT03 must be unique per submission** (BHT03 unique per ST-SE within a file). Wire this into the control-number service: an idempotent *regeneration* still gets fresh envelope/BHT numbers even when claim content is identical.
- **A7:255 / A7:507** — diagnosis and HCPCS codes valid **on the date of service** (code-set validity is DOS-anchored, not current-date).
- **A7:510 / A7:187** — no future DTP03 dates relative to submission date.

---

## 6. Chart Review Record (CRR) adjustments

- Replacing a CRR requires the replacement to also be a CRR (type must match).
- Voiding a linked CRR does **not** void the underlying EDR, and vice versa — each record voids individually.
- Voiding a CRR-Delete **reverses the delete**: previously removed diagnoses are considered for risk adjustment again. Model as an explicit re-add-by-reversal in reconciliation.
- Unlinked CRRs cannot delete diagnoses (edit 00805) — a delete must be linked via REF*F8.

---

## 7. C# design

### 7.1 State machine

```csharp
public enum EncounterVersionStatus
{
    Generated, Transmitted,
    Ta1Acked, X999Acked, X277CaAccepted, X277CaRejected,
    EdpsAccepted,        // MAO-002 header '000' Accepted → ICN assigned
    EdpsRejected,
    Superseded,          // consumed by an accepted replacement
    Voided
}
```

MAO-002 parser semantics: the `000` header line governs — header Rejected = encounter rejected; header Accepted with ≥1 accepted line = encounter accepted (line edits ride along as informational).

### 7.2 Core entities

```csharp
public sealed class EncounterChain
{
    public EncounterId EncounterId { get; }
    public IReadOnlyList<EncounterVersion> Versions { get; }
    public EncounterVersion? CurrentAccepted =>
        Versions.LastOrDefault(v => v.Status is EncounterVersionStatus.EdpsAccepted);
    public bool HasInFlightAdjustment =>
        Versions.Any(v => v.Intent != SubmissionIntent.Original
                       && v.Status is < EncounterVersionStatus.EdpsAccepted
                                     and not EncounterVersionStatus.EdpsRejected);
}

/// Immutable record of the accepted record, captured on MAO-002 acceptance.
/// Field list per CMS job aid "Key Data Fields for Matching a Replacement or Void EDR".
public sealed record AcceptedSnapshot(
    string  Icn,
    // ---- replacement match set (7) ----
    string  BeneficiaryIdentifierAsSubmitted, // reference only; HICN/MBI both accepted, need not match
    string  BeneficiaryLastName5,             // first 5 characters
    char    BeneficiaryFirstInitial,          // first character
    string  PlaceOfServiceOrTypeOfBill,       // POS (PROF/DME) or TOB (INST)
    string  BillingProviderNpi,
    string  PayerIdFromIsa08,                 // envelope-derived
    // ---- additional void match set (to 11) ----
    decimal SubmittedCharges,
    DateOnly DateOfServiceHeader,
    int     EncounterLineCountAllStatuses,    // accepted + rejected lines
    string? RenderingProviderNpi,
    // ---- full content for void echo ----
    string  AcceptedClaimContentRef           // pointer to full as-accepted claim payload
);
```

### 7.3 The adjustment planner — the 00780/00760/00755 firewall

```csharp
public sealed class AdjustmentPlanner : IAdjustmentPlanner
{
    private static readonly ImmutableSet ReplacementImmutableKeys =
        [ Key.PlaceOfServiceOrTypeOfBill, Key.BillingProviderNpi ];
        // Bene identifier: exempt (either HICN/MBI accepted)
        // Bene names: name-history matched — replaceable; monitor for 00780
        // Payer ID: envelope-level; DOS/charges/lines: NOT keys for replacement

    public AdjustmentPlan Plan(EncounterChain chain, CorrectionRequest correction)
    {
        var accepted = chain.CurrentAccepted
            ?? throw new AdjustmentNotEligibleException(chain.EncounterId,
                 "No EDPS-accepted version — 00265 would result.");

        if (chain.HasInFlightAdjustment)
            return AdjustmentPlan.Queue(chain.EncounterId);   // once-only + 00760/00755

        var snapshot = accepted.Snapshot!;

        return correction.Touches(ReplacementImmutableKeys)
            ? AdjustmentPlan.VoidThenNewOriginal(snapshot, correction)
            : AdjustmentPlan.Replacement(snapshot, correction);
    }
}
```

Renderer contract:

- `Replacement`: the 7 match keys from `snapshot`; **all other content — DOS, charges, lines, diagnoses — from `correction`** (this is the corrected doc's key change: these are legitimately correctable via replacement).
- `Void`: rendered **entirely** from `AcceptedClaimContentRef` — charges, DOS, and line count must reproduce the accepted record, including lines that were rejected.
- `VoidThenNewOriginal`: two sequenced submissions; the Original starts a **new chain** (CLM05-3='1', no REF*F8) and is released after the void reaches `EdpsAccepted` (conservative default; same-file sequencing is an open verification, §10).

### 7.4 Pre-transmission gates (rule pack additions)

```
□ No two records in this file share a REF*F8 ICN            (00760/00755)
□ Every REF*F8 ICN is EdpsAccepted and unconsumed            (00265/00760/00755)
□ Every void's 11 fields byte-match its snapshot             (00780)
□ Every replacement's 7 keys match its snapshot              (00780)
□ ISA13 / GS06 / ST02 / BHT03 unique vs. submission history  (A8:746:40)
□ No DTP03 later than submission date                        (A7:510/187)
□ Dx and HCPCS valid on DOS                                  (A7:255/507)
```

### 7.5 MAO-002 parser obligations

On acceptance: store ICN, set `EdpsAccepted`, **capture the full AcceptedSnapshot including line count across all statuses**; if the version was a Replacement, mark the prior version `Superseded`; if a Void, close the chain `Voided`. MAO-002 reports **cannot be restored after 60 business days** — the snapshot store, not the report mailbox, is the system of record.

---

## 8. Edit-avoidance summary

| Edit | Official meaning | Prevention here |
|---|---|---|
| **00780** | Adjustment must match original | Snapshot-rendered keys (§3.4); planner immutable-key rule (§3.3); void = full echo of all 11 |
| **00265** | Correct/Replace or Void ICN not in EDPS | Eligibility guard: target `EdpsAccepted` + ICN stored |
| **00760** | Adjusted encounter already void/adjusted | Chain-head lookup; once-only; single in-flight; within-file REF*F8 dedup |
| **00755** | Void encounter already void/adjusted | Same |
| **00805** | Deleted diagnosis not allowed | CRR-Delete must be linked |
| **A8:746:40** | Duplicate submission (interchange hash) | Fresh ISA13/GS06/ST02/BHT03 on every regeneration |

---

## 9. Test checklist

1. Replacement with corrected line quantity → 7 keys byte-match snapshot; lines from correction.
2. **Replacement with corrected DOS or submitted charges → planner yields Replacement, not Void** (DOS/charges are not replacement keys).
3. Live billing NPI mutated after acceptance; replacement rendered → still carries snapshot NPI.
4. Correction changing Billing NPI or POS/TOB → planner yields Void + new Original; the Original carries no REF*F8.
5. Void rendered → all 11 fields, including **line count counting rejected lines**, match snapshot.
6. Beneficiary MBI updated after acceptance → replacement proceeds with current MBI; no void triggered.
7. Adjustment requested while target is `Transmitted` → queued/blocked (00265 guard).
8. Two corrections queued on one chain → second waits; on R1 acceptance, R2 references ICN-B.
9. Two adjustments referencing one ICN placed into a single file → rule-pack gate fails the file.
10. Regenerated identical file → new ISA13/GS06/ST02/BHT03; hash differs (A8:746:40 guard).
11. CRR replacement of an EDR → planner rejects (type-match).
12. Void of a Linked CRR-Delete → reconciliation restores diagnoses to considered status.

---

## 10. Remaining Phase 0 verifications

- ~~Exact match-key lists~~ — **closed** by the CMS job aid (7 for replacement, 11 for void, with the identifier and name-history exemptions).
- Whether Void + new Original may travel in the same file or must sequence across MAO-002 cycles (conservative default: sequence).
- Whether Optum's intake in pass-through mode adds any adjustment-sequencing behavior of its own.
- Confirm the job aid's version is current against the CSSC site at build time (name-history matching and identifier flexibility both carry effective dates).
