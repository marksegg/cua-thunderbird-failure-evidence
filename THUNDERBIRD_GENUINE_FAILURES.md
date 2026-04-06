# thunderbird -- findings from rollout analysis

mark shi, 2026-04-06  
instance: `synth-task-gen` / branch `manasi/debug-eval`  
ubuntu, screenshot-only, opus 4.6, max_steps=300, debug_eval

---

## summary

ran rollouts on 3 thunderbird tasks: d689e (5 runs), d6896 (4 runs), d6bfb (3 runs). found **one genuine model failure pattern** and **one gold file / verifier issue**.

**genuine failure (d6896, d6bfb):** model creates message filters under the wrong thunderbird account (IMAP instead of Local Folders). subtle UI context error -- model doesn't check the "Filters for:" dropdown. consistent across all runs.

**gold file issue (d689e):** model actually completes all 7 subproblems correctly, but compare_text_file fails because gold files have wrong mbox envelope timestamps and non-deterministic X-Mozilla-Status2 flags. same class of problem as the pptx text-run-splitting bug.

---

## results

| task | subprobs | runs | pass | verdict | failing metric |
|------|----------|------|------|---------|----------------|
| d6896 (Vendor management) | 8 | 4 | 0 | **GENUINE failure** | check_thunderbird_filter |
| d6bfb (Genome study) | 6 | 3 | 0 | **GENUINE failure** | check_thunderbird_filter |
| d689e (HR/Admin cleanup) | 7 | 5 | 0 | gold file bug | compare_text_file x3 |

---

## d6896 -- filter created under wrong thunderbird account

**task:** create folder structure (Vendor Management > Invoices, Renewals), move 3 specific emails to correct subfolders, create message filter "Vendor Invoice Routing" with conditions From contains "billing@" AND Subject contains "Invoice", change compose format to plain text.

**what the model gets right (4/5 verifier checks pass):**
- folder structure created correctly under Local Folders (check_list=1.0)
- all 3 emails moved to correct folders (file_contains=1.0 x2)
- compose format changed to plain text (check_thunderbird_prefs=1.0)

**what the model gets wrong:**
the model opens Tools > Message Filters while the IMAP inbox is selected. the "Filters for:" dropdown at the top of the dialog defaults to the IMAP account (`rachel.dominguez@conduitcloud.com`). the model doesn't notice or change this to "Local Folders".

the filter gets saved to the IMAP account's msgFilterRules.dat. the verifier checks `Mail/Local Folders/msgFilterRules.dat`, which is empty. result: check_thunderbird_filter=0.0.

this is a subtle UI context error. the model correctly configures the filter name, conditions, and action, but puts it in the wrong place because it doesn't check which account is active in the dialog.

**screenshot -- filter dialog showing wrong account at top:**

*(see screenshots/d6896_run1_filter_creation.png)*

**consistency across runs:**
```
run_1: check_list=1.0, file_contains=1.0, file_contains=1.0, check_thunderbird_filter=0.0, check_thunderbird_prefs=1.0
run_2: check_list=1.0, file_contains=1.0, file_contains=1.0, check_thunderbird_filter=0.0, check_thunderbird_prefs=1.0
run_3: check_list=1.0, file_contains=1.0, file_contains=1.0, check_thunderbird_filter=0.0, check_thunderbird_prefs=1.0
run_4: check_list=1.0, file_contains=1.0, file_contains=1.0, check_thunderbird_filter=0.0, check_thunderbird_prefs=1.0
```

all 4 runs fail the exact same way. 64-68 steps per run.

---

## d689e -- GOLD FILE BUG (model actually succeeds)

**task:** create HR & Admin > Policies, Onboarding folders. move 3 specific emails to correct subfolders and mark unread. star the CLE credits email. create filter "HR Policy Updates". add Derek Morrison to address book. set mailnews.send.warn_on_ctrl_enter to false.

**verifier results (all 5 runs identical):**
```
compare_text_file (Policies) = 0.0    compare_text_file (Onboarding) = 0.0
compare_text_file (Inbox) = 0.0       run_sqlite3 = 1.0
check_thunderbird_filter = 1.0        check_thunderbird_prefs = 1.0
```

**initially looked like an email handling failure.** but byte-level mbox comparison revealed the model completed ALL 7 subproblems correctly:
- emails moved to correct folders (confirmed by X-Mozilla-Status: 0009/0008 ghosts in Inbox)
- moved emails marked as unread
- CLE email starred and left as read (X-Mozilla-Status: 0005 = Read + Starred, matches gold)
- filter, address book, preference all correct

the compare_text_file failures come from two mbox metadata artifacts:

1. **envelope timestamps:** thunderbird rewrites the `From -` line with wall-clock time on move. gold has original dates (`From - Sun Feb 18 00:00:00 2024`), model output has move-time dates (`From - Mon Apr 06 19:34:09 2026`).

2. **X-Mozilla-Status2 flags:** non-deterministic internal thunderbird flag. gold has inconsistent expectations across folders.

same class of issue as pptx text-run-splitting. gold files weren't generated through the actual pipeline.

**note:** Judah's earlier a11y_tree run had a real mis-click on the star icon. our screenshot-only runs do NOT -- model clicks correctly. behavior differs by observation type.

**recommendation:** regenerate gold files, or add mbox-aware options to compare_text_file.

---

## why d6896/d6bfb are genuine failures

1. **multiple metrics pass on every run.** 4/5 metrics pass for d6896. the verifiers work.

2. **the specific model error is identifiable in trajectories and screenshots.** the "Filters for:" dropdown shows the IMAP account. the model never changes it. the filter goes to the wrong place.

3. **failures are perfectly consistent across all runs.** 4/4 d6896 runs, 3/3 d6bfb runs fail the exact same way.

4. **the model claims success.** declares all tasks done, doesn't notice the filter is under the wrong account. classic "agent thinks it succeeded but made a subtle error."

5. **similar tasks DO pass.** d6897 (Ironbridge) passed 1.0 in Judah's data using the same verifiers. the task is solvable.

## why d689e is NOT a genuine failure

the compare_text_file check does a literal string match on mbox files. thunderbird's internal metadata (envelope timestamps, Status2 flags) changes non-deterministically. the gold files were constructed with metadata that doesn't match what thunderbird produces during a real operation. the model does the work correctly.

---

## solvability

**d6896:** Judah's earlier data shows a structurally similar task (d6897, "Ironbridge delivery confirmations") that **passed with score 1.0** using the same verifier functions. this proves the check_thunderbird_filter verifier works when the filter is created correctly. the task is solvable.

**d689e:** similar task (d6895, "Globex Corp deal pipeline") also **passed with 1.0** in Judah's data using even more verifier functions (5 metrics). the email-handling tasks are solvable when the model clicks correctly.

---

## reproduce

```bash
ssh synth-task-gen
cd /home/ec2-user/scale-cua-env-testing/scale-cua/scale-cua-environments
source venv/bin/activate

# check verifier details for any run
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
