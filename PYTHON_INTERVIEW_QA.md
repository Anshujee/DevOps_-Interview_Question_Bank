# Python (for DevOps) — Real Interview Questions & Answers

> This file is a personal log of actual Python questions asked to me by interviewers in real interviews.
> Questions and answers are added after each interview as they happened.
> Since Python is a supporting skill for DevOps roles (scripting, automation, tooling) rather than the primary focus, each answer here also breaks down the Python language mechanics in plain English — not just the DevOps use case — so it doubles as a learning reference.

---

## Table of Contents

- [Interview #1 — Coforge | DevOps Engineer | Technical Round 1](#interview-1)
  - [Q1. A folder has 10 files, 3 of them have IP addresses as filenames — how do you read only those files? (+ Follow-up: how do you specify the folder location in Python?)](#q1-a-folder-has-10-files-3-of-them-have-ip-addresses-as-filenames--how-do-you-read-only-those-files--follow-up-how-do-you-specify-the-folder-location-in-python)
  - [Q2. Can you write a Lambda function to read a file from S3? (+ Follow-up: reading S3 objects with boto3)](#q2-can-you-write-a-lambda-function-to-read-a-file-from-s3--follow-up-reading-s3-objects-with-boto3)
  - [Q3. A Python application isn't working properly — what Linux commands would you use to debug it? (+ Follow-up: what does the top command do?)](#q3-a-python-application-isnt-working-properly--what-linux-commands-would-you-use-to-debug-it)

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

#### Q2. Can you write a Lambda function to read a file from S3? (+ Follow-up: reading S3 objects with boto3)

**Answer:**

This is one of the most common real-world Lambda patterns — react to a new file landing in S3, read it, do something with the content. I'll write the version triggered automatically by an **S3 event** first, since that's the most typical setup, then cover the follow-up on the general boto3 patterns for reading S3 objects, including a couple of things that go wrong in practice if you don't know about them.

---

**The basic Lambda function**

```python
import boto3
import urllib.parse

s3_client = boto3.client("s3")

def lambda_handler(event, context):
    record = event["Records"][0]
    bucket_name = record["s3"]["bucket"]["name"]
    object_key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])

    response = s3_client.get_object(Bucket=bucket_name, Key=object_key)
    file_content = response["Body"].read().decode("utf-8")

    print(f"File content from s3://{bucket_name}/{object_key}:")
    print(file_content)

    return {
        "statusCode": 200,
        "body": f"Successfully read {object_key} from {bucket_name}"
    }
```

**Breaking this down line by line, since some of this is Lambda-specific and not obvious even if you know basic Python:**

- `s3_client = boto3.client("s3")` — I create this **outside** the `lambda_handler` function, at the top of the file, not inside it. This matters for a Lambda-specific reason: AWS keeps a Lambda function "warm" between invocations when traffic is frequent, reusing the same execution environment. Code outside the handler runs once per warm environment, not once per invocation — so the S3 client connection gets reused across multiple triggers instead of being recreated every single time, which is faster and is considered a Lambda best practice.
- `def lambda_handler(event, context):` — this exact function signature is what Lambda requires. `event` is a dictionary containing whatever triggered the function — for an S3 trigger, that's details about which bucket and file caused it. `context` carries runtime metadata (request ID, how much execution time is left, etc.) — not used in this example, but always part of the signature.
- `event["Records"][0]` — S3 event notifications always wrap the triggering file(s) inside a `"Records"` list, even when there's normally just one file per invocation. This is a fixed structure AWS uses across several event sources (S3, SQS, DynamoDB Streams), so it's worth recognizing.
- `record["s3"]["bucket"]["name"]` and `record["s3"]["object"]["key"]` — this is just navigating a nested dictionary to pull out the bucket name and the file's key (its path/name inside the bucket).
- **`urllib.parse.unquote_plus(...)` — a real gotcha, not optional.** S3 event notifications URL-encode the object key. A file named `Order Export 2026-08-01.csv` arrives in the event as `Order+Export+2026-08-01.csv`. If you use that encoded key directly in `get_object`, S3 can't find a file by that literal name and you get a `NoSuchKey` error — even though the file is right there. `unquote_plus` decodes it back to the real filename before you use it.
- `s3_client.get_object(Bucket=bucket_name, Key=object_key)` — the actual API call to S3. It returns a dictionary; the file's contents aren't a plain string in there, they're under `response["Body"]`, which is a **stream**, not text.
- `.read()` — pulls the raw bytes out of that stream, the same way you'd read any file-like object in Python.
- `.decode("utf-8")` — converts those raw bytes into a readable text string. This step is necessary because `.read()` on its own gives you `bytes`, not `str` — skip this and `print()`/string operations will show something like `b'order_id,total\n...'` instead of clean text.

---

**Where does the permission to call S3 come from? — no access keys, same principle as before**

The Lambda function never has an AWS access key anywhere in its code or environment variables set by you. It authenticates through its **IAM execution role** — the same "no static credentials" pattern that applies to everything in AWS I've touched on before. `boto3.client("s3")` automatically picks up temporary credentials that the Lambda service injects into the environment at runtime, tied to whatever execution role the function was configured with.

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject"],
      "Resource": "arn:aws:s3:::cloudcart-uploads/*"
    }
  ]
}
```

Scoped down to `s3:GetObject` only, on just the one bucket the function actually needs — not `s3:*` on `Resource: "*"`, which would be way more access than the function's actual job requires.

---

**Handling errors — what happens if the file isn't there, or the role lacks permission**

```python
from botocore.exceptions import ClientError

def lambda_handler(event, context):
    record = event["Records"][0]
    bucket_name = record["s3"]["bucket"]["name"]
    object_key = urllib.parse.unquote_plus(record["s3"]["object"]["key"])

    try:
        response = s3_client.get_object(Bucket=bucket_name, Key=object_key)
    except ClientError as e:
        error_code = e.response["Error"]["Code"]
        if error_code == "NoSuchKey":
            print(f"File {object_key} not found in {bucket_name}")
        elif error_code == "AccessDenied":
            print("Lambda execution role lacks s3:GetObject permission on this bucket")
        raise
```

`ClientError` is boto3's general exception for any failed AWS API call. `e.response["Error"]["Code"]` gives you the specific reason — `NoSuchKey` (wrong/missing file), `AccessDenied` (IAM problem), `NoSuchBucket`, etc. — so you can log something actually useful instead of a generic failure, and then `raise` re-throws it so Lambda correctly marks the invocation as failed (important if this is hooked up to retries or a dead-letter queue — swallowing the exception silently would hide real failures).

---

**Follow-up — reading S3 objects with boto3, more generally**

This follow-up usually comes to check whether you understand boto3 beyond the one pattern above, and whether you know when `get_object` isn't actually the right tool.

**1. Client API vs. Resource API**

```python
# Client — low-level, returns plain dictionaries (what I used above)
s3 = boto3.client("s3")
response = s3.get_object(Bucket="cloudcart-uploads", Key="orders.csv")
content = response["Body"].read().decode("utf-8")

# Resource — higher-level, object-oriented wrapper over the same API
s3 = boto3.resource("s3")
obj = s3.Object("cloudcart-uploads", "orders.csv")
content = obj.get()["Body"].read().decode("utf-8")
```
Both call the same underlying S3 API — `client` is more explicit and slightly more common in Lambda code; `resource` reads a bit more naturally when you're working with the object as a "thing" rather than a raw API call.

**2. `download_file` — when you need an actual file on disk, not bytes in memory**

```python
s3_client.download_file("cloudcart-uploads", "orders.csv", "/tmp/orders.csv")

with open("/tmp/orders.csv") as f:
    content = f.read()
```
Useful when a library you're using needs a real file path (some CSV/Excel/PDF parsing libraries expect a path, not raw bytes), or when the file is large enough that you'd rather stream it to disk than hold the whole thing in memory. Inside Lambda, `/tmp` is the only writable local disk — it's limited to **512 MB by default**, configurable up to **10 GB**, and it's wiped between cold starts, so it's scratch space, not persistent storage.

**3. Streaming large files instead of loading them fully into memory**

```python
response = s3_client.get_object(Bucket="cloudcart-uploads", Key="big-orders-export.csv")
for line in response["Body"].iter_lines():
    process(line)
```
`response["Body"]` behaves like a file object — `iter_lines()` reads it line by line instead of pulling the entire file into memory with one `.read()` call. This matters because Lambda has a hard memory ceiling (up to 10 GB, but most functions are configured well below that), and a multi-GB file read all at once with `.read()` can exhaust it and crash the function — I always ask "how big can this file realistically get?" before deciding between `get_object().read()` and streaming.

**4. S3 Select — querying a large file without downloading all of it**

For a large CSV/JSON/Parquet file where you only need a subset of rows or columns, `select_object_content()` lets you run a SQL-like query against the object directly inside S3 and get back only the matching data — genuinely useful when "reading the file" really means "reading a small piece of a huge file," and worth mentioning to show you know it exists, even if it's less common than plain `get_object`.

---

**Real-world example — CloudCart**

CloudCart has a Lambda function triggered whenever a new order-export CSV lands in an S3 bucket from an overnight batch job — it reads the file, validates the header row matches the expected schema, and either forwards it to the reporting pipeline or alerts the team in Slack if the format looks wrong.

We hit exactly the URL-encoding gotcha described above, early on. The batch job occasionally produced filenames like `Order Export 2026-08-01.csv` — a space in the name. The Lambda function worked fine in testing (test events were typed by hand, always clean filenames) but failed in production with `NoSuchKey` errors on any file with a space in its name, because the S3 event's object key arrived as `Order+Export+2026-08-01.csv` and we were passing that directly into `get_object` without decoding it. It took a confusing hour of "the file is right there, why can't Lambda find it" before someone spotted the `+` in the CloudWatch logs and connected it to URL encoding. `urllib.parse.unquote_plus` has been in every S3-triggered Lambda we've written since, as a standing rule.

---

**Complete thought process — how I approach this in the interview**

```
Lambda + S3 read — the standard shape:

1. Get the bucket/key out of the event
   → S3-triggered: event["Records"][0]["s3"]["bucket"]["name"] / ["object"]["key"]
   → Manually invoked: however the caller structured the event (e.g., event["bucket"])
   → ALWAYS urllib.parse.unquote_plus() the key — S3 event keys are URL-encoded

2. Call S3
   → get_object() → response["Body"] is a stream, not text
   → .read().decode("utf-8") to get a usable string

3. Handle failure explicitly
   → botocore.exceptions.ClientError, check e.response["Error"]["Code"]
   → re-raise after logging so Lambda correctly reports the failure

4. How big is the file, realistically?
   → Small → get_object().read() is fine
   → Large → stream with iter_lines(), or download_file() to /tmp
     (512 MB default, up to 10 GB), or S3 Select if you only need
     part of the data

5. Where do S3 permissions come from?
   → The Lambda's IAM execution role — no access keys anywhere,
     boto3 picks up temporary credentials automatically
```

---

**Summary (what to say if time is short):**

*"I'd write the handler to pull the bucket name and object key out of `event['Records'][0]['s3']`, making sure to URL-decode the key with `urllib.parse.unquote_plus` first, since S3 event keys are URL-encoded and filenames with spaces will otherwise fail with `NoSuchKey`. Then I'd call `s3_client.get_object()`, which returns the file content as a stream under `response['Body']`, not as plain text — so I'd call `.read().decode('utf-8')` to get a usable string. I'd wrap that in a try/except for `botocore.exceptions.ClientError` so I can log the actual reason — missing file versus an IAM permissions problem — rather than a generic failure. Permissions come entirely from the Lambda's IAM execution role, scoped to just `s3:GetObject` on the specific bucket, no access keys involved. For the follow-up on reading S3 more generally — if the file is small, `get_object().read()` is fine, but for large files I'd stream it with `iter_lines()` instead of loading it all into memory at once, since Lambda has a hard memory ceiling, or use `download_file()` to Lambda's `/tmp` directory if a library needs an actual file path rather than raw bytes."*

---

#### Q3. A Python application isn't working properly — what Linux commands would you use to debug it?

**Answer:**

This question is deliberately vague — "isn't working properly" could mean crashed, hanging, slow, silently doing nothing, or throwing errors intermittently — and that's the point. I don't answer with a random list of commands; I answer with a **workflow**, narrowing down from "is it even running" to "what exactly is it stuck on," using different tools at each stage. Let me walk through it in the order I'd actually work through it.

---

**Stage 1 — Is the process even running?**

```bash
ps aux | grep python
# or, more precisely, if you know the process name/service:
pgrep -fla python
```

```
appuser    4821  0.3  1.2 412300 98432 ?  Sl   09:14   0:12 python3 /opt/cloudcart/app.py
```

If it's managed by **systemd** (the normal case for a real production service), I'd check its status first — this alone often tells you most of the story:

```bash
systemctl status cloudcart-app.service
```

```
● cloudcart-app.service - CloudCart Backend API
   Loaded: loaded (/etc/systemd/system/cloudcart-app.service; enabled)
   Active: failed (Result: exit-code) since Fri 2026-08-22 09:20:11 UTC; 2min ago
  Process: 4821 ExecStart=/usr/bin/python3 /opt/cloudcart/app.py (code=exited, status=1/FAILURE)
```

That one command already tells me: it's not running, it exited with status 1, and roughly when. If it says `Active: running` instead, the process exists but is doing something wrong internally — different investigation path, covered below.

---

**Stage 2 — What do the logs actually say?**

This is where most real answers are found, and I always check it before reaching for anything more advanced.

```bash
journalctl -u cloudcart-app.service --since "10 min ago"
journalctl -u cloudcart-app.service -f          # follow live, like tail -f
tail -n 200 /var/log/cloudcart/app.log           # if it logs to a file instead of/alongside journald
```

If the process was killed unexpectedly with no clear application error, I specifically check whether the **kernel OOM killer** ended it — a very common, very silent cause of "the app just disappears":

```bash
dmesg | grep -i "killed process"
journalctl -k --since "10 min ago" | grep -i oom
```

```
Out of memory: Killed process 4821 (python3) total-vm:2100000kB, anon-rss:1980000kB
```

That single line changes the entire investigation — it's not an application bug, it's a memory/sizing problem.

---

**Stage 3 — Is it a resource problem?**

```bash
free -h          # memory — is the box actually out of RAM, or swapping heavily?
df -h             # disk space — a full disk breaks logging, temp files, SQLite, etc.
uptime            # load average — is the whole box under heavy CPU load?
vmstat 1          # live snapshot: CPU, memory, swap, I/O — repeats every 1 second
```

```
$ df -h
Filesystem      Size  Used Avail Use% Mounted on
/dev/xvda1       20G   20G     0 100% /
```

A `100%` full disk is a genuinely common, easy-to-miss root cause — the app can fail to write logs, fail to write temp files, or a SQLite/local database file can't be written to, and the resulting Python error is often something unhelpfully generic like `OSError: [Errno 28] No space left on device`, which people sometimes misread as an application bug rather than an infrastructure one.

---

**Stage 4 — If it's a service/API — is it actually listening, and is the network path open?**

```bash
ss -tulnp | grep python
# or the older equivalent:
netstat -tulnp | grep python
```

```
tcp   LISTEN  0  128  0.0.0.0:8080  0.0.0.0:*  users:(("python3",pid=4821,fd=6))
```

If nothing shows up on the expected port, the app either crashed before binding, or it's binding to the wrong interface/port (`127.0.0.1` instead of `0.0.0.0`, which would make it invisible to anything outside the box itself — a classic misconfiguration).

```bash
curl -v http://localhost:8080/health     # does it actually respond locally?
lsof -i :8080                             # what process, if any, owns that port
```

`lsof -i :8080` is specifically useful for the "port already in use" startup failure — it tells you exactly which other process is squatting on the port the app is trying to bind to.

---

**Stage 5 — Environment and permission problems**

This is the category of bug I've personally lost the most time to, and it's almost always the same shape: **"it works when I run it manually, but fails as a service."** The cause is nearly always that the app's actual runtime environment (systemd, cron) doesn't have the same environment variables, user, or working directory as your interactive shell.

```bash
# What user is the process actually running as?
ps -o user= -p 4821

# What environment variables does the RUNNING process actually have —
# not what YOUR shell has, what IT has:
cat /proc/4821/environ | tr '\0' '\n'
```

```
PATH=/usr/bin:/bin
HOME=/home/appuser
# Notice: no DATABASE_URL here — that's the bug
```

That's usually the "aha" moment — comparing `/proc/<PID>/environ` against what the app actually expects (e.g., in its config or `os.getenv()` calls) exposes missing environment variables immediately, without guessing.

```bash
which python3 && python3 --version    # is it running the interpreter/version you expect?
pip list                              # do installed package versions match requirements.txt?
ls -la /opt/cloudcart/app.py          # file permissions/ownership correct for the service user?
sudo -u appuser python3 /opt/cloudcart/app.py   # reproduce it running as the exact service account
```

---

**Stage 6 — It's running but hanging or stuck — you need to see what it's doing right now**

This is where I bring in more specialized tools, and specifically mention **`py-spy`**, because it's Python-native and doesn't require restarting or modifying the running process — genuinely one of the most useful tools for this exact situation:

```bash
py-spy dump --pid 4821
```

```
Thread 4821 (idle): "MainThread"
    get (requests/sessions.py:555)
    fetch_inventory (app.py:112)
    main (app.py:45)
```

That output tells you **exactly** which line of Python code the process is currently sitting on — in this example, it's stuck inside a `requests` HTTP call at `app.py:112`, which immediately points you toward "this is hanging on a network call to something else, not looping in its own logic." `py-spy top --pid <PID>` gives a live, `top`-style view instead of a one-time snapshot, useful when the app is busy rather than fully stuck.

For lower-level visibility — what actual system calls the process is making, useful when the hang isn't inside Python's own code but in a file/network operation underneath it:

```bash
strace -p 4821
```

```
read(6, ...) = -1 EAGAIN (Resource temporarily unavailable)
poll([{fd=6, events=POLLIN}], 1, -1
```

Seeing it stuck in a `read()`/`poll()` loop on a specific file descriptor confirms it's genuinely waiting on I/O (a socket, a file) rather than spinning in a CPU loop — different root cause, different fix.

---

**Real-world example — CloudCart**

CloudCart's backend API service worked perfectly when engineers ran it manually to test (`python3 app.py` from their own SSH session), but failed immediately every time it was started as a systemd service, with a generic `KeyError: 'DATABASE_URL'` in the logs.

The investigation went exactly through Stages 1–5 above: `systemctl status` confirmed it exited with a failure code immediately on start; `journalctl -u cloudcart-app.service` showed the `KeyError` traceback; and the fix came from comparing environment variables. Engineers were exporting `DATABASE_URL` in their personal `~/.bashrc`, which is loaded for interactive shells — but systemd services don't read `.bashrc` at all, they only see environment variables explicitly defined in the unit file (or an `EnvironmentFile=`). Running `cat /proc/<pid>/environ` on a similar service that *was* working made the difference obvious side-by-side — the working one had an `EnvironmentFile=/etc/cloudcart/app.env` line in its systemd unit; the broken one didn't.

We added a pre-deployment check that explicitly diffs the required environment variables (documented in the app's README) against what the systemd unit actually provides, specifically so this exact class of "works manually, fails as a service" bug gets caught before it reaches production again.

---

**Complete thought process — how I approach this in the interview**

```
"Isn't working properly" — narrow it down step by step, don't guess:

1. Is it even running?
   → ps aux / pgrep, or systemctl status if it's a managed service

2. What do the logs say?
   → journalctl -u <service> / tail -f <logfile>
   → dmesg / journalctl -k — was it OOM-killed?

3. Is a resource exhausted?
   → free -h (memory), df -h (disk), uptime / vmstat (CPU/load)

4. Is it network/port related (if it's a service)?
   → ss -tulnp — is it even listening?
   → curl locally, lsof -i :<port> — who owns the port?

5. Environment/permission mismatch — "works manually, fails as a service"?
   → cat /proc/<PID>/environ — compare against what the app expects
   → ps -o user= -p <PID>, sudo -u <serviceuser> to reproduce exactly

6. It's running but stuck/hanging — need to see it live?
   → py-spy dump --pid <PID> — Python-native, shows exact line it's on
   → strace -p <PID> — shows the actual syscall it's blocked on
```

---

**Summary (what to say if time is short):**

*"I wouldn't jump straight to a specific command — I'd narrow it down step by step. First, is it even running — `ps aux` or `systemctl status` if it's a managed service, which also tells me the exit code if it crashed. Then the logs — `journalctl -u` or the app's log file — and I'd specifically check `dmesg` for an OOM kill, since a process silently disappearing is often the kernel killing it for memory, not an application bug. If it's still unclear, I'd check resources with `free -h` and `df -h` — a full disk causes surprisingly generic-looking Python errors. If it's a service that should be listening on a port, `ss -tulnp` tells me if it's actually bound, and `curl` locally confirms it responds. A huge category of real bugs is 'works when I run it manually, fails as a service' — for that I'd compare `cat /proc/<PID>/environ` against what the app expects, since systemd doesn't load your shell's environment variables. And if the process is running but stuck, I'd reach for `py-spy dump --pid <PID>` first, since it's Python-native and shows exactly which line of code it's sitting on without needing to restart anything, and `strace -p <PID>` if I need to see what system call it's actually blocked on underneath that."*

---

**Follow-up — What does the `top` command do?**

`top` is a **real-time, interactive** process monitor — it shows you a live, auto-refreshing view (every 3 seconds by default) of every running process on the machine, ranked by resource usage, along with an overall summary of CPU, memory, and load at the top of the screen. It's usually the very first command I run when someone says "the server feels slow" or "something's eating CPU/memory," precisely because it answers "what, specifically, is the problem" in one glance rather than guessing.

```bash
top
```

```
top - 10:42:15 up 14 days,  3:21,  2 users,  load average: 2.15, 1.98, 1.42
Tasks: 148 total,   2 running, 145 sleeping,   0 stopped,   1 zombie
%Cpu(s): 78.3 us,  4.1 sy,  0.0 ni, 15.2 id,  0.8 wa,  0.0 hi,  1.6 si,  0.0 st
KiB Mem :  8165536 total,   412300 free,  6982104 used,   771132 buff/cache
KiB Swap:  2097148 total,  1245600 free,   851548 used.   980224 avail Mem

  PID USER      PR  NI    VIRT    RES    SHR S  %CPU  %MEM     TIME+ COMMAND
 4821 appuser   20   0  2100000 1980000  12400 R  97.3  24.3   4:12.55 python3
 1102 root      20   0  168400  10200   8100 S   0.3   0.1   0:02.11 sshd
```

**Breaking down the header — this alone often tells you what's wrong before you even look at individual processes:**

- **`load average: 2.15, 1.98, 1.42`** — average number of processes wanting CPU time, over the last 1/5/15 minutes. On a single-core machine, anything consistently above `1.0` means processes are queuing for CPU; on a 4-core machine, that threshold is `4.0`. Rising numbers left-to-right (like above) means load is climbing right now, which matters — it tells you whether a problem is getting worse or already recovering.
- **`Tasks: ... 1 zombie`** — a **zombie** process has finished running but its exit status hasn't been collected by its parent process yet. A handful appearing briefly is normal; a growing, persistent number of zombies usually points to a bug in the parent process not reaping its children properly.
- **`%Cpu(s)` line** — breaks total CPU usage down by category: `us` (user processes — your app), `sy` (kernel/system calls), `id` (idle — how much CPU is free), `wa` (waiting on I/O — high `wa` means CPU is idle specifically because it's stuck waiting on disk or network, not because there's no work to do — an important distinction when diagnosing "slow" vs. "busy"). In the example above, `78.3% us` with a Python process at `97.3% CPU` tells you immediately: this is a CPU-bound application problem, not an idle/waiting system.
- **`KiB Mem` / `KiB Swap` lines** — total/free/used memory and swap. Heavy swap usage (`used` swap growing) is a strong signal the box doesn't have enough RAM for what's running on it, and things will be slow even if CPU looks fine, since swapping to disk is orders of magnitude slower than RAM.

**The process table columns that matter most for debugging:**

| Column | Meaning |
|---|---|
| **PID** | Process ID — what you'd hand to `kill`, `strace -p`, `py-spy dump --pid`, etc. |
| **USER** | Which user is running it — useful for spotting a process running as the wrong user |
| **VIRT** | Total virtual memory the process has mapped — usually not the number to worry about |
| **RES** | **Resident memory** — actual physical RAM the process is using right now. This is the number I watch, refreshing over time, to confirm a memory leak — if `RES` keeps climbing for a PID that should have a stable memory footprint, that's a leak, not a guess |
| **S** | Process state — `R` (running), `S` (sleeping, waiting for something), `D` (uninterruptible sleep — almost always waiting on disk/network I/O; a process stuck in `D` state can't even be killed with a normal `kill` until the I/O completes), `Z` (zombie), `T` (stopped) |
| **%CPU / %MEM** | Live percentage of a CPU core / total system memory this process is using right now |
| **TIME+** | Total accumulated CPU time the process has consumed since it started — not wall-clock time, actual CPU-seconds used |
| **COMMAND** | The process name/command line |

**Useful interactive keys while `top` is running (you don't quit and re-run it — you press keys live):**

| Key | Effect |
|---|---|
| `P` | Sort by `%CPU` (default) — put the biggest CPU consumer at the top |
| `M` | Sort by `%MEM` — put the biggest memory consumer at the top |
| `1` | Show per-core CPU breakdown instead of one aggregate line |
| `k` | Kill a process by PID, without leaving `top` |
| `u` | Filter the view to just one user's processes |
| `q` | Quit |

**For scripting/logging instead of interactive use:**

```bash
top -b -n 1                 # batch mode, one snapshot, then exit — safe to pipe/redirect/cron
top -b -n 1 -p 4821          # batch mode, watching one specific PID
```

`top` without `-b` is meant for a human watching a terminal — it redraws the screen in place, which produces garbage output if you try to pipe it into a file or a script. `-b -n 1` (batch mode, one iteration) is what I'd actually use inside a monitoring script or a cron job that logs resource usage over time.

**`top` vs. `htop`:** `htop` is the friendlier, color-coded, scrollable, mouse-clickable version of the same idea — genuinely nicer to use — but it's a separate package that has to be installed, and it isn't guaranteed to be present on a minimal server or container image. `top` ships with essentially every Linux system by default, no installation required, which is exactly why it's the one I reach for first when I'm on an unfamiliar box and don't know what's already installed.

**Tying back to the earlier debugging workflow:** if I suspected a memory leak in the CloudCart backend, `top`, sorted by memory (`M`) and left running for a few minutes, watching one PID's `RES` column climb steadily upward with no plateau, is exactly how I'd confirm it's a real leak rather than normal memory usage — before going further and using `py-spy` to find *where* in the code the leak is actually coming from.

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
