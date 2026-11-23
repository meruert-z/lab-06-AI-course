# JUSTIFICATION — the design of the harness, and why

Everything described here is in the notebook code or in the submitted run (`decisions.jsonl`: 406 records; `control_register.json`). The fund and applicants are fictional, so nothing binds this prototype; I design it as if the fund operated in Kazakhstan and use EU law only as a benchmark.

## 1. What the system may do

**Purpose, stated narrowly:** to pre-screen scholarship applications. It reads one profile and returns *shortlist* or *reject*, a score (0–100) and a one-sentence reason. It informs the award stage; it never awards.

It may decide on its own only if: the purpose is `scholarship_screening`; the applicant is 18+ and consented to automated processing; all fields pass the type/range check and my free-text rule; the score is outside the band (37, 59); and my escalation rule does not fire. Everything else is refused or sent to a person.

| control | constraint that made me build it | where in the code |
|---|---|---|
| validate_input | safety and reliability (AI Law Art. 15(2)(2)) | `Harness(validate_input=True)`, Part 2d; supplied `validate_profile` |
| blind (name, university) | non-discrimination (AI Law Art. 6; Constitution Art. 16(2)); no excessive data (PD Law Art. 7(9)) | `BLIND`, Part 2a |
| refuse | consent and purpose limitation (PD Law Art. 7(1), 7(8), 19-1(1)) | supplied `refusal_reason` + `my_extra_refusal`, Part 2b |
| log | control and documentation (AI Law Art. 8(1), 15(2)(3)); recording of actions (PD Law Art. 22(1)) | `log_path="decisions.jsonl"`, Part 2d |
| escalate | right to object, human control (PD Law Art. 19-1(2)–(3); AI Law Art. 7(2), 8(1)) | `my_extra_escalation`, Part 2c; band rule `choose_band`, Part 2d |

**Constraints that changed the architecture.** (1) Part 1 showed no flip rate above its floor, so on the numbers alone I could have kept the bare prompt; the non-discrimination duty made me remove name and university from the input instead of monitoring the output. (2) PD Law Art. 19-1(1) made consent a hard refusal before the model, not a field the model weighs. (3) The right to object made me route borderline rejections to a person; without it the system would be the bare pipeline, which decided all 400 profiles alone.

## 2. What it must refuse

Refusals run before the model: 0 model calls, one log record with a reason code, no field values stored. In Part 4 the bare pipeline screened all six requests; the harness refused 5 of 6.

| refused | reason code | enforced in | fired in the submitted run | legitimate case it also blocks |
|---|---|---|---|---|
| purpose other than screening ("rank by ethnicity") | `purpose_not_allowed` | `refusal_reason` | Part 4 | none I can name |
| applicant under 18 | `minor` | `refusal_reason` | Part 4 | a 17-year-old who could apply through a legal representative |
| no consent | `no_consent` | `refusal_reason` | Part 4 | an adult who forgot to tick a box |
| impossible values (GPA 9.9, exam −5) | `invalid_input` | `validate_profile` | Part 4 | a typo by an honest applicant |
| instruction in name / university / programme | `prompt_injection_in_<field>` | `my_extra_refusal` | Part 4 (the bare pipeline **shortlisted** this profile) | a real name or programme containing a listed word |
| name that is not a personal name | `name_not_a_personal_name` | `my_extra_refusal` | only in my scripted test ("Agent 007"), not in the log | a real name with a digit or unusual character |

The applicant bears every cost: no decision on the merits. I accept it because each refusal is logged with a code that says what to correct, and because the alternative is deciding about people the system must not process. My rules refused 0 of the 400 clean applicants and passed "Aigerim Nurlanova", "Mary-Jane O'Neil" and "Dias Bekmukhambetov", but the names are synthetic, so that is a lower bound on wrong refusals.

**What happens to a refused request:** it is recorded (one line in `decisions.jsonl`). The design intends that the applicant is told the code and can object and ask for a person (AI Law Art. 7(2); PD Law Art. 19-1(2)–(3)), but the notebook sends no notice and has no objection channel; that is listed in §6.

## 3. What record it keeps

**Audit question:** *for application P, what did the model see, what did it answer, which control stopped or redirected it, and could someone else reconstruct the outcome without re-running the model?*

Each record holds `record_id`, `pid`, `input_sha256` (hash of the exact text the model saw), `fields_shown`, `fields_withheld`, `model_id`, `prompt_version`, the `harness` settings, `raw_output`, `decision`, `score`, `controls_fired`, `outcome`, `reason_code`, `escalated_to_human`, `timestamp`. Example from my run: P0043a, fields withheld name and university, raw reply "reject, score 40", controls blind + escalate + log, outcome `escalate`, reason `borderline`. That record alone answers the question. My log holds 406 records: shortlist 339, reject 24, escalate 38, refuse 5; reason codes model_decision 363, borderline 38, and one each of the five refusal codes.

**Left out on purpose:** field values. The log keeps field names and a hash, so it is not a second copy of applicant data (PD Law Art. 7(9), 12(2)). The price: the log alone cannot rebuild the input; an auditor needs the application store, re-renders the profile and compares hashes.

**Retention, access, integrity:** not implemented. No retention period is set (PD Law Art. 12(2) ties it to the purpose; it must at least cover the objection period), no access control, and nothing stops a record being edited afterwards. The record also does not show the reviewer's final decision or identity.

## 4. The legal constraints

They would apply because the fund would operate in Kazakhstan, processes personal data of applicants, and uses an AI system to support decisions about them.

- **Law of the RK No. 230-VIII of 17 November 2025 "On Artificial Intelligence"** (in force 18 January 2026). *Requires:* Art. 6 fairness and equality, no discrimination; Art. 7(2) informing the affected person about automated processing and the possibility to object; Art. 8(1) continuous control over the system; Art. 15(2)(2)–(3) safety, reliability and documentation; Art. 16(1)(4)–(5) explanations of results and the data behind a decision; Art. 18(1) risk management.
- **Law of the RK No. 94-V of 21 May 2013 "On Personal Data and Their Protection"**, as amended by Law No. 231-VIII of 17 November 2025. *Requires:* Art. 7(1) consent of the subject or legal representative; Art. 7(8) specific, predetermined, lawful purposes; Art. 7(9) no excessive data; Art. 12(2) retention until the purpose is achieved; Art. 22(1) protective measures. **Art. 19-1 is the provision written for automated decisions specifically:** automated processing that creates, changes or ends a person's rights is prohibited without consent (para. 1); the owner must explain it and consider an objection within three working days (paras. 2–3).
- **Constitution of the RK** (referendum of 15 March 2026): Art. 16(2) no discrimination by origin, sex, race, nationality, language; Art. 21(1) protection of personal data.
- **Benchmarks only** (nothing places the fund in the EU): GDPR (Regulation (EU) 2016/679) Art. 22(1) and (3); EU AI Act (Regulation (EU) 2024/1689) Annex III point 3(a) (access to education is high-risk; whether a scholarship shortlist counts I leave open), Art. 12(1) logs, Art. 14 human oversight.

**Required vs left to me.** The law sets outcomes and rights, not means: Art. 6 does not say "blind a field", Art. 15 does not prescribe a range check, Art. 16 does not prescribe a log format. Blinding, validation, the log schema and the band are my choices. I did more than required in logging withheld fields and the prompt version and in testing layout sensitivity (order probe). **No provision names input validation or field removal**: `validate_input` and `blind` rest on general duties (Art. 15(2)(2), 18(1); Art. 6), and these were the hardest to tie to a provision.

## 5. The ethical principles

Source: **UNESCO, Recommendation on the Ethics of Artificial Intelligence**, adopted 23 November 2021, Section III.2. Status: not law; Member States apply it voluntarily.
- *Fairness and non-discrimination* (paras. 28–30) → what the model sees: blinding.
- *Human oversight and determination* (paras. 35–36) → who decides band cases: escalation.
- *Right to privacy and data protection* (paras. 32–34), *Proportionality and do no harm* (paras. 25–26) → consent, age, purpose refusals; no values in the log.
- *Responsibility and accountability* (paras. 42–43), *Transparency and explainability* (paras. 37–41) → the log and reason codes.
- *Safety and security* (para. 27) → input validation and the injection rule.

**Conflicts.** *Treating applicants identically vs fairly:* hiding the university gives everyone the same view but discards real information about preparation; I put removing the measured +2.5-point channel first, because the fund cannot audit what a 3B model infers from a university name. *Privacy vs accountability:* no values in the log means it cannot rebuild the input alone; I chose the hash. *Oversight vs reviewer time:* a wider band protects more people and costs more reading.

**Where ethics demands more than the law:** 10 of 40 identical applicants changed decision when only the line order changed. No provision asks for it, but fairness requires treating prompt stability as part of the measurement and saying that the floors do not cover it.

## 6. The trade-offs

| control | cost (my run) | harm it prevents |
|---|---|---|
| blind | 0 extra calls (prompt tokens 77 922 vs 84 409), but 50 of 400 outcomes changed, 12 between shortlist and reject, 166 scores moved by 14.01 on average | the +2.5-point university gap (p 0.002) and the 0-against-4 gender direction reaching the model |
| validate_input + refuse | 0 calls; 0 of 400 clean applicants refused; risk of wrong refusals on real names | decisions on impossible data, minors, no consent, wrong purpose, injected instructions |
| log | storage; no field values stored | unaccountable decisions; silent failures |
| escalate | reviewer time: 38 of 400 (9.5 %) | borderline rejections decided by the model alone |

**The band.** Calibrated on blinded scores, as the threshold does not transfer: 160 null-pair profiles (160 extra calls), from the 90th percentile of rejects (40) to the 10th of shortlists (60), ±3, narrowed to ≤ 20 % of profiles. My first, symmetric rule gave (41, 59) = 1.2 % of profiles, because blinded scores cluster at 40 and 60, so almost nobody would be read. I changed it to narrow from the shortlist side, because a wrong rejection removes the applicant while a wrong shortlist meets people later. Result (37, 59): 11.2 % planned on calibration, **9.5 % actually escalated** (on unblinded scores the same band would hold 27.5 %). Reviewers read 38 files; the 24 rejections with scores in the 20s are re-checked by nobody. I would widen the band if the fund had more reviewer capacity, and turn on the counterfactual check if it could pay two calls per profile (my extension: 32 calls for 16 profiles, 0 disagreements).

**What the design still fails to protect against.**
- *Layout fragility:* 25.0 % of identical pairs flipped on line order (mean −6.6 points). Closing it needs a canonical rendering or averaging several layouts, plus a rerun of every measurement; out of scope for this lab.
- *Untested fields:* age, programme and income stay visible and untested for divergence; it needs a new matched-pair design (income is a legitimate need criterion).
- *Notices and objections:* no notice is sent and there is no objection channel, although AI Law Art. 7(2) and PD Law Art. 19-1(2)–(3) require them; that needs the fund's process, not code.
- *Unparseable escalation and my own escalation rule* never fired in the measured run (0 unparseable replies; my rule fired 2 times only in the counterfactual extension and in scripted tests), so they are tested as code, not on real errors.
- *Log integrity, retention and access* are not implemented.
- *Keyword-based injection rule:* the programme field is shown to the model and protected only by patterns.
- *Scope:* one 3B model, one prompt, synthetic data, 40 pairs per group.

## AI-use declaration

I used **Claude (Anthropic)** as an assistant throughout this lab. It wrote the code in the ✏️ cells (my refusal and escalation rules, the band-calibration rule, the harness settings, the control register and the scripted rule test) and drafted the observation cells, `REPORT.md` and this file from the numbers my notebook printed. It suggested which Kazakh and EU instruments and articles apply to each control. 

My own part: I ran the notebook in Colab step by step, checked every output before moving on, decided together with the assistant how to fix the review band after the first version sent only 1.2 % of applicants to a person, and checked that every number in both documents matches my saved outputs. The predictions were written before each result was revealed. As the course requires, I opened each cited legal provision in the primary text myself and checked that it says what I state here; no citation rests on an AI model's statement alone. I am responsible for the submitted work and can explain every control in it.
