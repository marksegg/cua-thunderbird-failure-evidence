# thunderbird rollout findings

mark shi, 2026-04-06  
instance: `synth-task-gen` / branch `manasi/debug-eval`  
ubuntu, screenshot-only, opus 4.6, max_steps=300

---

## bottom line (after adversarial stress-testing)

ran rollouts, analyzed results, then launched 3 adversarial agents to challenge my own conclusions. here's where things actually landed:

| task | runs | initial call | after stress-test | honest verdict |
|------|------|-------------|-------------------|----------------|
| d6bfb (genome study) | 3 | genuine failure | **confirmed** | model overrode correct default. cleanest sample we have |
| d6896 (vendor mgmt) | 4 | genuine failure | **partially overturned** | instruction ambiguous + verifier condition-order bug |
| d689e (HR/admin) | 5 | gold file bug | **confirmed** | model succeeds, mbox metadata mismatch |

**one clean genuine failure (d6bfb)**, one messy mixed-blame case (d6896), one gold file bug (d689e). not as many genuine failures as i hoped to find.

## what i ran

5 rollouts on d689e, 4 on d6896, 3 on d6bfb. all thunderbird email client tasks. rollout process died partway through (server contention) but got enough data for the top candidates.

## verifier results at a glance

| task | runs | pass | failing check | passing checks |
|------|------|------|---------------|----------------|
| d6bfb (genome study) | 3 | 0 | check_thunderbird_filter | file_contains |
| d6896 (vendor mgmt) | 4 | 0 | check_thunderbird_filter | check_list, file_contains x2, check_thunderbird_prefs |
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

## adversarial stress-test results

ran 3 agents specifically to challenge my own claims. here's what they found:

### d6896: claim partially overturned

the "genuine failure" label doesn't fully hold. two independent problems found:

**task instruction is ambiguous.** step 7 says "Create a message filter called 'Vendor Invoice Routing'" but never says "under Local Folders." the passing task d6897 (Ironbridge) explicitly said "on the 'Local Folders'" and the model correctly switched accounts for that one. without that phrase, the model's IMAP placement is a defensible interpretation.

**condition order is a verifier bug.** the instruction says "From contains 'billing@' AND Subject contains 'Invoice'" (From first). the model followed this exact order. but the verifier expects Subject first, From second. confirmed that `_match_record` does order-sensitive list equality. even if the model had the right account, it would still fail. the instruction and verifier contradict each other.

bottom line: ~40% task design, ~30% verifier bug, ~30% model error. not a clean sample.

### d6bfb: claim CONFIRMED genuine

the challenge agent found this is actually stronger than i initially thought:

- the instruction says "create a filter to move future invitations...to this folder" (Local Folders/Research). not ambiguous.
- the Message Filters dialog **defaulted to Local Folders** (the correct answer). the model deliberately switched it to the IMAP account, reasoning "incoming mail filters should be applied there."
- only 1 filter condition (`body contains "genome study"`), so no order issue.
- similar tasks with explicit "Local Folders" instructions pass, proving the verifier works.

one nuance worth flagging: in real thunderbird, IMAP account filters process incoming mail while Local Folders filters do not. the model's choice is arguably more functionally correct for "automatically moving future invitations." but within the evaluation framework, the model is wrong. the model overrode the correct default based on a heuristic that doesn't apply to this task setup.

### d689e: CONFIRMED gold file bug

independent re-verification found:
- the model completed all 7 subproblems correctly. 98 steps, no errors or misclicks.
- CLE email correctly starred (X-Mozilla-Status: 0005 = Read + Starred, matches gold exactly)
- right emails moved to right folders, marked unread
- filter, address book, preference all correct
- the ONLY diffs in the 3 failing mbox comparisons are `From -` envelope timestamps (runtime vs gold construction time) and `X-Mozilla-Status2` internal flags
- `run_sqlite3` passed 1.0, independently confirming the address book entry (not the star though, contrary to earlier assumption)

true score should be 1.0. the 0.0 is from `compare_text_file` doing naive string equality on files that contain non-deterministic thunderbird metadata.

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
