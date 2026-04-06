# thunderbird -- genuine model failures found

mark shi, 2026-04-06  
instance: `synth-task-gen` / branch `manasi/debug-eval`  
ubuntu, screenshot-only, opus 4.6, max_steps=300, debug_eval

---

## summary

ran 5 rollouts on d689e and 4 on d6896 (thunderbird email client tasks). both show **consistent genuine model failures** with specific, identifiable error patterns. unlike pptx where every failure was a verifier bug, thunderbird verifiers check actual application state and the failures are real.

both tasks have multiple verifier metrics where some PASS and some FAIL, confirming the verifiers work correctly and the failures are on specific subproblems.

---

## results

| task | subprobs | runs | pass | failing metric | passing metrics |
|------|----------|------|------|----------------|-----------------|
| d689e (HR/Admin cleanup) | 7 | 5 | 0 | compare_text_file x3 | run_sqlite3, check_thunderbird_filter, check_thunderbird_prefs |
| d6896 (Vendor management) | 8 | 4 | 0 | check_thunderbird_filter | check_list, file_contains x2, check_thunderbird_prefs |
| d6bfb (Genome study) | 6 | 3 | 0 | check_thunderbird_filter | file_contains |

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

## d689e -- email handling failures

**task:** create HR & Admin > Policies, Onboarding folders. move 3 specific emails to correct subfolders and mark unread. star the CLE credits email. create filter "HR Policy Updates". add Derek Morrison to address book. set mailnews.send.warn_on_ctrl_enter to false.

**what the model gets right (3/6 verifier checks pass):**
- filter created correctly for rachel.nguyen@cbhlaw.com account (check_thunderbird_filter=1.0)
- address book contact added, email starring/unread flags correct in sqlite (run_sqlite3=1.0)
- preference set correctly in config editor (check_thunderbird_prefs=1.0)

**what the model gets wrong:**
the compare_text_file checks for Policies, Onboarding, and Inbox mbox files all fail. the model's email file contents don't match the gold files. this indicates problems with how the emails were moved between folders.

from Judah's earlier single-rollout analysis (using screenshot+a11y_tree), the root cause was: the model mis-clicked when trying to star the CLE credits email (clicked in the subject column at x=651 instead of the star column at x~180). a context menu appeared. **the model then hallucinated that the star was applied** ("The Karen O'Shea email is now starred") and moved on. the missing star flag changed the X-Mozilla-Status header in the Inbox mbox file, causing the compare_text_file check to fail.

in our screenshot-only runs, the failure pattern is identical across all 5 runs: all 3 compare_text_file metrics fail, but the 3 state-based checks pass.

**screenshot -- model working on email operations (mid-task):**

*(see screenshots/d689e_run1_mid.png -- shows folder structure created, inbox emails visible)*

**consistency across runs:**
```
run_1: cmp=0.0, cmp=0.0, cmp=0.0, sqlite=1.0, filter=1.0, prefs=1.0
run_2: cmp=0.0, cmp=0.0, cmp=0.0, sqlite=1.0, filter=1.0, prefs=1.0
run_3: cmp=0.0, cmp=0.0, cmp=0.0, sqlite=1.0, filter=1.0, prefs=1.0
run_4: cmp=0.0, cmp=0.0, cmp=0.0, sqlite=1.0, filter=1.0, prefs=1.0
run_5: cmp=0.0, cmp=0.0, cmp=0.0, sqlite=1.0, filter=1.0, prefs=1.0
```

all 5 runs fail the exact same way. 98-103 steps per run.

---

## why these are genuine failures (not verifier bugs)

1. **multiple metrics pass on every run.** the verifiers demonstrably work. if this were a verifier bug, we'd see 0/6 or 0/5 metrics failing, not a specific subset.

2. **failures are on different subproblems for different tasks.** d6896 fails on the filter, d689e fails on email handling. the verifiers catch different types of model errors.

3. **failures are perfectly consistent across runs.** all 4 d6896 runs and all 5 d689e runs show the exact same pass/fail pattern. this rules out random environment issues.

4. **the specific model errors are identifiable in trajectories.** d6896: model doesn't change the "Filters for:" dropdown. d689e: model mis-clicks the star icon and hallucinates success.

5. **the model claims success in both cases.** classic "agent thinks it completed the task but made a subtle error" pattern -- exactly the failure mode PKJA is looking for.

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
