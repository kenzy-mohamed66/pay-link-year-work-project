# Prompt Log

| Date | Author | Stage | Tool and model | Prompt | Summary of answer | Verdict (accepted, fixed, rejected) and why |
|---|---|---|---|---|---|---|

| 2026-11-25 | Mariam (Member 1) | Requirements | Claude | Role: reviewer.<br>Context: our 41 user stories (US-1 to US-41) and the brief's FR priorities.<br>Task: check the stories against the brief and convert them to docs/requirements.md.<br>Constraints: keep our wording, do not add new stories, report every mismatch.<br>Output: markdown file plus a list of problems. | A markdown file and a list of mismatches. | Accepted after checking: the AI found that FR-17 and FR-19 were marked Must but the brief says Should, a stray sentence in US-34, and missing "As a" lines in US-35 and US-41. We fixed them in the Word file. |
