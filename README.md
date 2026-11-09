# Assigned-TUE-11-03-26---Due-MON-11-09-26

# SEIR Foundations — Module 1, Lesson 2: Balerica AI Slack Alerts

**Student:** Cassiem Davids
**Project path (EC2):** `/opt/balerica-bank`
**Slack workspace:** SEIR-Foundations-kazapes
**Alert channel:** `#balerica-alerts`

---

## 1. Lab Setup Summary

| Setting | Value | Secret? |
|---|---|---|
| Workspace URL | `https://<my-workspace>.slack.com` (set in `slack.py`) | No |
| Alert channel | `#balerica-alerts` | No |
| Slack app | Balerica AI | No |
| Incoming Webhook URL | Stored **only** in password manager + `.env` | **YES** |
| Alert threshold | `INFO` for homework (`ERROR` for production) | No |

`.env` lives next to `slack.py`, is `chmod 600`, and is listed in `.gitignore`:

```dotenv
SLACK_WEBHOOK_URL=<real webhook — never shared>
SLACK_ALERTS_ENABLED=true
SLACK_ALERT_SEVERITY=INFO
SLACK_ALERT_CHANNEL=balerica-alerts
```

Test: `python slack.py --test` → `SUCCESS: Chewbacca approves!`

---

## 2. Troubleshooting Lessons Learned

### "FAILED: Slack did not confirm delivery"
- **Cause:** `.env` still had the placeholder `PASTE-YOUR-REAL-WEBHOOK-HERE`.
- **Clues:** file size was exactly 169 bytes (the template's size), and the timestamp didn't change after `nano` — I never saved.
- **Lesson:** `grep '^SLACK_WEBHOOK_URL=https://hooks.slack.com/services/'` passes for the placeholder too. A better check:
  ```bash
  grep -c 'PASTE-YOUR-REAL' .env                                        # should be 0
  grep -c '^SLACK_WEBHOOK_URL=https://hooks.slack.com/services/T' .env  # should be 1
  ```

### "FAILED: Configure SLACK_WEBHOOK_URL in .env"
- **Cause:** I put the URL in the `read -p "..."` prompt text instead of typing it as input, and pasted several lines at once, so `sed` wrote an **empty** value.
- **Lesson:** paste secrets into `nano` (then `Ctrl+O`, `Enter`, `Ctrl+X` to save) — simplest and least error-prone.

### Webhook leaked
- **Lesson:** a webhook URL is a password. If it's posted anywhere (chat, screenshot, GitHub), **revoke it** in Slack → Incoming Webhooks and create a new one.

### Direct webhook test (bypasses `slack.py`)
```bash
set -a; source .env; set +a
curl -s -X POST -H 'Content-type: application/json' --data '{"text":"curl test"}' "$SLACK_WEBHOOK_URL"; echo
```
- `ok` → webhook works
- `invalid_token` / `no_service` → wrong or revoked URL
- timeout → no outbound HTTPS (port 443)

### Pasting Python into bash
- **Error:** `-bash: from: command not found`, `syntax error near unexpected token '('`
- **Lesson:** at the `$` prompt, type **commands**. Python code goes **in a file**, then run `python file.py`.

### Python syntax fixes (HW1)
- Strings need quotes: `MY_NAME = "Cassiem Davids"`
- f-strings use the **variable name**: `f"Hello from {MY_NAME}!"` — not `{Cassiem_Davids}`

### Changing the threshold quickly
```bash
sed -i 's/^SLACK_ALERT_SEVERITY=.*/SLACK_ALERT_SEVERITY=INFO/' .env
grep '^SLACK_ALERT_SEVERITY' .env
```

---

## 3. Homework Notes

### HW1 — Hello Slack (`hw1_hello_slack.py`)
- **Practiced:** calling a function with arguments, f-strings.
- **Edit made:** `MY_NAME = "Cassiem Davids"`
- Needs `SLACK_ALERT_SEVERITY=INFO`, since the alert is INFO.

### HW2 — Severity Tour (`hw2_severity_tour.py`)
- **Practiced:** `for` loops, severity filtering.
- **Results:**

| Severity | Threshold = INFO | Threshold = ERROR |
|---|---|---|
| INFO | Delivered | Skipped |
| WARNING | Delivered | Skipped |
| ERROR | Delivered | Delivered |
| CRITICAL | Delivered | Delivered |

- **Why:** severities are ranked INFO < WARNING < ERROR < CRITICAL. The threshold is a **minimum** — alerts at or above it are sent; anything below is skipped *before* reaching Slack.

### HW3 — Server Time (`hw3_server_time.py`)
- **Practiced:** `datetime`, `strftime`, UTC.
- **Observation:** EC2 runs on **UTC**. I'm on US Eastern — EST (UTC−5) since DST ended Nov 1, so the server is **5 hours ahead** (4 hours in summer under EDT). My Saturday 9:39 PM was the server's Sunday 2:39 AM.
- **Why UTC:** logs from servers in different regions line up without time-zone math. Convert alert timestamps to local time during incidents.

### HW4 — Disk Watch (`hw4_disk_watch.py`)
- **Practiced:** `shutil.disk_usage`, byte math, `if/elif/else`.
- **Logic:** ≥ 90% → ERROR, ≥ 75% → WARNING, else INFO. Order matters: `if/elif` stops at the first match, so the highest limit is checked first.
- **Why a quiet INFO is good design:** the check still ran; it decided things are healthy. Paging on every healthy check causes **alert fatigue** — people start ignoring the channel and miss real problems. Silence should mean "checked and fine"; a message should mean "act now."

### HW5 — Load Check (`hw5_load_check.py`)
- **Practiced:** `os.getloadavg()`, tuple unpacking, comparisons.
- **Concept:** load ≈ how many CPUs' worth of work is queued. Load above CPU count = overloaded.
- Uses the **5-minute** average to ignore short spikes.
- **What could spike load on Balerica bank:**
  - Traffic surges (payday, month-end)
  - Batch jobs (end-of-day reconciliation, interest, reports)
  - Slow database queries piling up
  - Runaway process / infinite loop
  - Attacks: brute-force logins, DDoS, cryptomining malware
- **Security note:** an unexplained, sustained spike on a quiet server is a red flag.
- **Test a WARNING:** `yes > /dev/null &` (×2), wait ~5 min, rerun, then `pkill yes`.

### HW6 — Incident Drill (`hw6_incident_drill.py`)
- **Practiced:** lists of tuples, `random.choice`, `random.randint`, building incident IDs.
- **Rule:** never send real customer data to Slack — drills use fake data.
- **Which incidents deserve a 3 AM wake-up:**

| Incident | Wake someone? | Why |
|---|---|---|
| CRITICAL: ledger file not reachable | **Yes, now** | No balances or transactions — bank effectively down |
| ERROR: `/payments` returning 500 | **Yes** | Customers can't pay |
| ERROR: rejection rate > 5% | **Yes, probably** | Payments failing at scale (bad deploy or feed?) |
| WARNING: SWIFT queue at 42 | Not yet | Watch; page if it keeps climbing |
| WARNING: batch job 12 min late | Not yet | Escalate if it threatens the morning deadline |
| INFO: backup completed | No | Belongs in the daily report |

- **Rule of thumb:** wake a human when **customers or money are affected right now**. That's the line `SLACK_ALERT_SEVERITY=ERROR` draws.

### HW7 — Morning Report (`hw7_morning_report.py`)
- **Practiced:** combining datetime, disk, load, and `platform` into one message.
- `load_1, _, _ = os.getloadavg()` — `_` means "value not needed."
- Adjacent strings inside `( )` join into one string; `\n` makes new lines in Slack.
- **Screenshot:** safe — the message shows server stats only, no secrets.
- **Next step — schedule it daily** (8:00 AM Eastern = 13:00 UTC in winter):
  ```bash
  crontab -e
  0 13 * * * cd /opt/balerica-bank && /opt/balerica-bank/venv/bin/python hw7_morning_report.py
  ```

---

## 4. Key Takeaways

1. **Secrets stay in `.env`** — `chmod 600`, in `.gitignore`, never pasted anywhere. Leaked = rotate.
2. **Verify, don't assume** — check file size/timestamp and grep for placeholders after editing.
3. **Save in nano:** `Ctrl+O`, `Enter`, `Ctrl+X`.
4. **Shell vs. Python:** commands at the `$` prompt, code in files.
5. **Monitoring pattern:** measure → decide severity → alert.
6. **Thresholds prevent alert fatigue:** INFO for learning, ERROR for production.
7. **Servers speak UTC.**

**Next lesson:** call `send_alert()` from monitoring agents like `health.py` and `analyze_logs.py`.