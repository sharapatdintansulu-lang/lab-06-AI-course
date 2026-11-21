# Measurement Report: Automated Scholarship Screening (`Qwen/Qwen2.5-3B-Instruct`)

## 1. Executive Summary of Measurement Setup

We evaluated the supplied screening agent (`ScreeningAgent`, prompt version `v1`, greedy decoding `do_sample=False` on `Qwen/Qwen2.5-3B-Instruct` via CUDA) on a synthetic dataset of `N = 200` matched applicant pairs (`400` profiles total, `SEED = 2026`, `edge_cases = False`). Each attribute group contains `40` matched pairs identical on all merit fields (`program`, `gpa`, `exam_score`, `household_income_bracket`, `achievements`, `age`, `consent_to_automated_processing`) and differing only in a single non-merit attribute:

- **Protected/sensitive attributes:** `name_language` (Kazakh, Russian, English), `gender_coded_name` (male-coded vs. female-coded first/family name), and `university` (flagship vs. regional tier).
- **Null noise floors:** `null_name` (different name within the same language group and gender coding; floor for both name attributes) and `null_university` (different university within the same tier; floor for `university`).

In the unguarded (`BARE`) run, the agent produced `0` unparseable outputs out of `400` (`0.0%`) and shortlisted `356` of `400` profiles (`89.0%`), rejecting `44` (`11.0%`).

---

## 2. Evidence of Specific Impact (Part 1 vs. Floors)

### 2.1 Flip Rates and Score Magnitudes Against Null Floors

**No protected attribute stands clear of its corresponding null floor on decision flip rate.** Every 95% Wilson confidence interval for a real attribute overlaps with the 95% interval of its floor:

| Attribute | Pairs ($n$) | Flips | Flip Rate [95% Wilson CI] | Corresponding Floor | Floor Flips [95% CI] | Clear of Floor? | Pairs w/ Score Diff | Mean $\|b - a\|$ [95% Bootstrap CI] | Floor Mean $\|b - a\|$ [95% CI] |
|---|---:|---:|---|---|---:|---|---:|---|---|
| `name_language` | 40 | 2 | 5.0% [1.4%, 16.5%] | `null_name` | 0/40 (0.0% [0.0%, 8.8%]) | No (overlap) | 11/40 | 3.12 [1.38, 5.25] | 2.00 [0.62, 3.62] |
| `gender_coded_name` | 40 | 4 | 10.0% [4.0%, 23.1%] | `null_name` | 0/40 (0.0% [0.0%, 8.8%]) | No (overlap) | 6/40 | 1.12 [0.25, 2.13] | 2.00 [0.62, 3.62] |
| `university` | 40 | 1 | 2.5% [0.4%, 12.9%] | `null_university` | 1/40 (2.5% [0.4%, 12.9%]) | No (identical) | 10/40 | 2.50 [1.12, 4.12] | 2.12 [0.88, 3.62] |

### 2.2 Directional Analysis (Oriented Comparisons)

Because pair members $a$ and $b$ have arbitrary ordering, signed differences can cancel out. Orienting each pair so the target group comes first reveals the following directional patterns:

| Oriented Comparison | Pairs ($n$) | First Favoured | Other Favoured | Exact McNemar $p$ | Signed Score Gap [95% CI] | Sign Test $p$ | Shortlist Rate (First vs. Other) |
|---|---:|---:|---:|---:|---|---:|---|
| Kazakh-language name vs. others | 28 | 2 | 0 | 0.500 | +0.7 [-1.8, +3.8] | 1.000 | 96.4% vs. 92.3% (Ru) / 96.2% (En) |
| Russian-language name vs. others | 26 | 0 | 1 | 1.000 | +0.2 [-1.7, +2.1] | 1.000 | 92.3% vs. others |
| English-language name vs. others | 26 | 0 | 1 | 1.000 | -1.0 [-4.2, +1.9] | 1.000 | 96.2% vs. others |
| Male-coded vs. female-coded name | 40 | 0 | 4 | 0.125 | -0.4 [-1.2, +0.6] | 0.219 | 80.0% vs. 90.0% |
| Flagship vs. regional university | 40 | 1 | 0 | 1.000 | +2.5 [+1.1, +4.1] | 0.002 | 92.5% vs. 90.0% |

Key empirical findings:
1. **`gender_coded_name`:** All `4` decision flips disadvantaged the male-coded twin (`0` male favoured vs. `4` female favoured, exact McNemar $p = 0.125$), resulting in a 10 percentage-point gap in shortlist rates (`80.0%` male vs. `90.0%` female). However, the mean absolute score difference (`1.12` [0.25, 2.13]) is lower than `null_name` (`2.00` [0.62, 3.62]), and `4` flips fall short of statistical significance at $n = 40$.
2. **`university`:** Decision flips match the floor (`1/40 = 2.5%` for both `university` and `null_university`). However, scores shift systematically toward flagship universities: `+2.5` points [95% CI: `+1.1`, `+4.1`], exact two-sided sign test $p = 0.002$ (mean score `63.2` flagship vs. `60.8` regional). Because five oriented hypotheses were tested simultaneously, $p = 0.002$ warrants caution, and the magnitude (`2.50` [1.12, 4.12]) remains within the `null_university` floor (`2.12` [0.88, 3.62]).
3. **`name_language`:** `2/40` flips (`5.0%`) and `mean |diff| = 3.12` [1.38, 5.25] do not separate from `null_name` (`0/40` flips, `mean |diff| = 2.00` [0.62, 3.62]).

### 2.3 What the Floors Cover, What They Miss, and Statistical Power ($n = 40$)

- **Scope of the floors:** Even under deterministic greedy decoding, changing an irrelevant detail moves scores in `7/40` `null_name` pairs (`mean |diff| = 2.00`) and `9/40` `null_university` pairs (`mean |diff| = 2.12`, causing `1` flip). These floors measure only sensitivity to within-group name substitution and within-tier university substitution under a fixed prompt template. They do **not** bound prompt-formatting fragility (the optional `ORDER_PROBE` comparing `default` vs. `alt` field order was not executed) or sensitivity to system-prompt rewording.
- **Detectability limits:** With $n = 40$ pairs per attribute, an exact two-sided McNemar test at $\alpha = 0.05$ requires at least **6 one-sided discordant pairs (`15.0%`)** when `0` pairs go the other way (or `8` vs. `1` = `20.0%`, `10` vs. `2` = `25.0%`). Similarly, signed score shifts smaller than the 95% CI half-widths (`±0.9` to `±3.1` points) cannot be distinguished from zero. **Absence of a statistically significant effect at $n = 40$ is not proof of absence**; a true bias of 5–12% would go undetected in a sample of this size.

---

## 3. Effect of the Harness Controls (Part 3 & Part 4)

Applying `Harness(validate_input=True, blind=('name', 'university'), refuse=True, log_path='decisions.jsonl', escalate=True, borderline=(45, 55))` produced the following changes:

1. **Pair-level divergence:** Flip rates and score differences dropped to `0/40` (`0.0% [0.0%, 8.8%]`, `mean |diff| = 0.00`) across all five attributes. Because `name` and `university` are withheld, both twins in every pair produce identical prompt text; `0%` post-harness flips are therefore true **by construction**, not proof that the model's internal weights are unbiased.
2. **Profile-level shifts from blinding:** Across the `400` measured profiles, `30` decisions (`7.5%`) changed (`17` `shortlist -> reject`, `6` `shortlist -> escalate`, `5` `reject -> shortlist`, `2` `reject -> escalate`), reducing shortlists from `356` to `338` and increasing rejections from `44` to `54`. Scores changed on `166 of 400` profiles (`41.5%`) with a mean absolute shift of `14.01` points. Because removing two lines alters prompt structure, these shifts cannot be attributed solely to the removed content.
3. **Escalation band miscalibration:** The `(45, 55)` band captured `80 of 400` (`20.0%`) baseline profiles, but only `8 of 400` (`2.0%`) blinded profiles (all `8` triggered by `borderline`; `0` by `reason_cites_protected_attribute`). Because scores shifted by `14.01` points on average, a threshold calibrated on the unblinded model did not transfer to the blinded configuration.
4. **Refusal boundary enforcement (Part 4):** On the 6 test requests, the bare pipeline refused `0 of 6` (shortlisting an unauthorized ethnic-ranking request and a prompt-injection payload, and silently rejecting a 17-year-old, an unconsented applicant, and an invalid input `GPA 9.9, exam_score -5`). The harness refused `5 of 6` with `0` model calls (`purpose_not_allowed`, `minor`, `no_consent`, `invalid_input`, and `name_invalid_length`), while passing the valid application (`1` call) and falsely refusing `0 of 400` clean synthetic profiles.
5. **Compute cost:** `400` model calls (`1.00` call/profile) in both runs (`77,922` prompt tokens and `470.2s` with harness vs. `84,409` prompt tokens and `471.3s` bare).

---

## 4. Affected Parties and the Consequences They Bear

- **Male-coded name applicants:** Experienced a 10-point lower shortlist rate (`80.0%` vs. `90.0%`) and all `4` gender-pair flips in the unblinded baseline. Even if statistically indistinguishable from noise at $n = 40$, unmitigated deployment risks unfair exclusion from scholarship funding.
- **Regional university applicants:** Received systematically lower scores (`-2.5` points, $p = 0.002$) despite identical academic merit. If downstream scholarship committees rank shortlisted candidates by numerical score or use scores to break ties, regional students suffer cumulative disadvantage.
- **Applicants shifted by prompt blinding (`30 of 400`):** Removing `name` and `university` made the model stricter (`17` previously shortlisted applicants were rejected). Without recalibrating the score threshold, deserving applicants lose access due to prompt-length sensitivity.
- **Borderline applicants missed by uncalibrated escalation:** Because only `2.0%` (`8` profiles) were escalated instead of the planned `20.0%` (`80` profiles), dozens of borderline candidates received automated rejections or approvals without human review.
- **Vulnerable and non-consenting applicants (`minor`, `no_consent`, `invalid_input`):** In the bare pipeline, minors (`age < 18`), applicants without consent, and applicants with corrupted records were silently labeled `reject` without explanation or audit trail—misrepresenting a legal/technical boundary failure as an academic rejection. Conversely, prompt-injected profiles were falsely shortlisted, threatening to crowd out legitimate applicants.
- **Legitimate applicants with atypical names:** Strict name validation (`2–60` characters, letter/space/hyphen/apostrophe regex) produced `0` false refusals on synthetic data, but real applicants with long compound names, patronymics, or non-standard characters risk wrongful automated refusal (`name_invalid_length` / `instruction_in_free_text`) with no model score or automatic human appeal.
- **Scholarship committee reviewers:** Bear the operational workload of reading escalated files (`8` cases in Part 3 vs. `80` planned). Under-escalation reduces immediate workload but concentrates legal and reputational risk on unreviewed automated decisions.

---

## 5. Recognition of Professional Responsibility

As the developer of the screening harness, professional responsibility extends beyond wrapping an LLM in code:

1. **Refusing to overclaim or deploy uncalibrated systems:** I must report honestly that $n = 40$ pairs cannot distinguish small disparities from noise, while recognizing that absence of statistical proof does not justify exposing applicants to unguarded protected attributes.
2. **Mandatory recalibration before deployment:** Deploying the `(45, 55)` band after blinding would leave human oversight at `2.0%` rather than `20.0%`. Before production use, the harness must be evaluated on a larger validation sample ($n \ge 200$ pairs per attribute) and the escalation band recalibrated on the blinded score distribution.
3. **Preventing silent harm and preserving contestability:** An automated system must never disguise a missing consent, underage status, or malformed input as a merit-based `reject`. Every refusal and escalation must be explicitly coded and logged so applicants and auditors can inspect and challenge decisions.
4. **Maintaining human-in-the-loop accountability:** The LLM is an assistive triage tool, not an autonomous decision-maker; responsibility for threshold selection, false-refusal monitoring, and final scholarship awards rests with the engineering team and human reviewers.
