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
  - [Q4. Write a script to monitor a directory and print the names of new files added every minute (+ Follow-up: difference between set and list?)](#q4-write-a-script-to-monitor-a-directory-and-print-the-names-of-new-files-added-every-minute)
  - [Q5. Write a function that takes a list of job log dictionaries and returns the job IDs where status is "FAILED"](#q5-write-a-function-that-takes-a-list-of-job-log-dictionaries-and-returns-the-job-ids-where-status-is-failed)
- [Interview #2 — Wipro | DevOps Engineer | Technical Round 1](#interview-2)
  - [Q1. What is the difference between a List and a Tuple in Python?](#q1-what-is-the-difference-between-a-list-and-a-tuple-in-python)
  - [Q2. What Python libraries or packages have you used for automation?](#q2-what-python-libraries-or-packages-have-you-used-for-automation)
  - [Q3. How would you copy a file from a remote server using Python?](#q3-how-would-you-copy-a-file-from-a-remote-server-using-python)
  - [Q4. How would you connect to an AWS EC2 Linux server using Python and execute a shell command such as `ls -lt`?](#q4-how-would-you-connect-to-an-aws-ec2-linux-server-using-python-and-execute-a-shell-command-such-as-ls--lt)
  - [Q5. What is the difference between SSH, SCP, and SFTP?](#q5-what-is-the-difference-between-ssh-scp-and-sftp)

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

#### Q4. Write a script to monitor a directory and print the names of new files added every minute

**Answer:**

This is a **polling** problem: check the directory's contents, wait, check again, and figure out what's different between the two checks. The core trick is using a Python **`set`**, because sets make "what's new" a single, fast operation — subtracting one set from another gives you exactly the items that are in the second but not the first.

---

**The script**

```python
import pathlib
import sys
import time


def get_current_files(folder_path: pathlib.Path) -> set[str]:
    return {f.name for f in folder_path.iterdir() if f.is_file()}


def monitor_directory(folder_path: pathlib.Path, interval_seconds: int = 60) -> None:
    previous_files = get_current_files(folder_path)
    print(f"Monitoring {folder_path} — {len(previous_files)} file(s) found at start.")

    while True:
        time.sleep(interval_seconds)
        current_files = get_current_files(folder_path)
        new_files = current_files - previous_files

        if new_files:
            for filename in sorted(new_files):
                print(f"New file detected: {filename}")
        else:
            print("No new files this minute.")

        previous_files = current_files


if __name__ == "__main__":
    folder = pathlib.Path(sys.argv[1]) if len(sys.argv) > 1 else pathlib.Path(".")
    monitor_directory(folder)
```

```bash
python3 monitor_folder.py /var/uploads
```

---

**Breaking down every new concept here, since this pulls in a few things Q1–Q3 didn't need:**

- **`set[str]`** — a Python `set` is an unordered collection with **no duplicates**. I'm using one specifically because sets support fast subtraction: `current_files - previous_files` returns every filename that's in `current_files` but **not** in `previous_files` — which is exactly the definition of "a file that's new since last time." Doing the equivalent with plain lists would mean manually looping and comparing, which is slower and more code for the same result.
- **`{f.name for f in folder_path.iterdir() if f.is_file()}`** — this is a **set comprehension**: like a `for` loop that builds a collection in one line. It's the same idea as a list comprehension (`[... for ... in ...]`), just with curly braces instead of square brackets, which tells Python to build a `set` instead of a `list`. `if f.is_file()` filters out subdirectories, so only actual files count.
- **`while True:`** — an infinite loop. It runs forever, checking the directory over and over, until the script is manually stopped (Ctrl+C) or killed. This is the right structure for a long-running monitoring script — it's meant to keep running indefinitely, not finish and exit.
- **`time.sleep(interval_seconds)`** — pauses execution for that many seconds without using CPU while it waits. Placing it at the top of the loop, before the check, means the script waits a full minute before its *first* comparison — which is correct here, since the very first snapshot (`previous_files`, taken before the loop starts) already captured whatever existed at startup; there's nothing "new" to report until a minute has actually passed.
- **`sys.argv`** — the list of command-line arguments passed to the script. `sys.argv[0]` is always the script's own filename; `sys.argv[1]` is the first real argument, if the user provided one. The line `pathlib.Path(sys.argv[1]) if len(sys.argv) > 1 else pathlib.Path(".")` means: use the folder the user specified, or default to the current directory if they didn't pass one.

---

**A real gotcha worth mentioning — "new" doesn't always mean "ready"**

If a file appears in the directory because something is actively **writing** to it — a large export, an upload still in progress via SFTP or `rsync` — this script will report it as "new" the moment it appears, even though its contents aren't finished yet. For a lot of real automation (like triggering a processing pipeline the moment a new file shows up), reading a half-written file is a genuine bug waiting to happen. A more robust version checks that the file's **size has stopped changing** across two consecutive checks before treating it as truly ready:

```python
def is_file_stable(file_path: pathlib.Path, wait_seconds: float = 2.0) -> bool:
    size_before = file_path.stat().st_size
    time.sleep(wait_seconds)
    size_after = file_path.stat().st_size
    return size_before == size_after
```

I'd only mention this if the interviewer's follow-up pushes toward "what could go wrong with this" — it's not part of the literal ask, but it's the kind of detail that shows real production experience rather than a textbook answer.

---

**Follow-up worth raising proactively — polling vs. event-driven monitoring**

The script above **polls**: it actively checks the directory on a fixed schedule, even if nothing changed. That's fine for a once-a-minute check, but it has two real limits: detection is delayed by up to the full interval (a file added right after a check waits almost a full minute to be reported), and it gets slower as the directory grows, since `iterdir()` has to list *everything* on every single check.

The alternative is **event-driven monitoring**, using the third-party **`watchdog`** library, which taps into the operating system's own filesystem notification mechanism (`inotify` on Linux, `FSEvents` on macOS) instead of repeatedly listing the directory:

```python
from watchdog.observers import Observer
from watchdog.events import FileSystemEventHandler
import time

class NewFileHandler(FileSystemEventHandler):
    def on_created(self, event):
        if not event.is_directory:
            print(f"New file detected: {event.src_path}")

if __name__ == "__main__":
    handler = NewFileHandler()
    observer = Observer()
    observer.schedule(handler, path=".", recursive=False)
    observer.start()
    try:
        while True:
            time.sleep(1)
    except KeyboardInterrupt:
        observer.stop()
    observer.join()
```

This detects a new file **almost instantly**, rather than waiting for the next poll, and doesn't get slower as the directory grows, since the OS itself pushes an event rather than the script re-scanning everything. I'd bring this up specifically because the question said "every minute" — which is exactly the kind of fixed-interval requirement that suggests polling is genuinely what's being asked for — but knowing the event-driven alternative exists, and why you'd reach for it instead, is exactly the kind of thing that separates "I can write the requested script" from "I understand the trade-off I'm making by writing it this way."

---

**Real-world example — CloudCart**

CloudCart has a folder where partner vendors drop daily inventory CSV files via SFTP, which triggers a processing pipeline. The first version of that watcher script was almost identical to the one above — poll, diff the file list, report anything new. It worked in testing, then broke in production: one vendor's upload was large enough that the SFTP transfer took a few seconds, and the watcher's poll happened to land in the middle of that transfer, picking up the file the moment it appeared and handing a **partial, truncated CSV** to the processing pipeline — which failed with a confusing parsing error that had nothing obviously to do with "the file wasn't finished uploading yet."

The fix was exactly the stability check described above — before treating a detected file as ready to process, the script waits a couple of seconds and re-checks its size, only proceeding once the size has stopped changing. It's a small addition, but it's the difference between a script that works in a quick test and one that's actually safe to run against real, unpredictable upload timing in production.

---

**Complete thought process — how I approach this in the interview**

```
"New files" = set difference between two snapshots in time

1. Take a snapshot of filenames now (a set, for fast comparison)
2. Sleep for the interval (60s here)
3. Take a new snapshot
4. new_files = current_snapshot - previous_snapshot
5. Report them, then the current snapshot becomes the new baseline
6. Repeat forever (while True) — this is a long-running process,
   not a one-shot script

Proactively worth mentioning:
  → "New" doesn't mean "finished being written" — a stability
    check (size unchanged across two checks) avoids reading a
    half-written file
  → Polling has a real alternative — event-driven monitoring via
    the `watchdog` library — worth knowing the trade-off even
    though a fixed "every minute" requirement points at polling
```

---

**Summary (what to say if time is short):**

*"I'd take a snapshot of the directory's filenames as a Python set, sleep for 60 seconds, take another snapshot, and use set subtraction — current minus previous — to get exactly the files that are new since the last check, printing those and then updating the baseline for the next loop. I'd wrap that in a `while True` loop since this needs to run indefinitely, not just once. One thing I'd flag proactively: a file showing up as 'new' doesn't mean it's finished being written — if something's still uploading to that folder, I'd add a quick check that the file's size has stopped changing across two checks before treating it as ready, since I've actually seen a script pick up a partially-written file and fail downstream because of it. I'd also mention that this is a polling approach — since the requirement is a fixed one-minute interval, that's the right tool here, but for something needing near-instant detection instead, I'd reach for the `watchdog` library instead, which uses the OS's own filesystem event notifications rather than repeatedly re-scanning the directory."*

---

**Follow-up — What is the difference between `set` and `list` in Python?**

This is a natural follow-up to the script above, since it specifically used a `set` rather than a `list` — and the reason why is really the core of the answer.

**Side-by-side**

| Aspect | `list` | `set` |
|---|---|---|
| **Order** | Preserved — stays in the order you added items | Not guaranteed — treat it as unordered |
| **Duplicates** | Allowed | Automatically removed — adding an existing value does nothing |
| **Indexing** | Yes — `my_list[0]` works | **No** — a set isn't subscriptable, `my_set[0]` raises `TypeError` |
| **Membership test (`x in ...`)** | **O(n)** — checks every element one at a time until it finds a match | **O(1)** on average — a hash lookup, effectively instant regardless of size |
| **What it can hold** | Anything, including other lists/dicts | Only **hashable** (effectively, immutable) items — no lists or dicts as elements |
| **Built-in set operations** | None — you'd write a loop | Union `\|`, intersection `&`, difference `-`, symmetric difference `^` — built in |
| **Syntax** | `[1, 2, 3]`, empty is `[]` | `{1, 2, 3}`, empty is `set()` — **`{}` alone creates an empty `dict`, not a set!** |

**A quick demonstration of the actual behavior:**

```python
my_list = [1, 2, 2, 3]
my_set = {1, 2, 2, 3}

print(my_list)   # [1, 2, 2, 3]  — duplicate 2 kept
print(my_set)    # {1, 2, 3}     — duplicate 2 silently dropped

my_list[0]        # 1 — works fine, lists are indexed
my_set[0]         # TypeError: 'set' object is not subscriptable

{1, 2, 3} - {2, 3}   # {1}         — set difference
{1, 2, 3} | {3, 4}   # {1, 2, 3, 4} — union
{1, 2, 3} & {2, 3, 4} # {2, 3}      — intersection
```

**Why the script above specifically needed a `set`, not a `list`**

Computing "what's new" is fundamentally a **repeated membership comparison** between two collections — for every filename in the current snapshot, you're effectively asking "was this already in the previous snapshot?" With a `set`, the built-in `-` operator does that whole comparison in one step, and because set membership checks are O(1) (a hash lookup) rather than O(n) (scanning the whole list), the comparison stays fast even if the directory has thousands of files. Doing the same thing with two `list`s would mean checking each filename against every filename in the other list — proportional to the size of *both* lists multiplied together — which gets noticeably slower as the directory grows, for no reason other than choosing the wrong data structure.

**Real-world example — CloudCart**

We once had a script that checked incoming order IDs against a list of already-processed order IDs to avoid double-processing a webhook that fired twice — written with a plain Python `list`, since early on there were only a few hundred processed IDs and the `in` check was instant either way. As that list grew into the tens of thousands over a few months of production traffic, the check `if order_id in processed_ids` became measurably slower — visible in the service's own latency metrics, since every single incoming webhook was now scanning a list with tens of thousands of entries one item at a time. Switching `processed_ids` from a `list` to a `set` made the check effectively instant again, completely independent of how many IDs had accumulated — a one-line change (`processed_ids = set(...)` instead of `processed_ids = [...]`) that fixed a real, gradually-worsening performance problem.

**When I'd actually reach for each:**
- **`list`** — when order matters, duplicates are meaningful, or I need to access items by position
- **`set`** — when I only care about uniqueness, I'm doing frequent membership checks, or I need fast mathematical set operations like union/intersection/difference

---

#### Q5. Write a function that takes a list of job log dictionaries and returns the job IDs where status is "FAILED"

**Answer:**

**Corrected example data first** — the pasted example had a few transcription issues (parentheses `(...)` where it should be curly braces `{...}`, `job_jd` instead of `job_id`, `imestamp` instead of `timestamp`, a stray `*`, and a missing comma before the last entry). Here's what it should actually look like as valid Python:

```python
logs = [
    {"job_id": 101, "status": "SUCCESS", "timestamp": "2025-06-10T10:00:00"},
    {"job_id": 102, "status": "FAILED",  "timestamp": "2025-06-10T10:05:00"},
    {"job_id": 103, "status": "FAILED",  "timestamp": "2025-06-10T10:10:00"},
    {"job_id": 104, "status": "SUCCESS", "timestamp": "2026-06-10T10:15:00"},
]
```

This is a filtering problem: loop through the logs, keep only the ones where `status` is `"FAILED"`, and pull out their `job_id`.

---

**The straightforward solution**

```python
def get_failed_job_ids(logs: list[dict]) -> list[int]:
    return [log["job_id"] for log in logs if log["status"] == "FAILED"]
```

```python
print(get_failed_job_ids(logs))
# [102, 103]
```

**Breaking this down:** this is a **list comprehension with a filter condition** — the general shape is `[expression for item in iterable if condition]`. Read left to right: "for each `log` in `logs`, if `log["status"] == "FAILED"`, include `log["job_id"]` in the result." `log["job_id"]` and `log["status"]` are plain dictionary key lookups — square brackets, the key name as a string.

For anyone who finds the comprehension harder to read at a glance, this is functionally identical, written as an explicit loop:

```python
def get_failed_job_ids(logs: list[dict]) -> list[int]:
    failed_ids = []
    for log in logs:
        if log["status"] == "FAILED":
            failed_ids.append(log["job_id"])
    return failed_ids
```

---

**The gotcha I'd raise proactively — and it's directly relevant given how the example data arrived**

Using `log["status"]` and `log["job_id"]` with square brackets **crashes the entire function** with a `KeyError` the moment it hits even one dictionary that's missing that key — for example, a malformed log entry (exactly the kind of typo the pasted example itself had, like `job_jd` instead of `job_id`). One bad record shouldn't be able to take down the processing of the other 99 good ones.

The fix is using **`.get()`** instead of direct key access — `.get("key")` returns `None` if the key doesn't exist, instead of raising an exception:

```python
def get_failed_job_ids(logs: list[dict]) -> list[int]:
    failed_ids = []
    for log in logs:
        if log.get("status") == "FAILED":
            job_id = log.get("job_id")
            if job_id is not None:
                failed_ids.append(job_id)
    return failed_ids
```

A record missing `"status"` entirely just silently doesn't match `"FAILED"` (since `None != "FAILED"`) and gets skipped, rather than crashing the whole batch. I'd mention this as the more production-appropriate version, especially for something like log processing, where malformed records are a realistic, expected occurrence rather than a hypothetical edge case.

---

**A clarifying question worth asking the interviewer, not assuming**

The spec doesn't say what should happen if the **same `job_id` appears more than once** with `"FAILED"` status — for example, a job that was retried and failed twice, logged as two separate entries. Should the result contain that ID twice (one per failed log entry), or once (unique failed job IDs)? I'd ask rather than guess. If uniqueness is wanted, this connects directly to the set-vs-list discussion from the previous question — I'd track "already seen" IDs in a `set` for fast O(1) lookups, while still returning an order-preserving `list`:

```python
def get_failed_job_ids_unique(logs: list[dict]) -> list[int]:
    seen = set()
    failed_ids = []
    for log in logs:
        if log.get("status") == "FAILED":
            job_id = log.get("job_id")
            if job_id is not None and job_id not in seen:
                seen.add(job_id)
                failed_ids.append(job_id)
    return failed_ids
```

---

**Real-world example — CloudCart**

We have a nightly script that pulls batch job results (a list of dicts, structurally identical to this example) from a job-runner's status API and posts a Slack alert listing any failed job IDs. The very first version used direct key access (`log["status"]`), and it broke in production the first time a legacy job runner — a slightly older version still running for one team that hadn't upgraded yet — emitted a log entry with a differently-named field instead of `"status"`. One malformed record from that one legacy runner caused a `KeyError` that killed the entire nightly report for **every** team, not just the one with the odd log format — nobody got their failure alerts that night because of a single bad record buried among hundreds of good ones. Switching to `.get()` fixed it immediately: a genuinely malformed record just gets skipped (and now also logged separately as "couldn't parse this entry" for visibility), while every well-formed record is still processed and reported correctly.

---

**Complete thought process — how I approach this in the interview**

```
1. Basic filter → list comprehension with a condition:
   [log["job_id"] for log in logs if log["status"] == "FAILED"]

2. Is malformed/incomplete input realistic here (log data usually is)?
   → Yes → use .get() instead of direct [] access, so one bad
     record doesn't crash the whole batch with a KeyError

3. Could the same job_id appear more than once?
   → Ask, don't assume — if uniqueness is wanted, track "seen"
     IDs in a set (O(1) lookup) while building an order-preserving
     list of results
```

---

**Summary (what to say if time is short):**

*"The core of it is a list comprehension with a filter: for each log entry, if status equals FAILED, include its job_id in the result. The detail I'd add without being asked is using `.get()` instead of direct dictionary key access, since real log data realistically includes malformed entries, and I don't want one bad record — missing the status or job_id key — to raise a KeyError and crash the processing of every other valid record in the batch. I'd also ask whether the same job_id could legitimately appear more than once, for something like a retried job that failed twice — if duplicates should be collapsed, I'd track already-seen IDs in a set for fast lookups while still building the result as an order-preserving list."*

---

<!-- Add more scenario questions as Scenario 1, Scenario 2... -->

---

## Interview #2

**Company:** Wipro
**Date:** 23-08-2026
**Role Applied For:** DevOps Engineer
**Round:** Technical Round 1
**Interviewer Level:** Not specified

---

### Questions Asked

#### Q1. What is the difference between a List and a Tuple in Python?

**Answer:**

The one-line version is "lists are mutable, tuples aren't" — true, but it undersells *why* that one difference cascades into several other, practically important differences: performance, what each can be used for, and even what each communicates about intent when someone reads the code.

---

**Side-by-side**

| | List | Tuple |
|---|---|---|
| **Syntax** | `[1, 2, 3]` | `(1, 2, 3)` |
| **Mutability** | Mutable — items can be added, removed, or changed after creation | **Immutable** — once created, it cannot be changed, resized, or reordered at all |
| **Methods available** | `.append()`, `.remove()`, `.sort()`, `.pop()`, `.insert()`, etc. | Only `.count()` and `.index()` — nothing that would modify it, because nothing can |
| **Hashable?** | **No** — can't be used as a dict key or put inside a `set` | **Yes**, if every element inside it is also hashable — so it *can* be a dict key or set member |
| **Performance** | Slightly slower to create, more memory overhead — has to support resizing | Slightly faster to create and iterate, smaller memory footprint — fixed size means less overhead |
| **Typical intent signaled** | "A collection that may grow, shrink, or change" | "A fixed, small record — this shouldn't change" |

---

**The hashability difference is the one people most often miss**

Because a list can be mutated, Python can't compute a stable hash for it — so it's explicitly disallowed as a dictionary key or a member of a `set`. A tuple *can* be hashed, provided everything inside it is itself hashable, which makes it usable in places a list simply can't go:

```python
# This raises TypeError: unhashable type: 'list'
cache = {}
cache[[40.7128, -74.0060]] = "New York"   # TypeError

# This works fine — a tuple of hashable values is itself hashable
cache = {}
cache[(40.7128, -74.0060)] = "New York"   # OK

# Same reason sets of coordinates use tuples, not lists
seen_coordinates = {(40.7128, -74.0060), (34.0522, -118.2437)}
```

---

**Tuple immutability is shallow, not deep — a common gotcha worth knowing**

A tuple itself can't be reassigned or resized, but if one of its elements is itself a **mutable** object, that inner object can still be changed — the tuple's immutability only guarantees the tuple's own references don't change, not that everything reachable through it is frozen:

```python
t = (1, 2, [3, 4])
t[0] = 99          # TypeError — can't reassign a tuple element
t[2].append(5)      # Works fine — the LIST inside the tuple is still mutable
print(t)             # (1, 2, [3, 4, 5])
```
This is the detail that separates a surface-level "tuples are immutable" answer from actually understanding what that immutability covers.

---

**When I'd reach for each**

- **List** — a collection I expect to grow, shrink, filter, or sort: results from a query, items being accumulated in a loop, anything processed with list comprehensions.
- **Tuple** — a small, fixed-shape record that shouldn't change after creation: coordinates `(lat, lon)`, a `(status_code, message)` pair returned from a function, or **dictionary keys/set members** when the immutability is a hard requirement, not just a preference. Tuples are also what a function returns when it "returns multiple values" — `return status, message` is actually returning a tuple, just without explicit parentheses.

```python
def get_health_check():
    return "OK", 200   # implicitly a tuple: ("OK", 200)

status, code = get_health_check()   # tuple unpacking
```

---

**Real-world example — CloudCart**

In the job-log processing function from Q5 in this interview, each job's result is built as a **list** of job IDs, because the whole point is a collection that gets filtered and can vary in length run to run — a tuple would be the wrong tool there, since nothing about that output is fixed-size. By contrast, CloudCart's health-check endpoint cache keys off `(service_name, region)` — a **tuple** — specifically because that pair needs to be a dictionary key (`health_cache[("order-service", "us-east-1")] = last_check_result`), and a list literally cannot be used as one; the immutability isn't just a style preference there, it's the reason a tuple is the only correct choice.

---

**Complete thought process — how I approach this in the interview**

```
Core difference: mutable (list) vs immutable (tuple) — but don't
stop there, walk through what THAT actually causes:

  → Methods: list has mutating methods, tuple only has read-only ones
  → Hashability: only tuples (with hashable contents) can be dict
    keys / set members — lists explicitly cannot
  → Performance: tuples are slightly faster/smaller due to fixed size
  → Gotcha: tuple immutability is shallow — a mutable object INSIDE
    a tuple can still be mutated; only the tuple's own slots are frozen

When to use which:
  → List: collection that changes size/order/contents over its life
  → Tuple: fixed-shape record, multiple return values, or anything
    that needs to be hashable (dict key / set member)
```

---

**Summary (what to say if time is short):**

*"The core difference is mutability — lists can be changed after creation, tuples can't — but that one difference has real downstream consequences. Because tuples can't change, they're hashable if their contents are, which means they can be used as dictionary keys or set members, while lists explicitly cannot — that's usually the detail people miss when they just say 'lists are mutable, tuples aren't' and stop there. Tuples are also slightly faster and use less memory, since Python doesn't need to support resizing them. One gotcha worth mentioning: tuple immutability is shallow — if a tuple contains a mutable object like a list, that inner list can still be modified, only the tuple's own slots are frozen. I'd use a list for anything I expect to grow, shrink, or reorder — query results, items accumulated in a loop — and a tuple for a small, fixed-shape record, multiple return values from a function, or specifically whenever I need something hashable, like a dictionary key built from more than one value."*

---

#### Q2. What Python libraries or packages have you used for automation?

**Answer:**

I'd group these by what problem they solve, since "automation" libraries aren't one category — they span cloud SDKs, system/process interaction, file/config handling, HTTP, and scheduling. Naming a grouped list shows I understand *why* each one gets reached for, not just that I've seen the name before.

---

**Cloud & infrastructure automation**

| Library | What it's for |
|---|---|
| **`boto3`** | The AWS SDK for Python — used throughout this interview already: the S3-reading Lambda in Q2, and it's what actually implements the IAM-role-based access pattern discussed across the AWS interview. This is the single most-used library for AWS automation, full stop. |
| **`kubernetes`** (the official Python client) | Programmatic access to the Kubernetes API — listing pods, checking Deployment health, triggering a rolling restart — exactly what the auto-remediation Lambda in the DevOps interview (Q4) would use instead of shelling out to `kubectl`. |
| **`docker`** (Docker SDK for Python) | Building, running, and inspecting containers programmatically, without shelling out to the `docker` CLI — useful for tooling that needs to manage containers as part of a larger script rather than a one-off command. |

**System, process, and file interaction**

| Library | What it's for |
|---|---|
| **`os` / `pathlib`** | Filesystem paths and operations — `pathlib` is the modern, object-oriented way to do this (used in Q1's folder-scanning answer), `os` is the older, still-common lower-level API. |
| **`subprocess`** | Running shell commands from Python and capturing their output — the bridge between "Python automation" and "the Linux commands I'd normally run by hand," like the debugging commands from Q3. |
| **`shutil`** | Higher-level file operations — copying, moving, archiving — that `os` doesn't handle directly. |
| **`watchdog`** | Filesystem event monitoring — the *real* tool for "watch a directory and react to new files," as opposed to the polling-loop approach I used in Q4's answer for simplicity. `watchdog` reacts to OS-level filesystem events instead of repeatedly listing a directory, which is both faster to react and lighter on resources for something watched continuously. |

**Configuration, data, and APIs**

| Library | What it's for |
|---|---|
| **`PyYAML`** | Reading/writing YAML — parsing Kubernetes manifests or Terraform-adjacent config files programmatically, rather than treating them as opaque text. |
| **`json`** (standard library) | Almost every cloud API — including boto3 under the hood — communicates in JSON; this is the baseline for handling that. |
| **`requests`** | Making HTTP calls — hitting an internal API, posting a Slack/Teams webhook notification (exactly what the auto-remediation example's "notify SNS" step could alternatively be, if the target were a chat webhook instead of SNS), or calling a REST API a cloud SDK doesn't cover. |
| **`python-dotenv`** | Loading configuration from a `.env` file into environment variables during local development — never used for how production actually gets its secrets (that's Secrets Manager/instance roles, as covered throughout the AWS interview), but genuinely useful for keeping local dev config out of the shell profile. |

**Scheduling, CLI, and testing**

| Library | What it's for |
|---|---|
| **`argparse`** (standard library) | Turning a script into a proper CLI tool with named flags and `--help` output, instead of positional `sys.argv` parsing — makes an automation script something a teammate can actually run without reading the source first. |
| **`schedule`** | A lightweight, in-process way to run a function on a recurring interval — fine for something simple; for anything that needs to survive a process restart or run reliably in production, a real scheduler (cron, a Kubernetes CronJob, EventBridge) is the better tool, and I'd say that explicitly rather than oversell `schedule`'s use case. |
| **`pytest`** | Testing the automation scripts themselves — automation that's never tested is exactly the kind of thing that silently breaks and isn't noticed until it's needed during an incident. |

---

**Real-world example — CloudCart**

Most of CloudCart's Python automation is genuinely simple, boring `boto3` and standard-library code — that's not a weakness, it's usually the right call for scripts that need to be maintainable by whoever's on call, not clever. The S3-reading Lambda (Q2) uses `boto3` alone. The directory-watcher (Q4) could be upgraded from its polling-loop approach to `watchdog` if it ever needed to scale to watching many directories continuously rather than a small, low-frequency check — a change I'd make if profiling showed the polling interval was actually a bottleneck, not preemptively. The one library that came up outside pure scripting was `requests`, used in a small internal tool that posts deployment notifications to a Slack channel via webhook whenever the CI/CD pipeline (DevOps interview, Q2) promotes an image to PROD — a much simpler alternative to standing up an SNS+Lambda chain when the only consumer is a Slack channel, not multiple downstream systems.

---

**Complete thought process — how I approach this in the interview**

```
Group by problem, don't just list names:

  Cloud/infra    → boto3 (AWS), kubernetes client, docker SDK
  System/files   → os/pathlib, subprocess, shutil, watchdog
  Config/data    → PyYAML, json, requests, python-dotenv
  Scheduling/CLI → argparse, schedule (with its real limits stated
                    honestly), pytest for testing the automation itself

For each, be ready to say WHEN I'd reach for it vs. an alternative —
e.g., watchdog vs. a polling loop, schedule vs. a real scheduler like
cron/EventBridge — shows judgment, not just familiarity with the name
```

---

**Summary (what to say if time is short):**

*"I'd group them by problem rather than list names. For cloud automation, boto3 for AWS — the same library behind the S3 Lambda I described earlier — plus the official kubernetes and docker Python clients when automation needs to talk to those APIs directly instead of shelling out to a CLI. For system-level work, pathlib and os for filesystem paths, subprocess for running shell commands from Python, and watchdog specifically for reacting to filesystem events rather than polling a directory in a loop. For config and APIs, PyYAML for parsing YAML manifests, and requests for anything that's a plain HTTP call — like posting a deployment notification to a Slack webhook. And for making a script production-usable rather than a one-off, argparse for a proper CLI interface and pytest for actually testing the automation itself, since untested automation tends to fail silently right when it's needed most. Most of what I've actually used day to day is intentionally simple — boto3 and the standard library — because maintainability by whoever's on call matters more than using a fancier library for its own sake."*

---

#### Q3. How would you copy a file from a remote server using Python?

**Answer:**

The first thing I'd clarify is **what kind of "remote server"** — the right library depends entirely on the protocol involved, and picking wrong is the difference between clean, maintainable code and something fragile. I'll cover the most common case first, then the others briefly.

---

**Case 1 — A generic remote server over SSH/SFTP (the most common interpretation)**

This is what "remote server" usually means in a DevOps context — pulling a file off an EC2 instance, an on-prem box, or any server reachable over SSH. The right tool is **`paramiko`**, a pure-Python SSH client library — no shelling out to the system's `scp`/`ssh` binaries required.

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.load_system_host_keys()
ssh.set_missing_host_key_policy(paramiko.RejectPolicy())  # fail closed on an unknown host, don't silently trust it
ssh.connect(
    hostname="10.0.1.25",
    username="deploy",
    key_filename="/home/deploy/.ssh/id_rsa",   # key-based auth, never a hardcoded password
)

sftp = ssh.open_sftp()
sftp.get("/var/log/myapp/app.log", "/local/backup/app.log")
sftp.close()
ssh.close()
```

**Two details worth calling out explicitly, because they're exactly what separates a "works on my laptop" answer from a production-minded one:**

- **Host key policy.** `paramiko.AutoAddPolicy()` shows up in almost every tutorial, but it silently trusts and accepts *any* server's host key on first connect — that's a real MITM exposure in a security-conscious environment. `load_system_host_keys()` + `RejectPolicy()` (or explicitly pinning the expected host key) means an unrecognized/changed host key fails the connection loudly instead of connecting anyway.
- **Authentication.** Key-based (`key_filename`, or better, an SSH agent) — never a password baked into the script. Same no-standing-credentials principle as every AWS answer in this repo, just applied to SSH instead of IAM.

**Cleaner version using a context manager**, so the connection always closes even if the transfer fails partway:

```python
from contextlib import closing

with closing(paramiko.SSHClient()) as ssh:
    ssh.load_system_host_keys()
    ssh.set_missing_host_key_policy(paramiko.RejectPolicy())
    ssh.connect(hostname="10.0.1.25", username="deploy", key_filename="/home/deploy/.ssh/id_rsa")
    with ssh.open_sftp() as sftp:
        sftp.get("/var/log/myapp/app.log", "/local/backup/app.log")
```

**If it genuinely needs to feel like `scp`** (recursive directory copies, progress callbacks), the `scp` package layers on top of the same `paramiko` transport:

```python
from scp import SCPClient

with closing(SCPClient(ssh.get_transport())) as scp:
    scp.get("/var/log/myapp/", "/local/backup/", recursive=True)
```

---

**Case 2 — The "remote server" is actually cloud storage (S3), not a machine**

If the file lives in S3 rather than on a server with an OS and an SSH daemon, this is a completely different, much simpler problem — the same `boto3` from Q2 in this interview, no SSH involved at all:

```python
import boto3

s3 = boto3.client("s3")
s3.download_file("cloudcart-logs-archive", "app-logs/app.log", "/local/backup/app.log")
```
Worth stating this distinction explicitly in the interview — conflating "a server" with "an S3 bucket" is a common mix-up, and using `paramiko` against S3 (or vice versa) simply doesn't work; they're not the same kind of "remote."

---

**Case 3 — A plain HTTP(S) download endpoint**

If "remote server" means a file served over an HTTP endpoint rather than SSH access to a machine, that's `requests`, streamed rather than loaded fully into memory for anything non-trivially sized:

```python
import requests

with requests.get("https://internal-artifact-server/build/app.tar.gz", stream=True) as r:
    r.raise_for_status()
    with open("/local/backup/app.tar.gz", "wb") as f:
        for chunk in r.iter_content(chunk_size=8192):
            f.write(chunk)
```
`stream=True` plus chunked writing is the detail that matters here — downloading the whole response into memory first (`requests.get(url).content`) works fine for a small file, but for anything large enough to matter, that's an avoidable memory spike.

---

**Side-by-side — picking the right one**

| "Remote server" means... | Library | Auth |
|---|---|---|
| A machine reachable over SSH | `paramiko` (+ `scp` package if recursive/progress needed) | SSH key, never a password |
| An S3 bucket | `boto3` | IAM role/instance profile, never static keys |
| An HTTP(S) file endpoint | `requests` (streamed) | API key/token in a header, or none if public |

---

**Real-world example — CloudCart**

Before CloudCart's log pipeline moved to the CloudWatch Agent → Firehose → S3 approach (AWS interview, Q1), an early, ad-hoc need was pulling nightly config backups off a legacy on-prem file server that had no AWS integration at all — a `paramiko`-based script, run as a scheduled job, connected over SFTP with a dedicated service account's SSH key and pulled the previous day's backup file down to a jump host, from which a separate step uploaded it to S3. That script deliberately used `RejectPolicy()` with a pre-loaded known-hosts file rather than `AutoAddPolicy()`, specifically because it was pulling from a server outside CloudCart's own VPC — the "we don't fully trust the network path" scenario is exactly when a strict host-key policy earns its keep, versus a same-VPC transfer where the blast radius of a mistake is smaller.

---

**Complete thought process — how I approach this in the interview**

```
First question: what IS the "remote server" — the answer branches
completely depending on the protocol:

  SSH-reachable machine → paramiko (SFTP), scp package if recursive
  S3 bucket              → boto3 — NOT paramiko, different kind of
                            "remote" entirely
  HTTP(S) endpoint        → requests, streamed for anything non-trivial

For the SSH case specifically, two production-minded details to
volunteer without being asked:
  → Host key policy: RejectPolicy()/pinned keys, not AutoAddPolicy()
    — the latter is a real MITM exposure, common in tutorials
  → Auth: SSH key, never a hardcoded password — same principle as
    IAM roles vs. static access keys elsewhere in this repo
```

---

**Summary (what to say if time is short):**

*"It depends what 'remote server' actually means, so I'd clarify that first. If it's a machine reachable over SSH, I'd use paramiko to open an SFTP session and pull the file with key-based authentication, never a hardcoded password — and I'd explicitly set a strict host key policy like RejectPolicy with a pre-loaded known-hosts file, rather than AutoAddPolicy, which a lot of tutorials use but which silently trusts any server's host key on first connect, a real man-in-the-middle exposure. If the file actually lives in S3 rather than on a server with an OS, that's a completely different tool — boto3's download_file, no SSH involved at all. And if it's served over a plain HTTP endpoint, that's requests, streamed in chunks rather than loaded fully into memory if the file is any real size. The main thing I'd want to get across is that 'remote server' isn't one problem — picking the right tool depends entirely on what's actually on the other end."*

---

#### Q4. How would you connect to an AWS EC2 Linux server using Python and execute a shell command such as `ls -lt`?

**Answer:**

There are genuinely two right answers here depending on the environment, and I'd volunteer both rather than just the first one that comes to mind — because the better of the two is specific to this being an **AWS EC2** instance rather than a generic server, which is exactly the distinction the question is testing for.

---

**Approach A — paramiko over SSH (the generic, works-anywhere answer)**

Same connection setup as Q3, but using `exec_command()` instead of SFTP:

```python
import paramiko

ssh = paramiko.SSHClient()
ssh.load_system_host_keys()
ssh.set_missing_host_key_policy(paramiko.RejectPolicy())
ssh.connect(hostname="10.0.1.25", username="ec2-user", key_filename="/home/deploy/.ssh/id_rsa")

stdin, stdout, stderr = ssh.exec_command("ls -lt")

exit_status = stdout.channel.recv_exit_status()   # blocks until the command finishes
output = stdout.read().decode()
errors = stderr.read().decode()

if exit_status == 0:
    print(output)
else:
    print(f"Command failed (exit {exit_status}): {errors}")

ssh.close()
```

**The detail that trips people up:** `exec_command()` returns immediately — reading `stdout`/`stderr` and calling `stdout.channel.recv_exit_status()` is what actually blocks until the remote command finishes and gives you a real exit code. Skipping that and just reading `stdout.read()` alone usually still works in practice, but doesn't give a reliable way to know whether the command actually **succeeded** — you'd have no exit status to check.

This approach requires: the instance reachable on port 22, an SSH key deployed to it, and — if it's in a private subnet — a bastion host or equivalent path to reach it at all.

---

**Approach B — AWS Systems Manager Run Command via boto3 (the better answer, specific to this being EC2)**

Since this is explicitly an **EC2** instance, not just "a Linux server," the stronger answer is: **don't use SSH at all.** This connects directly to the SSM Agent discussed in the AWS interview (Q9) — the same mechanism behind Session Manager, IAM-authenticated, with no open port 22, no SSH key to manage or rotate, and no bastion host required, since the SSM Agent on the instance initiates an **outbound** connection rather than needing anything inbound:

```python
import boto3
import time

ssm = boto3.client("ssm")

response = ssm.send_command(
    InstanceIds=["i-0abc123def456789"],
    DocumentName="AWS-RunShellScript",
    Parameters={"commands": ["ls -lt"]},
)
command_id = response["Command"]["CommandId"]

# Run Command is async — poll for the result
while True:
    result = ssm.get_command_invocation(CommandId=command_id, InstanceId="i-0abc123def456789")
    if result["Status"] in ("Success", "Failed", "Cancelled", "TimedOut"):
        break
    time.sleep(1)

if result["Status"] == "Success":
    print(result["StandardOutputContent"])
else:
    print(f"Command failed: {result['StandardErrorContent']}")
```

**What this requires instead of SSH:** the instance has the SSM Agent running (installed by default on Amazon Linux 2/2023 AMIs) and an **IAM instance profile** with the `AmazonSSMManagedInstanceCore` policy — no key pair, no security group rule for port 22, and access is governed entirely by IAM permissions on the *caller's* side (`ssm:SendCommand` scoped to specific instances/tags), which is also centrally logged in CloudTrail — every command run this way is auditable by identity, unlike SSH access which typically isn't logged with the same granularity unless session logging is separately configured.

---

**Side-by-side**

| | paramiko (SSH) | boto3 + SSM Run Command |
|---|---|---|
| Requires port 22 open? | Yes | No |
| Requires an SSH key? | Yes — deployed, rotated, managed | No |
| Works through a private subnet with no bastion? | No — needs a path to port 22 | Yes — SSM Agent connects outbound |
| Access control | Whoever holds the SSH key | IAM policy, per-instance/per-tag, auditable |
| Execution model | Synchronous (blocks on the SSH channel) | Asynchronous (send, then poll for the result) |
| Works on any Linux server, not just EC2? | Yes | No — AWS-specific |

---

**Real-world example — CloudCart**

CloudCart's fleet fully moved off SSH-key-based access years ago, for exactly the reasons in Q9 of the AWS interview — every EC2 instance runs the SSM Agent with an IAM instance profile carrying `AmazonSSMManagedInstanceCore`, and there's no security group anywhere in the account with an inbound rule for port 22. A small internal tool runs `ssm.send_command()` across instances tagged `role=order-processing` to pull `ls -lt /var/log/myapp/` output as a quick sanity check after a deploy, fanning the same command out to a whole tagged fleet at once — something that would otherwise mean a for-loop of individual paramiko SSH connections, each needing its own reachability and key management. The one place `paramiko` still shows up is the legacy on-prem SFTP script from Q3 — because that target genuinely isn't an EC2 instance, so SSM isn't an option there at all; the two tools aren't interchangeable, they solve different environments.

---

**Complete thought process — how I approach this in the interview**

```
Two valid answers, but ONE is specifically better because this is EC2:

paramiko/exec_command — works on ANY SSH-reachable Linux box, AWS
or not, but needs an SSH key, port 22 open, and a bastion for
private subnets — same reachability problem as every other SSH-
based approach
  → Gotcha to mention: must read recv_exit_status() to actually
    know if the command succeeded, exec_command() alone doesn't
    block or give a reliable success signal

boto3 + SSM Run Command — the AWS-native answer, no SSH key, no
open port 22, works through private subnets with no bastion (SSM
Agent connects outbound), access controlled and audited via IAM
  → This is the answer that shows I know this is EC2 specifically,
    not just "a Linux server" — same SSM Agent/Session Manager
    story as AWS Q9
  → Execution model is async: send_command() then poll
    get_command_invocation() for the result
```

---

**Summary (what to say if time is short):**

*"There are two right answers, and I'd give both, but lead with the one specific to this being EC2. The generic answer is paramiko — open an SSH connection, call exec_command with 'ls -lt', then read stdout and specifically call recv_exit_status on the channel to get a real success/failure signal, since exec_command itself doesn't block or guarantee the command finished. But since this is explicitly an AWS EC2 instance, the stronger answer is boto3's SSM Run Command instead — no SSH key, no open port 22, works even through a private subnet with no bastion host, because the SSM Agent on the instance connects outbound rather than needing anything inbound. It's asynchronous, so I'd call send_command, then poll get_command_invocation until the status is Success or Failed, and read StandardOutputContent from the result. The reason I'd lead with SSM for EC2 specifically is the same reasoning behind Session Manager replacing SSH key access across our fleet — it's IAM-governed and centrally logged in CloudTrail, versus SSH access which isn't audited with the same granularity by default."*

---

#### Q5. What is the difference between SSH, SCP, and SFTP?

**Answer:**

The relationship that matters most here: **SCP and SFTP are both file-transfer protocols that run *on top of* SSH** — SSH itself is the secure transport layer underneath both, not a competing alternative to them. The real question is really "what's the difference between the two file-transfer protocols," since SSH isn't in the same category as the other two at all.

---

**SSH — Secure Shell: the transport and remote-access protocol**

SSH's actual job is establishing an **encrypted, authenticated connection** to a remote machine, and giving you an **interactive shell** or the ability to run a single remote command — exactly what Q4's `paramiko.SSHClient().exec_command()` uses. It's the foundation everything else in this answer is built on: authentication (key-based, as covered in Q3/Q4), encryption of everything sent over the connection, and the underlying transport that SCP and SFTP both tunnel through.

```bash
ssh deploy@10.0.1.25          # interactive shell
ssh deploy@10.0.1.25 "ls -lt"  # single remote command, no shell session
```

---

**SCP — Secure Copy: simple, fast, one-directional file transfer**

SCP is a **minimal** file-copy protocol layered on SSH — its entire job is "move this file/directory from A to B," nothing more. It's non-interactive by design: no listing directories, no resuming an interrupted transfer, no renaming/deleting on the remote side — just copy and done.

```bash
scp app.log deploy@10.0.1.25:/var/log/myapp/
scp -r deploy@10.0.1.25:/var/log/myapp/ ./local-backup/   # recursive
```

**Worth knowing as an interview detail:** the original SCP protocol has been effectively **deprecated by OpenSSH** in recent versions in favor of SFTP under the hood, due to some long-standing security/parsing concerns in how the original `scp` protocol handled filenames — modern `scp` clients often actually speak SFTP internally now even though the command is still called `scp`. Worth mentioning to show current awareness, not just textbook knowledge.

---

**SFTP — SSH File Transfer Protocol: a full, interactive file-management protocol**

Despite the similar name, SFTP isn't "SCP with extra steps" — it's a genuinely more capable, **interactive, stateful protocol** for remote file operations, also running over SSH, but supporting real file-management operations, not just copy:

```
sftp> ls              # list remote directory
sftp> cd /var/log      # change remote directory
sftp> get app.log      # download
sftp> put backup.tar   # upload
sftp> rm old.log        # delete a remote file
sftp> mkdir archive      # create a remote directory
```

This is exactly why `paramiko`'s `.open_sftp()` (used throughout Q3) is the natural choice for programmatic file operations — it exposes a real API (`.get()`, `.put()`, `.listdir()`, `.remove()`, `.stat()`) rather than just a single copy action, and unlike SCP, SFTP transfers can be resumed and support seeking within a file.

---

**Side-by-side**

| | SSH | SCP | SFTP |
|---|---|---|---|
| **What it is** | Secure remote access/transport protocol | Simple file-copy protocol, runs over SSH | Full file-management protocol, runs over SSH |
| **Interactive?** | Yes — shell access | No — copy and exit | Yes — browse, list, delete, rename remotely |
| **Resume interrupted transfer?** | N/A | No | Yes |
| **Typical Python library** | `paramiko.SSHClient` (`.exec_command()`) | `scp` package (built on paramiko) | `paramiko.SSHClient().open_sftp()` |
| **Current status** | Standard, unchanged | Effectively deprecated in favor of SFTP under the hood in modern OpenSSH | The modern default for file transfer over SSH |

---

**Real-world example — CloudCart**

In this same interview, both Q3 and Q4 map directly onto this distinction: Q4's `exec_command("ls -lt")` is pure **SSH** — running a remote command, no file transfer involved at all — while Q3's `sftp.get(...)` is **SFTP**, an actual file-management session over that same SSH connection. The legacy on-prem backup script from Q3 specifically used SFTP rather than SCP, even though SCP would have been "simpler" for a single-file pull, because the same script also needed to `sftp.listdir()` the remote directory first to find the correct dated backup file before downloading it — an operation plain SCP has no way to do at all, since it has no concept of browsing, only copying something you already know the exact path to.

---

**Complete thought process — how I approach this in the interview**

```
The key framing: SSH is the transport, SCP and SFTP are both file-
transfer protocols that run ON TOP of it — not three peers, two of
them are built on the first one

SSH  → secure remote shell/command execution — the foundation
SCP  → minimal, one-shot file copy, no browsing/listing/resuming
SFTP → full interactive file-management protocol — list, browse,
       delete, resume — the more capable of the two

Worth mentioning for depth: modern OpenSSH has deprecated the
original SCP protocol due to filename-handling security issues,
and current scp clients often actually speak SFTP under the hood

Ties directly back to my own answers: Q4's exec_command is SSH,
Q3's sftp.get() is SFTP — not SCP, specifically because that script
needed to list/browse the remote directory first, which SCP can't do
```

---

**Summary (what to say if time is short):**

*"SSH is the secure transport and remote-access protocol underneath both of the others — it's what gives you an authenticated, encrypted connection and the ability to run a remote command or shell, which is exactly what I used in the previous question to run ls -lt. SCP and SFTP are both file-transfer protocols that run on top of SSH, not alternatives to it. SCP is minimal — copy a file from A to B and that's it, no browsing, no resuming an interrupted transfer. SFTP is a full, interactive file-management protocol over the same SSH connection — list directories, delete, rename, resume a transfer — which is why paramiko's open_sftp gives you a real API with methods like get, put, and listdir, rather than just a single copy action. Worth mentioning: modern OpenSSH has actually deprecated the original SCP protocol due to some old security issues in how it handled filenames, and current scp clients often speak SFTP under the hood now even though the command is still called scp. In the file-copy question earlier in this interview, I specifically used SFTP rather than SCP because that script needed to list the remote directory to find the right file first — something SCP simply has no way to do."*

---

<!-- Add more scenario questions as Scenario 1, Scenario 2... -->

---

<!--
To add a new interview, copy the block below and paste it at the bottom:

## Interview #3

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
