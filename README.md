# Lab 04: Capture Both Output and Error — `&>`, `2>&1`

**Series:** File Operations & Shell Fundamentals · **Lab 4 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (foundational), RHCE EX294 (every Ansible path reference), CKA (every kubelet/etcd/containerd path), RHCA building blocks (RH342 troubleshooting, RH358 services, RH236 storage)  
**Prerequisite:** Bash basics (variables, pipes)  
**Time Estimate:** 30–40 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–12 practical · 13–17 advanced · 18–20 exam-realistic

---

## 🎯 Objective

Capture everything a command produces — both **standard output** (success messages, file listings, query results) and **standard error** (warnings, permission denied, errors) — into the same file or `tee` stream. By the end of this lab you will never again miss an error message because it "didn't show up in the log."

---

## 🧠 Concept: Three File Descriptors Always Open

Every process in Linux starts with **three** pre-opened file descriptors (FDs):

| FD | Stream | Default destination | When it's used |
|---|---|---|---|
| **0** | **stdin** | Keyboard | Reading input |
| **1** | **stdout** | Terminal screen | Normal output (success) |
| **2** | **stderr** | Terminal screen | Error/warning output |

Even though stdout and stderr both **display** on the same screen, they are **separate streams**. `>` only redirects stdout. Errors slip past it onto the screen unless you capture them deliberately.

```
ls /etc /nope
  ├── stdout (FD 1) → "adjtime  alternatives  ..."     → screen by default
  └── stderr (FD 2) → "ls: cannot access '/nope': ..."  → screen by default

ls /etc /nope > out.log
  ├── stdout → out.log
  └── stderr → STILL screen (not captured!)

ls /etc /nope > out.log 2>&1
  ├── stdout → out.log
  └── stderr → out.log (combined)

ls /etc /nope &> out.log
  ├── stdout → out.log
  └── stderr → out.log (shorthand — identical result)
```

> **Why this matters on every cert:** RHCSA Task 14 says "save the output to a file." If you forget `2>&1`, your `find / -mtime -30` produces a partial file plus a wall of `Permission denied` on screen — graders mark this wrong. Ansible's `command:` and `shell:` modules capture both streams by default but expose them as separate keys; you must know which to inspect.

---

## 📚 Redirection Reference

| Operator | Meaning | File state if exists | File state if missing |
|---|---|---|---|
| `> file` | stdout → file | **Overwrites** | Creates |
| `>> file` | stdout → file (append) | Appends | Creates |
| `2> file` | stderr → file | Overwrites | Creates |
| `2>> file` | stderr → file (append) | Appends | Creates |
| `> file 2>&1` | stdout AND stderr → file | Overwrites | Creates |
| `&> file` | stdout AND stderr → file (shorthand) | Overwrites | Creates |
| `&>> file` | stdout AND stderr → file (append) | Appends | Creates |
| `> /dev/null` | Discard stdout | — | — |
| `2> /dev/null` | Discard stderr | — | — |
| `&> /dev/null` | Discard both | — | — |
| `2>&1` | "Send stream 2 to where stream 1 is going" | depends on `>` placement | depends on `>` placement |
| `\| tee FILE` | Display AND save stdout | Overwrites (use `-a` to append) | Creates |
| `2>&1 \| tee FILE` | Display AND save both streams | Overwrites | Creates |

> **The single most important rule:** `> file 2>&1` works. `2>&1 > file` does **not**. The order matters because `2>&1` copies wherever stdout is currently pointing — so stdout must already point to the file when `2>&1` runs.

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 11, 13, 14, 18, 19 — every "save the output" task requires this |
| **RHCE EX294** | Ansible `command:` / `shell:` modules expose `stdout` and `stderr` separately — knowing which to check matters |
| **CKA** | `kubelet --v=4` and `journalctl -u kubelet` produce both streams; capturing both is required for postmortems |
| **RHCA — RH342 (Troubleshooting)** | Capture cmd output during incident — `cmd &> /tmp/incident-$(date +%s).log` |
| **RHCA — RH358 (Services)** | Service start-failure forensics: `systemctl status SERVICE 2>&1 \| tee output.log` |
| **RHCA — RH236 (Storage)** | Gluster heal/repair commands emit warnings to stderr — must be captured |

---

## 🔧 The 20 Tasks

> Each task ends with three callouts: **Switches** (every operator/flag explained), **Output decoded** (what each line means), and **Troubleshoot** (what to do if it goes wrong).

---

### Task 1 — Set up the lab workspace

**Purpose:** Build a clean directory so each subsequent task has predictable filenames.

```bash
mkdir -p ~/redir-lab
cd ~/redir-lab
pwd
```

**Expected output:**

```
/home/ec2-user/redir-lab
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create with missing parents; do not error if it exists |
| `cd` | Move into the directory |
| `pwd` | Confirm location |

**Output decoded**

| Line | Meaning |
|---|---|
| `/home/ec2-user/redir-lab` | Workspace ready |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` on `mkdir` | Use a path under your own home |

---

### Task 2 — See stdout and stderr both on screen

**Purpose:** Demonstrate that even though they look identical on screen, the two streams are separate.

```bash
ls /etc /nonexistent
```

**Expected output (interleaved):**

```
ls: cannot access '/nonexistent': No such file or directory
/etc:
adjtime  alternatives  audit  bashrc  ...
```

**Switches**

| Token | Meaning |
|---|---|
| `ls A B` | List two paths; valid entries go to stdout, errors to stderr |

**Output decoded**

| Line | Stream |
|---|---|
| `ls: cannot access ...` | stderr (FD 2) |
| `/etc:` header and file names | stdout (FD 1) |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Only one stream displayed | Some shells buffer differently — they still arrive at the terminal |

---

### Task 3 — Redirect stdout only with `>`

**Purpose:** Most basic redirection — only success output goes to file; errors stay on screen.

```bash
ls /etc /nonexistent > out.log
cat out.log
```

**Expected output (errors on screen during `ls`, then `cat` shows only stdout):**

```
ls: cannot access '/nonexistent': No such file or directory
/etc:
adjtime
alternatives
audit
bashrc
...
```

**Switches**

| Token | Meaning |
|---|---|
| `> FILE` | Redirect FD 1 (stdout) to FILE (overwrite) |

**Output decoded**

| Phase | What you see |
|---|---|
| During `ls` | stderr line displayed on screen (`>` didn't capture it) |
| `cat out.log` | Only the `/etc:` listing — stderr was not saved |

**Why this matters:** This is the #1 beginner mistake — "I redirected the command and the error STILL appeared." Yes, because `>` doesn't capture stderr.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Errors clutter screen | Add `2>&1` (Task 7) or `2> /dev/null` (Task 12) |

---

### Task 4 — Redirect stderr only with `2>`

**Purpose:** Send errors to a file while letting normal output appear on screen.

```bash
ls /etc /nonexistent 2> err.log
cat err.log
```

**Expected output (stdout on screen, then err.log content via cat):**

```
/etc:
adjtime
...
ls: cannot access '/nonexistent': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `2> FILE` | Redirect FD 2 (stderr) to FILE (overwrite) |
| `2>> FILE` | Same but append |

**Output decoded**

| Phase | What you see |
|---|---|
| During `ls` | The `/etc:` listing on screen — stdout untouched |
| `cat err.log` | Just the error message — stderr was redirected here |

**Why a sysadmin needs this on RHCA RH342:** When you want to see results LIVE but log errors separately for review.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wrong file got the error | Make sure there's no space: `2>err.log` and `2> err.log` both work; `2 > err.log` does **not** |

---

### Task 5 — The classic mistake: `2>&1 > file`

**Purpose:** See the wrong order so you instantly recognize it on the exam.

```bash
ls /etc /nonexistent 2>&1 > wrong.log
echo "----"
cat wrong.log
```

**Expected output:**

```
ls: cannot access '/nonexistent': No such file or directory
----
/etc:
adjtime
alternatives
...
```

**Switches**

| Token | Meaning |
|---|---|
| `2>&1 > FILE` | **Wrong order** — `2>&1` runs FIRST, when stdout still points to the terminal; THEN `> FILE` redirects stdout to FILE |
| Result | stderr still goes to terminal; only stdout reaches the file |

**Output decoded**

| Line | Meaning |
|---|---|
| `cannot access ...` on screen | stderr leaked because `2>&1` copied the terminal-pointing FD |
| `cat wrong.log` shows stdout only | Errors NOT captured |

**Why a sysadmin needs to recognize this:** This pattern is in lots of bad Stack Overflow answers. Spot it and fix it.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Errors keep appearing on screen | Move `2>&1` AFTER the file redirect (Task 6) |

---

### Task 6 — The correct form: `> file 2>&1`

**Purpose:** Capture both streams into one file. Memorize this exact pattern.

```bash
ls /etc /nonexistent > right.log 2>&1
echo "---- (nothing on screen above)"
cat right.log
```

**Expected output:**

```
---- (nothing on screen above)
ls: cannot access '/nonexistent': No such file or directory
/etc:
adjtime
alternatives
...
```

**Switches**

| Token | Meaning |
|---|---|
| `> FILE` | Send stdout to FILE first |
| `2>&1` | THEN send stderr to wherever stdout is going (i.e., FILE) |

**Output decoded**

| Phase | What you see |
|---|---|
| During `ls` | Nothing on screen — both streams captured |
| `cat right.log` | Errors AND listing — interleaved order depends on buffering |

**Read `2>&1` literally as:** *"file descriptor 2, go to where file descriptor 1 is currently going."*

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Errors still on screen | You wrote `2>&1` before `>` — reverse the order |

---

### Task 7 — Decode `2>&1` syntactically

**Purpose:** Lock down the syntax so you never misread it.

```bash
# Equivalent forms
ls /etc /nope > a.log 2>&1
ls /etc /nope 2>&1 1> b.log    # same effect, different writing
ls /etc /nope 1> c.log 2>&1    # explicit 1>

ls -l a.log b.log c.log
```

**Expected output:**

```
-rw-r--r--. 1 ec2-user ec2-user 4321 Sep 12 16:00 a.log
-rw-r--r--. 1 ec2-user ec2-user 4321 Sep 12 16:00 b.log
-rw-r--r--. 1 ec2-user ec2-user 4321 Sep 12 16:00 c.log
```

**Switches**

| Token | Reading |
|---|---|
| `>` | Same as `1>` (stdout) — the `1` is implicit |
| `1>` | Explicit stdout redirection |
| `2>` | stderr redirection |
| `&` (in `2>&1`) | "the same destination as" |
| `2>&1` | "send FD 2 to where FD 1 is going" |

**Output decoded**

| Token | Meaning |
|---|---|
| Three files, same size | All three syntactic forms are equivalent |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Confused by `&` | The `&` here is NOT background — only inside `>&N` does it mean "the same file descriptor" |

---

### Task 8 — The `&>` shorthand

**Purpose:** Bash-specific shorthand for `> file 2>&1`.

```bash
ls /etc /nonexistent &> both.log
cat both.log
```

**Expected output:**

```
ls: cannot access '/nonexistent': No such file or directory
/etc:
adjtime
alternatives
...
```

**Switches**

| Token | Meaning |
|---|---|
| `&> FILE` | Bash shorthand for `> FILE 2>&1` (capture both streams, overwrite) |

**Output decoded**

| Token | Meaning |
|---|---|
| Combined output in one file | Same result as Task 6, fewer characters |

> **Portability note:** `&>` is bash/zsh-specific. POSIX `/bin/sh` (dash on Debian/Ubuntu) does not understand it. For maximum portability use `> FILE 2>&1`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `&> not found` in a `sh` script | Switch script to `#!/bin/bash` or use the long form |

---

### Task 9 — Append both streams with `&>>`

**Purpose:** When you want to keep adding to a log file instead of overwriting.

```bash
ls /etc /nope &> combined.log
ls /tmp /alsonope &>> combined.log
cat combined.log
wc -l combined.log
```

**Expected output (excerpt):**

```
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
...
ls: cannot access '/alsonope': No such file or directory
/tmp:
systemd-private-...
12  combined.log    (approximate)
```

**Switches**

| Token | Meaning |
|---|---|
| `&>>` | Append both stdout and stderr (bash 4+) |
| Long form | `>> FILE 2>&1` |

**Output decoded**

| Phase | Effect |
|---|---|
| First `&>` | Created/overwrote `combined.log` |
| Second `&>>` | Appended to the same file |
| `wc -l` | Confirms both runs are captured |

**Why a sysadmin needs this on RHCA RH358:** Long-running deploys want to **append** to a log, not overwrite each step.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `&>>` not recognized | Use `>> FILE 2>&1` (works in older bash too) |

---

### Task 10 — Append the long way: `>> file 2>&1`

**Purpose:** Portable append form.

```bash
ls /var/log >> portable.log 2>&1
ls /etc/ssh >> portable.log 2>&1
ls /missing >> portable.log 2>&1
tail -3 portable.log
```

**Expected output:**

```
sshd_config
sshd_config.d
ls: cannot access '/missing': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `>>` | Append stdout |
| `2>&1` | Send stderr to wherever stdout is going (the append) |

**Output decoded**

| Line | Stream |
|---|---|
| `sshd_config` rows | stdout, appended |
| `cannot access` row | stderr, also appended |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| File missing | Append creates if absent — same as `&>>` |

---

### Task 11 — Send streams to separate files

**Purpose:** When you want stdout and stderr in different logs for easier filtering.

```bash
ls /etc /nope > stdout.log 2> stderr.log
echo "==== stdout.log ===="
head -3 stdout.log
echo "==== stderr.log ===="
cat stderr.log
```

**Expected output:**

```
==== stdout.log ====
/etc:
adjtime
alternatives
==== stderr.log ====
ls: cannot access '/nope': No such file or directory
```

**Switches**

| Token | Meaning |
|---|---|
| `> stdout.log` | stdout to one file |
| `2> stderr.log` | stderr to a different file |
| Both can be combined freely | Both redirections happen for one command |

**Output decoded**

| File | Contains |
|---|---|
| `stdout.log` | Normal listing |
| `stderr.log` | Just the access error |

**Why a sysadmin needs this on RHCA RH342:** Run a complex job; tail stdout for progress, watch stderr for problems — different windows, different priorities.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Wanted both in one file too | Add `tee` per stream (Task 14) — or run twice |

---

### Task 12 — Discard a stream with `/dev/null`

**Purpose:** Throw away noise (often unwanted `Permission denied` warnings).

```bash
find / -name "sshd_config" 2> /dev/null
find / -name "sshd_config" > /dev/null
find / -name "sshd_config" &> /dev/null; echo "exit=$?"
```

**Expected output:**

```
/etc/ssh/sshd_config
(no output — stdout was discarded)
exit=0
```

**Switches**

| Token | Meaning |
|---|---|
| `/dev/null` | Linux's "black hole" device — anything written is discarded |
| `2> /dev/null` | Discard stderr (commonly used with `find`) |
| `> /dev/null` | Discard stdout |
| `&> /dev/null` | Discard both — used when you only care about the exit code |

**Output decoded**

| Form | Effect |
|---|---|
| First | Real hits shown, errors hidden — most common |
| Second | Errors visible, stdout hidden — rare |
| Third | Nothing visible, exit code preserved — for tests |

**Why a sysadmin needs this on RHCSA Task 14:** `find / 2>/dev/null` hides the inevitable `Permission denied` floods on `/proc` and `/sys`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Errors still appear | You discarded the wrong stream — re-check `2>` vs `>` |

---

### Task 13 — Tee both streams so you see AND save

**Purpose:** `tee` writes its stdin to both stdout AND a file. Pair with `2>&1` to capture both streams.

```bash
ls /etc /nope 2>&1 | tee teed.log
echo "---- file content ----"
head -5 teed.log
```

**Expected output (live during ls + the tee'd file):**

```
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
...
---- file content ----
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
alternatives
audit
```

**Switches**

| Token | Meaning |
|---|---|
| `\| tee FILE` | Read stdin, write to terminal AND FILE |
| `tee -a FILE` | Append to FILE instead of overwriting |
| `2>&1 \| tee FILE` | Combine streams first, then tee both |

**Output decoded**

| Phase | What you see |
|---|---|
| Live | Combined output on screen — `tee` mirrors to terminal |
| File | Identical content saved |

**Why on RHCSA Task 18:** "Install package X and save the install log." `dnf install -y httpd 2>&1 | tee /var/tmp/install.log` shows progress AND saves it.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `tee` overwriting prior log | Use `tee -a` |
| Need root-owned destination | `sudo tee` — your shell can't pipe into a sudo redirection, but it can pipe into sudo tee |

---

### Task 14 — Tee with `sudo` correctly

**Purpose:** `sudo command > /root/file` fails because the shell redirects as YOUR user. `sudo tee` is the canonical fix.

```bash
echo "hello root" | sudo tee /root/sudoed.log > /dev/null
sudo cat /root/sudoed.log
echo "more" | sudo tee -a /root/sudoed.log > /dev/null
sudo cat /root/sudoed.log
```

**Expected output:**

```
hello root
hello root
more
```

**Switches**

| Token | Meaning |
|---|---|
| `sudo tee FILE` | Tee running as root, writing to a root-only file |
| `> /dev/null` (after tee) | Don't echo what tee already printed |
| `tee -a` | Append |

**Output decoded**

| Line | Meaning |
|---|---|
| First content | Written by first run |
| Subsequent lines | Appended thanks to `-a` |

**Why a sysadmin needs this on RHCSA Task 19:** "Save grep output of `/etc/shadow` to `/var/tmp/file`." Either `sudo grep ... > /var/tmp/file` (need to use `sudo bash -c`) or `sudo grep ... | sudo tee /var/tmp/file`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `Permission denied` on `sudo cmd > file` | The `>` runs as you. Use `sudo tee` or `sudo bash -c 'cmd > file'` |

---
# Task 15 — Process Substitution Breakdown

## First: The "File Descriptor" Mental Model

Before anything else, you need this mental model:

**Every running process has 3 built-in "pipes" (called file descriptors):**

| Number | Name | What it is |
|---|---|---|
| `0` | stdin | the pipe data comes *in* from |
| `1` | stdout | the pipe normal output goes *out* to |
| `2` | stderr | the pipe error messages go *out* to |

Think of them as **numbered output ports on the back of a machine.**

---

## The `>` and `2>` translations (plain English)

| Syntax | Human English |
|---|---|
| `> file.log` | "Send port 1 (stdout) into this file" |
| `2> file.log` | "Send port 2 (stderr) into this file" |
| `2>&1` | "Plug port 2 into wherever port 1 is currently going" |
| `> >(cmd)` | "Send port 1 into this *command* instead of a file" |
| `2> >(cmd)` | "Send port 2 into this *command* instead of a file" |

**`2>&1` decoded word by word:**
- `2>` = "redirect stderr"
- `&1` = "to the same destination as file descriptor 1"
- Together: *"merge errors into the same stream as regular output"*

---

## What is `>(cmd)` actually doing?

Normally `>` expects a **filename**. Process substitution tricks bash into creating a **temporary named pipe (FIFO)** and gives you back a fake filename pointing to it — so the redirected data flows into `cmd`'s stdin instead of a file on disk.

```
Without process substitution:   ls /etc  >  myfile.log
With process substitution:      ls /etc  >  >(grep something)
                                              ^^^^^^^^^^^^^^^^^
                                         bash creates a temp FIFO here
```


## Lab 15a — Baseline: see stdout and stderr raw

**Goal:** Understand what the original command produces before any filtering.

```bash
ls /etc /nope
```

**What you'll see:**
- A wall of `/etc` contents on stdout
- `ls: cannot access '/nope': No such file or directory` on stderr

**Key point:** Both streams are mixed in your terminal right now. The labs below separate them.

---

## Lab 15b — Redirect stdout to a file (basic `>`)

```bash
ls /etc /nope > stdout.log
cat stdout.log
```

**What happened:** Port 1 (stdout) went to the file. Port 2 (stderr) still printed to your terminal.

---

## Lab 15c — Redirect stderr to a file (`2>`)

```bash
ls /etc /nope 2> errors.log
cat errors.log
```

**What happened:** Port 2 (stderr) went to the file. Port 1 (stdout) still printed to terminal.

---

## Lab 15d — Redirect both to the same file (`2>&1`)

```bash
ls /etc /nope > both.log 2>&1
cat both.log
```

**Plain English:** "Send stdout to `both.log`, then send stderr to wherever stdout is going." Order matters — `2>&1` must come *after* `>`.

---

## Lab 15e — What happens if you reverse the order (common gotcha)

```bash
ls /etc /nope 2>&1 > reversed.log
cat reversed.log
```

**You'll notice:** stderr still prints to terminal — it doesn't go to the file. That's because `2>&1` ran first (when stdout was still the terminal), *then* stdout got redirected to the file. **Stderr was already wired up and didn't follow.**

---

## Lab 15f — Separate stdout and stderr to different files

```bash
ls /etc /nope > stdout-only.log 2> stderr-only.log
echo "STDOUT:" && cat stdout-only.log
echo "STDERR:" && cat stderr-only.log
```

**What happened:** Each stream went to its own file independently.

---

## Lab 15g — Introduce `tee` (see it AND save it)

**`tee` = splits a stream: saves to file AND passes it through to stdout simultaneously**

```bash
ls /etc /nope 2> >(tee errors-tee.log)
cat errors-tee.log
```

**What happened:** stderr went into `tee`, which both printed it to your screen and saved it to the log.

---

## Lab 15h — Process substitution on stdout: `> >(cmd)`

```bash
ls /etc /nope > >(cat > piped-stdout.log)
cat piped-stdout.log
```

**What happened:** Instead of writing directly to a file, stdout flowed into `cat`, which wrote it to the file. This proves `>(cmd)` works — same result, one extra hop.

---

## Lab 15i — Filter stdout with grep via process substitution

```bash
ls /etc /nope > >(grep "^a" > starts-with-a.log) 2>/dev/null
cat starts-with-a.log
```

**What happened:** stdout piped into `grep`, which only kept lines starting with `a`. `2>/dev/null` silences errors for now.

---

## Lab 15j — The `^/` grep filter explained

```bash
ls /etc /nope > >(grep -v '^/' > stdout-noslash.log) 2>/dev/null
cat stdout-noslash.log
```

**`-v '^/'` means:** "exclude lines that *start with* `/`"

When `ls` lists multiple directories, it prints headers like `/etc:` — this filter strips those headers, leaving only filenames.

---

## Lab 15k — Process substitution on stderr: `2> >(cmd)`

```bash
ls /etc /nope 2> >(grep "nope" > nope-errors.log)
cat nope-errors.log
```

**What happened:** stderr flowed into a `grep` that only kept lines mentioning "nope".

---

## Lab 15l — The `>&2` inside a subshell (echo errors back to terminal)

```bash
ls /etc /nope 2> >(tee errors.log >&2)
```

**Plain English of `>&2` here:** After `tee` saves the line to the file, `>&2` says "also send it back out to stderr so the user still sees it on screen."

Without `>&2`, tee would silently write to the file and you'd never see the error in your terminal.

---

## Lab 15m — Combine both streams into process substitution simultaneously

```bash
ls /etc /nope \
  > >(grep -v '^/' > stdout-noslash.log) \
  2> >(tee errors-only.log >&2)
```

**This is the full Task 15 command.** Now you've built up to it piece by piece.

---

## Lab 15n — Verify both output files look correct

```bash
echo "---- stdout-noslash.log (first 3 lines) ----"
head -3 stdout-noslash.log

echo "---- errors-only.log ----"
cat errors-only.log
```

**Expected:**
- `stdout-noslash.log` → filenames only, no `/etc:` header lines
- `errors-only.log` → the `/nope` error message

---

## Lab 15o — Break it on purpose (test the bash-only restriction)

```bash
sh -c 'ls /etc /nope > >(cat)'
```

**Expected error:** `syntax error near unexpected token`

**Why:** `>(...)` is a **bash/zsh feature only**. POSIX `sh` doesn't support it. On RHCA you must confirm your shebang or interactive shell is `bash`.

---

## Summary Cheat Sheet

```
Port 1 = stdout    → redirect with >
Port 2 = stderr    → redirect with 2>
2>&1               → "merge stderr into stdout's current destination"
>(cmd)             → "use a command as if it were a file"
> >(cmd)           → stdout into a command
2> >(cmd)          → stderr into a command
tee                → save AND pass through simultaneously
>&2 inside a sub   → echo back to stderr so user still sees it

---

### Task 16 — Whole-script redirection with `exec`

**Purpose:** Redirect every subsequent command in a script with one statement.

```bash
cat > script-with-exec.sh <<'EOF'
#!/bin/bash
exec > script-output.log 2>&1
echo "Start: $(date)"
ls /etc /nope
echo "End: $(date)"
EOF
chmod +x script-with-exec.sh
./script-with-exec.sh
cat script-output.log
```

**Expected output:**

```
Start: Thu May 21 14:00:00 EDT 2026
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
...
End: Thu May 21 14:00:00 EDT 2026
```

**Switches**

| Token | Meaning |
|---|---|
| `exec > FILE 2>&1` | From this point forward, all stdout AND stderr in the running shell go to FILE |
| `<<'EOF' ... EOF` | Heredoc — embed script content inline |
| `chmod +x` | Make script executable |

**Output decoded**

| Line | Meaning |
|---|---|
| Each line | All commands' output captured automatically — no per-command redirection needed |

**Why a sysadmin needs this on RHCA RH358:** Service start scripts; capture everything for postmortem with one `exec` line at the top.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `bash: exec: too many arguments` | Don't put commands after the FILE — `exec` only takes the redirection |

---

### Task 17 — Suppress permission noise on `find /`

**Purpose:** Classic exam pattern — search the whole filesystem without drowning in errors.

```bash
find / -name 'passwd' 2>/dev/null | head -5
find / -name 'passwd' > /var/tmp/passwd-hits.txt 2>&1
echo "lines: $(wc -l < /var/tmp/passwd-hits.txt)"
head -5 /var/tmp/passwd-hits.txt
```

**Expected output:**

```
/etc/passwd
/etc/pam.d/passwd
/usr/bin/passwd
/usr/share/bash-completion/completions/passwd
/usr/share/man/man1/passwd.1ssl.gz
lines: 5  (approximate, hit count varies)
/etc/passwd
/etc/pam.d/passwd
...
```

**Switches**

| Token | Meaning |
|---|---|
| `find /` | Search from root |
| `-name 'passwd'` | Match exact filename |
| `2>/dev/null` | Hide permission denied / proc errors |
| `> file 2>&1` | Capture both — useful when the grader wants the full picture |

**Output decoded**

| Phase | What you see |
|---|---|
| First | Clean, filtered list (no noise) |
| Second (file) | Hits AND any errors — useful if grader checks the file |

**Why a sysadmin needs this on RHCSA Task 14:** Different graders want different things — know both patterns.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Find hangs forever | `/proc` and `/sys` have weird files — add `-xdev` to stay on one filesystem |

---

### Task 18 — Capture into a variable

**Purpose:** Use a command's output (including errors) in a script variable.

```bash
result=$(ls /etc /nope 2>&1)
echo "result has $(echo "$result" | wc -l) lines"
echo "first 3:"
echo "$result" | head -3
```

**Expected output:**

```
result has 184 lines    (number varies)
first 3:
ls: cannot access '/nope': No such file or directory
/etc:
adjtime
```

**Switches**

| Token | Meaning |
|---|---|
| `$( ... )` | Command substitution — runs the command and captures its stdout |
| `2>&1` inside | Merge stderr into stdout so both are captured into the variable |
| `"$result"` | Quote so newlines are preserved when echoed |

**Output decoded**

| Line | Meaning |
|---|---|
| `lines: N` | Total lines captured |
| First three | First three lines of combined output |

**Why a sysadmin needs this on RHCE EX294:** Same pattern Ansible's `command:` module uses internally — knowing it helps debug `register: result` outputs.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Trailing newline missing | Bash strips trailing newlines from `$()` — usually harmless |

---

### Task 19 — Cron job with full capture

**Purpose:** Cron emails errors by default. Most production jobs send everything to a log file instead.

```bash
crontab -l > my-current-cron.bak 2>/dev/null || true
cat > my-new-cron.txt <<'EOF'
# Captures both stdout AND stderr into /var/log/myjob.log
*/5 * * * *  /usr/local/bin/myjob.sh >> /var/log/myjob.log 2>&1

# Or bash 4+ shorthand:
# */5 * * * *  /usr/local/bin/myjob.sh &>> /var/log/myjob.log
EOF
cat my-new-cron.txt
```

**Expected output:**

```
# Captures both stdout AND stderr into /var/log/myjob.log
*/5 * * * *  /usr/local/bin/myjob.sh >> /var/log/myjob.log 2>&1

# Or bash 4+ shorthand:
# */5 * * * *  /usr/local/bin/myjob.sh &>> /var/log/myjob.log
```

**Switches**

| Token | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes (m/h/dom/mon/dow) |
| `>> FILE 2>&1` | Append both streams to FILE |
| `&>>` | Bash shorthand |
| `crontab -l` | List current user's crontab |

**Output decoded**

| Line | Meaning |
|---|---|
| The `*/5 ...` row | A working cron entry that loses NOTHING |

**Why a sysadmin needs this on RHCSA Task 13:** Cron jobs that "silently failed" almost always lack `2>&1`. Adding it is the #1 fix.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Cron emails you every error | Add `>> log 2>&1` and email stops, log grows |
| Log doesn't update | Path wrong, permissions wrong, or job exits before running — check `MAILTO=root` for now |

---

### Task 20 — Exam-style scenario: capture, display, and verify

**Task statement (RHCSA-style):** *"Run `find / -mtime -30` and capture both the file list AND any permission errors into `/var/tmp/modfiles.txt`. Also show progress on screen as it runs. Then verify with a line count and a sample."*

```bash
sudo find / -mtime -30 2>&1 | sudo tee /var/tmp/modfiles.txt | head -20
echo "----"
sudo wc -l /var/tmp/modfiles.txt
sudo grep -c "Permission denied" /var/tmp/modfiles.txt
sudo head -5 /var/tmp/modfiles.txt
```

**Expected output (excerpts):**

```
/etc/resolv.conf
/etc/hostname
find: '/proc/1/task/1/fd/4': No such file or directory
...
----
12345 /var/tmp/modfiles.txt   (line count varies)
422   (Permission denied count varies)
/etc/resolv.conf
/etc/hostname
/var/log/messages
...
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `find / -mtime -30` | Files modified in the last 30 days |
| `2>&1` | Merge stderr into stdout so tee sees both |
| `\| sudo tee /var/tmp/modfiles.txt` | Display AND save in one go (sudo needed because `/var/tmp` writes as caller; here actually fine but kept for habit) |
| `\| head -20` | Show only first 20 lines on screen during the run |
| `wc -l` | Audit: total lines captured |
| `grep -c "Permission denied"` | Audit: how many errors got captured (confirms `2>&1` worked) |

**Output decoded**

| Token | Meaning |
|---|---|
| File listing rows | stdout of `find` |
| `Permission denied` / `No such file...` rows | stderr — captured thanks to `2>&1` |
| `wc -l` | Total entries in the log |
| `grep -c` result > 0 | Proves errors were captured (not lost) |

**Cleanup**

```bash
cd ~
rm -rf ~/redir-lab
sudo rm -f /var/tmp/modfiles.txt /var/tmp/passwd-hits.txt /root/sudoed.log
```

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Tee shows the listing but file is empty | Pipe targets only stdout; ensure `2>&1` is BEFORE the pipe |
| Permission denied to write `/var/tmp/...` | Use `sudo tee` |

---

## 🔍 Redirection Decision Guide

```
Do you want to SAVE the output?
  ├── Stdout only? → > file        (>> to append)
  ├── Stderr only? → 2> file       (2>> to append)
  ├── Both?
  │     ├── Overwrite → > file 2>&1     or     &> file
  │     └── Append    → >> file 2>&1    or     &>> file
  ├── Discard?     → cmd > /dev/null   /   2> /dev/null   /   &> /dev/null
  └── Want to see AND save?  → cmd 2>&1 | tee file       (use tee -a to append)

Need to write as root?
  └── sudo cmd 2>&1 | sudo tee /root/file

Whole script captured?
  └── exec > /var/log/myscript.log 2>&1   (at the top)

Separate streams to different files?
  └── cmd > stdout.log 2> stderr.log
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Set up `~/redir-lab`
- [ ] 02 See stdout and stderr both on screen with `ls /etc /nonexistent`
- [ ] 03 Capture stdout only with `>`
- [ ] 04 Capture stderr only with `2>`
- [ ] 05 Recognize the wrong order `2>&1 > file`
- [ ] 06 Apply the correct order `> file 2>&1`
- [ ] 07 Read `2>&1` as "send 2 to where 1 is going"
- [ ] 08 Use the `&>` shorthand
- [ ] 09 Append both streams with `&>>`
- [ ] 10 Append the portable way with `>> file 2>&1`
- [ ] 11 Send streams to two separate files
- [ ] 12 Discard a stream with `/dev/null`
- [ ] 13 Use `2>&1 \| tee file` to see + save
- [ ] 14 Write as root with `sudo tee`
- [ ] 15 Process substitution `> >(grep ...)` and `2> >(tee ...)`
- [ ] 16 `exec > log 2>&1` for whole-script capture
- [ ] 17 Suppress `find /` permission errors with `2>/dev/null`
- [ ] 18 Capture combined output into a variable with `$(cmd 2>&1)`
- [ ] 19 Cron entry that captures both streams with `>> log 2>&1`
- [ ] 20 Exam scenario: `find / -mtime -30` captured + teed + audited

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `2>&1 > file` (wrong order) | Errors still on screen, file has only stdout | Write `> file 2>&1` |
| `>` when you meant `>>` | Existing log destroyed | Use `>>` or `&>>` for appends |
| `sudo cmd > /root/file` | `Permission denied` | `sudo cmd \| sudo tee /root/file` or `sudo bash -c 'cmd > /root/file'` |
| Forgetting `2>&1` in cron | Errors disappear (cron emails them instead) | Always `>> log 2>&1` in cron lines |
| Confusing `&` for background | `2>&1` is **not** background; `&` alone at end is | Read `&` in `>&` as "the same FD" |
| `&>` on `/bin/sh` | Syntax error | Switch to `#!/bin/bash` or use long form |
| `> file 2> file` (same file, separate redirects) | Race conditions — content can interleave badly | Use `> file 2>&1` instead |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Every "save the output" task = `> file 2>&1` (or `&> file`). Default to capturing BOTH.
- `find / ... > /var/tmp/file 2>&1` is the literal answer for Task 14.
- Use `sudo tee` whenever the destination is owned by root.

**RHCE EX294 (Ansible)**
- `command:` and `shell:` modules capture both streams. Access them as `result.stdout` and `result.stderr`.
- `failed_when:` often checks `'error' in result.stderr` — knowing how streams flow matters.

**CKA**
- `kubectl logs POD &> /tmp/pod.log` to grab everything for triage.
- `journalctl -u kubelet --since "10 min ago" &> /tmp/kubelet.log` is the canonical postmortem capture.

**RHCA**
- RH342: `exec > /var/log/incident-$(date +%s).log 2>&1` at the top of remediation scripts.
- RH358: service forensics — `systemctl status SERVICE 2>&1 \| tee /tmp/svc.log`.
- RH236: Gluster ops emit warnings to stderr — always combine with `2>&1`.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 05 — Directory navigation | You must know your path before redirecting into it |
| Lab 06 — `ls -lZ` | Verifying log file ownership and SELinux context after creation |
| Lab 08 — `cp` | Capturing `cp -av` output for audit |
| Lab 11 — `rm` | `rm -rfv DIR > /tmp/deleted.log 2>&1` for audit-friendly deletion |
| Lab 14 — `find` | The canonical command that needs `2>&1` |
| Lab 18 — `dnf` install | `dnf install -y X 2>&1 \| tee /var/tmp/install.log` |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
