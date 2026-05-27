# Lab 04: Capture Both Output and Error — `&>`, `2>&1`

- **Series:** linux-ops-mastery — File Operations & Shell Fundamentals
- **Career arcs covered:** RHCSA EX200 (Tasks 11, 13, 14, 18, 19), RHCE EX294 (Ansible `command:`/`shell:` registers), CKA (kubelet/journalctl postmortems), RHCA building blocks (RH342 troubleshooting capture, RH358 service forensics, RH236 storage warnings)
- **Prerequisite:** Lab 01 (`>`, `>>`), Lab 02 (`2>`, `2>/dev/null`), Lab 03 (`|`, `tee`, `wc -l`)
- **Time Estimate:** 50–70 minutes (20 tasks)
- **Difficulty arc:** Tasks 1–6 foundation · 7–12 practical · 13–17 advanced · 18–20 exam-realistic
- **Practice Directory (lab-wide rotation #04):** `/lib64`
- **Sandbox:** `/tmp/redir-lab`

> **This lab's practice directory is: `/lib64`** — every task references it in at least two commands.

---

## 🎯 Objective

Capture everything a command produces — both **stdout** (success output) and **stderr** (errors and warnings) — into the same file, the same pipeline, or the same variable. By the end you will never again miss an error message because it "didn't show up in the log."

---

## 🧠 Concept: Three File Descriptors Always Open

Every process in Linux starts with **three** pre-opened file descriptors (FDs):

| FD | Stream | Default destination | When it is used |
|---|---|---|---|
| **0** | **stdin** | Keyboard | Reading input |
| **1** | **stdout** | Terminal screen | Normal output (success) |
| **2** | **stderr** | Terminal screen | Error / warning output |

```
ls /lib64 /nope
  ├── stdout (FD 1) → "ld-linux-x86-64.so.2  libc.so.6  ..."  → screen
  └── stderr (FD 2) → "ls: cannot access '/nope': ..."          → screen

ls /lib64 /nope > out.log
  ├── stdout → out.log
  └── stderr → STILL screen (not captured!)

ls /lib64 /nope > out.log 2>&1
  ├── stdout → out.log
  └── stderr → out.log (combined)

ls /lib64 /nope &> out.log
  ├── stdout → out.log
  └── stderr → out.log (shorthand — identical result)
```

> **The single most important rule:** `> file 2>&1` works. `2>&1 > file` does **not**. The order matters because `2>&1` copies wherever stdout is *currently* pointing — so stdout must already point to the file when `2>&1` runs.

---

## 📚 Redirection Reference

| Operator | Meaning | If file exists | If file missing |
|---|---|---|---|
| `> file` | stdout → file | **Overwrites** | Creates |
| `>> file` | stdout → file (append) | Appends | Creates |
| `2> file` | stderr → file | Overwrites | Creates |
| `2>> file` | stderr → file (append) | Appends | Creates |
| `> file 2>&1` | stdout AND stderr → file | Overwrites | Creates |
| `&> file` | stdout AND stderr → file (bash shorthand) | Overwrites | Creates |
| `&>> file` | stdout AND stderr → file (append) | Appends | Creates |
| `> /dev/null` | Discard stdout | — | — |
| `2> /dev/null` | Discard stderr | — | — |
| `&> /dev/null` | Discard both | — | — |
| `2>&1` | "send FD 2 to where FD 1 is currently going" | depends on `>` placement | depends on `>` placement |
| `\| tee FILE` | Show AND save stdout | Overwrites (use `-a` to append) | Creates |
| `2>&1 \| tee FILE` | Show AND save both streams | Overwrites | Creates |
| `>\|` | Force `>` even under `set -o noclobber` | Overwrites | Creates |

---

## 🛣️ Career Pathway

| Cert | Why this lab matters |
|---|---|
| **RHCSA EX200** | Tasks 11, 13, 14, 18, 19 — every "save the output to a file" task requires this |
| **RHCE EX294** | Ansible `command:` / `shell:` modules expose `stdout` and `stderr` separately |
| **CKA** | `journalctl -u kubelet &> /tmp/kubelet.log` is the canonical postmortem capture |
| **RHCA — RH342** | `exec > /var/log/incident-$(date +%s).log 2>&1` at the top of remediation scripts |
| **RHCA — RH358** | `systemctl status SERVICE 2>&1 \| tee /tmp/svc.log` for service forensics |

---

## 🚦 Lab-Wide Setup — Run This BEFORE Task 1

```bash
sudo -i
mkdir -p /tmp/redir-lab
cd /tmp/redir-lab

cat > /tmp/redir-lab/THIS_DIRECTORY.txt <<'EOF'
/lib64 — 64-bit shared libraries

/lib64 holds the 64-bit ELF shared libraries every binary in /bin and /sbin
depends on at runtime. The dynamic loader (ld-linux-x86-64.so.2) lives here
and resolves library symbols when a program starts.

Why it exists: 64-bit binaries cannot link against 32-bit libraries, so
RHEL ships them in a separate path. On a pure 64-bit RHEL system /lib is
typically a symlink to /lib64 — they are effectively the same.

What lives inside it: libc.so.6, libpthread.so.0, libm.so.6, libdl.so.2,
and the dynamic loader. Without /lib64 not a single command on the system
can launch — even /bin/bash cannot start.

Why RHCSA cares: filesystem hierarchy questions, library-troubleshooting
recovery boots, and rd.break / chroot rescue work all require knowing
that /lib64 is on the root partition and must mount before /usr.
EOF

cat /tmp/redir-lab/THIS_DIRECTORY.txt
echo "exit was: $?"
```

> **STOP — wait for output before starting Task 1.**

---

# 🔧 The 20 Tasks

---

## Task 1 — Set up the lab workspace

**Practice directory this task:** `/lib64`
*That directory holds the 64-bit shared libraries every command depends on at runtime.*

### 🔁 Warm-Up (rotation: `>`, `>>`, `wc -l`, `$(date -Is)`)

```bash
date -Is > /tmp/redir-lab/warmup.log
date -Is >> /tmp/redir-lab/warmup.log
ls /lib64 | wc -l > /tmp/redir-lab/libcount.log
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Confirm the sandbox is ready and record the size of `/lib64` before any redirection happens.

### Main Command Block

```bash
cd /tmp/redir-lab
pwd
ls /lib64 | wc -l > task01-libcount.txt
cat task01-libcount.txt
test -s task01-libcount.txt && echo "OK: file has content"
```

### Human-Readable Breakdown

- `cd /tmp/redir-lab` — move into the sandbox.
- `pwd` — confirm you are exactly where you think you are.
- `ls /lib64 | wc -l` — list `/lib64` and let `wc -l` count the lines, i.e. the number of entries.
- `> task01-libcount.txt` — redirect that single number to a file (overwrite).
- `cat task01-libcount.txt` — print the saved number.
- `test -s task01-libcount.txt` — exit 0 if the file exists AND is non-empty.

### Reading It Left to Right

`ls /lib64 | wc -l > task01-libcount.txt`
1. `ls /lib64` runs first; its stdout becomes the input of the next stage.
2. `|` connects stdout-of-left to stdin-of-right.
3. `wc -l` counts lines; its stdout would normally go to the screen.
4. `>` redirects that stdout to `task01-libcount.txt` (truncate / create).

### The Story

Every redirection lab needs a stable workspace. `/tmp` is the RHCSA-blessed sandbox — it is a tmpfs cleared on reboot and writable without sudo. Anchoring the lab in `/tmp/redir-lab` guarantees a clean slate each session.

### Expected Output

```
/tmp/redir-lab
OK: file has content
```

### Switches Table

| Token | Meaning |
|---|---|
| `cd DIR` | Change working directory |
| `pwd` | Print working directory |
| `ls DIR` | List contents of `DIR` |
| `\|` | Pipe stdout of the left command into stdin of the right |
| `wc -l` | Count input lines |
| `>` | Truncate stdout to file (overwrite) |
| `cat FILE` | Print file contents to stdout |
| `test -s FILE` | True if FILE exists and is **s**ize > 0 |
| `&&` | Run the next command only if the previous exited 0 |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `>` | Redirect stdout, truncate target |
|  | `>>` | Redirect stdout, append to target |
|  | `\|` | Connect stdout → stdin between two commands |
|  | `wc -l` | Line counter |
|  | `test -s` | True only when file is non-empty |
|  | `$(date -Is)` | ISO-8601 timestamp via command substitution |
|  | `$(whoami)` | Current login name via command substitution |
|  | `$?` | Exit status of the previous command |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task01-libcount.txt /tmp/redir-lab/warmup.log /tmp/redir-lab/libcount.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `Permission denied` on `mkdir` | Use a path under `/tmp` — every user can write there |
| `cat: ... No such file` | Check `pwd`; you may not be in `/tmp/redir-lab` |
| `test -s` returned 1 | File is empty — the pipe failed; re-run with `set -o pipefail` to surface why |

> **STOP — paste your output before Task 2.**

---

## Task 2 — See stdout and stderr both on screen

**Practice directory this task:** `/lib64`
*Linux prints normal listings and error messages to the same terminal even though they are two separate streams.*

### 🔁 Warm-Up (rotation: `|`, `tee`, `grep -c`, `$(whoami)`)

```bash
ls /lib64 | tee /tmp/redir-lab/lib64.list | wc -l
grep -c 'libc' /tmp/redir-lab/lib64.list
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Demonstrate that stdout and stderr look identical on screen but are dispatched through different file descriptors.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent
echo "----"
ls /lib64 /nonexistent ; echo "exit was: $?"
```

### Human-Readable Breakdown

- `ls /lib64 /nonexistent` — try to list two paths; `/lib64` succeeds (stdout), `/nonexistent` fails (stderr).
- `echo "----"` — separator so you can spot the two runs.
- `; echo "exit was: $?"` — print the **exit code** of the previous `ls` (will be non-zero when *any* argument fails).

### Reading It Left to Right

`ls /lib64 /nonexistent`
1. `ls` is the command.
2. `/lib64` is argument 1 — found, contents go to FD 1 (stdout).
3. `/nonexistent` is argument 2 — missing, error goes to FD 2 (stderr).
4. Both end up at the controlling terminal because no redirection was applied.

### The Story

Before you can route streams you must *see* them. The two streams share a screen so beginners assume they are one. Tasks 3–6 prove they are not.

### Expected Output

```
ls: cannot access '/nonexistent': No such file or directory
/lib64:
ld-linux-x86-64.so.2
libc.so.6
libpthread.so.0
...
----
ls: cannot access '/nonexistent': No such file or directory
/lib64:
ld-linux-x86-64.so.2
...
exit was: 2
```

### Switches Table

| Token | Meaning |
|---|---|
| `ls A B` | List two paths; valid → stdout, invalid → stderr |
| `;` | Command separator — run sequentially regardless of exit status |
| `$?` | Exit status of the previous command (`ls` exits 2 on partial failure) |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | FD 1 | stdout — normal output |
|  | FD 2 | stderr — errors and warnings |
|  | `;` | Sequential command separator |
|  | `$?` | Exit status of the last command |
|  | `tee FILE` | Mirror stdin to FILE and stdout |
|  | `grep -c PATTERN` | Count matching lines |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/lib64.list
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Only one stream appears | Buffering — pipe through `stdbuf -oL` or just wait |
| `exit was: 0` after a missing path | You ran a different command between `ls` and `echo` — `$?` only sees the *immediately* previous one |

> **STOP — paste your output before Task 3.**

---

## Task 3 — Redirect stdout only with `>`

**Practice directory this task:** `/lib64`
*Because `/lib64` is large and noisy, it's the perfect target for proving that `>` does not catch errors.*

### 🔁 Warm-Up (rotation: `2>`, `2>/dev/null`, `wc -w`, `$(hostname)`)

```bash
ls /lib64/no-such-file 2> /tmp/redir-lab/err.log
ls /lib64/no-such-file 2>/dev/null ; echo "exit was: $?"
wc -w /tmp/redir-lab/err.log
echo "Warm-up done by $(whoami) at $(date -Is) on $(hostname)"
echo "exit was: $?"
```

### Purpose

Confirm that `>` redirects **only stdout** — stderr still hits the screen.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent > task03-out.log
echo "==== task03-out.log ===="
head -n 5 task03-out.log
echo "lines captured: $(wc -l < task03-out.log)"
```

### Human-Readable Breakdown

- `ls /lib64 /nonexistent > task03-out.log` — capture stdout (the `/lib64` listing) to file; the error for `/nonexistent` still prints on screen.
- `head -n 5 task03-out.log` — show first five captured lines.
- `wc -l < task03-out.log` — count lines using stdin redirection so `wc` does not print the filename.
- `$(wc -l < ...)` — capture the number for the echo statement.

### Reading It Left to Right

`ls /lib64 /nonexistent > task03-out.log`
1. `ls` is invoked with two arguments.
2. `> task03-out.log` redirects FD 1 (stdout) to the file, truncating it.
3. FD 2 (stderr) was never touched — it still points at the terminal.
4. The error message therefore prints to the screen, not the file.

### The Story

This is the #1 beginner trap: *"I redirected it but the error still appears."* That is correct behavior — `>` is shorthand for `1>`. The next four tasks build the muscle memory to fix it.

### Expected Output

```
ls: cannot access '/nonexistent': No such file or directory
==== task03-out.log ====
/lib64:
ld-linux-x86-64.so.2
libBrokenLocale.so.1
libanl.so.1
libc.so.6
lines captured: 188   (number varies by RHEL minor version)
```

### Switches Table

| Token | Meaning |
|---|---|
| `> FILE` | Redirect FD 1 to FILE (truncate / create) |
| `head -n N` | Print only the first N lines |
| `<` | Redirect stdin from a file |
| `wc -l < FILE` | Count lines without printing the filename |
| `$(cmd)` | Run cmd; substitute its stdout inline |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `> FILE` | stdout → FILE (overwrite) |
|  | stderr is NOT captured by `>` | Still goes to screen |
|  | `head -n N` | First N lines |
|  | `<` | Stream a file into a command's stdin |
|  | `$(wc -l < f)` | Pure-number line count via substitution |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task03-out.log /tmp/redir-lab/err.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Errors clutter the screen | Add `2>&1` (Task 6) or `2> /dev/null` (Task 12) |
| `wc -l` printed a filename | You did `wc -l file` instead of `wc -l < file` |
| `> file` blocked by noclobber | Use `>\|` to force overwrite (Task 8) |

> **STOP — paste your output before Task 4.**

---

## Task 4 — Redirect stderr only with `2>`

**Practice directory this task:** `/lib64`
*A missing `/lib64/<file>` is a controlled way to manufacture a stderr line without affecting any real binary.*

### 🔁 Warm-Up (rotation: `grep -v`, `grep -i`, `wc -c`, `$(uname -r)`)

```bash
grep -v '^#' /etc/passwd | grep -i bash | wc -l
wc -c /etc/hostname
echo "kernel: $(uname -r)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Send only stderr to a file so normal stdout still appears live on the terminal.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent 2> task04-err.log
echo "==== task04-err.log ===="
cat task04-err.log
echo "stderr bytes: $(wc -c < task04-err.log)"
```

### Human-Readable Breakdown

- `2> task04-err.log` — redirect FD 2 (stderr) to file, truncating it.
- The `/lib64` listing prints on the screen live because stdout was never redirected.
- `cat` then reveals the captured error.
- `wc -c < file` reports byte size of stderr (proves it actually got written).

### Reading It Left to Right

`ls /lib64 /nonexistent 2> task04-err.log`
1. `ls` runs.
2. `/lib64` succeeds → FD 1 → terminal.
3. `/nonexistent` fails → FD 2.
4. `2> task04-err.log` redirects FD 2 to the file (truncate / create).

### The Story

Sysadmins often want stdout *live* while errors land in a side log: think `tail -f` of a deploy script next to a separate error log. `2> file` is the canonical pattern.

### Expected Output

```
/lib64:
ld-linux-x86-64.so.2
libc.so.6
libpthread.so.0
...
==== task04-err.log ====
ls: cannot access '/nonexistent': No such file or directory
stderr bytes: 60   (approximate)
```

### Switches Table

| Token | Meaning |
|---|---|
| `2> FILE` | Redirect FD 2 (stderr) to FILE (truncate) |
| `2>> FILE` | Same but append |
| `wc -c` | Count bytes |
| `<` | Send file into stdin |
| `grep -v PATTERN` | Invert match (exclude) |
| `grep -i PATTERN` | Case-insensitive match |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `2>` | stderr → file (overwrite) |
|  | `2>>` | stderr → file (append) |
|  | `wc -c` | Byte count |
|  | `grep -v` | Invert match |
|  | `grep -i` | Case-insensitive match |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task04-err.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Wrong file got the error | Make sure there's no space between digit and `>`: `2> err.log` and `2>err.log` both work; `2 > err.log` does **not** |
| `task04-err.log` is empty | The command had no stderr — try `ls /lib64 /definitely-missing` |

> **STOP — paste your output before Task 5.**

---

## Task 5 — The classic mistake: `2>&1 > file`

**Practice directory this task:** `/lib64`
*We deliberately use `/lib64` and a fake path so the difference between right and wrong ordering is obvious.*

### 🔁 Warm-Up (rotation: `find`, `-name`, `2>/dev/null`, `${PIPESTATUS[@]}`)

```bash
find /lib64 -name 'libc*' -type f 2>/dev/null | head -n 3
find /etc -type f -name '*.conf' 2>/dev/null | wc -l
echo "PIPESTATUS=${PIPESTATUS[@]}"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Recognize the wrong order so you spot it instantly on an exam or in someone else's script.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent 2>&1 > task05-wrong.log
echo "----"
cat task05-wrong.log
echo "stderr STILL on screen above the dashes."
```

### Human-Readable Breakdown

- `2>&1` first — at that moment FD 1 still points at the terminal, so FD 2 also gets pointed at the terminal.
- `> task05-wrong.log` next — redirects FD 1 to the file; FD 2 keeps its earlier (terminal) target.
- Result: stdout in file, stderr on screen.

### Reading It Left to Right

`ls /lib64 /nonexistent 2>&1 > task05-wrong.log`
1. `2>&1` — "FD 2 follows FD 1." FD 1 currently = terminal. So FD 2 → terminal.
2. `> task05-wrong.log` — FD 1 now points at the file; FD 2 still at terminal.
3. Outcome: stderr printed live; stdout captured.

### The Story

A surprising number of Stack Overflow answers paste `2>&1 > file`. It looks right and even "works" most of the time when there are no errors. The day there *is* an error, the log is incomplete.

### Expected Output

```
ls: cannot access '/nonexistent': No such file or directory
----
/lib64:
ld-linux-x86-64.so.2
libc.so.6
...
stderr STILL on screen above the dashes.
```

### Switches Table

| Token | Meaning |
|---|---|
| `2>&1` | "FD 2 → wherever FD 1 currently points" |
| `>` | stdout → file (truncate) |
| Order matters | `2>&1` copies the *current* destination of FD 1 — apply it **after** `>` |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `2>&1` | Duplicate FD 1's destination into FD 2 |
|  | Order rule | Redirections evaluated left-to-right |
|  | Anti-pattern | `2>&1 > file` ⇒ stderr stays on screen |
|  | `find -type f` | Match only regular files |
|  | `find -name PATTERN` | Glob-match file names |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task05-wrong.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Errors keep appearing on screen | Move `2>&1` AFTER the file redirect (Task 6) |
| You wanted *only* stderr captured | Use `2> file` (Task 4); don't merge streams |

> **STOP — paste your output before Task 6.**

---

## Task 6 — The correct form: `> file 2>&1`

**Practice directory this task:** `/lib64`
*This is the exam-correct way to capture every line `/lib64`'s listing plus any error in the same log.*

### 🔁 Warm-Up (rotation: `>`, `>>`, `find -user`, `$(uname -a)`)

```bash
date > /tmp/redir-lab/wu.log
date >> /tmp/redir-lab/wu.log
find /lib64 -user root -type f 2>/dev/null | head -n 3
echo "kernel: $(uname -a)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Capture both stdout AND stderr into one log file — the canonical RHCSA pattern.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent > task06-right.log 2>&1
echo "==== task06-right.log ===="
head -n 5 task06-right.log
echo "...lines: $(wc -l < task06-right.log)"
grep -c "cannot access" task06-right.log
```

### Human-Readable Breakdown

- `> task06-right.log` — FD 1 (stdout) now points at the file.
- `2>&1` — FD 2 (stderr) duplicates the *current* destination of FD 1, which is the file.
- Both streams land in `task06-right.log`.
- `grep -c "cannot access"` proves the stderr line actually got captured.

### Reading It Left to Right

`ls /lib64 /nonexistent > task06-right.log 2>&1`
1. `ls /lib64 /nonexistent` — generates one success listing and one error.
2. `> task06-right.log` — FD 1 → file (truncate).
3. `2>&1` — FD 2 → wherever FD 1 currently points = file.
4. Both streams now write to the same file.

### The Story

Read `2>&1` aloud as: *"file descriptor 2, go to where file descriptor 1 is currently going."* Apply that mental script every time and you will never get the order wrong again.

### Expected Output

```
==== task06-right.log ====
ls: cannot access '/nonexistent': No such file or directory
/lib64:
ld-linux-x86-64.so.2
libc.so.6
libpthread.so.0
...lines: 189
1
```

### Switches Table

| Token | Meaning |
|---|---|
| `> FILE` | FD 1 → FILE (truncate) |
| `2>&1` | FD 2 → wherever FD 1 *currently* points |
| `grep -c PATTERN` | Count matching lines (number only) |
| `head -n N` | First N lines |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `> file 2>&1` | Capture both streams in one file |
|  | `grep -c` | Count matching lines |
|  | `wc -l < file` | Line count without filename |
|  | `find -user USER` | Match files owned by USER |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task06-right.log /tmp/redir-lab/wu.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Errors still on screen | You wrote `2>&1` before `>`; reverse the order |
| File missing | `> file` creates it — re-check `pwd` and spelling |

> **STOP — paste your output before Task 7.**

---

## Task 7 — Decode `2>&1` syntactically

**Practice directory this task:** `/lib64`
*Three syntactic forms produce identical results — and proving it on `/lib64` listings makes them memorable.*

### 🔁 Warm-Up (rotation: `$?`, `${PIPESTATUS[@]}`, `false`, `true`)

```bash
true ; echo "true exit: $?"
false ; echo "false exit: $?"
ls /nope /lib64 2>&1 | wc -l
echo "PIPESTATUS=${PIPESTATUS[@]}"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Lock down the syntax so you never misread it — `>`, `1>`, and `> ... 2>&1` are all kin.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nope > task07-a.log 2>&1
ls /lib64 /nope 2>&1 1> task07-b.log
ls /lib64 /nope 1> task07-c.log 2>&1
ls -l task07-a.log task07-b.log task07-c.log
md5sum task07-a.log task07-b.log task07-c.log
```

### Human-Readable Breakdown

- Version A: standard `> file 2>&1` — both streams in file.
- Version B: `2>&1 1> file` — at first glance looks wrong, but `1>` explicitly redirects FD 1 to the file BEFORE the implicit copy of `2>&1` finalises (left-to-right, with `2>&1` evaluated when seen, but stdout duplication is consulted at command-launch). In practice it yields the same merged file because the shell processes redirections in order and the final destinations resolve the same way.
- Version C: `1> file 2>&1` — explicit version of Task 6. Identical to A.
- `md5sum` proves the three files are byte-identical.

> **Note for bash purists:** in *some* obscure shells `2>&1 1>file` differs from `1>file 2>&1`; in bash both yield identical files because all redirections resolve before the command runs but with descriptor copy happening at the moment `2>&1` is parsed. Test on RHEL bash 5.x — they match.

### Reading It Left to Right

`ls /lib64 /nope 1> task07-c.log 2>&1`
1. `1> task07-c.log` — FD 1 → file.
2. `2>&1` — FD 2 → wherever FD 1 currently points (file).
3. Both streams merge into `task07-c.log`.

### The Story

You will read other people's scripts forever. They use `>`, `1>`, and `2>&1` interchangeably. Recognizing them prevents you from "fixing" working code.

### Expected Output

```
-rw-r--r--. 1 root root 4321 May 27 11:30 task07-a.log
-rw-r--r--. 1 root root 4321 May 27 11:30 task07-b.log
-rw-r--r--. 1 root root 4321 May 27 11:30 task07-c.log
<hash>  task07-a.log
<hash>  task07-b.log
<hash>  task07-c.log
```

### Switches Table

| Token | Meaning |
|---|---|
| `>` | Same as `1>` (stdout) — the `1` is implicit |
| `1>` | Explicit stdout redirection |
| `2>` | stderr redirection |
| `&` in `>&N` | "the same destination as FD N" |
| `2>&1` | FD 2 → wherever FD 1 is going |
| `md5sum` | Hash a file — same hash ⇒ same content |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `>` ↔ `1>` | Equivalent, implicit vs explicit |
|  | `2>&1` | FD 2 copies FD 1's destination |
|  | `md5sum FILE` | Content fingerprint |
|  | `true` / `false` | Exit-0 / exit-1 builtins |
|  | `${PIPESTATUS[@]}` | Array of each pipe stage's exit code |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task07-a.log /tmp/redir-lab/task07-b.log /tmp/redir-lab/task07-c.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Confused by `&` | The `&` here is NOT background — only inside `>&N` does it mean "the same FD" |
| Files have different sizes | Buffering — re-run; the merge order can shift between runs |

> **STOP — paste your output before Task 8.**

---

## Task 8 — The `&>` shorthand and `noclobber`

**Practice directory this task:** `/lib64`
*Bash gives you a one-character shortcut for merging both streams — and `noclobber` protects you from accidental overwrites.*

### 🔁 Warm-Up (rotation: `set -o noclobber`, `set +o noclobber`, `head -n`, `>\|`)

```bash
set -o noclobber
echo "first" > /tmp/redir-lab/clob.log
echo "second" > /tmp/redir-lab/clob.log || echo "blocked by noclobber"
echo "third" >| /tmp/redir-lab/clob.log
set +o noclobber
head -n 2 /tmp/redir-lab/clob.log
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Master the bash shorthand `&>` while learning how `set -o noclobber` prevents accidental data loss.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nonexistent &> task08-both.log
head -n 5 task08-both.log
echo "lines: $(wc -l < task08-both.log)"

set -o noclobber
ls /lib64 &> task08-both.log 2>/dev/null || echo "noclobber refused overwrite"
ls /lib64 &>| task08-forced.log
set +o noclobber

ls -l task08-both.log task08-forced.log
```

### Human-Readable Breakdown

- `ls ... &> task08-both.log` — bash shorthand for `> task08-both.log 2>&1`.
- `set -o noclobber` — forbid `>` from overwriting an existing file.
- `&>| FILE` (also written `>| FILE`) — force overwrite, bypassing noclobber.
- `set +o noclobber` — turn the protection back off.

### Reading It Left to Right

`ls /lib64 /nonexistent &> task08-both.log`
1. `&>` — bash sees the combined redirection token.
2. Internally rewrites as `> task08-both.log 2>&1`.
3. Both streams land in the file.

### The Story

`&>` keeps scripts concise — once you've drilled the long form (Task 6) you can use the short form everywhere. `noclobber` is how senior admins avoid `> file` accidents that destroy production logs.

### Expected Output

```
ls: cannot access '/nonexistent': No such file or directory
/lib64:
ld-linux-x86-64.so.2
libc.so.6
libpthread.so.0
lines: 189
noclobber refused overwrite
-rw-r--r--. 1 root root 4321 May 27 11:32 task08-both.log
-rw-r--r--. 1 root root 4012 May 27 11:32 task08-forced.log
```

### Switches Table

| Token | Meaning |
|---|---|
| `&> FILE` | Bash shorthand for `> FILE 2>&1` |
| `>\| FILE` | Force overwrite, bypassing `noclobber` |
| `set -o noclobber` | Refuse to overwrite existing files with `>` |
| `set +o noclobber` | Turn the rule off |
| `\|\|` | Run the next command only if the previous failed (exit ≠ 0) |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `&>` | Capture both streams |
|  | `noclobber` | Safety against `>` overwrite |
|  | `>\|` | Force overwrite under noclobber |
|  | `\|\|` | Run-on-failure operator |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task08-both.log /tmp/redir-lab/task08-forced.log /tmp/redir-lab/clob.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `&> not found` in a script | Switch shebang to `#!/bin/bash` — POSIX `/bin/sh` does not implement it |
| `cannot overwrite existing file` | `noclobber` is on — use `>\|` or unset with `set +o noclobber` |

> **STOP — paste your output before Task 9.**

---

## Task 9 — Append both streams with `&>>`

**Practice directory this task:** `/lib64`
*Appending lets us run two `/lib64` queries and keep both transcripts in one log.*

### 🔁 Warm-Up (rotation: `head -n`, `tail -n`, `tail -f`, `$(uname -r)`)

```bash
head -n 3 /etc/passwd
tail -n 3 /etc/passwd
ls /lib64 | tail -n 3 > /tmp/redir-lab/lib64-tail.log
echo "kernel: $(uname -r)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Keep adding to an existing log instead of overwriting it on each invocation.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nope &> task09-combined.log
ls /tmp /alsonope &>> task09-combined.log
echo "==== final task09-combined.log size ===="
wc -l task09-combined.log
tail -n 6 task09-combined.log
```

### Human-Readable Breakdown

- First `&>` creates or truncates `task09-combined.log`.
- Second `&>>` appends to the same file.
- `tail -n 6` shows the last six lines, proving both runs are present.

### Reading It Left to Right

`ls /tmp /alsonope &>> task09-combined.log`
1. `&>>` — bash combined operator: stdout+stderr append.
2. Equivalent to `>> task09-combined.log 2>&1`.
3. File pointer opens in append mode; both streams write there.

### The Story

CI pipelines and cron jobs need a *cumulative* log. Truncating on every run hides earlier failures. `&>>` is the bash-friendly way to keep history.

### Expected Output

```
==== final task09-combined.log size ====
12 task09-combined.log   (number varies)
ls: cannot access '/alsonope': No such file or directory
/tmp:
redir-lab
systemd-private-...
...
```

### Switches Table

| Token | Meaning |
|---|---|
| `&>>` | Append stdout and stderr (bash 4+) |
| Long form | `>> FILE 2>&1` |
| `tail -n N` | Print the last N lines |
| `head -n N` | Print the first N lines |
| `tail -f FILE` | Follow file as it grows (for live logs) |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `&>>` | Bash combined append |
|  | `>> FILE 2>&1` | Portable equivalent |
|  | `tail -f` | Live-follow growing file |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task09-combined.log /tmp/redir-lab/lib64-tail.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `&>>` not recognized | Use `>> FILE 2>&1` (Task 10) — older bash or dash |
| Log grows forever | `truncate -s 0 file` or `: > file` resets it without changing inode |

> **STOP — paste your output before Task 10.**

---

## Task 10 — Append the portable way: `>> file 2>&1`

**Practice directory this task:** `/lib64`
*The portable append form works on every POSIX shell — useful when targeting Alpine or BusyBox.*

### 🔁 Warm-Up (rotation: `sort -n`, `sort -t: -k3 -n`, `wc -l`, `$(hostname)`)

```bash
sort -t: -k3 -n /etc/passwd | head -n 3
ls /lib64 | sort | head -n 3
echo "hostname: $(hostname)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Write portable redirections that survive on `/bin/sh`, dash, and BusyBox.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 >> task10-portable.log 2>&1
ls /etc/ssh >> task10-portable.log 2>&1
ls /missing >> task10-portable.log 2>&1
echo "==== tail of task10-portable.log ===="
tail -n 3 task10-portable.log
```

### Human-Readable Breakdown

- Each `ls` appends both streams to the same file using the portable form.
- Final `tail -n 3` confirms the last action — the `ls /missing` error.

### Reading It Left to Right

`ls /missing >> task10-portable.log 2>&1`
1. `>> task10-portable.log` — FD 1 opens file in append mode.
2. `2>&1` — FD 2 mirrors FD 1.
3. Error message appends to file; no overwrite of prior contents.

### The Story

Container base images often ship with `/bin/sh` → `dash`. `&>>` then fails silently. Defaulting to the portable form keeps your scripts working everywhere.

### Expected Output

```
==== tail of task10-portable.log ====
sshd_config
sshd_config.d
ls: cannot access '/missing': No such file or directory
```

### Switches Table

| Token | Meaning |
|---|---|
| `>>` | Append stdout |
| `2>&1` | Merge stderr into stdout's destination |
| `tail -n N` | Last N lines |
| `sort -t: -k3 -n` | Sort numerically by 3rd colon-delimited field |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `>> file 2>&1` | Portable both-stream append |
|  | `sort -n` | Numeric sort |
|  | `sort -t: -k3 -n` | Sort by 3rd colon field, numeric |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task10-portable.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| File missing | Append creates if absent — works the same as `&>>` |
| Output order surprising | Buffering — flush with `stdbuf -oL -eL` or accept interleave |

> **STOP — paste your output before Task 11.**

---

## Task 11 — Send streams to separate files

**Practice directory this task:** `/lib64`
*Sometimes you want `/lib64`'s entries in one file and any access errors in another for separate review.*

### 🔁 Warm-Up (rotation: `test -s`, `test -f`, `test -d`, `$(date -Is)`)

```bash
test -d /lib64 && echo "lib64 is a directory"
test -f /lib64/libc.so.6 && echo "libc.so.6 is a regular file (symlink target)"
test -s /etc/passwd && echo "/etc/passwd has content"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Direct stdout and stderr to two different files so each can be filtered independently.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nope > task11-out.log 2> task11-err.log
echo "==== task11-out.log (head) ===="
head -n 5 task11-out.log
echo "==== task11-err.log ===="
cat task11-err.log
echo "out lines: $(wc -l < task11-out.log)  err lines: $(wc -l < task11-err.log)"
```

### Human-Readable Breakdown

- `> task11-out.log` — FD 1 → first file (overwrite).
- `2> task11-err.log` — FD 2 → second file (overwrite).
- Two distinct files; nothing merged.

### Reading It Left to Right

`ls /lib64 /nope > task11-out.log 2> task11-err.log`
1. `> task11-out.log` — FD 1 → first file.
2. `2> task11-err.log` — FD 2 → second file.
3. Each stream stays in its own lane.

### The Story

When debugging a long deploy, you usually `tail -f stdout.log` for progress in one window and `tail -f stderr.log` in another for errors. This separation is its own discipline.

### Expected Output

```
==== task11-out.log (head) ====
/lib64:
ld-linux-x86-64.so.2
libc.so.6
libpthread.so.0
...
==== task11-err.log ====
ls: cannot access '/nope': No such file or directory
out lines: 189  err lines: 1
```

### Switches Table

| Token | Meaning |
|---|---|
| `> FILE` | FD 1 → FILE |
| `2> FILE` | FD 2 → FILE |
| `head -n N` | First N lines |
| `wc -l < FILE` | Line count without filename |
| `test -d` | True if directory |
| `test -f` | True if regular file |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `> file 2> file2` | Split streams |
|  | `test -d/-f/-s` | Directory / regular file / non-empty checks |
|  | `head -n N` / `tail -n N` | Top / bottom N |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task11-out.log /tmp/redir-lab/task11-err.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Wanted both streams in one file | Use Task 6 / Task 8 / Task 9 instead |
| Race condition (`> file 2> file` to same file) | Use `> file 2>&1` instead |

> **STOP — paste your output before Task 12.**

---

## Task 12 — Discard a stream with `/dev/null`

**Practice directory this task:** `/lib64`
*`find /lib64` is noisy; `/dev/null` lets you throw away the noise.*

### 🔁 Warm-Up (rotation: `chmod +x`, `chmod 644`, `chmod 755`, `$(uname -m)`)

```bash
echo '#!/bin/bash' > /tmp/redir-lab/wu.sh
echo 'echo hi' >> /tmp/redir-lab/wu.sh
chmod 644 /tmp/redir-lab/wu.sh && ls -l /tmp/redir-lab/wu.sh
chmod 755 /tmp/redir-lab/wu.sh && ls -l /tmp/redir-lab/wu.sh
chmod +x /tmp/redir-lab/wu.sh
echo "arch: $(uname -m)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Send unwanted output to the kernel's bit bucket — most common with `find /` permission errors.

### Main Command Block

```bash
cd /tmp/redir-lab
find /lib64 -name 'libc*' -type f 2> /dev/null
find /lib64 -name 'libc*' -type f > /dev/null
find /lib64 -name 'libc*' -type f &> /dev/null ; echo "exit=$?"
```

### Human-Readable Breakdown

- First `find` — hits printed, errors discarded.
- Second `find` — hits discarded, errors visible (rare).
- Third `find` — both discarded, exit code preserved (for tests in scripts).

### Reading It Left to Right

`find /lib64 -name 'libc*' -type f 2> /dev/null`
1. `find /lib64` — descend into `/lib64`.
2. `-name 'libc*'` — match libc-family files.
3. `-type f` — regular files only.
4. `2> /dev/null` — discard any permission errors.

### The Story

`/dev/null` is the canonical "discard" device. On the RHCSA exam, `find / -name pattern 2>/dev/null` is the idiom for searching from `/` without drowning in `Permission denied`.

### Expected Output

```
/lib64/libc.so.6
/lib64/libc_malloc_debug.so.0
/lib64/libcap.so.2.48
... (errors silenced)

... (stdout silenced; nothing printed)

exit=0
```

### Switches Table

| Token | Meaning |
|---|---|
| `/dev/null` | Black-hole device — writes are discarded |
| `2> /dev/null` | Discard stderr |
| `> /dev/null` | Discard stdout |
| `&> /dev/null` | Discard both |
| `find -name PATTERN` | Glob-match filename |
| `find -type f` | Regular files only |
| `chmod 755` | rwxr-xr-x |
| `chmod 644` | rw-r--r-- |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `/dev/null` | Discard sink |
|  | `find -name` | Glob match |
|  | `find -type f` | Regular files |
|  | `chmod 644/755/+x` | Standard permissions |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/wu.sh
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Errors still appear | You discarded the wrong stream — `2>` vs `>` |
| `find` hangs | Add `-xdev` to stay on one filesystem |

> **STOP — paste your output before Task 13.**

---

## Task 13 — Tee both streams so you see AND save

**Practice directory this task:** `/lib64`
*`/lib64` listings make a perfect tee target — long enough to demonstrate `tee`, short enough to inspect.*

### 🔁 Warm-Up (rotation: heredoc `<<'EOF'`, `1>&2`, `2>&1 |`, `$(whoami)`)

```bash
cat <<'EOF' > /tmp/redir-lab/note.txt
demo heredoc body
EOF
echo "this is stderr" 1>&2
ls /nope /lib64 2>&1 | wc -l
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Use `tee` to copy the merged stream to both the screen and a file in one shot.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nope 2>&1 | tee task13-teed.log | head -n 5
echo "---- file content ----"
head -n 5 task13-teed.log
ls /lib64 2>&1 | tee -a task13-teed.log > /dev/null
wc -l task13-teed.log
```

### Human-Readable Breakdown

- `ls ... 2>&1` — merge stderr into stdout *before* the pipe so `tee` sees both.
- `| tee task13-teed.log` — `tee` writes its stdin to the file AND its own stdout.
- `| head -n 5` — only the first five lines reach the terminal live.
- `tee -a` — append on the second run.
- `> /dev/null` — silently append without re-printing.

### Reading It Left to Right

`ls /lib64 /nope 2>&1 | tee task13-teed.log | head -n 5`
1. `ls ...` — generates stdout + stderr.
2. `2>&1` — FD 2 follows FD 1 (which is still the terminal at the time the pipe is set up — but bash sets up redirections before launching).
3. `| tee task13-teed.log` — tee writes to file AND passes through to its stdout.
4. `| head -n 5` — keep only the first five lines.

### The Story

When you install a package live on a customer's box, you want them to *see* progress AND keep a transcript. `dnf install -y httpd 2>&1 | tee /var/tmp/install.log` is the muscle-memory pattern.

### Expected Output

```
ls: cannot access '/nope': No such file or directory
/lib64:
ld-linux-x86-64.so.2
libBrokenLocale.so.1
libanl.so.1
---- file content ----
ls: cannot access '/nope': No such file or directory
/lib64:
ld-linux-x86-64.so.2
libBrokenLocale.so.1
libanl.so.1
378  task13-teed.log
```

### Switches Table

| Token | Meaning |
|---|---|
| `\| tee FILE` | Mirror stdin to FILE and stdout |
| `tee -a` | Append to FILE instead of truncate |
| `2>&1 \|` | Merge stderr into stdout BEFORE the pipe |
| `cat <<'EOF' ... EOF` | Heredoc with single-quoted delimiter (no expansion) |
| `echo TEXT 1>&2` | Send your own message to stderr |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `tee FILE` | Save and display |
|  | `tee -a` | Append while teeing |
|  | `2>&1 \| tee` | Merge then mirror |
|  | `1>&2` | Send a literal echo to stderr |
|  | `<<'EOF'` | Heredoc, no expansion |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task13-teed.log /tmp/redir-lab/note.txt
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `tee` overwrote prior log | Use `tee -a` |
| Need root-owned destination | `sudo tee` — shell can't pipe into a sudo redirection, but it can pipe into `sudo tee` |

> **STOP — paste your output before Task 14.**

---

## Task 14 — Tee with `sudo` correctly

**Practice directory this task:** `/lib64`
*We will reference `/lib64` again while exercising sudo-tee to drop a file under `/root`.*

### 🔁 Warm-Up (rotation: `false`, `true`, `${PIPESTATUS[@]}`, `set -o pipefail`)

```bash
set -o pipefail
false | true ; echo "pipefail on, exit=$? PIPESTATUS=${PIPESTATUS[@]}"
set +o pipefail
false | true ; echo "pipefail off, exit=$? PIPESTATUS=${PIPESTATUS[@]}"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Write to a root-owned destination by piping into `sudo tee` — the canonical pattern.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 | sudo tee /root/task14-libs.log > /dev/null
sudo head -n 3 /root/task14-libs.log
ls /lib64 /nope 2>&1 | sudo tee -a /root/task14-libs.log > /dev/null
sudo tail -n 3 /root/task14-libs.log
sudo wc -l /root/task14-libs.log
```

### Human-Readable Breakdown

- `| sudo tee /root/file > /dev/null` — root-elevated write, suppress duplicate echo on screen.
- `| sudo tee -a /root/file > /dev/null` — append while still root-owned.
- The shell would refuse `sudo cmd > /root/file` because the redirection is applied by *your* shell, not by sudo.

### Reading It Left to Right

`ls /lib64 | sudo tee /root/task14-libs.log > /dev/null`
1. `ls /lib64` — generates the library listing on its stdout.
2. `|` — pipe to next stage.
3. `sudo tee /root/task14-libs.log` — run tee with root privilege, writing to `/root`.
4. `> /dev/null` — discard tee's own stdout so the screen stays clean.

### The Story

This is the answer to one of the most common interview/exam pitfalls: *"How do I write to a root-owned file when piping?"* The answer is `| sudo tee`. Bookmark it.

### Expected Output

```
/lib64:
ld-linux-x86-64.so.2
libBrokenLocale.so.1
...
ls: cannot access '/nope': No such file or directory
192 /root/task14-libs.log
```

### Switches Table

| Token | Meaning |
|---|---|
| `sudo tee FILE` | Tee running as root |
| `sudo tee -a FILE` | Append as root |
| `> /dev/null` after tee | Don't double-print |
| `set -o pipefail` | Pipeline exit = first non-zero stage's exit |
| `set +o pipefail` | Turn pipefail off |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `\| sudo tee` | Write to root-owned files via pipe |
|  | `\| sudo tee -a` | Same, but append |
|  | `set -o pipefail` | Surface failures buried in pipes |
|  | `false \| true` | Demonstrates exit-code propagation |

### 🧹 Cleanup

```bash
sudo rm -f /root/task14-libs.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `Permission denied` on `sudo cmd > file` | The `>` runs as you. Use `sudo tee` or `sudo bash -c 'cmd > file'` |
| Lost an error in a pipe | Turn on `set -o pipefail` so the pipeline reports failure |

> **STOP — paste your output before Task 15.**

---

## Task 15 — Process substitution `> >(cmd)`

**Practice directory this task:** `/lib64`
*Branch each stream to its own filter without writing intermediate files.*

### 🔁 Warm-Up (rotation: `>`, `>>`, `wc -l`, `$(date -Is)`)

```bash
ls /lib64 > /tmp/redir-lab/wu.list
ls /lib64 >> /tmp/redir-lab/wu.list
wc -l /tmp/redir-lab/wu.list
echo "stamp: $(date -Is)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Send stdout and stderr through *different* commands using bash process substitution.

### Main Command Block

```bash
cd /tmp/redir-lab
ls /lib64 /nope \
  > >(grep -v '^/' > task15-stdout.log) \
  2> >(tee task15-errors.log >&2)
echo "==== task15-stdout.log ===="
head -n 3 task15-stdout.log
echo "==== task15-errors.log ===="
cat task15-errors.log
```

### Human-Readable Breakdown

- `> >(cmd)` — process substitution; bash builds a FIFO and pipes the redirected stream into `cmd`.
- `> >(grep -v '^/' > task15-stdout.log)` — stdout filtered to drop lines starting with `/`, the result saved.
- `2> >(tee task15-errors.log >&2)` — stderr teed into a file AND echoed back to the original stderr so the user still sees it.
- `>&2` inside the tee — duplicates tee's stdout back onto the parent's FD 2.

### Reading It Left to Right

`> >(grep -v '^/' > task15-stdout.log)`
1. `>` — redirect FD 1.
2. `>(...)` — process-substitution target; bash creates a `/dev/fd/N` FIFO.
3. `grep -v '^/'` — invert match: exclude lines beginning with `/`.
4. `> task15-stdout.log` — inside the subshell, save filtered output.

### The Story

Process substitution lets you build a one-line "stream router" instead of writing intermediate files. RHCA RH342 (Troubleshooting) labs love this for incident triage scripts.

### Expected Output

```
==== task15-stdout.log ====
ld-linux-x86-64.so.2
libBrokenLocale.so.1
libanl.so.1
==== task15-errors.log ====
ls: cannot access '/nope': No such file or directory
```

### Switches Table

| Token | Meaning |
|---|---|
| `>(cmd)` | Process substitution as a write target |
| `<(cmd)` | Process substitution as a read source |
| `grep -v PATTERN` | Inverted match |
| `grep -n PATTERN` | Show line numbers with matches |
| `>&2` | Send a stream back to the parent's FD 2 |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `>(cmd)` | FIFO write target |
|  | `<(cmd)` | FIFO read source |
|  | `grep -v` | Invert match |
|  | `grep -n` | Show line numbers |
|  | `>&2` | Push back to parent stderr |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task15-stdout.log /tmp/redir-lab/task15-errors.log /tmp/redir-lab/wu.list
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `syntax error near unexpected token >` | `>(...)` is bash/zsh only — re-shebang the script `#!/bin/bash` |
| `task15-errors.log` empty | The command produced no stderr — try a path that doesn't exist |

> **STOP — paste your output before Task 16.**

---

## Task 16 — Whole-script redirection with `exec`

**Practice directory this task:** `/lib64`
*One `exec` line at the top of a script captures every later command — perfect for an incident-response script that surveys `/lib64` health.*

### 🔁 Warm-Up (rotation: heredoc `<<'EOF'`, `chmod +x`, `test -x`, `$(uname -a)`)

```bash
cat <<'EOF' > /tmp/redir-lab/probe.sh
#!/bin/bash
echo "probe by $(whoami) on $(uname -n)"
EOF
chmod +x /tmp/redir-lab/probe.sh
test -x /tmp/redir-lab/probe.sh && /tmp/redir-lab/probe.sh
echo "kernel: $(uname -a)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Redirect every subsequent command of a script with a single statement.

### Main Command Block

```bash
cd /tmp/redir-lab
cat > task16-script.sh <<'EOF'
#!/bin/bash
set -euo pipefail
exec > /tmp/redir-lab/task16-out.log 2>&1
echo "START: $(date -Is)"
ls /lib64 | head -n 3
ls /nope
echo "END:   $(date -Is)"
EOF
chmod +x task16-script.sh
./task16-script.sh ; echo "outer exit=$?"
cat task16-out.log
```

### Human-Readable Breakdown

- `exec > FILE 2>&1` — from this line on, every command in the current shell has stdout AND stderr redirected to `FILE`.
- `set -euo pipefail` — strict mode: exit on error, unset variables fail, pipeline status surfaces.
- Outer `echo "outer exit=$?"` — the script ran but `ls /nope` returned non-zero; under `-e` the script exits early, so we use a wrapper to keep the lab moving.

### Reading It Left to Right

`exec > /tmp/redir-lab/task16-out.log 2>&1`
1. `exec` — replace current shell's redirections, not a new process.
2. `> /tmp/redir-lab/task16-out.log` — FD 1 → file.
3. `2>&1` — FD 2 follows FD 1.
4. All future commands inherit these.

### The Story

Production runbooks and post-incident scripts want a single transcript file. `exec ... 2>&1` is the senior-admin idiom — write it once, capture everything.

### Expected Output

```
outer exit=1
START: 2026-05-27T11:40:00-04:00
ld-linux-x86-64.so.2
libBrokenLocale.so.1
libanl.so.1
ls: cannot access '/nope': No such file or directory
```

(The `END:` line is missing because `set -e` aborted on the failing `ls /nope`. Remove `-e` if you want the script to continue.)

### Switches Table

| Token | Meaning |
|---|---|
| `exec > FILE 2>&1` | Set the shell's redirections from now on |
| `set -e` | Exit immediately on first error |
| `set -u` | Treat unset variables as errors |
| `set -o pipefail` | Pipeline exit = last non-zero stage |
| `<<'EOF' ... EOF` | Heredoc; single-quoted delimiter prevents expansion |
| `test -x` | True if executable |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `exec > file 2>&1` | Whole-shell redirection |
|  | `set -euo pipefail` | Strict-mode trio |
|  | `<<'EOF'` heredoc | Inline file content |
|  | `chmod +x` | Make script executable |
|  | `test -x` | Test executable bit |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task16-script.sh /tmp/redir-lab/task16-out.log /tmp/redir-lab/probe.sh
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `bash: exec: too many arguments` | `exec` only takes the redirection — no commands on the same line |
| Script exits silently | `set -e` aborted on a non-zero — read the captured log |

> **STOP — paste your output before Task 17.**

---

## Task 17 — Suppress permission noise on `find /`

**Practice directory this task:** `/lib64`
*The classic exam pattern — search from `/` (which touches `/lib64`) while hiding the inevitable `Permission denied`.*

### 🔁 Warm-Up (rotation: pipes `|`, `2>&1 |`, `grep -c`, `${PIPESTATUS[@]}`)

```bash
ls /lib64 /nope 2>&1 | grep -c .
echo "PIPESTATUS=${PIPESTATUS[@]}"
find /lib64 -name 'libc*' 2>/dev/null | wc -l
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Run a `find /` search cleanly: see the hits, suppress the noise, and still record both streams when needed.

### Main Command Block

```bash
cd /tmp/redir-lab
find / -name 'passwd' 2>/dev/null | head -n 5
find / -name 'passwd' > task17-hits.log 2>&1
echo "lines: $(wc -l < task17-hits.log)"
grep -c 'Permission denied' task17-hits.log || true
head -n 5 task17-hits.log
```

### Human-Readable Breakdown

- `2>/dev/null` — discard `Permission denied` lines from `/proc`, `/sys`, etc.
- `> task17-hits.log 2>&1` — alternative capture-everything variant for graders.
- `grep -c "Permission denied" ... || true` — count of suppressed errors; `|| true` prevents `grep`'s exit-1 on zero matches.

### Reading It Left to Right

`find / -name 'passwd' 2>/dev/null | head -n 5`
1. `find /` — search from root.
2. `-name 'passwd'` — exact filename match.
3. `2>/dev/null` — discard permission errors.
4. `| head -n 5` — keep the first five hits.

### The Story

`find / 2>/dev/null` is the muscle-memory pattern for exam tasks 14 and 19. Memorize it; you will type it many times.

### Expected Output

```
/etc/passwd
/etc/pam.d/passwd
/usr/bin/passwd
/usr/share/bash-completion/completions/passwd
/usr/share/man/man1/passwd.1.gz
lines: 5   (varies)
0
/etc/passwd
/etc/pam.d/passwd
/usr/bin/passwd
/usr/share/bash-completion/completions/passwd
/usr/share/man/man1/passwd.1.gz
```

### Switches Table

| Token | Meaning |
|---|---|
| `find / -name` | Search from root by name |
| `2>/dev/null` | Discard stderr |
| `\| head -n 5` | First five lines |
| `grep -c PATTERN` | Count matches |
| `\|\| true` | Force exit-0 so pipelines under `set -e` keep going |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `find / -name P 2>/dev/null` | Clean filesystem search |
|  | `\|\| true` | Soften non-zero exits |
|  | `grep -c` | Count matching lines |
|  | `head -n N` | Limit output |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task17-hits.log
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `find` runs forever | Add `-xdev` to stay on one filesystem |
| Many permission errors leak | You only redirected stdout — add `2>/dev/null` or `> file 2>&1` |

> **STOP — paste your output before Task 18.**

---

## Task 18 — Capture into a variable

**Practice directory this task:** `/lib64`
*Capture `/lib64` listings AND any errors into a single shell variable for further processing.*

### 🔁 Warm-Up (rotation: command substitution `$(...)`, `wc -l`, `$(uname -r)`, `$(hostname)`)

```bash
echo "$(ls /lib64 | wc -l) entries in /lib64 on $(hostname) ($(uname -r))"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Store the combined output of a command in a variable so a script can branch on it.

### Main Command Block

```bash
cd /tmp/redir-lab
result=$(ls /lib64 /nope 2>&1)
echo "lines: $(echo "$result" | wc -l)"
echo "first 3:"
echo "$result" | head -n 3
echo "$result" | grep -c 'cannot access' || true
```

### Human-Readable Breakdown

- `result=$(... 2>&1)` — capture the merged stream as one string. The `2>&1` is inside the subshell so stderr is included.
- `"$result"` — quote it so newlines survive when echoed.
- `echo "$result" | wc -l` — line count of captured text.
- `echo "$result" | grep -c 'cannot access'` — prove the error was captured.

### Reading It Left to Right

`result=$(ls /lib64 /nope 2>&1)`
1. `$(...)` — command substitution: run the subshell, capture stdout as a string.
2. Inside: `ls /lib64 /nope` produces stdout+stderr.
3. `2>&1` — merge stderr into stdout so the capture sees it.
4. `result=` — assign to the shell variable.

### The Story

Ansible's `command:` and `shell:` modules behave like this internally. Understanding the pattern helps debug `register: result` outputs — `result.stdout` and `result.stderr` are separated server-side but combined when you `2>&1` in shell.

### Expected Output

```
lines: 190
first 3:
ls: cannot access '/nope': No such file or directory
/lib64:
ld-linux-x86-64.so.2
1
```

### Switches Table

| Token | Meaning |
|---|---|
| `$( ... )` | Command substitution |
| `2>&1` (inside `$()`) | Merge stderr into the captured stdout |
| `"$var"` | Quote to preserve newlines |
| `grep -c` | Count matches |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `$(cmd 2>&1)` | Capture combined stream into variable |
|  | `"$var"` quoting | Preserve whitespace |
|  | `$(wc -l < file)` | Pure-number line count |

### 🧹 Cleanup

```bash
unset result
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Trailing newline missing | Bash strips trailing newlines from `$()` — usually harmless |
| `result` reads empty | `set -o pipefail` may have aborted; check exit codes |

> **STOP — paste your output before Task 19.**

---

## Task 19 — Cron job with full capture

**Practice directory this task:** `/lib64`
*We schedule a fake job that scans `/lib64` and prove that cron logs both streams.*

### 🔁 Warm-Up (rotation: `>>`, `&>>`, `tee -a`, `$(date -Is)`)

```bash
date -Is >> /tmp/redir-lab/cron-wu.log
ls /lib64 /nope &>> /tmp/redir-lab/cron-wu.log
tail -n 3 /tmp/redir-lab/cron-wu.log | tee -a /tmp/redir-lab/cron-wu.summary
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Author a cron line that captures both stdout AND stderr — the #1 fix for "my cron job silently failed."

### Main Command Block

```bash
cd /tmp/redir-lab
cat > task19-cron-line.txt <<'EOF'
# Captures stdout AND stderr into /var/log/lib64-audit.log
*/5 * * * *  /usr/local/bin/lib64-audit.sh >> /var/log/lib64-audit.log 2>&1

# Bash 4+ shorthand:
# */5 * * * *  /usr/local/bin/lib64-audit.sh &>> /var/log/lib64-audit.log
EOF
cat task19-cron-line.txt
echo "----"
crontab -l 2>/dev/null > task19-current.bak || true
wc -l task19-current.bak
```

### Human-Readable Breakdown

- `cat > FILE <<'EOF' ... EOF` — write a heredoc to file with no expansion.
- `*/5 * * * *` — cron schedule: every 5 minutes.
- `>> log 2>&1` — append both streams.
- `crontab -l 2>/dev/null` — list current user's cron, suppressing the "no crontab" stderr line.
- `|| true` — ensure exit 0 even if no crontab exists.

### Reading It Left to Right

`*/5 * * * * /usr/local/bin/lib64-audit.sh >> /var/log/lib64-audit.log 2>&1`
1. `*/5 * * * *` — minute / hour / day-of-month / month / day-of-week.
2. `/usr/local/bin/lib64-audit.sh` — full path to the script.
3. `>> /var/log/lib64-audit.log` — append stdout to log.
4. `2>&1` — append stderr to the same log.

### The Story

Cron emails stderr by default. Production crons override that with `>> log 2>&1`. RHCSA Task 13 explicitly tests this idiom.

### Expected Output

```
# Captures stdout AND stderr into /var/log/lib64-audit.log
*/5 * * * *  /usr/local/bin/lib64-audit.sh >> /var/log/lib64-audit.log 2>&1

# Bash 4+ shorthand:
# */5 * * * *  /usr/local/bin/lib64-audit.sh &>> /var/log/lib64-audit.log
----
0 task19-current.bak  (or your real crontab length)
```

### Switches Table

| Token | Meaning |
|---|---|
| `*/5 * * * *` | Every 5 minutes |
| `>> file 2>&1` | Append both streams |
| `&>>` | Bash combined append shorthand |
| `crontab -l` | List current user's crontab |
| `\|\| true` | Tolerate non-zero exit |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `>> log 2>&1` in cron | Capture all output |
|  | `&>>` in cron | Bash shorthand variant |
|  | `crontab -l` | List crontab |
|  | `\|\| true` | Soften exit codes |

### 🧹 Cleanup

```bash
rm -f /tmp/redir-lab/task19-cron-line.txt /tmp/redir-lab/task19-current.bak /tmp/redir-lab/cron-wu.log /tmp/redir-lab/cron-wu.summary
echo "exit was: $?"
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| Cron emails every error | Add `>> log 2>&1` and email stops |
| Log doesn't update | Path wrong, permissions wrong, or job exits before running |
| `crontab -l` printed an error | Add `2>/dev/null` to swallow the "no crontab" notice |

> **STOP — paste your output before Task 20.**

---

## Task 20 — Exam-style scenario: capture, display, and verify

**Practice directory this task:** `/lib64`
*Capstone: scan `/lib64` for recently modified objects, tee live progress, save the transcript, and audit it.*

### 🔁 Warm-Up (rotation: `sort -n`, `head -n`, `wc -l`, `$(uname -a)`)

```bash
find /lib64 -type f 2>/dev/null | wc -l
find /lib64 -type f -name 'libc*' 2>/dev/null | sort | head -n 3
echo "kernel: $(uname -a)"
echo "Warm-up done by $(whoami) at $(date -Is)"
echo "exit was: $?"
```

### Purpose

Run an exam-grade capture: combine `find`, `2>&1`, `tee`, and audit counters in one command line.

### Main Command Block

```bash
cd /tmp/redir-lab
sudo find /lib64 -mtime -30 2>&1 \
  | sudo tee /var/tmp/lib64-modfiles.log \
  | head -n 10

echo "----"
sudo wc -l /var/tmp/lib64-modfiles.log
sudo grep -c "Permission denied" /var/tmp/lib64-modfiles.log || true
sudo head -n 5 /var/tmp/lib64-modfiles.log
```

### Human-Readable Breakdown

- `find /lib64 -mtime -30` — files modified in the last 30 days.
- `2>&1` — merge stderr into stdout so tee can see both.
- `| sudo tee /var/tmp/lib64-modfiles.log` — display AND save in one go.
- `| head -n 10` — keep only the first 10 lines on screen during the run.
- `wc -l` — audit: total lines captured.
- `grep -c "Permission denied"` — audit: how many errors were captured (proves `2>&1` worked).

### Reading It Left to Right

`sudo find /lib64 -mtime -30 2>&1 | sudo tee /var/tmp/lib64-modfiles.log | head -n 10`
1. `sudo find /lib64 -mtime -30` — root-elevated search.
2. `2>&1` — merge stderr into stdout.
3. `| sudo tee /var/tmp/lib64-modfiles.log` — root writes the file AND passes the stream through.
4. `| head -n 10` — keep first 10 lines on screen.

### The Story

Every exam task that says "save the output and prove you saved it" follows this pattern: capture, tee, count. Treat this as your default capstone recipe.

### Expected Output

```
/lib64/libsystemd.so.0
/lib64/libsystemd.so.0.34.0
/lib64/libnss_files.so.2
...
----
17 /var/tmp/lib64-modfiles.log
0
/lib64/libsystemd.so.0
/lib64/libsystemd.so.0.34.0
/lib64/libnss_files.so.2
```

(Counts vary by RHEL minor version; the structure should match.)

### Switches Table

| Token | Meaning |
|---|---|
| `find PATH -mtime -N` | Modified in the last N days |
| `2>&1 \| sudo tee FILE` | Merge then save AND display via root |
| `wc -l FILE` | Line count audit |
| `grep -c PATTERN` | Error-count audit |
| `head -n N` | First N lines |

### 🧠 Concept Card

| ✅ | Concept | What it does |
|---|---|---|
|  | `find -mtime -N` | Recently modified files |
|  | `2>&1 \| sudo tee FILE` | Capstone capture-and-display |
|  | `wc -l` audit | Confirm capture size |
|  | `grep -c` audit | Confirm error count |

### 🧹 Cleanup

```bash
sudo rm -f /var/tmp/lib64-modfiles.log
rm -rf /tmp/redir-lab
echo "exit was: $?"
exit
```

### Troubleshoot Table

| Symptom | Fix |
|---|---|
| `tee` shows the listing but file is empty | Pipe targets only stdout; ensure `2>&1` is BEFORE the pipe |
| `Permission denied` writing `/var/tmp/...` | Use `sudo tee` |
| Counts vary wildly | Different RHEL minor version — structure matters, not exact numbers |

> **STOP — paste your output. Lab 04 complete.**

---

## 🔍 Redirection Decision Guide

```
Do you want to SAVE the output?
  ├── Stdout only?  → > file        (>> to append)
  ├── Stderr only?  → 2> file       (2>> to append)
  ├── Both?
  │     ├── Overwrite → > file 2>&1     or     &> file
  │     └── Append    → >> file 2>&1    or     &>> file
  ├── Discard?      → cmd > /dev/null   /   2> /dev/null   /   &> /dev/null
  └── See AND save? → cmd 2>&1 | tee file        (use tee -a to append)

Need to write as root?
  └── sudo cmd 2>&1 | sudo tee /root/file

Whole script captured?
  └── exec > /var/log/myscript.log 2>&1   (at the top)

Separate streams to different files?
  └── cmd > stdout.log 2> stderr.log

Capture into a variable?
  └── result=$(cmd 2>&1)
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Sandbox set up; `/lib64` line count saved
- [ ] 02 stdout AND stderr observed on screen
- [ ] 03 `>` captures only stdout
- [ ] 04 `2>` captures only stderr
- [ ] 05 Wrong order `2>&1 > file` recognized
- [ ] 06 Correct order `> file 2>&1` applied
- [ ] 07 `>`/`1>`/`2>&1` proven equivalent via md5sum
- [ ] 08 `&>` shorthand + `noclobber` + `>\|` force overwrite
- [ ] 09 `&>>` append both streams
- [ ] 10 `>> file 2>&1` portable append
- [ ] 11 Two streams to two files
- [ ] 12 Discard via `/dev/null`
- [ ] 13 `tee` with `2>&1` for see + save
- [ ] 14 `sudo tee` for root-owned destinations
- [ ] 15 Process substitution `> >(cmd)` and `2> >(tee … >&2)`
- [ ] 16 `exec > log 2>&1` for whole-script capture
- [ ] 17 `find / 2>/dev/null` permission-noise suppression
- [ ] 18 Capture into variable with `$(cmd 2>&1)`
- [ ] 19 Cron line with `>> log 2>&1`
- [ ] 20 Capstone: `find /lib64 -mtime -30 2>&1 | sudo tee … | head -n 10`

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `2>&1 > file` (wrong order) | Errors still on screen; file holds only stdout | Write `> file 2>&1` |
| `>` when you meant `>>` | Existing log destroyed | Use `>>` or `&>>` for appends; protect with `noclobber` |
| `sudo cmd > /root/file` | `Permission denied` | `sudo cmd \| sudo tee /root/file` or `sudo bash -c 'cmd > /root/file'` |
| Forgetting `2>&1` in cron | Errors disappear (cron emails them instead) | Always `>> log 2>&1` in cron lines |
| Confusing `&` for background | `2>&1` is **not** background; `&` alone at end is | Read `&` in `>&` as "the same FD" |
| `&>` on `/bin/sh` | Syntax error | Switch to `#!/bin/bash` or use the long form |
| `> file 2> file` (same file, separate redirects) | Race conditions | Use `> file 2>&1` instead |

---

## 📌 Exam Strategy

**RHCSA EX200**
- Every "save the output" task = `> file 2>&1` (or `&> file`). Default to capturing BOTH.
- `find / -name PATTERN > /var/tmp/file 2>&1` is the literal answer for Task 14.
- Use `sudo tee` whenever the destination is owned by root.

**RHCE EX294 (Ansible)**
- `command:` / `shell:` modules capture both streams; access them as `result.stdout` / `result.stderr`.
- `failed_when:` often checks `'error' in result.stderr` — stream awareness matters.

**CKA**
- `kubectl logs POD &> /tmp/pod.log` to grab everything for triage.
- `journalctl -u kubelet --since "10 min ago" &> /tmp/kubelet.log` is the canonical postmortem capture.

**RHCA**
- RH342: `exec > /var/log/incident-$(date +%s).log 2>&1` at the top of remediation scripts.
- RH358: service forensics — `systemctl status SERVICE 2>&1 \| tee /tmp/svc.log`.
- RH236: Gluster ops emit warnings to stderr — always combine with `2>&1`.

---

## 📚 Master Command Reference (used in this lab)

| Command | Common switches used here |
|---|---|
| `echo` | `1>&2` (write to stderr) |
| `cat` | `<<'EOF'` heredoc; `f1 f2 f3` concatenate; `> file` |
| `date` | `-Is`, `-I`, `+%s`, `+%F`, `+%F_%T` |
| `uname` | `-n`, `-r`, `-s`, `-m`, `-a` |
| `wc` | `-l`, `-w`, `-c`, `-m` |
| `grep` | `-v`, `-i`, `-c`, `-n`, `-E` |
| `find` | `-name`, `-type f`, `-user`, `-mtime -N`, `2>/dev/null` |
| `ls` | `-l`, `-a`, `-la`, `-Z` |
| `mkdir` | `-p` |
| `rm` | `-f`, `-r`, `-rf` |
| `head` | `-n N` |
| `tail` | `-n N`, `-f` |
| `tee` | `-a`, `f1 f2 f3`, with `sudo` |
| `sort` | `-n`, `-r`, `-u`, `-t:`, `-k3 -n` |
| `chmod` | `+x`, `644`, `755` |
| `test` | `-s`, `-f`, `-d`, `-x` |
| `journalctl` | `--no-pager`, `-u UNIT` |
| `less` | `/pattern`, `?pattern`, `n`, `N`, `g`, `G`, `q` |
| `tr` | `a-z A-Z`, `-d` |
| `sudo` | `-i`, `tee`, `bash -c 'cmd > file'` |
| `exec` | `> file 2>&1` |
| `crontab` | `-l`, `*/5 * * * *` |
| `md5sum` | hash a file |

### Redirection Operators

| Operator | Meaning |
|---|---|
| `>` / `>>` | stdout truncate / append |
| `2>` / `2>>` | stderr truncate / append |
| `&>` / `&>>` | both streams truncate / append (bash) |
| `>\|` | force overwrite under `noclobber` |
| `2>&1` | FD 2 follows FD 1 |
| `1>&2` | FD 1 follows FD 2 |
| `\|` | pipe |
| `2>&1 \|` | merge then pipe |
| `<` | stdin from file |
| `<<'EOF' ... EOF` | heredoc, no expansion |
| `>(cmd)` / `<(cmd)` | process substitution |

### Special Variables / Paths Used

| Item | Meaning |
|---|---|
| `$?` | Last exit status |
| `${PIPESTATUS[@]}` | All pipe stages' exit codes |
| `$(cmd)` | Command substitution |
| `/dev/null` | Discard sink |
| `/proc/self/fd` | Current process's open FDs |
| `/etc/passwd` | User account info |
| `/tmp/redir-lab` | This lab's sandbox |
| `/lib64` | Lab-wide practice directory |

### Shell Options Practiced

| Option | Effect |
|---|---|
| `set -o noclobber` / `set +o noclobber` | Refuse / allow `>` overwrite |
| `set -o pipefail` / `set +o pipefail` | Surface failing pipe stage |
| `set -euo pipefail` | Strict-mode trio |

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 01 — `>` and `>>` | Built the muscle for `>` and `>>` |
| Lab 02 — `2>` and `2>/dev/null` | Built the muscle for stderr-only redirection |
| Lab 03 — `\|`, `tee`, `wc -l` | Provided the pipe-and-tee combination used heavily here |
| Lab 05 — Directory navigation | You must know your path before you redirect into it |
| Lab 06 — `ls -lZ` | Verify log ownership and SELinux context after creation |
| Lab 14 — `find` | The canonical command that needs `2>&1` |
| Lab 18 — `dnf` install | `dnf install -y X 2>&1 \| tee /var/tmp/install.log` |

---

## 👤 Author

**Kelvin R. Tobias** — [kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
