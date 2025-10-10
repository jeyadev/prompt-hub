Here’s how you can structure the same content as a Confluence page — clean, scannable, and in your typical SRE documentation tone. You can paste this directly into Confluence’s editor (use headings, info panels, and code blocks for readability):


---

🧭 Repository Size Optimization — Guidelines & Action Plan

Owner: Site Reliability Engineering
Last Updated: {{Insert Date}}
Scope: Application Teams — Repository Size Audit


---

📈 Overview

Recent analysis of our two main Bitbucket repositories shows that the majority of storage is consumed by Git history rather than code.
The .git directory contributes to 90–99% of total repository size, causing slower clone/fetch times, increased CI checkout durations, and unnecessary Bitbucket storage costs.


---

🔍 Root Causes

Issue	Description	Impact

Excessive Branch Count	Over 1,000 branches, including merged and abandoned ones, retained in Git history.	Large .git pack files and slow fetches.
High Tag Volume	Hundreds of tags maintained across releases.	Inflated metadata storage.
Deep Commit History	Long-lived branches with frequent merges.	Heavy repository graph traversal.
Cross References	Git maintains links across all tags and branches.	Larger index and slow operations.



---

🧩 Space Utilization Snapshot

.git folder: ~91–99% of total size

Working directory (actual source): <10%



---

⚙️ Immediate Cleanup Actions

Perform the following safe cleanup operations to reduce repo size:

1. Prune stale remote branches

git fetch --prune
git branch -r | grep 'origin/' | grep -vE 'main|develop|release' | xargs -n 1 git push origin --delete

2. Remove unused tags

git tag -l | xargs git tag -d && git fetch origin --prune --tags

3. Optimize repository

git gc --aggressive --prune=now

> 💡 Tip: Always perform these steps on a fresh clone or a non-production branch first.




---

🧠 Long-Term Best Practices

Category	Recommendation	Implementation

Branch Hygiene	Delete merged branches automatically post-release.	Configure a Bitbucket pipeline or scheduled cleanup script.
Retention Policy	Remove inactive branches >90 days old.	Use Bitbucket REST API cleanup job.
Shallow Clones	Use shallow clones (--depth=50) for local development.	Recommended for all developer machines.
Git LFS	Move binaries, media, or large files to LFS.	Enable at repo level under Settings → Git LFS.
Repository Segmentation	Split large mono-repos into smaller service-specific ones.	Evaluate based on service boundaries.



---

📊 Expected Impact

Metric	Before	After Cleanup	Improvement

Repo Size	~2 GB	~300–400 MB	↓ 80%
Clone Time	2–3 mins	<30 sec	↓ 70–80%
CI Checkout	2–4 mins	~45 sec	↓ 75%



---

🚀 Next Steps

1. SRE Team to run one-time cleanup with application teams.


2. Teams to confirm branches or tags needed for audit/release traceability.


3. Policy rollout: Branch retention and auto-pruning scripts integrated into Bitbucket pipelines.




---

📘 References

Bitbucket: Repository Size Management Guide

Git: Maintenance and Garbage Collection

Atlassian REST API — Branch Cleanup



---

✅ Summary

Reducing Git history bloat improves:

Developer productivity (faster fetch/pull/checkout).

CI/CD pipeline efficiency.

Repository health and maintainability.


A branch and tag hygiene policy will now be standard practice across all Bitbucket projects.


---

Would you like me to append a Bitbucket Pipeline YAML snippet at the bottom that automates stale-branch cleanup weekly (runs via cron or scheduled pipeline)? It would make this page actionable.

