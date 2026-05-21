# Lab 24: Stream Editing with `sed`

**Series:** File Operations & Shell Fundamentals · **Lab 24 of the Novice → RHCA path**  
**Certifications covered:** RHCSA EX200 (in-place config edits, scripted changes), RHCE EX294 (`lineinfile`/`replace` mental model), CKA (manifest transforms before `kubectl apply`), RHCA building blocks (RH342 quick fixes, RH358 mass service config edits, RH236 brick config rewrites)  
**Prerequisite:** Labs 05–23 (filesystem, `cat`, `less`, `grep`, `diff`)  
**Time Estimate:** 50–70 minutes  
**Difficulty arc:** Tasks 1–6 foundation · 7–13 practical · 14–18 advanced · 19–20 exam-realistic

---

## 🎯 Objective

By the end of this lab you will use `sed` (the **s**tream **ed**itor) to find and replace text, delete lines, insert content, transform fields, and edit files **in place** — all without ever opening a text editor. You'll know exactly when to reach for `sed` instead of `vi`, when to use `-i` (in place) safely with a backup, and how to think in terms of *addresses* and *commands* the way `sed` does.

---

## 🧠 Concept: `sed` Is a Tiny Programming Language

`sed` reads input one line at a time, applies a small program to each line, and writes the result. The program is built from **addresses** + **commands**.

```
sed '5,10s/foo/bar/' file
   │   │  │
   │   │  └── command  : s/foo/bar/  (substitute)
   │   └────── address : 5,10        (only lines 5 through 10)
   └────────── single-quoted script for safety from the shell
```

### The four most-used commands

| Command | Purpose | Example |
|---|---|---|
| `s/RE/REPL/` | **S**ubstitute first match | `s/foo/bar/` |
| `s/RE/REPL/g` | Substitute **all** matches on the line | `s/foo/bar/g` |
| `d` | **D**elete a line | `/^#/d` |
| `p` | **P**rint a line (usually with `-n`) | `-n '5p'` |
| `i\TEXT` | **I**nsert before | `5i\New header` |
| `a\TEXT` | **A**ppend after | `5a\See above` |
| `c\TEXT` | **C**hange whole line | `5c\Replaced` |

### Address forms

| Form | Means |
|---|---|
| `5` | Line 5 |
| `5,10` | Lines 5–10 |
| `$` | Last line |
| `5,$` | Line 5 to end |
| `/regex/` | Every line matching regex |
| `/start/,/end/` | From the line matching `start` to the line matching `end` |
| `0~3` | Every third line starting from 1 (`0~N` = stride) |
| `5,+3` | Line 5 and the next 3 (line 5–8) |
| `!` after address | **NOT** that address (e.g., `5d` = delete line 5; `5!d` = delete everything except line 5) |

### BRE vs ERE in `sed`

| Dialect | Flag |
|---|---|
| BRE | (default) — `\+`, `\?`, `\|`, `\(`, `\)`, `\{,\}` need backslashes |
| ERE | `-E` (or `-r` on old `sed`) — `+`, `?`, `\|`, `(`, `)`, `{,}` are first-class |

> **Senior-engineer default:** Use `sed -E` for new scripts. Always single-quote the script (`'…'`) so the shell doesn't expand `$`, `*`, `!`.

---

## 📚 Command Reference

| Flag | Long form | Purpose |
|---|---|---|
| `-e SCRIPT` | `--expression` | Add a script (use multiple times) |
| `-f FILE` | `--file` | Read script from a file |
| `-n` | `--quiet` | Don't auto-print every line — print only via `p` |
| `-E` / `-r` | `--regexp-extended` | Use ERE |
| `-i` | `--in-place` | Edit files in place |
| `-i.bak` | — | Edit in place, backup original to `FILE.bak` |
| `-s` | `--separate` | Treat multiple input files as separate streams |
| `--follow-symlinks` | — | Edit the target of a symlink (not the link) |
| `--posix` | — | Disable GNU extensions (portability) |
| `-z` | `--null-data` | Use null-byte as line separator |
| `-l N` | `--line-length=N` | Wrap `l` output |
| `-u` | `--unbuffered` | Flush after each line (live pipes) |

### Substitute (`s`) flags

| Flag | Meaning |
|---|---|
| `g` | Global — every match on the line |
| `N` | Replace only the **N**th match on the line |
| `gN` | Replace match N and every match after |
| `p` | Print the line if substitution succeeded |
| `w FILE` | Write changed lines to FILE |
| `i` / `I` | Case-insensitive |
| `e` | Execute the result as a shell command (rare; dangerous) |

---

## 🛣️ RHCA Pathway Sidebar

| Cert level | Why this lab matters |
|---|---|
| **Foundation** | Substitution + in-place edit are the everyday script tools |
| **RHCSA EX200** | "Change PermitRootLogin to no in /etc/ssh/sshd_config" — `sed -i` is the fastest |
| **RHCE EX294** | Ansible's `lineinfile`/`replace` modules are literally `sed` wrappers |
| **CKA** | Stamp a namespace/image into a manifest before `kubectl apply` |
| **RHCA — RH342 (Troubleshooting)** | Strip BOM, fix CRLF, redact secrets, normalize comments |
| **RHCA — RH358 (Services)** | Mass edits across `/etc/httpd/conf.d/*.conf`, etc. |
| **RHCA — RH236 (Storage)** | Brick config rewrites on N nodes — `for h in nodes; do ssh "$h" sed -i …; done` |

---

## 🔧 The 20 Tasks

> Each task ends with three short callouts: **Switches / address+command**, **Output decoded**, and **Troubleshoot**.

---

### Task 1 — Build the lab file

**Purpose:** Have a small but realistic config to edit.

```bash
mkdir -p ~/sed-lab && cd ~/sed-lab
cat > app.conf <<'EOF'
# Application configuration
host = web01
port = 80
ssl  = off
log  = /var/log/app.log

# Database
db_host = db01
db_port = 5432
db_name = appdb

# Feature flags
feature_x = enabled
feature_y = disabled
feature_z = disabled
EOF
cat -n app.conf
```

**Switches**

| Token | Meaning |
|---|---|
| `mkdir -p` | Create directory tree |
| `cat > FILE <<'EOF'` | Heredoc with literal `$` |
| `cat -n` | Verify with line numbers |

**Output decoded**

| Line | Content section |
|---|---|
| 1 | Comment header |
| 2–5 | App settings |
| 6 | Blank separator |
| 7–11 | Database section |
| 12–16 | Feature flags |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Heredoc missing trailing newline | Add an explicit `echo`, or use `printf` |

---

### Task 2 — First substitute: `s/old/new/`

**Purpose:** The canonical `sed` operation. Replace the **first** match per line and print the result to STDOUT.

```bash
cd ~/sed-lab
sed 's/web01/web02/' app.conf
```

**Expected output (excerpt):**

```
# Application configuration
host = web02
…
```

**Switches / addr+cmd**

| Token | Meaning |
|---|---|
| `s/web01/web02/` | Substitute `web01` → `web02` (first match per line only) |
| No address before `s` | Apply to every line |
| File is **not** modified | Output goes to STDOUT |

**Output decoded**

| Line | Meaning |
|---|---|
| Each line | Either modified (if matched) or printed unchanged |

**Why a sysadmin needs this:** Always preview a `sed` script by running it without `-i` first.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| No change | Match might be case-sensitive — try `s/web01/web02/i` |
| `sed: -e expression #1, char N: unknown option to ``s''` | You used `/` in the pattern/replacement — switch delimiters: `s#/old#/new#` |

---

### Task 3 — Replace **all** matches with the `g` flag

**Purpose:** Without `g`, `sed` replaces the **first** match per line only. With `g`, it replaces every occurrence.

```bash
echo "foo bar foo bar foo" | sed 's/foo/BAZ/'
echo "foo bar foo bar foo" | sed 's/foo/BAZ/g'
```

**Expected output:**

```
BAZ bar foo bar foo
BAZ bar BAZ bar BAZ
```

**Switches**

| Token | Meaning |
|---|---|
| `s/foo/BAZ/` | First-only |
| `s/foo/BAZ/g` | **G**lobal — every match on each line |
| `s/foo/BAZ/2` | The **second** match on each line |
| `s/foo/BAZ/2g` | The second and every later match |

**Output decoded**

| Line | Meaning |
|---|---|
| First | Only the leftmost `foo` replaced |
| Second | All three replaced |

**Why a sysadmin needs this:** Most real config edits want `g` — otherwise you'll leave duplicates behind on shared lines.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Some matches missed | Forgot the `g` flag |

---

### Task 4 — Use alternative delimiters

**Purpose:** Paths contain `/`. Picking a different delimiter avoids forests of backslashes.

```bash
sed 's|/var/log/app.log|/var/log/myapp/app.log|' ~/sed-lab/app.conf | grep log
sed 's#/var/log/app.log#/var/log/myapp/app.log#' ~/sed-lab/app.conf | grep log
```

**Expected output (both):**

```
log  = /var/log/myapp/app.log
```

**Switches**

| Token | Meaning |
|---|---|
| `s|A|B|` | Substitute using `\|` as delimiter |
| `s#A#B#` | Substitute using `#` |
| Any non-newline char | Can be the delimiter — pick whatever doesn't appear in your strings |

**Output decoded**

| Line | Meaning |
|---|---|
| Single line | The `log = …` setting with the new path |

**Why a sysadmin needs this:** Path-heavy substitutions become readable.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Backslashing everywhere | Switch delimiter |

---

### Task 5 — Address ranges: edit only specific lines

**Purpose:** Apply a command only to certain lines (numeric or pattern-based).

```bash
cd ~/sed-lab
sed -n '7,11p' app.conf            # print lines 7–11 only
sed '/^# Database/,/^$/d' app.conf # delete the Database section
```

**Expected output:**

```
# Database
db_host = db01
db_port = 5432
db_name = appdb

# Application configuration
host = web01
port = 80
ssl  = off
log  = /var/log/app.log


# Feature flags
feature_x = enabled
feature_y = disabled
feature_z = disabled
```

**Switches**

| Token | Meaning |
|---|---|
| `-n '7,11p'` | Suppress default output (`-n`), then **p**rint lines 7–11 |
| `'/^# Database/,/^$/d'` | Delete from the line matching `# Database` through the next empty line |

**Output decoded**

| Block | Meaning |
|---|---|
| First | Slice viewer |
| Second | Database section removed (just like deleting a paragraph in `vi`) |

**Why a sysadmin needs this on RHCA RH358:** Removing a configuration block (a `<VirtualHost>` for httpd, a `server { … }` for nginx) from a generated config.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Multi-paragraph deletion misses second match | Address-pair stops at the first `/^$/` after the start match — that's expected |

---

### Task 6 — Negate an address with `!`

**Purpose:** "Apply this command to every line **except** these."

```bash
sed '1d' ~/sed-lab/app.conf | head -3
sed '1!d' ~/sed-lab/app.conf
```

**Expected output:**

```
host = web01
port = 80
ssl  = off
# Application configuration
```

**Switches**

| Token | Meaning |
|---|---|
| `1d` | Delete line 1 |
| `1!d` | Delete every line **except** line 1 |

**Output decoded**

| Run | Meaning |
|---|---|
| `1d` | Drop the header; the rest survives |
| `1!d` | Keep the header; drop everything else |

**Why a sysadmin needs this:** Quick "show me only the first line" without `head`.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `! ` mishandled by shell | Put script in single quotes; in zsh, sometimes escape `!` |

---

### Task 7 — Edit in place with `-i` and `-i.bak`

**Purpose:** The most powerful (and dangerous) `sed` option. Modifies the file directly.

```bash
cd ~/sed-lab
cp app.conf app.conf.original
sed -i.bak 's/^ssl  = off/ssl  = on/' app.conf
ls
diff app.conf.bak app.conf
```

**Expected output (excerpt):**

```
app.conf  app.conf.bak  app.conf.original
4c4
< ssl  = off
---
> ssl  = on
```

**Switches**

| Token | Meaning |
|---|---|
| `-i` | Edit in place — **no backup** |
| `-i.bak` | Edit in place — keep backup at `app.conf.bak` |
| `-i'_$(date +%s)'` | Timestamp-named backup |

**Output decoded**

| Element | Meaning |
|---|---|
| `app.conf.bak` | The original file, preserved |
| `diff` shows one-line change | `ssl  = off` → `ssl  = on` |

**Why a sysadmin needs this on RHCSA EX200:** Nearly every "modify this config" task is fastest with `sed -i.bak`.

> ⚠️ **macOS / BSD `sed`** requires an explicit extension argument: `sed -i '' 's/A/B/' FILE`. GNU `sed` accepts `-i` with no argument. Always test in your environment first.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `sed: -i may not be used with stdin` | Use a filename, not a pipe |
| Symlink replaced with a regular file | Add `--follow-symlinks` |
| Backup not desired | Use plain `-i`; but always back up first via `cp` |

---

### Task 8 — Print line ranges with `-n` and `p`

**Purpose:** `sed -n '/A/,/B/p'` is the classic "print between markers" idiom.

```bash
sed -n '/^# Database/,/^# Feature flags/p' ~/sed-lab/app.conf
```

**Expected output:**

```
# Database
db_host = db01
db_port = 5432
db_name = appdb

# Feature flags
```

**Switches**

| Token | Meaning |
|---|---|
| `-n` | Suppress automatic line printing |
| `/A/,/B/p` | Print from line matching `A` through line matching `B` |

**Output decoded**

| Block | Meaning |
|---|---|
| Database section + trailing blank + start of feature flags | The address range is inclusive of both endpoints |

**Why a sysadmin needs this on RHCA RH342:** Extract a single VirtualHost / server block from a long config for analysis.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Range extends past where you wanted | `/start/,/end/` stops at the **first** match of `end` after `start` |
| Want to exclude the boundaries | `sed -n '/A/,/B/{ /A/d; /B/d; p }'` |

---

### Task 9 — Capture groups and back-references

**Purpose:** Reuse parts of the matched text in the replacement.

```bash
echo "host = web01" | sed -E 's/^([a-z_]+)[[:space:]]*=[[:space:]]*(.*)$/key=\1 val=\2/'
```

**Expected output:**

```
key=host val=web01
```

**Switches / regex**

| Token | Meaning |
|---|---|
| `-E` | ERE — no backslashes on `(`, `)`, `+` |
| `([a-z_]+)` | Group 1 — the key |
| `(.*)$` | Group 2 — the value, everything to end-of-line |
| `\1`, `\2` | Back-references to groups |
| `&` (not shown) | The **entire** match |

**Output decoded**

| Token | Meaning |
|---|---|
| `key=host` | Group 1 reused |
| `val=web01` | Group 2 reused |

**Why a sysadmin needs this on RHCE EX294:** Reformat config lines for Ansible variable extraction.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `\1` shows up literally | You forgot `-E` in BRE you'd write `\(…\)` for groups |
| Greedy match swallows too much | Use `[^=]+` instead of `.*` where appropriate |

---

### Task 10 — Insert, append, change whole lines

**Purpose:** Add a header, footer, or replace an entire line based on context.

```bash
cd ~/sed-lab
sed '1i\# Edited by automation' app.conf | head -3
sed '$a\# End of file' app.conf | tail -3
sed '/^ssl/c\ssl  = required' app.conf | grep ssl
```

**Expected output:**

```
# Edited by automation
# Application configuration
host = web01

feature_z = disabled
# End of file

ssl  = required
```

**Switches / cmds**

| Token | Meaning |
|---|---|
| `Ni\TEXT` | **I**nsert TEXT before line N |
| `Na\TEXT` | **A**ppend TEXT after line N |
| `/PAT/c\TEXT` | **C**hange any line matching PAT — replace whole line |
| `$` | Last line address |

**Output decoded**

| Run | Meaning |
|---|---|
| First | Banner inserted at top |
| Second | Trailer appended at bottom |
| Third | The `ssl` line replaced wholesale |

**Why a sysadmin needs this on RHCSA EX200:** Add a license banner, replace a deprecated directive.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `\` not interpreted | Use single quotes around the whole script |
| Want multi-line insert | Each new line ends with `\` (POSIX) or use `i\` followed by literal newline in GNU sed |

---

### Task 11 — Delete blank lines and comments (production reading)

**Purpose:** Build the everyday "show me the active config" one-liner.

```bash
sed -E '/^[[:space:]]*(#|$)/d' ~/sed-lab/app.conf
```

**Expected output:**

```
host = web01
port = 80
ssl  = on
log  = /var/log/app.log
db_host = db01
db_port = 5432
db_name = appdb
feature_x = enabled
feature_y = disabled
feature_z = disabled
```

**Switches**

| Token | Meaning |
|---|---|
| `-E` | ERE |
| `/^[[:space:]]*(#\|$)/d` | Delete lines whose first non-whitespace char is `#` or that are empty |

**Output decoded**

| Element | Meaning |
|---|---|
| Only active settings | Comments and blanks gone |

**Why a sysadmin needs this:** Quick "what did the admin really configure?" view.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Indented comments survive | Make sure your character class includes `[[:space:]]*` |

---

### Task 12 — Case-insensitive substitution and multiple commands `-e`

**Purpose:** Stack commands with `-e`, and add `I` to the `s` for case-insensitivity.

```bash
echo "Error: thing failed. ERROR: again." \
  | sed -E -e 's/error/!!ERROR!!/gI' -e 's/again/once more/'
```

**Expected output:**

```
!!ERROR!!: thing failed. !!ERROR!!: again.
```

Wait — only the first line was edited; the `again` substitution missed. Let's expand:

```bash
echo "Error: thing failed. ERROR: again." \
  | sed -E -e 's/error/!!ERROR!!/gI' -e 's/again/once more/g'
```

**Expected output:**

```
!!ERROR!!: thing failed. !!ERROR!!: once more.
```

**Switches**

| Token | Meaning |
|---|---|
| `-e SCRIPT` | Add a script — `sed` runs them in order |
| `gI` | Global + case-**i**nsensitive |
| `g` on second | Global flag on the second `again` substitution |

**Output decoded**

| Token | Meaning |
|---|---|
| `!!ERROR!!` × 2 | Both case variants caught |
| `once more.` | Replaced via second `-e` |

**Why a sysadmin needs this:** Multi-step config rewrites in a single invocation.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Forgot order matters | `-e` runs sequentially — pipeline them carefully |

---

### Task 13 — Substitute with environment-variable values

**Purpose:** Inject `$VAR` from the shell into the `sed` replacement. Requires **double quotes**.

```bash
NEW_HOST=web99
sed -E "s/^host = .*$/host = $NEW_HOST/" ~/sed-lab/app.conf | head -3
```

**Expected output:**

```
# Application configuration
host = web99
port = 80
```

**Switches**

| Token | Meaning |
|---|---|
| Double quotes | Allow `$NEW_HOST` to expand |
| `^host = .*$` | Match the whole line starting with `host = ` |

**Output decoded**

| Line | Meaning |
|---|---|
| `host = web99` | Variable expanded into the replacement |

**Why a sysadmin needs this on RHCE EX294:** Templating config from shell variables when Ansible isn't available.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `$` was literal | You used single quotes — switch to double |
| Variable contains `&` or `/` | Escape them in the replacement, or pre-substitute: `sed "s|^host = .*|host = ${NEW_HOST//&/\\&}|"` |

---

### Task 14 — Multi-line scripts with `-f`

**Purpose:** Long scripts are unreadable on the command line. Put them in a file.

```bash
cat > ~/sed-lab/clean.sed <<'EOF'
# Strip comments and blank lines
/^[[:space:]]*(#|$)/d

# Normalize key/value spacing: "key = value"
s/^[[:space:]]*([A-Za-z_][A-Za-z_0-9]*)[[:space:]]*=[[:space:]]*/\1 = /

# Quote string values that look like words
s/= ([A-Za-z_][A-Za-z_0-9-]*)$/= "\1"/
EOF

sed -E -f ~/sed-lab/clean.sed ~/sed-lab/app.conf
```

**Expected output (excerpt):**

```
host = "web01"
port = 80
ssl = "on"
log = /var/log/app.log
db_host = "db01"
db_port = 5432
db_name = "appdb"
feature_x = "enabled"
feature_y = "disabled"
feature_z = "disabled"
```

**Switches**

| Token | Meaning |
|---|---|
| `-f FILE` | Read script from FILE |
| `-E` | ERE applies to the whole script |
| `#` in the script | Comment (with GNU sed; not always POSIX) |

**Output decoded**

| Element | Meaning |
|---|---|
| Normalized spacing | Step 2 of the script |
| Quoted word values | Step 3 of the script |
| Numbers untouched | Their regex didn't match step 3 |

**Why a sysadmin needs this on RHCA RH358:** Reusable cleanup scripts kept in version control.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Script does nothing | Missing `-E` on a script written for ERE |

---

### Task 15 — Use `&` to keep the matched text

**Purpose:** `&` in the replacement = "the whole match." Useful for adding decorations.

```bash
echo "PermitRootLogin yes" | sed -E 's/yes/&  # <-- INSECURE/'
```

**Expected output:**

```
PermitRootLogin yes  # <-- INSECURE
```

**Switches**

| Token | Meaning |
|---|---|
| `&` | The entire matched text (read-only) |
| `\1`, `\2` | Group back-refs |

**Output decoded**

| Element | Meaning |
|---|---|
| `yes  # <-- INSECURE` | Original `yes` preserved by `&`, comment appended |

**Why a sysadmin needs this:** Annotate findings without rewriting them.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want a literal `&` | Escape: `\&` |

---

### Task 16 — Combine `sed` with live tailing

**Purpose:** Live-edit a stream — e.g., redact secrets in real-time log output.

```bash
sudo tail -F /var/log/secure | sed -uE 's/(Failed password for )([A-Za-z0-9_-]+)/\1<USER>/'
```

(Run `ssh nobody@localhost` from another shell to trigger.)

**Switches**

| Token | Meaning |
|---|---|
| `-u` | **U**nbuffered — flush per line (live output) |
| `-E` | ERE |
| `(\Failed password for )([A-Za-z0-9_-]+)` | Capture the prefix and the username |
| `\1<USER>` | Keep the prefix, redact the username |

**Output decoded**

| Element | Meaning |
|---|---|
| `Failed password for <USER>` | Redacted line |

**Why a sysadmin needs this on RHCA RH342:** Live demos / screen sharing — never leak usernames.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Live but bursty | Some `sed` versions don't honor `-u`; use `stdbuf -oL sed` instead |

---

### Task 17 — Extract IPv4 addresses cleanly

**Purpose:** Combine `sed -n` and `s` to print just the matched group of a regex.

```bash
sudo cat /var/log/secure | sed -nE 's/.* from (([0-9]{1,3}\.){3}[0-9]{1,3}).*/\1/p' | sort -u | head
```

**Switches**

| Token | Meaning |
|---|---|
| `-n` | Suppress auto-print |
| `s/RE/\1/p` | Substitute the whole line with group 1, and **print** only if substitution happened |
| `\| sort -u` | Unique addresses |

**Output decoded**

| Lines | Meaning |
|---|---|
| Each line | One unique source IP from sshd events |

**Why a sysadmin needs this on RHCA RH342:** The pure-`sed` cousin of the `grep -oE` extraction from Lab 22.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Multiple IPs on one line | `sed` doesn't loop — feed through another `sed` or `grep -oE` |

---

### Task 18 — Edit only matching lines (combined address + command)

**Purpose:** Make a change scoped to lines that contain a pattern. Surgical, safe.

```bash
cd ~/sed-lab
sed -E -i.bak '/^feature_/ s/disabled/enabled/' app.conf
grep ^feature_ app.conf
```

**Expected output:**

```
feature_x = enabled
feature_y = enabled
feature_z = enabled
```

**Switches**

| Token | Meaning |
|---|---|
| `/^feature_/` | Address — only lines that start with `feature_` |
| `s/disabled/enabled/` | Substitute on those lines only |
| `-i.bak` | In place with backup |

**Output decoded**

| Element | Meaning |
|---|---|
| Only `feature_*` lines changed | Other `disabled` strings (if any) untouched |

**Why a sysadmin needs this on RHCE EX294:** "Enable every disabled feature flag" without touching look-alike strings elsewhere.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Want to revert | `mv app.conf.bak app.conf` |

---

### Task 19 — A reusable redaction script for log sharing

**Purpose:** Strip emails, IPs, and usernames before sharing a log with a vendor / on a forum.

```bash
cat > ~/sed-lab/redact.sed <<'EOF'
# Mask IPv4
s/([0-9]{1,3}\.){3}[0-9]{1,3}/<IP>/g

# Mask email addresses
s/[A-Za-z0-9._%+-]+@[A-Za-z0-9.-]+\.[A-Za-z]{2,}/<EMAIL>/g

# Mask usernames after "Failed password for "
s/(Failed password for )[A-Za-z0-9_-]+/\1<USER>/g

# Mask UUIDs
s/[0-9a-fA-F]{8}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{4}-[0-9a-fA-F]{12}/<UUID>/g
EOF

printf 'Failed password for alice from 10.0.0.5\nMail: bob@example.com\nID: 12345678-90ab-cdef-1234-567890abcdef\n' \
  | sed -E -f ~/sed-lab/redact.sed
```

**Expected output:**

```
Failed password for <USER> from <IP>
Mail: <EMAIL>
ID: <UUID>
```

**Switches**

| Token | Meaning |
|---|---|
| `-E` | ERE |
| `-f redact.sed` | Pull the script from a file |
| Pattern set | Four common PII categories |

**Output decoded**

| Element | Meaning |
|---|---|
| `<IP>`, `<EMAIL>`, `<USER>`, `<UUID>` | Privacy-safe placeholders |

**Why a sysadmin needs this on RHCA RH342:** Always redact before sharing — make it a habit, codify with a script.

**Troubleshoot**

| Symptom | Fix |
|---|---|
| Over-redacts | Tighten the regex; e.g., scope `<USER>` only after `Failed password for ` |
| Misses some patterns | Add more `s/.../...../g` lines to the script |

---

### Task 20 — Exam-style scenario: harden sshd in place

**Task statement (RHCSA / RH358-style):** *"In `/etc/ssh/sshd_config`, set `PermitRootLogin no`, `PasswordAuthentication no`, and `X11Forwarding no`. If a setting already exists (commented or not), update it. Keep a timestamped backup. Verify with `sshd -t` and `diff`."*

```bash
# 1. Backup with timestamp
TS=$(date +%Y%m%d-%H%M%S)
sudo cp /etc/ssh/sshd_config "/etc/ssh/sshd_config.bak.$TS"

# 2. Apply the three hardening edits.
#    For each: if a "#?Key value" line exists, set it; otherwise append.
for setting in 'PermitRootLogin no' 'PasswordAuthentication no' 'X11Forwarding no'; do
  KEY=${setting%% *}
  if sudo grep -Eq "^[[:space:]]*#?[[:space:]]*${KEY}[[:space:]]+" /etc/ssh/sshd_config; then
    sudo sed -i -E "s|^[[:space:]]*#?[[:space:]]*${KEY}[[:space:]]+.*|${setting}|" /etc/ssh/sshd_config
  else
    echo "$setting" | sudo tee -a /etc/ssh/sshd_config >/dev/null
  fi
done

# 3. Verify syntax (does NOT restart the service)
sudo sshd -t && echo "syntax OK"

# 4. Show what changed
sudo diff "/etc/ssh/sshd_config.bak.$TS" /etc/ssh/sshd_config

# 5. (Optional, only if you understand the risk) reload sshd
# sudo systemctl reload sshd
```

**Expected output (excerpt):**

```
syntax OK
40c40
< #PermitRootLogin prohibit-password
---
> PermitRootLogin no
65c65
< PasswordAuthentication yes
---
> PasswordAuthentication no
88c88
< X11Forwarding yes
---
> X11Forwarding no
```

**Step-by-step rationale**

| Step | Why |
|---|---|
| `cp` with timestamp | Reversible — known-good restore point |
| `grep -Eq` test | Detect whether the key is already present (commented or not) |
| `sed -i -E "s\|…\|…\|"` | Use `\|` as delimiter — path-free and clear |
| `^[[:space:]]*#?[[:space:]]*KEY[[:space:]]+` | Match indented, possibly commented setting |
| `tee -a` (when missing) | Append only when the key wasn't found |
| `sshd -t` | Syntax-check **before** restarting — never reload a broken sshd |
| `diff` against backup | Prove exactly what changed |
| Commented reload | Don't reload sshd mid-lab — test syntax only |

**Output decoded**

| Line | Meaning |
|---|---|
| `syntax OK` | Configuration parses; safe to reload |
| `40c40 …` | Lines that changed in the unified-or-normal diff |

**Troubleshoot**

| Symptom | Fix |
|---|---|
| `sshd -t` errors | Open the file and review — restore from backup if needed: `sudo cp /etc/ssh/sshd_config.bak.$TS /etc/ssh/sshd_config` |
| Duplicate keys appended | The `grep -Eq` test missed an indent variant — improve the regex |
| `sed -i` failed | Permission or missing `-E` |

---

## 🔍 sed Decision Guide

```
First substitute on a line              → sed 's/A/B/'
Every substitute on a line              → sed 's/A/B/g'
Case-insensitive                        → sed 's/A/B/gI'
Only on matching lines                  → sed '/pattern/ s/A/B/'
Only on line N                          → sed 'N s/A/B/'
Address range                           → sed '/start/,/end/ s/A/B/g'
Delete lines                            → sed '/regex/d'   or  sed 'N,Md'
Print a slice                           → sed -n '/A/,/B/p'   (or -n 'N,Mp')
Edit file in place                      → sed -i.bak 's/A/B/g' FILE
Multi-step                              → sed -e 'cmd1' -e 'cmd2'  OR -f script.sed
Use shell variable                      → double quotes, e.g.  sed "s/A/$VAR/"
Live (stream)                           → sed -u 's/...'    or  stdbuf -oL sed
Extract a capture group                 → sed -nE 's/.*X(GROUP).*/\1/p'
```

---

## ✅ Lab Checklist (20 Tasks)

- [ ] 01 Build the lab file with a heredoc
- [ ] 02 `s/old/new/` first-match substitute
- [ ] 03 `g` flag — global substitute
- [ ] 04 Alternative delimiters (`|`, `#`)
- [ ] 05 Numeric and pattern address ranges
- [ ] 06 `!` for "everything except this address"
- [ ] 07 `-i` and `-i.bak` in-place edits
- [ ] 08 Print ranges with `-n '/A/,/B/p'`
- [ ] 09 Capture groups `\1`, `\2`
- [ ] 10 Insert, append, change-line (`i\`, `a\`, `c\`)
- [ ] 11 Delete blanks + comments idiom
- [ ] 12 Case-insensitive (`I`) and multiple `-e`
- [ ] 13 Inject shell variables (double quotes)
- [ ] 14 Multi-line scripts with `-f`
- [ ] 15 `&` in the replacement
- [ ] 16 Live pipeline with `sed -u`
- [ ] 17 Extract IPs with `s/.../\1/p`
- [ ] 18 Scoped edits with `/match/ s/...`
- [ ] 19 Reusable redaction script
- [ ] 20 Exam combo: harden sshd_config with backup + verify

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `sed -i 's/A/B/' FILE` on macOS | Errors or weird in-place file | `sed -i '' 's/A/B/' FILE` (BSD) — or use GNU sed (`gsed`) |
| Using `/` in pattern/replacement when path has `/` | Endless backslashes | Switch delimiter: `s\|…\|…\|` |
| Single-quoting when you need `$VAR` | Literal `$VAR` substituted | Use double quotes |
| Forgetting `g` | Only first match per line replaced | Add `g` |
| Editing a symlink with `-i` | Symlink replaced by a regular file | Add `--follow-symlinks` |
| `!` swallowed by zsh history | `event not found` | Escape: `\!` or use single quotes |
| `\1` in BRE without `\(\)` | Literal `\1` | Either escape groups in BRE or use `-E` |
| Mass edit broke production | No undo | Always `-i.bak` and diff before applying widely |
| Greedy `.*` matches too much | Wrong substitution | Use `[^X]*` instead of `.*` for non-greedy effect |
| Mixing `-r` (BSD) and `-E` (POSIX) | Portability surprise | Use `-E`; for portability also check with `--posix` |

---

## 📌 Exam Strategy

**RHCSA EX200**
- `sudo cp FILE FILE.bak && sudo sed -i 's/X/Y/' FILE && sudo diff FILE.bak FILE` is the universal change pattern.
- Always run `sudo SERVICE -t` (e.g., `sshd -t`, `nginx -t`, `httpd -t`) before reloading.

**RHCE EX294 (Ansible)**
- `lineinfile` (single line) and `replace` (multi-line regex) wrap exactly the semantics of `sed`. Knowing `sed` makes your Ansible debugging faster.

**CKA**
- `sed -i 's|nginx:1.25|nginx:1.26|' deployment.yaml && kubectl diff -f deployment.yaml`.
- Avoid `sed`-editing live cluster state — prefer `kubectl edit` or GitOps.

**RHCA**
- RH342: Master scripted redactions and "delete-the-block" range deletes.
- RH358: Cross-host edits via `ansible -m shell -a 'sed -i …'` (or use `lineinfile`).
- RH236: Brick option tweaks across N nodes — `sed -i` inside a `for host in ...` loop.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 19 — `cat` | Use `cat -n` to find the line numbers `sed` should target |
| Lab 22 — `grep` | Identify which lines exist before scripting their edit with `sed` |
| Lab 23 — `diff` | Confirm exactly what `sed -i.bak` changed |
| Lab 25 — `awk` | When you need per-field logic, awk is the next step up |
| Lab 26 — `vi` | Interactive editing; mental model identical to `sed`'s `s///` |

---

## 👤 Author

**Kelvin R. Tobias**  
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
