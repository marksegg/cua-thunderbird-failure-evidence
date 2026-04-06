# thunderbird rollout findings

mark shi, 2026-04-06  
ubuntu, screenshot-only, opus 4.6, max_steps=300

---

## tldr

ran 12 rollouts across 3 thunderbird tasks. found **one genuine model failure** (d6bfb), **one task with mixed issues** (d6896), and **one gold file bug** (d689e).

| task | runs | verdict |
|------|------|---------|
| d6bfb (genome study filter) | 0/3 | genuine model error |
| d6896 (vendor management) | 0/4 | ambiguous instruction + verifier bug |
| d689e (HR/admin cleanup) | 0/5 | gold file mismatch, model actually succeeds |

---

## d6bfb: genuine model failure

the task asks the model to move a genome study email to Local Folders/Research, then create a filter to catch future ones.

the model moves the email correctly (file_contains=1.0). but when it opens the Message Filters dialog, the "Filters for:" dropdown defaults to **Local Folders** (the right answer). the model switches it to the IMAP account, reasoning that incoming mail filters belong there. creates the filter perfectly otherwise, just in the wrong place.

the verifier checks `Local Folders/msgFilterRules.dat`. that file is empty because the filter went to the IMAP account.

![d6896 filter dialog showing account dropdown](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d6896_run1_filter_creation.png)

*(screenshot from d6896 showing the same pattern -- IMAP account selected instead of Local Folders)*

all 3 runs fail the same way. the model overrides the correct default every time.

worth noting: in real thunderbird, IMAP filters do process incoming mail while Local Folders filters don't. the model's reasoning is sensible but wrong for this task setup.

---

## d6896: not a clean sample

same filter-account issue as d6bfb, but two problems make this one unreliable:

1. the instruction says "Create a message filter" without specifying Local Folders. a similar task (d6897) that passed explicitly said "on the 'Local Folders'." the model can reasonably interpret the instruction either way.

2. the verifier expects filter conditions in the order Subject, From. the instruction says "From contains 'billing@' AND Subject contains 'Invoice'" (From first). the model follows the instruction. the verifier contradicts it. even with the right account, this task would fail.

---

## d689e: gold file bug

the model completes all 7 subproblems correctly (folders, email moves, star, filter, address book, preference). the 3 state-based verifiers pass (sqlite, filter, prefs). the 3 compare_text_file checks fail.

the diffs are only mbox metadata: `From -` envelope timestamps (thunderbird writes wall-clock time on move, gold has original dates) and `X-Mozilla-Status2` internal flags. same class of issue as the pptx text-run-splitting bug. gold files need to be regenerated through the actual pipeline.

![d689e mid-task](https://raw.githubusercontent.com/marksegg/cua-thunderbird-failure-evidence/main/screenshots/d689e_run1_mid.png)

---

## verifier issues found

two things worth flagging to the team:

1. **compare_text_file on mbox files**: needs mbox-aware comparison that ignores `From -` envelope timestamps and `X-Mozilla-Status2` flags. affects d689e and probably other thunderbird tasks using this verifier.

2. **check_thunderbird_filter condition order**: uses order-sensitive list equality. AND conditions are commutative so order shouldn't matter. affects d6896 (instruction and verifier disagree on order).

---

## reproduce

```bash
ssh synth-task-gen
cd /home/ec2-user/scale-cua-env-testing/scale-cua/scale-cua-environments
source venv/bin/activate

python3 -c "
import json
for task, run in [('69bb15eeb46c7fadbf006896', '1'), ('69bb15eeb46c7fadbf00689e', '1'), ('69b8c4222cb4edbf3c9a6bfb', '1')]:
    path = 'results/mark/thunderbird_rollouts/claude_computer_use/screenshot/claude-opus-4-6/thunderbird/%s/run_%s/verifier_details.json' % (task, run)
    with open(path) as f:
        d = json.load(f)
    short = task[-5:]
    for m in d['metrics']:
        print('  %s %s: %s' % (short, m['func'], m['score']))
    print()
"
```
