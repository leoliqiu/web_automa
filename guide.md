This page defines how the team measures, tracks, and reports Defect Arrival Rate (DAR) to assess product quality and release readiness.
DAR = Total Valid Defects / Total Test Cases Executed
Only include valid defects (exclude duplicates, rejected)
Use executed test cases, not planned
Target Thresholds (Team Standard)
Range	Status	Interpretation	Action
< 10%	🟢 Very Stable	Possible under-testing	Review coverage
10% – 15%	🟡 Healthy	✅ Target operating range	Continue
15% – 20%	🟠 Warning	Quality risk increasing	Monitor closely
> 20%	🔴 Danger	Unstable build	Investigate / Block release
Team expectation: Operate around ~15% DAR
Above 20% = Danger Zone

Mandatory Fields for Every Defect
All defects must include:
Link to JIRA
Severity
Environment (QA / UAT / Prod)
