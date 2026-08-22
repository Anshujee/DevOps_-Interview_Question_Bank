# Python (for DevOps) — Real Interview Questions & Answers

> This file is a personal log of actual Python questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.
> Since Python is a supporting skill for DevOps roles (scripting, automation, tooling) rather than the primary focus, each answer here also breaks down the Python language mechanics in plain English — not just the DevOps use case — so it doubles as a learning reference.

---

## Table of Contents

- [Interview #1 — Coforge | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. A folder has 10 files, 3 of them have IP addresses as filenames — how do you read only those files? (+ Follow-up: how do you specify the folder location in Python?)](#q1-a-folder-has-10-files-3-of-them-have-ip-addresses-as-filenames--how-do-you-read-only-those-files--follow-up-how-do-you-specify-the-folder-location-in-python)

---

## Interview #1

**Company:** Coforge
**Date:** 22-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Senior DevOps Manager

---

### Questions Asked

#### Q1. A folder has 10 files, 3 of them have IP addresses as filenames — how do you read only those files? (+ Follow-up: how do you specify the folder location in Python?)

**Answer:**

This is a classic "can you actually script something" question — very common for DevOps roles, because this exact pattern shows up constantly in real automation: scanning a directory of per-host backup files, per-device config dumps, or per-server log files, where the filename itself is the identifying piece of data. There are three separate sub-problems hiding inside this one question: **(1) listing the files in a folder, (2) deciding which filenames are actually valid IP addresses, and (3) reading the content of just those files.** I'll go through each, and then the follow-up.

---

**Sub-problem 1 — Listing the files in a folder**

Python gives you a few ways to do this. I'll use `pathlib`, because it's the modern, recommended approach — it works the same way on Windows, Linux, and macOS, and it gives you file objects you can call methods on directly, instead of just plain strings.

```python
import pathlib

folder_path = pathlib.Path("/var/log/network-configs")

for file_path in folder_path.iterdir():
    print(file_path.name)
```

**Breaking this down for anyone newer to Python:**
- `import pathlib` — brings in Python's built-in module for working with file paths.
- `pathlib.Path("...")` — wraps a string into a `Path` object, which understands "this is a filesystem location" rather than treating it as plain text.
- `folder_path.iterdir()` — returns every item (file or subfolder) directly inside that folder, one at a time. It's a *generator*, meaning it doesn't load the entire list into memory at once — it hands you one item, you use it, then it hands you the next.
- `file_path.name` — just the filename itself, e.g. `"192.168.1.10"`, without the rest of the path.

(The older, still very common alternative is `os.listdir("/var/log/network-configs")`, which returns plain filename strings instead of `Path` objects — functionally similar, but you lose the convenient methods `Path` objects give you, like `.is_file()`, `.stem`, `.read_text()`, which I'll use below.)

---

**Sub-problem 2 — Deciding which filenames are actually valid IP addresses**

This is the part where I'd actively push back on the "obvious" first instinct, which is usually a quick regex like `\d+\.\d+\.\d+\.\d+`. That regex is wrong — it would happily match `999.999.999.999` or `300.1.1.1`, neither of which is a real IP address, because a naive digit-dot pattern doesn't understand that each octet has to be between 0 and 255. Writing a fully correct regex for that is possible but ugly and easy to get subtly wrong.

The better approach is Python's built-in **`ipaddress`** module, which does full, correct validation for you:

```python
import ipaddress

def is_valid_ip(text: str) -> bool:
    try:
        ipaddress.ip_address(text)
        return True
    except ValueError:
        return False
```

**Breaking this down:**
- `def is_valid_ip(text: str) -> bool:` — defines a function named `is_valid_ip` that takes one argument (`text`, expected to be a string) and returns `True` or `False`. The `-> bool` part is a *type hint* — it documents the expected return type but Python doesn't strictly enforce it at runtime.
- `ipaddress.ip_address(text)` — tries to parse `text` as either a valid IPv4 or IPv6 address. If it succeeds, it returns an IP address object. If `text` isn't a real, valid IP, it raises a `ValueError`.
- `try` / `except ValueError` — this is Python's error-handling structure. "Try to run this code; if it raises a `ValueError`, don't crash the program, just run the `except` block instead." Here, that means: if parsing fails, just return `False`.

This correctly rejects `999.999.999.999` and correctly accepts real addresses like `192.168.1.10` or even IPv6 addresses — something the naive regex wouldn't handle at all.

---

**A real gotcha — IP addresses and filename extensions don't mix cleanly with `.stem`**

Here's a subtlety that trips people up, and I always call it out because it's exactly the kind of thing that causes a silent, hard-to-notice bug in production. If you want to strip a file extension before checking the filename, the natural instinct is to use `Path.stem`:

```python
pathlib.Path("192.168.1.10").stem     # → '192.168.1'   ← WRONG, truncated!
pathlib.Path("192.168.1.10").suffix   # → '.10'
```

`pathlib` treats everything after the **last dot** as the file extension — but an IP address is *made of* dots. If the filename is the raw IP with no real extension (like `192.168.1.10`), `.stem` chops off the last octet, and `ipaddress.ip_address("192.168.1")` then fails validation (it's not a real IPv4 address — only 3 octets), silently skipping a file that should have matched.

The safe approach is to check the **full filename first**, and only fall back to the stem if that fails (which correctly handles filenames like `192.168.1.10.log`, where `.log` really is a separate extension):

```python
def extract_ip_from_filename(filename: str) -> str | None:
    for candidate in (filename, pathlib.Path(filename).stem):
        if is_valid_ip(candidate):
            return candidate
    return None
```

**Breaking this down:**
- `for candidate in (filename, pathlib.Path(filename).stem):` — tries two things in order: first the full filename exactly as-is, then the "stem" (filename minus its last extension) as a fallback.
- `str | None` — a type hint meaning "this function returns either a string, or `None` if nothing matched" (this syntax needs Python 3.10+; on older versions you'd write `Optional[str]` from the `typing` module instead).
- The function returns as soon as it finds a match (`return candidate`), or `None` if neither the full name nor the stem is a valid IP.

---

**Sub-problem 3 — Reading the content of just the matching files**

Once you know which filenames are valid IPs, reading them is the easy part. `pathlib` gives you a one-line shortcut, `read_text()`, that opens the file, reads the whole thing, and closes it automatically:

```python
content = file_path.read_text()
```

(The more "manual" way, which you'll also see constantly and should recognize, is the `with open(...)` context manager — `with` guarantees the file gets closed even if an error happens partway through reading:

```python
with open(file_path, "r") as f:
    content = f.read()
```

Both are correct; `read_text()` is just shorter when you don't need finer control over how the file is opened.)

---

**Putting it all together — the full script**

```python
#!/usr/bin/env python3
import argparse
import ipaddress
import pathlib


def is_valid_ip(text: str) -> bool:
    try:
        ipaddress.ip_address(text)
        return True
    except ValueError:
        return False


def extract_ip_from_filename(filename: str) -> str | None:
    for candidate in (filename, pathlib.Path(filename).stem):
        if is_valid_ip(candidate):
            return candidate
    return None


def main() -> None:
    parser = argparse.ArgumentParser(description="Read content of files named after IP addresses")
    parser.add_argument("--folder", required=True, help="Folder containing the files")
    args = parser.parse_args()

    folder_path = pathlib.Path(args.folder).resolve()
    if not folder_path.is_dir():
        raise NotADirectoryError(f"{folder_path} is not a valid directory")

    for file_path in sorted(folder_path.iterdir()):
        if not file_path.is_file():
            continue
        ip = extract_ip_from_filename(file_path.name)
        if ip is None:
            continue
        print(f"--- {file_path.name} (IP: {ip}) ---")
        print(file_path.read_text())


if __name__ == "__main__":
    main()
```

```bash
python3 read_ip_files.py --folder /var/log/network-configs
```

**A couple more lines worth explaining for a beginner:**
- `if __name__ == "__main__":` — a very common Python pattern. `__name__` is a special variable Python sets automatically; it equals `"__main__"` only when the file is run directly (`python3 script.py`), not when it's imported by another script. This lets you write reusable functions in a file that can *also* be run standalone.
- `if not file_path.is_file(): continue` — skips anything that isn't a regular file (e.g., a subfolder), then `continue` jumps straight to the next loop iteration.
- `sorted(folder_path.iterdir())` — processes files in a consistent, predictable (alphabetical) order rather than whatever order the operating system happens to return them in, which can vary.

---

**Follow-up — How do you specify the directory/folder location in Python?**

This follow-up is checking whether you'd hardcode a path like a beginner, or handle it the way a script meant for real, repeated use actually should. I'd walk through the options from worst to best:

| Approach | Example | When it's appropriate |
|---|---|---|
| **Hardcoded string** | `folder_path = "/var/log/network-configs"` | Quick one-off script, never reused |
| **`os.path.join`** (older style) | `os.path.join("/var", "log", "network-configs")` | Legacy codebases; builds a path piece by piece |
| **`pathlib.Path`** (modern, preferred) | `pathlib.Path("/var/log/network-configs")` | Default choice — cross-platform, has useful methods |
| **Relative to the script itself** | `pathlib.Path(__file__).resolve().parent` | When the folder should always be found relative to where the script lives, not wherever it's run from |
| **Configurable — CLI arg / env var** | `argparse`, `os.getenv(...)` | Any real, reusable DevOps script |

The two points I'd make sure to hit:

1. **Current working directory vs. script location are not the same thing.** If you write `pathlib.Path("configs")`, that's resolved relative to whatever directory the script was *run from* (`os.getcwd()`) — which can be completely different from where the `.py` file itself lives, especially if the script is triggered by cron, CI/CD, or called from another directory. If you need a path relative to the script's own location instead, you use:
   ```python
   script_dir = pathlib.Path(__file__).resolve().parent
   folder_path = script_dir / "network-configs"
   ```
   `__file__` is a special variable Python sets to the path of the currently running script; `.resolve()` converts it to an absolute path; `.parent` gives you its containing folder; and the `/` operator on `Path` objects joins path segments (this is `pathlib`'s equivalent of `os.path.join`).

2. **Never hardcode it in anything meant to be reused.** For a real script, I'd take the folder as a command-line argument (`argparse`) with an optional environment-variable fallback, and validate it before doing anything else:
   ```python
   parser.add_argument("--folder", default=os.getenv("CONFIG_FOLDER"), required=False)
   ```
   ```python
   if not folder_path.is_dir():
       raise NotADirectoryError(f"{folder_path} is not a valid directory")
   ```
   Failing loudly and immediately if the folder doesn't exist is much better than the script silently doing nothing partway through, which is a very easy mistake to ship in a first draft.

---

**Real-world example — CloudCart**

At CloudCart, we had a nightly job that backed up the running configuration of every network switch and router, one file per device, named by the device's management IP — no extension, just the raw IP as the filename (e.g., `10.0.4.12`). A separate compliance script scanned that folder every morning to check that each device's config still contained a required security banner, and alerted the team in Slack if any device was missing it.

That script was originally written using `Path(filename).stem` to "clean up" the filename before validating it as an IP — exactly the gotcha described above. For most switches this happened to work by coincidence (some filenames did have a trailing `.cfg` extension from an earlier version of the backup job), but one switch, `10.0.4.12`, had no extension at all. `.stem` truncated it to `10.0.4`, `ipaddress.ip_address("10.0.4")` correctly rejected that as invalid, and the file was silently skipped — no error, no crash, just quietly never checked. That switch's security banner check didn't run for **several months** before someone noticed during a manual audit that it wasn't showing up in any of the Slack reports.

The fix was exactly the "check the full filename first, fall back to stem" pattern shown above. We also added a check at the end of the run that logs a warning for every file in the folder that *didn't* match any valid IP, specifically so a silent skip like this would show up in the logs instead of just disappearing.

---

**Complete thought process — how I approach this in the interview**

```
Three sub-problems, one after another:

1. List files in the folder
   → pathlib.Path(folder).iterdir() — modern, gives you Path objects
     with useful methods (vs. os.listdir(), which gives plain strings)

2. Which filenames are REAL IP addresses?
   → Don't reach for a naive regex — \d+\.\d+\.\d+\.\d+ matches
     999.999.999.999, which isn't a valid IP
   → Use the `ipaddress` module — ipaddress.ip_address() does full,
     correct validation, wrapped in try/except ValueError

   Gotcha: does the filename have a real extension, or is the IP
   itself the whole filename?
     → IP addresses contain dots, so Path.stem can truncate the last
       octet if there's no real extension — check the full filename
       first, fall back to .stem second

3. Read the matching files
   → Path.read_text() (shortcut) or `with open(...) as f: f.read()`
     (manual, more control) — either is correct

Follow-up: how is the folder path specified?
   → Never hardcoded in a reusable script
   → Distinguish current working directory (os.getcwd()) from the
     script's own location (Path(__file__).resolve().parent)
   → Make it configurable — argparse and/or an environment variable
   → Validate it exists (folder_path.is_dir()) before using it
```

---

**Summary (what to say if time is short):**

*"I'd use `pathlib` to list the folder's contents, since it gives me `Path` objects with useful built-in methods instead of plain strings. To decide which filenames are real IP addresses, I wouldn't use a naive regex like `\\d+\\.\\d+\\.\\d+\\.\\d+`, because that would also match invalid values like `999.999.999.999` — I'd use Python's built-in `ipaddress` module instead, wrapped in a try/except, since it does full, correct validation. One gotcha I'd watch for: if the filename is just the raw IP with no extension, `Path.stem` can accidentally truncate the last octet, because `pathlib` treats everything after the last dot as a file extension — so I'd check the full filename first and only fall back to the stem if that fails. Once I know which files match, I'd read their contents with `Path.read_text()` or a `with open(...)` block. For the follow-up — how the folder path itself gets specified — I'd never hardcode it in anything meant to be reused; I'd take it as a command-line argument via `argparse`, possibly falling back to an environment variable, resolve it to an absolute path, and validate that it's actually a directory before doing anything else — and if the path needs to be relative to the script's own location rather than wherever it's invoked from, I'd use `Path(__file__).resolve().parent` rather than assuming the current working directory."*

---

<!-- Add more scenario questions as Scenario 1, Scenario 2... -->

---

<!--
To add a new interview, copy the block below and paste it at the bottom:

## Interview #2

**Company:**
**Date:**
**Role Applied For:**
**Round:**
**Interviewer Level:**

### Questions Asked

#### Q1.

**Answer:**

---
-->
