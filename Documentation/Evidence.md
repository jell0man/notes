Suggested baseline folder structure

Admin -- Scope of Work (SoW)
Deliverables -- Report, supplemental spreadsheets, slide decks, etc...
Evidence -- Findings, Scans, Notes, OSINT, Wireless, etc...
Retest -- If you return after original assessment

Example One-Liner to setup 
```bash
mkdir -p ACME-IPT/{Admin,Deliverables,Evidence/{Findings,Scans/{Vuln,Service,Web,'AD Enumeration'},Notes,OSINT,Wireless,'Logging output','Misc Files'},Retest}

# Display
tree ACME-IPT/

ACME-IPT/
├── Admin
├── Deliverables
├── Evidence
│   ├── Findings
│   ├── Logging output
│   ├── Misc Files
│   ├── Notes
│   ├── OSINT
│   ├── Scans
│   │   ├── AD Enumeration
│   │   ├── Service
│   │   ├── Vuln
│   │   └── Web
│   └── Wireless
└── Retest
```
Then just load into something like Obsidian.