# Windows-Event-Analyser
EventLens is a beginner friendly teaching tool I built to show how a security analyst looks at Windows Security logs. You export events from Event Viewer as a CSV, drop the file into the page, and get a plain English triage report that points out activity worth a closer look, explains why, offers a harmless explanation, and suggests what to check .

# EventLens (Windows Security Event Log Triage Tool, Built Offline in the Browser)

`Windows Event Viewer` · `Cline` · `SOC Triage` · `Event IDs` · `Log Correlation` · `Privacy by Design`

## Overview
EventLens is a beginner friendly teaching tool I built to show how a security analyst looks at Windows Security logs. You export events from Event Viewer as a CSV, drop the file into the page, and get a plain English triage report that points out activity worth a closer look, explains why, offers a harmless explanation, and suggests what to check next.

I am not a programmer, so I treated this project the way a product owner would. I wrote a detailed requirements prompt, gave it to an AI coding agent (Cline), and then spent my time testing the result with sample data and with real logs from my own laptop. What I cared about most was not the code. It was the rules the tool had to follow, especially the promise that it never claims a single event proves a system has been compromised.

## Objective
Build a tool that teaches the analyst mindset: look at authentication activity, account changes and privilege changes, connect related events together, and always explain a finding alongside its innocent explanations. Everything had to run locally so nobody ever needs to upload a sensitive log file anywhere.

## Environment
- **Build machine:** Windows laptop, project kept in a local Documents folder
- **Coding agent:** Cline running the DeepSeek v4.1 Flash model, given a requirements prompt of roughly 20,000 characters that I drafted in Notepad
- **The tool itself:** plain HTML, CSS and vanilla JavaScript. No framework, no build step and no backend. I open `index.html` straight from disk in Microsoft Edge
- **Test data:** fictional demo CSVs that ship with the tool, plus real exports from my own Event Viewer

## Tools I Used

| Tool | What It Does | Why I Used It |
|------|--------------|----------------|
| **Windows Event Viewer** | Built in Windows log viewer | Source of the real events. I used `eventvwr.msc`, Filter Current Log and Save All Events As to export data |
| **Cline** | AI coding agent | Wrote the tool from my plain English requirements and explained its decisions to me as it went |
| **Notepad** | Text editor | Where I wrote and refined the requirements prompt |
| **Microsoft Edge** | Browser | Opened the finished page locally to test every feature |
| **Demo CSV samples** | Fictional teaching data | Let me check that each detection fires when it should and stays quiet when it should |

## What I Did

### Writing the Requirements Prompt
1. Described the project as beginner friendly and educational, and told the agent to act as the developer while explaining technical and security decisions to me in plain English.
2. Laid out the workflow I wanted in nine steps, from collecting events in Event Viewer through to a report where every finding explains why it may deserve investigation, lists possible legitimate explanations and suggests next steps.
3. Set one rule in capital letters: the tool must never claim an event proves compromise. It should only flag activity that deserves investigation.
4. Made privacy a hard requirement. The CSV must never be uploaded to a server, sent to an AI provider or analytics platform, stored remotely or sent to an external API. No analytics, tracking, cookies, advertising or telemetry.

### Watching the Build
1. Let Cline work through the prompt and watched it generate realistic fictional sample logs, with made up users, a fictional domain and workstation, and private IP ranges, so the tool could be tested without touching real data.
2. Checked that the samples included the kinds of events a real Security log contains, such as logons (4624) and process creation (4688).

### How the Privacy Promise Is Enforced
I did not want the privacy claim to be just a sentence on a web page, so the design backs it up in three independent layers:
1. A **Content Security Policy** in `index.html` that blocks the browser from making outbound requests of any kind.
2. A **runtime guard** that replaces the browser's network functions with blocking stubs. If anything ever tried to send data, it would be refused and counted in a header badge labelled "Offline check". On my screenshots it reads "passed, no network calls attempted".
3. The code simply never calls those functions in the first place.

Uploaded CSV content is also treated as untrusted. Text from the file is displayed as plain text and never as raw HTML, so a malicious log line cannot run anything. There is even a test sample made of fake log rows packed with script payloads to prove that.

### Testing With the Normal Sample
1. Loaded the normal sample and checked the dashboard. It parsed 12 events and reported 0 High and 0 Medium findings, which is exactly what a quiet day should look like.
2. One failed logon, one deleted account and one installed service showed up, all rated Low, because on their own they are usually ordinary administration.
3. The tool leaves computer accounts and service accounts (names ending in `$`, SYSTEM and similar) out of failed logon correlation and tells the user why, since they are noisy and rarely meaningful alone.
4. It also explains that many logon types are local and legitimately have no source address, so a missing IP is not a red flag.
5. The timeline lists events in time order with the related finding next to each one, so you see the story and not just a pile of rows.

### Testing a Correlated Sample
Loading a sample where an account is created and then added to a privileged group produced a much louder result: 1 account created, 1 privilege change, 2 events requiring investigation and an investigation score of 50. The score is not a probability. The tool shows an itemised breakdown of exactly which behaviours added points, for example a security log being cleared adds the most, so the number is explainable rather than a mystery.

### Understanding the Supported Event IDs
The tool teaches a small set of events well instead of skimming hundreds.

| Event ID | What It Means | Default Severity |
|----------|---------------|------------------|
| 1102 | Security audit log cleared | High |
| 4104 | PowerShell script block logging | Informational |
| 4624 | Successful logon | Informational |
| 4625 | Failed logon | Informational |
| 4688 | Process created | Informational |
| 4720 | User account created | Medium |
| 4726 | User account deleted | Low |
| 4728 | Member added to global security group | Medium |
| 4732 | Member added to local security group | Medium |
| 7045 | Service installed | Low |

Anything outside this list is shown as Unrecognised, kept for context and never graded.

### The Detection Rules
The tool does not judge every event on its own. Where it can, it connects related events and records exactly which ones it used so the reasoning can be audited.
- **Repeated failed logons:** five or more 4625 events for the same account (adjustable, with a default 10 minute window).
- **Failures followed by a success:** a run of failed logons immediately followed by a successful one for the same account. Order matters here.
- **Account created, then privilege added:** a 4720 followed by a 4728 or 4732, raised higher when the group is a well known privileged one.
- **Audit log cleared:** any 1102 is High by design, because it is something to verify, not proof of an attack.
- **Administrative activity summary:** account changes, group changes, service installs and log clearing grouped in one section, with unrelated accounts never linked automatically.

### Testing With My Own Real Export
1. Opened my own Application log in Event Viewer (about 6,500 events) and saved it as `event 1022.evtx`.
2. Tried to load the `.evtx` file straight into the tool. It refused with a clear message that the file does not look like a CSV export and that nothing was uploaded. That was correct behaviour, since raw `.evtx` parsing is not part of version 1.
3. Exported the same log as CSV and loaded it. The tool parsed 5,984 events, skipped 602 rows it could not read, and reported 0 High and 0 Medium findings with an investigation score of 0.
4. Almost everything came back as Unrecognised and the dashboard showed 0 accounts, because I had exported the Application log and not the Security log. The most common IDs were things like 1004, 16384 and 642.
5. The missing accounts also taught me something about Event Viewer itself. Its own CSV export only includes a handful of columns (keywords, date and time, source, Event ID and task category). It leaves out account names and the message text, so correlation by account is impossible from that export. The tool says so openly instead of guessing, and the project documents a PowerShell export that includes the message text for people who want full detail.

## What's in This Repo

```
windows event/
├── index.html                  # The page: privacy notice, upload, settings, education
├── styles.css                  # Dark, accessibility first theme
├── engine.js                   # The brain: CSV parsing, field extraction, rules, scoring
├── script.js                   # The interface: upload, drag and drop, rendering, filters, privacy guard
├── README.md                   # This file
├── build-samples.ps1           # Regenerates the embedded sample data
├── samples/
│   ├── normal-windows-events.csv
│   ├── password-guessing.csv
│   ├── account-persistence.csv
│   ├── suspicious-admin-activity.csv
│   ├── eventviewer-native-export.csv
│   ├── injection-test.csv
│   └── sample-data.js          # Auto generated so the Demo Data buttons work offline
├── tests/
│   ├── selftest.html           # 115 automated checks
│   └── probe.html              # Quick check that the engine loads
└── screenshots/
    ├── 01-requirements-prompt.png
    ├── 02-cline-building-sample-data.png
    ├── 03-dashboard-normal-sample.png
    ├── 04-top-accounts-and-event-ids.png
    ├── 05-timeline-and-admin-activity.png
    ├── 06-supported-event-ids-and-settings.png
    ├── 07-limitations-and-severity-legend.png
    ├── 08-evtx-rejected-message.png
    ├── 09-event-viewer-export-dialog.png
    └── 10-real-application-log-results.png
```

The engine and the interface are kept in separate files on purpose. The engine contains no page code, so the detection logic can be tested on its own and changed without touching the interface. The project includes a self test page with 115 automated checks covering parsing, the detection rules, security behaviour and edge cases such as malformed CSV files and missing columns.

## Skills I Picked Up
- **Writing requirements for an AI agent,** turning a vague idea into clear rules, a defined workflow and firm limits, then checking whether the finished tool followed them.
- **Thinking in terms of triage,** understanding that an event is a clue and not a verdict. One failed logon is boring, a burst of them is interesting, and context decides everything.
- **Knowing the core Windows Security Event IDs,** including logons, failed logons, process creation, account creation and deletion, group membership changes, service installs and audit log clearing.
- **Exporting logs properly from Event Viewer,** including filtering by Event ID, saving as CSV rather than the native `.evtx` format, and understanding what that CSV leaves out.
- **Seeing why correlation matters,** since the same events mean very little alone and a lot when they happen in sequence for one account.
- **Privacy by design,** making local only processing a requirement from day one and building in checks that anyone can verify.
- **Testing with clean and messy data,** using fictional samples to confirm expected results and a real export to see how the tool copes with data it was never built for.

## How This Applies in the Real World
In a real SOC, analysts rarely see one dramatic event. They see thousands of ordinary ones and have to spot the few that deserve attention. Failed logons followed by a success, a new account that quickly joins an administrators group, a service that appears out of nowhere, or the security log being wiped are all patterns analysts learn to recognise, and EventLens walks a beginner through exactly that thinking.

The false positive side matters just as much. Every finding in the tool comes with a possible harmless explanation, because a typo, a stale saved password, a new starter or routine maintenance can all look alarming out of context. Learning to hold both ideas at once is a big part of the job.

The privacy design reflects something real too. Log files contain usernames, hostnames and IP addresses, so being able to analyse them without sending them anywhere is a genuine consideration and not a nice extra.

## Where I'm Coming From
I'm making the jump into cybersecurity from a background in **healthcare**. A lot of that experience carries over, such as following procedure carefully, protecting sensitive information and staying calm and methodical when something is not behaving the way it should. I am currently studying for **CompTIA Security+** and building projects like this one to get real hands on practice, since that is what I am missing on paper compared to my experience.

I'll be upfront about two things. I am not a programmer and I built this by directing an AI agent, so I am showing how I specified and tested it and not claiming I hand wrote the code. I also exported the wrong log on my first real test, and I left that in this write up because it taught me something useful about reading what a tool is actually telling me.

## What I Want to Learn Next
- Testing the tool with a proper export of the **Security** log using the PowerShell method, since that is the one it was built for
- Running the self test page myself and reading what each group of checks covers
- Raw `.evtx` parsing so the CSV step is no longer needed
- Sysmon support and deeper PowerShell 4104 analysis
- Mapping findings to MITRE ATT&CK techniques
- Exporting the triage report as Markdown or PDF so it can feed into an incident write up

## Limitations & What I'd Do Differently in Production
- **This is static, single file analysis with no baseline.** It cannot know what is normal for a given environment, so it is a learning tool and does not replace an EDR, a SIEM or a professional SOC platform.
- **Event IDs describe activity, never intent.** A finding means worth checking, not malicious. Only context can answer that.
- **What gets logged depends on audit policy.** A missing event is not proof that nothing happened.
- **Event Viewer's own CSV export is thin.** Without account names and message text, correlation is impossible, which is exactly what my real test showed.
- **Date handling is a weak spot.** Exports differ by language and region, and day and month order can be ambiguous. On my real export the reported time span ran into December 2026, which is later than the day I tested, so I suspect some dates were read the wrong way round. In production I would confirm the date format before trusting any ordering.
- **My real test used the wrong log type,** so it shows the tool fails gracefully on unfamiliar data but says nothing yet about how it performs on a real Security log.
- **Some counts need a second look.** The Application export showed six failed logons even though that is not an authentication log. I would check whether another provider reuses the same Event ID before trusting that number.
- **I have not independently audited the code.** I trust the tool as far as my testing and the automated checks go, and a production tool would need a proper review.
- **Only a small set of Event IDs is supported.** Everything else is shown for context and never graded.

## References
- [Microsoft Security Auditing Overview](https://learn.microsoft.com/en-us/windows/security/threat-protection/auditing/security-auditing-overview)
- [Windows Security Log Encyclopedia](https://www.ultimatewindowssecurity.com/securitylog/encyclopedia/)
- [MITRE ATT&CK](https://attack.mitre.org/)
- [CompTIA Security+ (SY0-701) Exam Objectives](https://www.comptia.org/certifications/security)
- Cline, the AI coding agent used to build the tool
