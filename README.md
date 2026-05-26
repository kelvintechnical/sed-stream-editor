# Lab: Stream Editing — `sed`

**Series:** linux-ops-mastery — RHCSA Text File Management
**Subjects covered:** `sed` as a non-interactive editor, the substitute command `s/PAT/REPL/FLAGS`, BRE vs `-E` ERE regex, address forms (`N`, `$`, `/PAT/`, `N,M`, `N~step`), in-place editing (`-i`, `-i.bak`), the `d` (delete), `p` (print), `i` (insert), `a` (append), `c` (change), `y` (translate), and `q` (quit) commands, multiple expressions (`-e`), script files (`-f`), capture groups (`\1..\9`), branching (`b`, `t`), the hold buffer (`h`, `H`, `g`, `G`, `x`), `sed -n` for selective printing, and how `sed` differs from `awk` and `grep`
**Career arcs covered:** RHCSA (rewriting config files in scripts), RHCE (template-free Ansible bootstrap with `lineinfile`/`replace` modeled on sed), SRE (log redaction, mass edits), DevOps (CI build-time substitutions), AI/MLOps (massage CSV/JSON before ingestion)
**Prerequisite:** Lab 22 (grep + regex)
**Time Estimate:** 35 to 45 minutes
**Difficulty arc:** Task 1 foundation (`s///`) · 2 addresses + `-n p` · 3 in-place edits with `.bak` · 4 multi-expression scripts · 5 capture groups + delete · 6 RHCSA exam-realistic capstone

---

## Objective

Edit files non-interactively. By the end of this lab you can replace text, delete lines, insert headers, comment out a config directive, and do it in-place with a safety-net backup.

The capstone is an exam-realistic prompt: *"Without opening an editor, edit `/etc/ssh/sshd_config` to set `PermitRootLogin no`, comment out any line containing `Banner`, insert a header comment at line 1, and keep a `.bak` of the original."*

> **Lab safety note:** All edits use `sed -i.bak` so the original file survives as `.bak`. The capstone runs against a local copy of `sshd_config` to avoid breaking SSH.

---

## Concept: `sed` Reads a Stream, Applies Commands per Line, Writes a Stream

`sed` works one line at a time. For each input line it runs the script (one or more commands), prints the (possibly modified) line, and moves on. It is **non-interactive** — perfect for shell scripts, CI, and Ansible.

```
   INPUT stream ─► sed -e CMD1 -e CMD2 -e ... ─► OUTPUT stream
                       │
                       ├─ s/PAT/REPL/FLAGS   substitute
                       ├─ d                  delete the current line
                       ├─ p                  print (use with -n)
                       ├─ i\ TEXT            insert before
                       ├─ a\ TEXT            append after
                       ├─ c\ TEXT            change (replace whole line)
                       ├─ q [N]              quit (with optional exit code)
                       └─ y/SRC/DST/         translate single characters
```

> **Why this matters:** Every Ansible module that touches a file (`lineinfile`, `replace`, `blockinfile`) is conceptually a managed `sed`. CI pipelines do `sed -i "s/VERSION_PLACEHOLDER/$VERSION/g"` on config templates. Knowing `sed` makes those automations transparent instead of magical.

---

## 📜 Why `sed` Exists — The Story

`sed` was written in 1973 by Lee McMahon at Bell Labs as a **stream editor** — an `ed`-syntax editor that didn't require loading the whole file into memory and didn't need a human typing commands. Where `ed` was interactive and line-addressed, `sed` was script-driven and stream-oriented. That made `sed` the first "filter" tool that could rewrite files inside a pipeline.

The substitute command (`s/PAT/REPL/`) is borrowed directly from `ed`. The `g` flag (global, replace all per line) and `p` flag (print) live on in every text editor today (`:%s/foo/bar/g` in vim is `sed`'s `s/foo/bar/g` over the whole buffer).

GNU added `-i` in-place editing (with optional `.bak` backup), `-E` for ERE syntax, and `-z` for NUL-delimited records.

> **The point of the story:** `sed -i.bak 's/old/new/g' FILE` is the canonical "automated config edit" command and has been for fifty years. Master it once; reuse for the rest of your career.

---

## 👪 The Sed Family — Who Lives There

| Tool | Notes |
|---|---|
| `sed` | The stream editor itself |
| `sed -E` | ERE regex — modern default |
| `sed -n` | Suppress automatic print; pair with `p` |
| `sed -i` | In-place edit (dangerous w/o backup) |
| `sed -i.bak` | In-place edit with `.bak` backup |
| `sed -f script.sed` | Read commands from a script file |
| `sed -e A -e B` | Multiple expressions |
| `awk` | Field-oriented sibling — Lab 25 |
| `tr` | Character translation only — simpler than `sed y///` |
| `ed` | Interactive line editor (`sed`'s ancestor) |
| `ex` / `vim -c` | Editor-mode scripting |
| `perl -pi -e` | More powerful in-place editing |
| `ansible lineinfile / replace / blockinfile` | Managed equivalents |

### `sed` command cheat sheet

| Command | Means |
|---|---|
| `s/PAT/REPL/` | Replace first match on the line |
| `s/PAT/REPL/g` | Replace ALL matches on the line |
| `s/PAT/REPL/2` | Replace 2nd match only |
| `s/PAT/REPL/i` | Case-insensitive |
| `s/PAT/REPL/I` | Same on GNU sed |
| `s/PAT/REPL/p` | Print after replacement (combine with `-n`) |
| `s|PAT|REPL|` | Same — alternate delimiter `|` (useful when REPL has `/`) |
| `d` | Delete the current line |
| `p` | Print the current line |
| `q` | Quit immediately |
| `q N` | Quit with exit code N |
| `i\TEXT` | Insert TEXT before the current line |
| `a\TEXT` | Append TEXT after |
| `c\TEXT` | Change the current line to TEXT |
| `y/abc/xyz/` | Translate a→x, b→y, c→z |
| `=` | Print line number |
| `n` | Read next line into pattern space |
| `N` | Append next line to pattern space (multi-line) |

### Address forms

| Address | Selects |
|---|---|
| `N` | Line N |
| `$` | Last line |
| `N,M` | Lines N through M |
| `N,+K` | Lines N through N+K |
| `N~step` | Every "step"th line starting at N (GNU) |
| `/REGEX/` | Lines matching REGEX |
| `/A/,/B/` | From first match of A to first match of B |
| `!` | Negate (e.g., `/PAT/!d` = delete non-matching) |

### Useful flags

| Flag | Meaning |
|---|---|
| `-n` | No auto-print |
| `-e` | One expression (use multiple) |
| `-f FILE` | Read script from FILE |
| `-E` | ERE regex |
| `-i` | In-place |
| `-i.bak` | In-place with backup |
| `-r` | GNU alias for `-E` |
| `-z` | NUL-separated records |

> **The point of the family tree:** Memorize `s///`, `d`, `p`, and the `-n` flag — 90% of real-world `sed` usage stops there.

---

## 🔬 The Anatomy of `sed -E -i.bak 's|^#?PermitRootLogin .*$|PermitRootLogin no|' sshd_config` — In One Diagram

```
sed -E -i.bak  's|^#?PermitRootLogin .*$|PermitRootLogin no|'  sshd_config
    │    │      │   │ │└──────┬───────┘                 │      │
    │    │      │   │ │       │                         │      └─ target file
    │    │      │   │ │       └─ rest of the line (.* is greedy)
    │    │      │   │ └─ optional '#' (commented-out directive)
    │    │      │   └─ start of line anchor
    │    │      └─ substitute command, delimited by `|` because REPL has `/` characters
    │    └─ in-place edit, save .bak copy of original
    └─ extended regex (ERE) so `?` works without backslash
```

> **Reading rule:** Anything inside the `s|...|...|` is regex (PAT) on the left and the replacement string (REPL) on the right. Delimiters can be any character (`/`, `|`, `#`) so pick one that doesn't appear in PAT or REPL.

---

## 📚 sed Reference Table

| Task | Command | Notes |
|---|---|---|
| Replace first match per line | `sed 's/old/new/' FILE` | |
| Replace all matches per line | `sed 's/old/new/g' FILE` | |
| Case-insensitive replace | `sed 's/OLD/new/gi' FILE` | |
| Replace only on matching lines | `sed '/^Listen/ s/80/8080/' FILE` | |
| Delete lines matching regex | `sed '/^#/d' FILE` | Drop comments |
| Delete blank lines | `sed '/^$/d' FILE` | |
| Delete comments and blanks | `sed -E '/^(#|$)/d' FILE` | |
| Delete lines N through M | `sed '5,10d' FILE` | |
| Keep only matching lines | `sed -n '/PAT/p' FILE` | grep-ish |
| Print lines N through M | `sed -n '5,10p' FILE` | |
| Add line number | `sed = FILE \| paste -d: - -` | crude |
| Insert before line 1 | `sed '1i\# managed file' FILE` | |
| Append after last line | `sed '$a\# end of file' FILE` | |
| Change line | `sed '3c\new line 3' FILE` | |
| In-place with backup | `sed -i.bak 's/old/new/g' FILE` | Always use a backup suffix |
| Multiple expressions | `sed -e 's/A/B/' -e '/^#/d' FILE` | |
| From a script file | `sed -f edits.sed FILE` | |
| Capture group | `sed -E 's/([0-9]+)/(\1)/g' FILE` | |
| Translate chars | `sed 'y/abc/ABC/' FILE` | |
| Quit early | `sed '100q' FILE` | Faster than `head -n 100` for huge files? same idea |
| Comment a directive | `sed -i.bak -E 's/^(PermitRootLogin .*)/#\1/' sshd_config` | Idempotent: only matches uncommented |

> **Rule one of sed:** Always test without `-i` first. Then add `-i.bak`. Then remove the `.bak` once you trust the change.

---

## 🎯 Career Pathway Sidebar

| Level | Why this lab matters |
|---|---|
| **RHCSA candidate** | Bootstrap scripts that fix configs use `sed -i.bak` everywhere. |
| **RHCE candidate** | The `ansible.builtin.replace` and `lineinfile` modules are managed `sed`. |
| **SRE / Platform** | Mass redaction of secrets in logs: `sed -E 's/(token=)[^ &]+/\1REDACTED/g'`. |
| **DevOps** | Build-time substitution of environment values into config files. |
| **AI / MLOps** | Pre-processing CSVs/JSONs before they hit your pipeline: `sed -E 's/,,/,0,/g'`. |

---

## 🔧 The 6 Tasks

> Six exam-realistic phases that build **replace → address → in-place → multi-expr → backref → capstone** muscle memory.

---

### Task 1 — Basic substitute

```bash
mkdir -p ~/sedlab && cd ~/sedlab
cat > pets.txt <<'EOF'
cat - mittens
dog - rex
cat - whiskers
fish - bubbles
EOF

sed 's/cat/CAT/' pets.txt           # only the FIRST cat on each line
sed 's/cat/CAT/g' pets.txt          # ALL cats on each line
sed 's/cat/CAT/gi' pets.txt         # case-insensitive
```

---

### Task 2 — Addresses + `-n` selective print

```bash
sed -n '2p' pets.txt                # print only line 2
sed -n '1,3p' pets.txt              # print lines 1-3
sed -n '/dog/p' pets.txt            # print only lines matching /dog/
sed '/^cat/d' pets.txt              # delete lines starting with cat
sed -n '$=' /etc/passwd             # print the line count (last line number)
```

---

### Task 3 — In-place with backup

```bash
cp pets.txt pets.txt.before
sed -i.bak 's/dog/DOG/g' pets.txt

# Compare before/after
diff -u pets.txt.bak pets.txt       # .bak holds the pre-edit content
diff -u pets.txt.before pets.txt    # identical to above
```

---

### Task 4 — Multiple expressions in one pass

```bash
cat > srv.conf <<'EOF'
# default config
Listen 80
ServerName localhost
#DocumentRoot /var/www/html
EOF

sed -i.bak -E \
  -e 's/^Listen .*/Listen 8080/' \
  -e 's|^#?DocumentRoot .*|DocumentRoot /srv/web|' \
  -e '/^#/d' \
  srv.conf

cat srv.conf
```

---

### Task 5 — Capture groups + delete + insert

```bash
# Wrap any IPv4 in parentheses
echo "Connection from 10.0.0.5 via gateway 192.168.1.1" \
  | sed -E 's/([0-9]{1,3}(\.[0-9]{1,3}){3})/(\1)/g'

# Comment out lines that start with PermitRootLogin (without duplicating # on already-commented)
printf 'PermitRootLogin yes\n#PermitRootLogin no\nOther line\n' \
  | sed -E 's/^(PermitRootLogin .*)$/#\1/'

# Insert a header into pets.txt
sed -i.bak '1i\# managed by sedlab — do not edit by hand' pets.txt
head -n 3 pets.txt
```

---

### Task 6 — Capstone: harden a copy of `sshd_config` non-interactively

**Task statement:** Copy `/etc/ssh/sshd_config` to `~/sedlab/sshd_config`, then with a single `sed -i.bak` invocation: (a) set `PermitRootLogin no` (idempotent — works whether the original was commented or not), (b) comment out any uncommented `Banner` line, (c) insert a top-of-file header. Verify the result and keep the `.bak` for rollback.

```bash
sudo -i
mkdir -p /root/sedlab
cp -a /etc/ssh/sshd_config /root/sedlab/sshd_config
cd /root/sedlab

# Single sed call, three edits in order
sed -i.bak -E \
  -e '1i\# Hardened by sedlab capstone' \
  -e 's|^#?PermitRootLogin .*$|PermitRootLogin no|' \
  -e 's|^(Banner .*)$|#\1|' \
  sshd_config

# Verify
head -n 5 sshd_config
grep -nE '^PermitRootLogin' sshd_config
grep -nE '^#?Banner' sshd_config

# Compare against original
diff -u sshd_config.bak sshd_config | head -n 40
```

**Expected verification output (sample):**

```text
# Hardened by sedlab capstone
#       $OpenBSD: sshd_config,v ...
...
38:PermitRootLogin no
86:#Banner /etc/issue.net
--- sshd_config.bak  ...
+++ sshd_config      ...
@@ -1,3 +1,4 @@
+# Hardened by sedlab capstone
 #       $OpenBSD: sshd_config,v ...
@@ -38,1 +38,1 @@
-#PermitRootLogin prohibit-password
+PermitRootLogin no
@@ -86,1 +86,1 @@
-Banner /etc/issue.net
+#Banner /etc/issue.net
```

**Cleanup**

```bash
rm -rf /root/sedlab
exit
```

---

## 🔍 Sed Decision Guide

```
What edit do you need?
  ├── "Replace text"                          → sed 's/OLD/NEW/g'
  ├── "Replace only on certain lines"         → sed '/PAT/ s/A/B/g'
  ├── "Delete lines"                          → sed '/PAT/d'
  ├── "Print only matching lines (grep-like)" → sed -n '/PAT/p'
  ├── "Insert line"                           → sed 'Ni\TEXT'   (or '$a\TEXT' for append)
  ├── "Edit in place with safety"             → sed -i.bak 'CMDS' FILE
  ├── "Multiple edits one pass"               → sed -e CMD1 -e CMD2 -e CMD3
  ├── "Edits from a file"                     → sed -f script.sed FILE
  ├── "Capture / backreference"               → sed -E 's/(GRP)/\1.../'
  └── "Idempotent comment-toggle"             → sed -E 's/^(KEY .*)/#\1/'   (won't double-#)
```

---

## ✅ Lab Checklist (6 Tasks)

- [ ] 01 `s///` and `s///g`
- [ ] 02 Addresses (`N`, `N,M`, `/PAT/`) + `-n p` + `d`
- [ ] 03 In-place edits with `-i.bak`
- [ ] 04 Multiple expressions via `-e`
- [ ] 05 Capture groups and insert
- [ ] 06 Capstone — harden a copy of `sshd_config` in one pass

---

## ⚠️ Common Pitfalls

| Mistake | Symptom | Fix |
|---|---|---|
| `sed -i 's/A/B/'` with no `.bak` | No safety net | Always `-i.bak` first |
| `sed 's/foo/bar/'` only changed first match | Forgot `g` flag | `s/foo/bar/g` |
| `sed 's/foo bar/baz/'` ate the space | Pattern wanted spaces literal | Fine, but quote whole expr |
| REPL contains `/` and sed errored | Delimiter clash | Use `|` or `#` as delimiter |
| Tried `(grp)` in default mode | BRE needs `\(grp\)` | Use `-E` |
| Tried `?` and got literal `?` | Same — BRE | `-E` |
| Lost original file | `-i` without `.bak` | Restore from VCS or `cp -a` next time |
| `sed` did not handle multi-line pattern | Default is line-by-line | Use `N` or `awk` / `perl -0` |
| `sed -i` failed on read-only file | Permission | `sudo sed -i.bak ...` |
| Hostname inside replacement got expanded | Used double quotes | Switch to single quotes to stop shell substitution |

---

## 🎯 Career & Interview Strategy

**RHCSA candidate**
- Bootstrap script template: `sed -i.bak -E 's|^#?Directive .*|Directive value|' /etc/file.conf`. Drill this until reflexive.

**RHCE candidate**
- Know which Ansible module replaces which sed pattern: `lineinfile` = match a regex and set a single line; `replace` = `sed s/PAT/REPL/g`; `blockinfile` = managed block between markers.

**SRE / Platform interview**
- "How would you redact API tokens from a 10 GB log?" → `sed -E 's/(token=)[A-Za-z0-9_-]+/\1REDACTED/g'` streamed via `pv` and `tee`.

**DevOps**
- CI templating without a templating engine: `sed -i "s/PLACEHOLDER_VERSION/${BUILD_VERSION}/g" deploy.yaml`.

**AI / MLOps**
- Massage messy CSVs: `sed -E 's/,(,)/,0\1/g'` to fill empty numeric columns.

---

## 🔗 Related Labs

| Lab | Connection |
|---|---|
| Lab 22 — grep | Same regex flavor |
| Lab 23 — diff | Verify the edit |
| Lab 25 — awk | Field-aware sibling |
| Lab 26 — vi | Same `:%s/old/new/g` syntax |
| Lab 27 — vipw/vigr | Safer for /etc/passwd & /etc/group |

---

## 👤 Author

**Kelvin R. Tobias**
[kelvinintech.com](https://kelvinintech.com) · [GitHub](https://github.com/kelvintechnical) · [LinkedIn](https://www.linkedin.com/in/kelvin-r-tobias-211949219)
