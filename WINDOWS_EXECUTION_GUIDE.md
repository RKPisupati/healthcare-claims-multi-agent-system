# Windows Community Edition — End-to-End Execution Guide

## What changed in the workflow design (and why)

| Change | Reason |
|---|---|
| **`Read Claims CSV`** node now points to `C:\Users\Admin\.n8n-files\sample_claims.csv` | Newer n8n versions sandbox the `Read/Write Files from Disk` node — by default it can **only** access a special `.n8n-files` folder inside your Windows user profile, regardless of what path you type. Trying to point it at `D:\Hackathon\Files\...` fails with "Access to the file is not allowed" even if the file genuinely exists there. Pointing the node at the allowed folder instead is the fastest, most reliable fix for a hackathon deadline. |
| **`Parse CSV to Items`** node converts the file into individual claim records | Each row becomes one n8n item that the loop processes one at a time. |
| **Fixed**: loop-back wiring | `Notification Agent` feeds back into `Loop Over Claims`'s single input (not a separate output index) — this is n8n's documented pattern: loop output → processing chain → back into the loop node, which auto-advances to the next item until the batch is exhausted. |

---

## Part 1 — Place the CSV file in n8n's allowed folder

n8n restricts file access to a special folder for security. You have two options:

### Option A (recommended — fastest, guaranteed to work)
1. Open File Explorer, navigate to `C:\Users\Admin\`
2. Look for a folder named `.n8n-files` (it may be hidden — enable **View → Show → Hidden items** if you don't see it). If it doesn't exist yet, create it manually.
3. Copy the provided `sample_claims.csv` into that folder, so the final path is exactly:
   ```
   C:\Users\Admin\.n8n-files\sample_claims.csv
   ```
4. This matches exactly what the workflow's `Read Claims CSV` node is already configured to look for — no further changes needed.

### Option B (if you specifically want to use D:\Hackathon\Files instead)
You can widen n8n's allowed paths via an environment variable, but this is less reliable on Windows npm installs (community reports show it's inconsistent depending on n8n version) and not worth the risk this close to a deadline. If you want to try it anyway:
1. Stop n8n if it's running (Ctrl+C in the terminal)
2. Set the environment variable before starting n8n:
   ```powershell
   $env:N8N_RESTRICT_FILE_ACCESS_TO = "D:\Hackathon\Files"
   n8n start
   ```
3. If this doesn't work, fall back to Option A — it's guaranteed to work since `.n8n-files` is n8n's own default allowed path.

**For your demo, just use Option A.** It's one folder + one file copy, zero configuration risk.

## Part 2 — Install & Start n8n on Windows (if not already running)

You said it's already running, so you can skip ahead to Part 3. For reference, here's how to start it if needed:

```powershell
npm install n8n -g
n8n start
```
Then open **http://localhost:5678** in your browser.

---

## Part 3 — Get your free Groq API key (2 min)
1. Go to https://console.groq.com → sign up (free, no card required)
2. Left sidebar → **API Keys** → **Create API Key**
3. Copy the key (starts with `gsk_...`) — you won't see it again, so paste it somewhere safe temporarily

---

## Part 4 — Set up credentials in n8n

### Groq credential
1. In n8n: left sidebar → **Credentials** → **Add Credential**
2. Search **"Groq"**
   - **If it exists natively**: paste your API key, save, name it e.g. `Groq account`
   - **If it does NOT exist** (depends on your n8n version): instead create a **Header Auth** credential:
     - Name: `Authorization`
     - Value: `Bearer gsk_your_actual_key_here`
     - Save it as e.g. `Groq Header Auth`
   - You will select this credential on **5 nodes**: Validation Agent, Policy Agent, Fraud Agent, Knowledge Agent, Decision Agent — all already set to use credential type `groqApi` in the imported JSON. If you had to use Header Auth instead, you'll need to change each of those 5 nodes' Authentication dropdown to "Header Auth" and pick your credential (4–5 clicks, one-time).

### SMTP (email) credential
1. **Credentials** → **Add Credential** → search **"SMTP"**
2. If using Gmail:
   - Host: `smtp.gmail.com`
   - Port: `465`
   - SSL/TLS: **On**
   - User: your full Gmail address
   - Password: a **Gmail App Password**, NOT your normal password
     - Generate one at https://myaccount.google.com/apppasswords (requires 2-Step Verification enabled on your Google account)
3. Save it, name it e.g. `Gmail SMTP`

> **Windows note**: corporate/college Wi-Fi sometimes blocks outbound port 465/587. If the email node times out during testing, try a personal hotspot, or switch port to `587` with STARTTLS instead of SSL.

---

## Part 5 — Import the workflow

1. Open `healthcare_claims_batch.json` in a text editor (Notepad is fine), **Ctrl+A → Ctrl+C** to copy all of it
2. In n8n, click **Add Workflow** (or open a blank canvas)
3. Click on the empty canvas area to focus it
4. Press **Ctrl+V** — n8n detects the JSON and imports all 14 nodes, fully wired

If paste doesn't work, check the **"⋯"** (three-dot) menu in the top-right of the canvas for **Import from File**, and select `healthcare_claims_batch.json` directly.

---

## Part 6 — Wire up credentials on the imported nodes

1. Click each of these 5 nodes and set **Authentication → your Groq credential**:
   - Validation Agent, Policy Agent, Fraud Agent, Knowledge Agent, Decision Agent
2. Click **Notification Agent** → under **Credentials**, select your SMTP credential
3. Also on **Notification Agent**, update the `fromEmail` field (currently `claims-system@yourdomain.com`) to your actual Gmail address — Gmail will reject sends where the "From" doesn't match the authenticated account
4. Click **Save** (top right, or Ctrl+S)

---

## Part 7 — Run it end to end

1. Click **Run Batch Job** (the Manual Trigger node) — or just hit **"Execute Workflow"** at the bottom of the canvas
2. Watch the canvas: `Read Claims CSV` reads the file → `Parse CSV to Items` turns it into 6 separate claim records → `Loop Over Claims` pulls claim #1 → 4 agent nodes light up **in parallel** → they merge → `Decision Agent` → `Format Final Result` → `Notification Agent` sends an email → flow loops back automatically to claim #2, and so on through all 6
3. Each full pass through the loop takes ~10–20 seconds (5 Groq calls per claim), so the whole batch of 6 finishes in roughly 1–2 minutes
4. You'll receive **6 separate decision emails** in your inbox, one per claim

---

## Part 8 — Verify results

Click on the **Format Final Result** node after the run finishes → open its **Output** panel. Since this node ran once per claim, click through each execution (use the input/output item selector) to see:
- `decision` (APPROVE / REJECT / MANUAL_REVIEW)
- `reasoning` (the Decision Agent's explanation)
- `match_expected` (true/false — confirms it matched our predicted outcome)

Expected results:
| Claim | Expected |
|---|---|
| CLM-2001 (Sarah Mitchell) | APPROVE |
| CLM-2002 (Robert Chen) | APPROVE |
| CLM-2003 (Amanda Torres) | REJECT |
| CLM-2004 (Mike Donovan) | REJECT |
| CLM-2005 (Lisa Park) | MANUAL_REVIEW |
| CLM-2006 (David Kim) | MANUAL_REVIEW |

LLM outputs aren't 100% deterministic, so an occasional borderline claim landing differently is normal — and a good live talking point ("here's why the agent reasoned this way," opening that node's JSON output).

---

## Quick Troubleshooting

| Symptom | Fix |
|---|---|
| `Read Claims CSV` errors "Access to the file is not allowed" | This is n8n's file-access sandbox. Confirm the file is at `C:\Users\Admin\.n8n-files\sample_claims.csv` — n8n only allows paths under `.n8n-files` by default in recent versions. |
| `Read Claims CSV` errors "file not found" or "ENOENT" | Confirm `sample_claims.csv` actually exists inside `C:\Users\Admin\.n8n-files\` (check for hidden `.txt` extension, typos in filename) |
| Groq node returns 401 Unauthorized | API key wrong/missing — recheck credential, make sure `Bearer ` prefix is included if using Header Auth |
| Email node hangs / times out | Try port 587 instead of 465, or switch networks (Wi-Fi may block SMTP ports) |
| `JSON.parse` error in Format Final Result | Occasionally Groq returns malformed JSON — rerun that one claim; `response_format: json_object` minimizes this but doesn't eliminate it entirely |
| Loop only processes 1 item then stops | Re-check the connection from `Notification Agent` goes back into `Loop Over Claims`'s input (not a different node) — this is the exact bug we fixed in this version |
| Paste (Ctrl+V) doesn't import workflow | Use the **⋯ menu → Import from File** instead, browse to `healthcare_claims_batch.json` |

---

## Demo Day Tips
- Run it live — the parallel agent fan-out per claim is the most visually impressive part and directly matches your architecture diagram.
- Have the **Format Final Result** output panel pre-opened in a second browser tab so you can flip to it instantly after the run finishes, showing all 6 decisions + reasoning side by side.
- If a judge asks about scale: point out that swapping the CSV for a Google Sheets/database/API source is a 2-minute change — the entire agent pipeline downstream is untouched.
