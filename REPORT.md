# REPORT — what the measurement showed, who it affects, and whose responsibility it is

Every number below was printed by my submitted run of `lab06_harness_says_no.ipynb` (Qwen2.5-3B-Instruct, greedy decoding, prompt v1, 40 pairs per group, SEED 2026, clean profiles).

## 1. Measurement read-out

The bare agent screened 200 matched pairs (400 profiles), shortlisted 356 of 400 (89.0 %) and returned 0 unparseable replies.

| group | flips / 40 | flip rate [95 % CI] | pairs with different scores | mean abs. score diff [95 % CI] |
|---|---|---|---|---|
| name_language | 2 | 5.0 % [1.4, 16.5] | 11 | 3.12 [1.38, 5.25] |
| gender_coded_name | 4 | 10.0 % [4.0, 23.1] | 6 | 1.12 [0.25, 2.13] |
| university | 1 | 2.5 % [0.4, 12.9] | 10 | 2.50 [1.12, 4.12] |
| null_name (floor for the name attributes) | 0 | 0.0 % [0.0, 8.8] | 7 | 2.00 [0.62, 3.62] |
| null_university (floor for university) | 1 | 2.5 % [0.4, 12.9] | 9 | 2.12 [0.88, 3.62] |

**No attribute stands clear of its floor**: every interval overlaps its floor's, so this run does not show that name language, gender coding or university moved a decision. With 40 pairs an exact McNemar test needs at least 6 one-sided pairs (15 %) with none the other way, 8 with one and 10 with two; a smaller effect is invisible here, so "no measured effect" is not "no effect".

- **Gender coding:** all 4 flips favoured the female-coded name (0 against 4, p 0.125); shortlisted 90.0 % female-coded vs 80.0 % male-coded; score gap −0.4 [−1.2, +0.6]. A direction to watch, not a demonstrated effect.
- **University:** 1 flip (p 1.000), but a consistent score shift: flagship minus regional +2.5 points [+1.1, +4.1], sign test p 0.002 (mean 63.2 vs 60.8). Its size (2.50 over 10 pairs) is close to the floor (2.12 over 9), and with 89 % shortlisted it almost never crosses the decision line: an effect on scores, not on decisions.
- **Name language:** no consistent direction (Kazakh 2 against 0, p 0.500; Russian 0 against 1; English 0 against 1; all score-gap intervals span zero).

My prediction was wrong: I expected university to flip 4–8 pairs and Kazakh names to fare worse.

**Noise.** The floors change only another name of the same group or another university of the same tier, and keep wording and layout fixed. The order probe shows what they leave out: in 40 pairs of **identical** applicants with only the line order changed, 10 decisions flipped (25.0 % [14.2, 40.2]), all one way (10 against 0, p 0.002), mean score −6.6 [−9.9, −3.5]. Layout alone moved more decisions than any attribute. The counterfactual check (8 name-language pairs, 32 calls) found 0 disagreements when the name was swapped. A stronger estimate needs more pairs near the line, several prompts and layouts, and a second model.

**After the harness.** Flips are 0 of 40 in every row, but **by construction**: with name and university blinded both members of a pair see identical text. Per profile: outcomes went from shortlist 356 / reject 44 / escalate 0 to shortlist 338 / reject 24 / escalate 38; 50 of 400 changed outcome, 12 (3.0 %) between shortlist and reject; 166 scores moved by a mean of 14.01 points, which shows unstable scores, not proof the model used the hidden fields. 38 of 400 (9.5 %) went to a person, all `borderline`. The band did not transfer: (37, 59) would hold 27.5 % of unblinded baseline profiles, 11.2 % of blinded calibration profiles, and sent 9.5 % in the run. Cost: 400 calls in both runs plus 160 once for calibration.

**Limits.** One prompt, one 3B model, synthetic names, fictional universities, 40 pairs per group; age, programme and income were not tested.

## 2. Who is affected

**Wrongly rejected:** applicants near the line. In the bare run the 40–49 row is the only one with both rejects (33) and shortlists (31), and there decisions turn on details with no merit: +2.5 points for a flagship university, −6.6 points for reordered lines. The groups to watch are regional-university applicants (60.8 vs 63.2) and male-coded names (80.0 % vs 90.0 %), neither significant. A wrong rejection costs the scholarship and possibly the chance to study, and in the bare pipeline nobody reads the file. The harness now sends the 38 borderline rejections to a person; 24 low-score rejections are still decided by the model alone.

**Wrongly shortlisted:** with 89.0 % shortlisted, places are scarce only at the award stage, where a wrong shortlist costs reviewer time and can displace a stronger applicant. The clearest case was manipulation: a name field holding an instruction was shortlisted by the bare pipeline. The costs are not symmetric: a wrong rejection removes a person, a wrong shortlist is still checked by people later.

**Not in the brief:** reviewers (38 of 400 files to read); refused applicants (minors, no consent, malformed fields), with 0 of 400 clean applicants refused as only a lower bound; applicants from institutions the model has never seen, now judged without that field; people who never apply because they expect an opaque rejection; and the fund, which carries the legal and reputational risk.

**If an effect exists that n = 40 cannot see** (up to 6–10 one-sided pairs), it falls on the same groups, every year. Used repeatedly or fed into later reviews and rankings, a 2.5-point gap is copied into every cut-off, and cut-offs drift (27.5 % vs 9.5 % for the same band).

## 3. Professional responsibility

**What I must do.** Recalibrate the band whenever what the model sees changes and report the share actually escalated; freeze and version the prompt layout (layout flipped 25.0 % of identical pairs); tell the fund that no divergence above its floor was measured, what n = 40 cannot detect, that blinding changed 50 of 400 outcomes, and what I did not test. I must not claim that 0 flips after the harness means an unbiased model.

**How, and what is done.** A test (rerun the order probe and matched pairs after any prompt change), a documented limit (this report and `JUSTIFICATION.md`), a stop condition (no new prompt goes live without that rerun). Done in this lab: five controls, band calibrated on blinded scores and corrected when the first version sent only 1.2 %, refusal path exercised (5 of 6 refused, 0 of 400 wrongly), a 406-record log, order probe, counterfactual check, scripted tests of my escalation rule. Left: programme, income and age, more pairs, other prompts and models, real names, a record of reviewers' final decisions.

**The institution's part.** The fund owns the purpose, who reviews escalated cases, notices to applicants, appeals and log retention. I own the correctness of the controls and the honesty of this report.

**Conflict.** If the fund wanted escalation removed to save time, I would show what it carries (38 applicants, 22 of whom the bare agent rejected), offer a narrower band or a review-time cap, and refuse to call the system human-reviewed if nobody reads them. If it wanted blinding removed, I would restate the +2.5-point gap (p 0.002) and the 0-against-4 direction and leave the decision to the fund in writing.
