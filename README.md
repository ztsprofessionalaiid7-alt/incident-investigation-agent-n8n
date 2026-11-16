# Autonomous Security Incident Investigation Agent (n8n)

An n8n workflow where an AI Agent (Gemini) investigates a security incident report: it checks the key indicator with a tool, follows the matching playbook from the database, and writes an incident report with type, priority, likely cause, impact and containment steps. P1 incidents go to a Slack war room.

▶ **Demo video:** https://drive.google.com/file/d/1O5hjQ3RrPyCnBsMY1rNGOzEYy0nmqCiI/view

## How it works

1. Mock incident reports arrive (phishing, encoded PowerShell, public storage bucket)
2. Playbooks are read from the Postgres `playbooks` table and the best match is added to the case by keyword
3. The AI Agent calls the `ioc_check` tool with the key indicator and writes the report as JSON, basing containment on the playbook
4. A Code node validates the output. Priority P1 (active compromise or exposed data) becomes `escalated`, others `logged`
5. Escalated incidents go to Slack, every report is saved to Postgres

```
Manual Trigger -> Mock Incidents -> Get Playbooks (Postgres) -> Build Case
-> AI Agent (Gemini + ioc_check tool) -> Parse Report -> P1 Incident?
   yes -> Slack Alert -> Save Report
   no  -> Save Report
```

## AI Agent

AI Agent node with one tool (`ioc_check`). Containment steps come from the playbook table, so the agent follows your procedure instead of inventing one.

## Tech stack

n8n, AI Agent, Gemini, Slack, Postgres (Supabase)

## Setup

1. Copy the JSON from `workflow.md` into a file named `security-investigation-aiagent-cloud.json` and import it into n8n
2. Create credentials (all have free tiers):
   - Google Gemini (API key)
   - Slack (Bot token and signing secret)
   - Postgres (Supabase Session pooler, SSL: Require)
3. In the Slack node set channel `incident-war-room`, then run `/invite @your-bot` in that channel
4. Create the database tables (below)
5. In the Postgres save node remove the `id` and `created_at` fields, the database fills them
6. Click **Execute workflow**: the mock inputs run through all branches

No other API keys are needed, the tool nodes are self-contained mock tools.

## Database

```sql
create table if not exists public.playbooks (
  id bigint generated always as identity primary key,
  incident_type text,
  keywords text,
  steps text
);

create table if not exists public.incident_reports (
  id bigint generated always as identity primary key,
  incident_id text,
  incident_type text,
  priority text,
  likely_cause text,
  impact text,
  containment text,
  status text,
  created_at timestamptz default now()
);

insert into public.playbooks (incident_type, keywords, steps) values
  ('phishing', 'phishing,verify,login,email,clicked,link', 'Block the sender and URL, reset passwords for users who clicked, search mailboxes for the same message, notify affected staff.'),
  ('malware', 'powershell,malware,edr,encoded,ransomware', 'Isolate the host from the network, capture memory and logs, kill the process, scan other hosts for the same indicator, rebuild if confirmed.'),
  ('data_exposure', 'bucket,public,storage,exposed,exports', 'Remove public access immediately, review access logs for downloads, rotate any exposed keys, assess what data was exposed, notify the data owner.');
```

## Demo data

| Mock input | Expected result |
|---|---|
| INC-501 payroll phishing, 3 staff clicked the link | Phishing playbook, likely P1 or P2 |
| INC-502 encoded PowerShell on file-server-02 at 3 AM | Malware playbook, likely P1 |
| INC-503 storage bucket customer-exports made public | Data exposure playbook, likely P1 |

## Notes

- The sample `playbooks` rows are examples, replace them with your real procedures
- Priority comes from the AI and is validated in code (P1, P2, P3 only, default P2)
- Replace Mock Incidents with a ticket or alert trigger and `ioc_check` with a real reputation API for client use
- Slack message text must be in **Expression** mode so `{{ }}` values are filled in
- Never commit API keys. Use n8n credentials only

## Author

Zarin Tasnim Shaily, AI Automation Engineer
[LinkedIn](https://linkedin.com/in/5ha1l72) | [GitHub](https://github.com/ztsprofessionalaiid7-alt)
