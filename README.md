# thunderbird rollout findings

mark shi, 2026-04-06  
instance: `synth-task-gen` / branch `manasi/debug-eval`  
ubuntu, screenshot-only, opus 4.6, max_steps=300

---

## what i ran

5 rollouts on d689e, 4 on d6896, 3 on d6bfb. all thunderbird email client tasks. rollout process died partway through (server contention with other experiments) but got enough data for the top candidates.

## results at a glance

| task | runs | pass | failing check | passing checks |
|------|------|------|---------------|----------------|
| d6896 (vendor mgmt) | 4 | 0 | check_thunderbird_filter | check_list, file_contains x2, check_thunderbird_prefs |
| d6bfb (genome study) | 3 | 0 | check_thunderbird_filter | file_contains |
| d689e (HR/admin) | 5 | 0 | compare_text_file x3 | run_sqlite3, check_thunderbird_filter, check_thunderbird_prefs |

every task has some metrics that pass, some that fail. the pattern is identical across all runs for each task.

---

## d6896: filter under wrong thunderbird account

8 subproblems. model completes 7 correctly. fails on the message filter.

the task asks the model to create folders under Local Folders, move emails there, create a filter that routes future emails to those folders, and change the compose format.

what the model does: opens Tools > Message Filters. the "Filters for:" dropdown at the top defaults to the IMAP account. the model either doesn't notice or rationalizes it as correct. creates the filter under the IMAP account instead of Local Folders.

in run_1, the model actually SAW the dropdown and said "the Filters for dropdown is showing rachel.dominguez@conduitcloud.com" but decided that was the right account. runs 2-4 didn't even mention it.

the filter itself is correctly configured (name, conditions, action). it's just in the wrong place.

### the filter dialog -- note the account dropdown at top

![d6896 filter creation](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d6896_run1_filter_creation.png)

### model navigating the task

![d6896 mid-task](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d6896_run1_mid.png)

### verifier scores (identical across 4 runs)

```
check_list (folder structure):      1.0
file_contains (emails in Invoices): 1.0
file_contains (emails in Renewals): 1.0
check_thunderbird_filter:           0.0  <-- filter not found in Local Folders
check_thunderbird_prefs:            1.0
```

### open questions i'm still investigating

- the task instruction says "create a message filter called 'Vendor Invoice Routing'" but doesn't explicitly say "under Local Folders." is the model's interpretation defensible?
- in thunderbird, a filter under an IMAP account CAN move emails to Local Folders. so functionally the model's filter might work. the verifier just checks a specific file path.
- the verifier also expects conditions in a specific order (Subject first, From second) but the task instruction lists From first. the model follows the instruction order. is that a verifier strictness issue or a model error?
- similar task d6897 (Ironbridge) passed 1.0. need to check if that task's filter was also supposed to be under Local Folders.

---

## d6bfb: same filter-account pattern

6 subproblems. email move works (file_contains=1.0). filter fails (check_thunderbird_filter=0.0).

model created the filter under `dr.carter@genomictechinnovations.com` account instead of Local Folders. same root cause as d6896. the model used "Body contains genome study" as the condition.

the msgFilterRules.dat at the verifier's expected path is 25 bytes (empty header).

same open questions apply: is the instruction clear about where the filter should go?

---

## d689e: model might actually succeed here

7 subproblems. the 3 state-based verifiers (sqlite, filter, prefs) all pass. the 3 compare_text_file checks (mbox folder files) all fail.

a deep byte-level comparison of the model's mbox files vs gold found that the content differences are:
- `From -` envelope timestamps (thunderbird uses wall-clock time on email move, gold has original dates)
- `X-Mozilla-Status2` flags (non-deterministic internal thunderbird metadata)

the actual emails appear to be in the right folders. the star appears to be applied. the model seems to have done everything correctly.

### model working on email operations

![d689e mid-task](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d689e_run1_mid.png)

### thunderbird startup

![d689e start](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d689e_run1_first.png)

### verifier scores (identical across 5 runs)

```
compare_text_file (Policies folder): 0.0  <-- mbox metadata mismatch
compare_text_file (Onboarding):      0.0  <-- mbox metadata mismatch
compare_text_file (Inbox):           0.0  <-- mbox metadata mismatch
run_sqlite3:                         1.0
check_thunderbird_filter:            1.0
check_thunderbird_prefs:             1.0
```

this looks like the same class of issue as the pptx text-run-splitting bug: gold files weren't generated through the actual evaluation pipeline, so internal application metadata doesn't match.

one wrinkle: Judah's earlier run (with screenshot+a11y_tree) showed the model mis-clicking the star icon. our screenshot-only runs don't show that. model behavior varies by observation type.

### open question

still need to independently verify the diff claim. running an adversarial agent to re-check whether the ONLY differences are truly metadata, or if there are actual content errors i missed.

---

## what i'm still checking

i've launched 3 adversarial agents to stress-test these findings:

1. is d6896 actually a task ambiguity problem (instruction doesn't say "Local Folders") rather than a model error?
2. same question for d6bfb
3. independent re-verification of d689e -- are the mbox diffs really just metadata?

will update this doc when those complete. not drawing final conclusions until then.

---

## reproduce

```bash
ssh synth-task-gen
cd /home/ec2-user/scale-cua-env-testing/scale-cua/scale-cua-environments
source venv/bin/activate

python3 -c "
import json
for task, run in [('69bb15eeb46c7fadbf006896', '1'), ('69bb15eeb46c7fadbf00689e', '1')]:
    path = 'results/mark/thunderbird_rollouts/claude_computer_use/screenshot/claude-opus-4-6/thunderbird/%s/run_%s/verifier_details.json' % (task, run)
    with open(path) as f:
        d = json.load(f)
    short = task[-5:]
    for m in d['metrics']:
        print('  %s %s: %s' % (short, m['func'], m['score']))
    print()
"
```

results at `results/mark/thunderbird_rollouts/` on the instance.
