# Lab: Capture Both Output and Error — `&>`, `2>&1`

**Series:** linux-ops-mastery — RHCSA Shells, Terminals & Redirection
**Subjects covered:** Merging stdout and stderr into a single stream, the bash shorthand `&>`, the POSIX-portable form `> file 2>&1`, append variants `&>>` and `>> file 2>&1`, the critical "order matters" rule (`2>&1` after `>`, not before), discarding both streams with `&> /dev/null`, exit-code preservation through combined redirection, and when to merge vs. split streams in forensic captures
**Career arcs covered:** RHCSA (every "save the output" task in EX200 — combining streams keeps the answer complete), RHCE (Ansible `command:` / `shell:` modules capture both into `result.stdout` + `result.stderr`), SRE (incident logging: `cmd &>> /var/log/incident.log` keeps a complete chronological record), DevOps (CI/CD job logs are nearly always combined-stream), AI/MLOps (`python train.py &> runs/exp-42.log` is the single command experiment-capture pattern)
**Prerequisite:** Labs 01 and 02 (you understand FD 1, FD 2, `>` and `2>` independently)
**Time Estimate:** 30 to 45 minutes
**Difficulty arc:** Task 1 foundation · 2–3 `&>` and `2>&1` core forms · 4–5 append variants and the order-matters rule · 6 RHCSA exam-realistic capstone

---

## Objective

Stop missing half the evidence. Until now you have redirected stdout *or* stderr but never both at the same time. By the end of this lab you will combine the two streams into a single file in two different syntaxes — bash's shorthand `&>` and the POSIX-portable `2>&1` form — and you will know exactly **why** the order of those operators matters. The next time a cron job "succeeded" but did the wrong thing, the combined log you saved will contain the warning that explains it.

The capstone is an exam-realistic prompt: *"Run a `find` against `/etc` and save both the matching paths and any permission errors to `/root/find-evidence.log`. Verify that the log contains both the successful matches and at least one `Permission denied` line."*

> **Lab safety note:** Every command in this lab reads from `/etc`, `/var/log`, or writes to your sandbox under `/tmp/combo-lab`. No system files are modified. The same operators you practice here are the ones you will type during exam Task 14 and during every real-world incident response.

---

## Concept: Two Streams, One Destination

You already know that stdout (FD 1) and stderr (FD 2) are independent. To capture both into one place, you have to **make FD 2 point at the same destination as FD 1** — or vice versa. Bash offers two syntaxes for this. They produce the same result; they differ only in portability and operator order.

```
   ┌─────────────────────────────────────────────────────┐
   │   Your command (find, ls, dnf, python ...)          │
   ├─────────────────────────────────────────────────────┤
   │   FD 1  stdout  ────┐                               │
   │                     ├──>  ONE file (or pipe, or /dev/null)
   │   FD 2  stderr  ────┘                               │
   └─────────────────────────────────────────────────────┘

   `cmd &> file`            bash shorthand — merge then write
   `cmd > file 2>&1`        POSIX — open file on FD1, THEN clone FD1 onto FD2
   `cmd 2>&1 > file`        WRONG — clones the terminal onto FD2, then redirects FD1 only
```

The third form is the trap. Most shell beginners type it once, see stderr still on screen, and assume it's broken — they were just one operator-order away from working. We'll do the experiment in Task 3 and never get it wrong again.

> **Why this matters:** Combined streams give you the **complete record** of what a command did. Cron jobs, package installs, training scripts, build pipelines — anything where the diagnostics matter as much as the data needs `&>` or `2>&1` to retain a full audit trail.

---

## 📜 Why Combined Streams Exist — The Story

When Dennis Ritchie split stderr off from stdout in **1974**, he immediately discovered a new problem: sometimes you actually *do* want them merged. Cron jobs, for instance, run unattended — if errors and data scroll past nobody's terminal, you need a single log that captures everything in the order it happened.

The Bourne shell (1977) introduced `2>&1` for this purpose: an explicit "duplicate FD 1 onto FD 2." It was deliberately verbose because the language designer (Stephen Bourne) wanted you to *think* about what you were doing. The order-matters rule is a direct consequence: redirections happen left-to-right, and `2>&1` copies wherever FD 1 currently points.

Bash later (around the mid-1990s) added `&>` as a convenience shortcut for the most common case: `> file 2>&1` is so common that bash gave it a single operator. Korn shell does not have it; dash does not have it; busybox sh does not have it. That is why every portable script on the planet still writes `> file 2>&1` instead of `&> file`.

> **The point of the story:** `2>&1` is the original, portable, slightly verbose form that exists in every Bourne-family shell since 1977. `&>` is bash sugar from the 1990s. Pick the right one for the target environment: production scripts that may run on `/bin/sh` get `> file 2>&1`; interactive bash one-liners get `&>`.

---

## 👪 The Combined-Stream Family — Who Lives There

A small but high-impact family. Memorize all of it.

### The four overwrite forms

| Syntax | Shell support | What it does |
|---|---|---|
| `cmd > file 2>&1` | POSIX (`sh`, `dash`, `bash`, `ksh`) | Open `file` on FD 1, then clone FD 1 onto FD 2 → both into `file` |
| `cmd &> file` | bash (and ksh93+) | Shorthand for the above |
| `cmd >& file` | csh / older bash | Legacy; avoid in new scripts |
| `cmd 2>&1 > file` | POSIX, but **wrong intent** | Clones the **terminal** onto FD 2, then redirects FD 1 — stderr stays on screen |

### The four append forms

| Syntax | Shell support | What it does |
|---|---|---|
| `cmd >> file 2>&1` | POSIX | Append both streams |
| `cmd &>> file` | bash | Shorthand append |
| `cmd >>file 2>>file` | POSIX, but **buggy** | Two separate appends — can interleave or drop bytes under fast writes |
| `cmd &>> /var/log/x` | bash | Best for unattended log accumulation |

### The discard forms

| Syntax | Shell support | What it does |
|---|---|---|
| `cmd > /dev/null 2>&1` | POSIX | Silent mode — everything discarded |
| `cmd &> /dev/null` | bash | Shorthand silent mode |
| `cmd 2>/dev/null 1>/dev/null` | POSIX, long form | Same effect, more typing |

> **The point of the family tree:** Pick the right operator family by asking three questions: *Is this script POSIX-portable?* (`2>&1`), *Am I appending or overwriting?* (`&>>` vs `&>`), and *Do I want the output at all?* (`/dev/null` vs file).

---

## 🔬 The Anatomy of `> file 2>&1` — Why Order Matters

```
$ cmd > file 2>&1
  │   │  │   │
  │   │  │   └─ "Send stream 2 to wherever stream 1 is CURRENTLY going."
  │   │  └─ This file is where stream 1 is going right now.
  │   └─ Redirect stream 1 to a file.
  └─ The command whose output we want to capture.

EVALUATION ORDER (left to right):
  1. open(file, O_WRONLY|O_CREAT|O_TRUNC) → FD now points at file
  2. dup2(opened_fd, 1)                   → FD 1 (stdout) → file
  3. dup2(1, 2)                           → FD 2 (stderr) → file (because FD 1 now points at file)
  4. exec(cmd ...)                        → both streams flow into file

—————————————————————————————————————————————————————————————————

$ cmd 2>&1 > file       ← WRONG ORDER
  │   │    │
  │   │    └─ Redirect stream 1 to a file.
  │   └─ "Send stream 2 to wherever stream 1 is CURRENTLY going." (stdout = terminal at this moment)
  └─ The command.

EVALUATION ORDER (left to right):
  1. dup2(terminal, 2)                    → FD 2 (stderr) → terminal (no change)
  2. open(file, O_WRONLY|O_CREAT|O_TRUNC) → FD now points at file
  3. dup2(opened_fd, 1)                   → FD 1 (stdout) → file
  4. exec(cmd ...)                        → stdout → file; stderr → terminal (still!)
```

> **Reading rule:** `2>&1` is a **snapshot** of where FD 1 currently points. Whatever you do to FD 1 *after* `2>&1` does **not** retroactively change FD 2. Put `2>&1` **after** the `>` operator, always.

---

## 📚 Combined-Stream Reference Table

| Task | Command | Notes |
|---|---|---|
| Combine streams to a new file | `cmd > file 2>&1` | POSIX — works in `/bin/sh` |
| Combine streams to a new file (bash) | `cmd &> file` | Bash shorthand |
| Combine streams, append | `cmd >> file 2>&1` | POSIX append |
| Combine streams, append (bash) | `cmd &>> file` | Bash append |
| Discard both streams | `cmd > /dev/null 2>&1` or `cmd &> /dev/null` | Silent mode |
| Pipe both streams to another command | `cmd 2>&1 \| next` | Merge before pipe |
| Pipe both streams (bash shorthand) | `cmd \|& next` | Bash 4+ shortcut |
| Combined stream to file AND screen | `cmd 2>&1 \| tee file` | Audit + display |
| Combined append to file AND screen | `cmd 2>&1 \| tee -a file` | Long-running jobs |
| Preserve exit code through redirection | `(cmd; echo "exit=$?") &> log` | Wrapper trick |
| Cron-job logging | `* * * * * /path/job &>> /var/log/job.log` | Captures everything per run |

> **Rule one of combined streams:** The operator that opens the file goes **first**. `2>&1` (or its bash sugar) goes **second**. Always.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Exam Task 14 says "save the output of `find / -mtime -30` to a file." Combined-stream capture (`&>` or `> file 2>&1`) avoids the broken half-capture that loses the `Permission denied` evidence. |
| **RHCE candidate** | Ansible `command:` / `shell:` exposes `result.stdout` and `result.stderr` separately. To replicate `&>` in Ansible, capture both and concatenate. |
| **SRE / Platform** | Incident response: `cmd &>> /tmp/incident-$(date +%s).log` is the unambiguous "I want everything that command says, in time order." |
| **DevOps** | CI/CD pipelines: every job's "Raw logs" panel is combined-stream — your local equivalent is `make build &> build.log`. |
| **AI / MLOps** | `python train.py &> runs/exp.log` captures epoch metrics (stdout) and PyTorch warnings (stderr) into one log for postmortem analysis. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build the **stdout + stderr → one file → verify** habit.

---

### Task 1 — Set up the sandbox and confirm streams are still independent

**Purpose:** Build a clean working directory, generate one well-defined success and one well-defined failure on the same command, and re-prove from Lab 02 that `>` alone misses the error.

```bash
mkdir -p /tmp/combo-lab && cd /tmp/combo-lab

ls /etc /nope                       # success on stdout, error on stderr
ls /etc /nope > out.txt              # `>` alone — error still on screen
cat out.txt
ls /etc /nope > out.txt 2> err.txt   # split
cat err.txt
```

**Human-Readable Breakdown:** Build the sandbox, run the same `ls /etc /nope` three different ways: default (both on screen), `>` only (stderr still on screen, file has only stdout), `> ... 2> ...` (split into two files).

**Reading it left to right:** Default `ls` writes paths to FD 1 and "cannot access" to FD 2; both happen to land on the terminal. `> out.txt` rebinds FD 1 only — FD 2 still hits the terminal. The split form rebinds both, but into different files.

**The story:** Every combined-stream conversation starts with this proof. If you cannot see the split, you will not appreciate the merge.

**Expected output:**

```text
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
aliases
...
ls: cannot access '/nope': No such file or directory
(out.txt has only the /etc listing)
ls: cannot access '/nope': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `ls A B` | List two paths |
| `> file` | Send stdout only |
| `2> file` | Send stderr only |
| `> a 2> b` | Send to two files |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `out.txt` has both streams | You used `&>` or `2>&1` — try plain `>` |
| `err.txt` is empty | The error did not happen — confirm with default `ls /etc /nope` |
| `cd: /tmp/combo-lab: No such file or directory` | `mkdir -p` it first |

---

### Task 2 — Merge with bash shorthand `&>`

**Purpose:** Use bash's `&>` operator to capture **both** streams into one file with the shortest possible syntax.

```bash
cd /tmp/combo-lab

ls /etc /nope &> both.txt
cat both.txt
wc -l both.txt

find /etc -name '*.conf' &> /tmp/combo-lab/find.log
wc -l find.log
head -n 3 find.log
tail -n 3 find.log
```

**Human-Readable Breakdown:** Replace `> file 2> file2` with a single `&> file` — bash opens the file once and binds both FD 1 and FD 2 to it. Confirm with `cat`/`wc -l`/`head`/`tail` that both streams ended up interleaved in time order.

**Reading it left to right:** `&>` is bash sugar for `> file 2>&1`. The shell opens `both.txt` with `O_TRUNC`, binds it to FD 1, then duplicates FD 1 onto FD 2. The kernel sees both `write(1, ...)` and `write(2, ...)` from `ls`/`find` and both writes go to the same file, in the order they happen.

**The story:** `&>` is the bash one-liner you reach for at the command line. Anything you would have written with two redirections collapses into one operator. The cost is that `&>` does not work in `/bin/sh` — that's what Task 3 is for.

**Expected output:**

```text
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
...
234 both.txt
358 find.log
/etc/dnf/dnf.conf
/etc/sysctl.conf
/etc/yum.conf
find: ‘/etc/audit’: Permission denied
find: ‘/etc/pki/CA/private’: Permission denied
find: ‘/etc/sudoers.d’: Permission denied
```

**Switches**

| Token | Meaning |
|---|---|
| `&> file` | Bash — both streams to file (overwrite) |
| `wc -l file` | Count lines |
| `head -n N` | First N lines |
| `tail -n N` | Last N lines |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: &>: command not found` | You are in `/bin/dash` or `/bin/sh` — switch to bash or use `> file 2>&1` |
| Lines look out of order | Buffered I/O — stderr is unbuffered, stdout is line-buffered to a pipe |
| File overwritten | `&>` is overwrite mode — use `&>>` for append |

---

### Task 3 — Merge with POSIX `> file 2>&1` and learn the order rule

**Purpose:** Use the POSIX-portable form that works in **every** Bourne-family shell, and prove to yourself that operator order matters.

```bash
cd /tmp/combo-lab

ls /etc /nope > correct.log 2>&1
cat correct.log

ls /etc /nope 2>&1 > wrong.log
cat wrong.log

echo "---compare wrong.log to correct.log---"
wc -l correct.log wrong.log
```

**Human-Readable Breakdown:** Run the same command twice with `2>&1` placed in two different positions. The "correct" form (operator at end) merges both streams into the file. The "wrong" form (operator before `>`) sends stderr to the terminal and only stdout to the file.

**Reading it left to right:** `> correct.log` opens the file on FD 1, then `2>&1` duplicates FD 1's new value (the file) onto FD 2. Both flow into the file. In the wrong form, `2>&1` runs first — FD 1 still points at the terminal at that moment, so FD 2 also points at the terminal. The subsequent `> wrong.log` only rebinds FD 1.

**The story:** This is the single most common shell bug. Every senior engineer learned this rule by debugging a "broken" cron log. Walk through the kernel calls in your head every time, until it's automatic.

**Expected output:**

```text
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
...
ls: cannot access '/nope': No such file or directory   ← printed to terminal
/etc:
adjtime
...                                                     ← printed to wrong.log only
---compare wrong.log to correct.log---
235 correct.log
234 wrong.log     ← one line short — the stderr line is missing
```

**Switches**

| Token | Meaning |
|---|---|
| `> file 2>&1` | POSIX — merge both into file |
| `2>&1 > file` | POSIX — but wrong: stderr stays on terminal |
| `wc -l a b` | Line counts for both files |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `wrong.log` and `correct.log` have the same line count | Re-check operator order — `2>&1` must come **after** `> file` |
| Both files are empty | Both streams produced nothing — verify with the bare command |
| `2>&1` shows on screen as text | You quoted it — `'2>&1'` is literal |

---

### Task 4 — Append both streams and build a cron-style log

**Purpose:** Use the append forms (`&>>` and `>> file 2>&1`) to accumulate the complete output of repeated commands into one growing log — the canonical cron-job logging pattern.

```bash
cd /tmp/combo-lab

> job.log
for i in 1 2 3; do
  echo "===== run $i $(date -Is) =====" &>> job.log
  ls /etc /nope/$i &>> job.log
done

cat job.log
wc -l job.log

dnf check-update >> job.log 2>&1 || true
tail -n 10 job.log
```

**Human-Readable Breakdown:** Truncate `job.log` with the empty `>`. Run a loop three times, each iteration appending both streams to the same log with `&>>`. Then append a `dnf check-update` (which exits 100 when updates are available — we use `|| true` to avoid breaking the script).

**Reading it left to right:** `> job.log` opens the file with `O_TRUNC` and immediately closes — net effect: file is now empty. Each loop iteration opens the log in append mode, binds both streams, and writes. Each `dnf check-update` line also appends both streams.

**The story:** Every cron job on every production server worth running uses this pattern — `* * * * * /path/job >> /var/log/job.log 2>&1`. When the job fails at 03:00 and you wake up at 09:00, the complete log is sitting there waiting for you.

**Expected output:**

```text
===== run 1 2026-05-26T13:10:00-04:00 =====
ls: cannot access '/nope/1': No such file or directory
/etc:
adjtime
...
===== run 2 2026-05-26T13:10:00-04:00 =====
ls: cannot access '/nope/2': No such file or directory
...
===== run 3 ... =====
...
720 job.log
Last metadata expiration check: ...
Dependencies resolved.
================================================================================
 Package          Architecture          Version                    Repository ...
================================================================================
...
```

**Switches**

| Token | Meaning |
|---|---|
| `> file` (no command) | Truncate a file to zero bytes |
| `&>> file` | Bash — both streams, append |
| `>> file 2>&1` | POSIX — both streams, append |
| `\|\| true` | Force overall exit 0 even if previous command failed |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Log overwritten between iterations | You wrote `&>` instead of `&>>` |
| Log entries interleave from parallel processes | `>>` is per-write atomic in append mode for small writes; for safety use `flock` |
| `dnf check-update` exit code 100 | Normal — it means updates are available; `\|\| true` handles it |

---

### Task 5 — Discard both streams with `&> /dev/null`

**Purpose:** Use `&> /dev/null` (or the POSIX `> /dev/null 2>&1`) to silence a command completely — useful for background services, cron jobs whose output you do not need, and noisy tools you only care about by exit code.

```bash
cd /tmp/combo-lab

dnf list installed &> /dev/null
echo "dnf exit: $?"

find / -name '*.conf' &> /dev/null
echo "find exit: $?"

ls /nope &> /dev/null
echo "ls exit: $?"

systemctl status sshd &> /dev/null && echo "sshd is running"
systemctl status fake-nonexistent &> /dev/null || echo "fake-nonexistent failed"
```

**Human-Readable Breakdown:** Run loud commands silently and rely on the **exit code** to know what happened. `&> /dev/null` discards both streams; `$?` and `&&` / `||` give you the truthful result.

**Reading it left to right:** `&> /dev/null` opens the null device on FD 1 and FD 2 — every byte the command writes is discarded by the kernel. `echo "ls exit: $?"` reports the prior command's exit status. `&& echo X` runs `echo` only if the prior command succeeded; `|| echo Y` runs only if it failed.

**The story:** `/dev/null` is the canonical Linux trash can. Inside services and scripts, silencing both streams while still using exit codes is how senior engineers write code that "just runs without spam." Pair with `set -e` and your scripts get loud only when something genuinely breaks.

**Expected output:**

```text
dnf exit: 0
find exit: 0
ls exit: 2
sshd is running
fake-nonexistent failed
```

**Switches**

| Token | Meaning |
|---|---|
| `&> /dev/null` | Bash — discard both streams |
| `> /dev/null 2>&1` | POSIX — discard both streams |
| `cmd && X` | Run X only if cmd succeeded |
| `cmd \|\| Y` | Run Y only if cmd failed |
| `systemctl status NAME` | Exit code reflects unit state |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Errors still on screen | You wrote `> /dev/null` (stdout only) — add `2>&1` |
| `$?` is `0` even on missing service | `systemctl status` exit codes vary — use `systemctl is-active` for boolean |
| `&&` and `\|\|` both fired | Operator-precedence trap — wrap with parens |

---

### Task 6 — Capstone: RHCSA-realistic combined-stream capture

**Task statement:** *"Run `find /etc -type f -name '*.conf'` and save **both** the matching paths and any `Permission denied` errors to `/root/find-evidence.log`. Verify that the log contains successful matches AND at least one `Permission denied` line."*

**Purpose:** Execute a full exam-style answer end-to-end using the combined-stream pattern, then verify the artifact the way a grader would.

```bash
sudo -i

# Run as a non-root account so we actually generate Permission denied lines
su - ec2-user -c "find /etc -type f -name '*.conf' &> /root/find-evidence.log"
# (run as root if you do not have a non-root user; the file still gets both
#  streams, there just won't be any Permission denied messages)

wc -l /root/find-evidence.log
echo "--- matches ---"
grep '\.conf$' /root/find-evidence.log | head -n 3
echo "--- errors ---"
grep 'Permission denied' /root/find-evidence.log | head -n 3

test -s /root/find-evidence.log && echo "VERIFY: file exists and is non-empty"
grep -q 'Permission denied' /root/find-evidence.log && echo "VERIFY: errors captured"
grep -q '\.conf$' /root/find-evidence.log && echo "VERIFY: data captured"
```

**Human-Readable Breakdown:** Run the `find` as a non-root user so it genuinely encounters unreadable directories, and merge both streams into `/root/find-evidence.log` with `&>`. Verify that the file is non-empty, contains real `.conf` paths, and contains at least one `Permission denied` line.

**Layer stack you built:**

```text
/root/find-evidence.log           <- the artifact a grader reads
  ├── stdout of find ...          <- captured by `&>` (FD 1 → file)
  └── stderr of find ...          <- captured by `&>` (FD 2 → same file)
```

**The story:** This is the **canonical 60-second exam answer** when the prompt says "save the output" without telling you which stream. Default to combined-stream capture — you lose nothing and you may save the answer when stderr was the real data source. Memorize the spine: `cmd &> /path/file` or the POSIX `cmd > /path/file 2>&1`, then verify with `grep -q` and `test -s`.

**Expected verification output:**

```text
532 /root/find-evidence.log
--- matches ---
/etc/dnf/dnf.conf
/etc/ssh/sshd_config.d/50-redhat.conf
/etc/yum.conf
--- errors ---
find: ‘/etc/audit’: Permission denied
find: ‘/etc/pki/CA/private’: Permission denied
find: ‘/etc/sudoers.d’: Permission denied
VERIFY: file exists and is non-empty
VERIFY: errors captured
VERIFY: data captured
```

**Cleanup**

```bash
rm -rf /tmp/combo-lab
rm -f /root/find-evidence.log
exit
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| File has only `.conf` paths, no errors | You ran as root — try as a non-root user |
| File has only errors, no `.conf` paths | The `find` filter is wrong, or the path has no `.conf` files |
| `&>` syntax error | Running in `/bin/sh` not `/bin/bash` — switch shell or use `> file 2>&1` |
| `grep -q` printed nothing but echo did not fire | The `&&` chain detected a failure — read each `grep` output by hand |

---

## 🔍 Combined-Stream Decision Guide

```
Got command output you want to capture COMPLETELY?
  │
  ├── "Bash one-liner, just merge them"
  │       └── ✅ cmd &> file
  │
  ├── "Same, append form"
  │       └── ✅ cmd &>> file
  │
  ├── "Must run in /bin/sh or busybox"
  │       └── ✅ cmd > file 2>&1
  │
  ├── "Same, append form"
  │       └── ✅ cmd >> file 2>&1
  │
  ├── "I want to see it AND save it"
  │       └── ✅ cmd 2>&1 | tee file       (or `| tee -a file` to append)
  │
  ├── "Discard everything; rely on exit code"
  │       └── ✅ cmd &> /dev/null
  │       └── ✅ cmd > /dev/null 2>&1
  │
  ├── "Pipe combined stream to another command"
  │       └── ✅ cmd 2>&1 | next            (POSIX)
  │       └── ✅ cmd |& next                (bash 4+)
  │
  └── "Preserve exit code of cmd even after the redirect"
          └── ✅ Just check `$?` immediately after — redirection never changes it
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 Set up `/tmp/combo-lab`, reproduce the stdout-vs-stderr split, and prove `>` alone is incomplete
- [ ] 02 Merge with bash `&>` and verify both streams landed in the same file
- [ ] 03 Merge with POSIX `> file 2>&1` and demonstrate the broken `2>&1 > file` order
- [ ] 04 Build a cron-style log with `&>>` (or `>> file 2>&1`) across multiple runs
- [ ] 05 Discard both streams with `&> /dev/null` and lean on `$?`, `&&`, `\|\|`
- [ ] 06 Execute the RHCSA capstone — capture `find /etc -name '*.conf'` data **and** errors into `/root/find-evidence.log` and verify

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `2>&1` placed before `>` | Stderr stays on screen | `2>&1` always **after** the `>` |
| Used `&>` in `/bin/sh` | `bash: &>: command not found` | Use `> file 2>&1` for portability |
| `&>>` typo as `&> >` | Bash parse error | Single token `&>>` |
| Two separate appends `>>file 2>>file` | Bytes can interleave/drop | Use `>>file 2>&1` |
| Discarded both streams of a long-running job | No way to debug failures | Capture to a file, not `/dev/null` |
| Forgot exit code is preserved | Assumed `&>` zeros `$?` | `$?` is the **command's** exit, not the redirection's |
| Combined stream in a pipe but only stderr matters | Hard to filter | `cmd 2>&1 \| grep -i error` |
| `tee` after merge captures both | Expected screen to remain quiet | `tee` writes to its own stdout — use `tee file > /dev/null` to silence screen |
| Cron job with no redirection | Mail spool fills up | Always `&>> /var/log/job.log` for cron |
| Output truncated mid-line | `cmd` killed by SIGKILL mid-write | Use `set -o pipefail` and check `$?` |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- When the prompt says "save the output," default to combined-stream. You can always grep down later. `cmd &> /root/answer.txt` is the safe one-character upgrade from `cmd > /root/answer.txt`.

**RHCE candidate**
- Ansible exposes `stdout` and `stderr` separately. To emulate `&>`, register the result and concatenate `result.stdout ~ "\n" ~ result.stderr` into a `copy: content:` task.

**SRE / Platform interview**
- "How would you debug a cron job that 'works fine' but produces wrong results?" → "I would change the crontab line to `* * * * * /path/job &>> /var/log/job.log` and read the next run's log for stderr warnings."

**DevOps**
- Container `ENTRYPOINT` should not redirect — let stdout and stderr flow into the container runtime's log driver, which combines them. But local `make` jobs benefit from `make build &> build.log`.

**AI / MLOps**
- Long training runs: `python train.py --epochs 100 &> runs/exp-$(date +%s).log`. Combines metrics and warnings — when accuracy drops on epoch 47, the warning that explains it is in the same log.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 01 — Standard Output Redirection | Half of this lab — the FD 1 side |
| Lab 02 — Standard Error Redirection | The other half — the FD 2 side |
| Lab 03 — Pipe Text Streams | Combine `2>&1` with `\|` for stderr-aware pipelines |
| Lab 14 — File Searching with `find` | The command most often paired with combined-stream capture |
| Lab 20 — Scrolling Through Large Files | Read the combined log you just produced with `less` |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
