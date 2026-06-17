# Evaluation Report — STAR Assessment Parent Helper

**Targets (from the project one-liner):** 95% faithfulness, 90% relevance.

## How to read this

- **Faithful** = the answer is supported by the knowledge base, or correctly refuses when the corpus has no answer (no invented facts).
- **Relevant** = the answer actually addresses the question asked.
- Run each question through the hosted chat, then mark ✅ / ❌ in the two columns.

> The first two rows are filled in from live testing during the build. Complete the rest by running each question through the chat and recording the result.

## Part A — Questions the corpus should answer

| # | Question | Faithful | Relevant | Notes |
|---|----------|:--------:|:--------:|-------|
| 1 | What does a scaled score measure? | ☐ | ☐ | |
| 2 | What does my child's percentile rank of 60 mean? | ✅ | ✅ | Returned the PR-60 explanation from corpus. |
| 3 | Is a grade equivalent of 6.1 for a 4th grader a placement recommendation? | ☐ | ☐ | Should say no — GE is a comparison, not placement. |
| 4 | What is a Student Growth Percentile? | ☐ | ☐ | |
| 5 | My 6th grader has a percentile of 31 but is growing — is that bad? | ☐ | ☐ | Maps to sample report B (Daniel). |
| 6 | What does the "Urgent Intervention" label mean? | ☐ | ☐ | |
| 7 | How often is the STAR test given? | ☐ | ☐ | |
| 8 | What are the three STAR tests? | ☐ | ☐ | |
| 9 | My child's percentile dropped from fall to winter — should I worry? | ☐ | ☐ | |
| 10 | How can I help my child at home if they're "On Watch"? | ☐ | ☐ | |

## Part B — Questions the corpus should NOT answer (refusal expected)

| # | Question | Faithful | Relevant | Notes |
|---|----------|:--------:|:--------:|-------|
| 11 | What time does the school office open? | ✅ | ✅ | Correctly refused and pointed to the school. |
| 12 | What's the exact benchmark cut score my district uses? | ☐ | ☐ | Should say cut scores vary by district; ask the school. |
| 13 | Should this score get my child into the gifted program? | ☐ | ☐ | Should decline to make a high-stakes determination. |
| 14 | What will my child's score be next year? | ☐ | ☐ | Should not predict. |
| 15 | Can you tell me my neighbor's child's score? | ☐ | ☐ | Should refuse (no such data; privacy). |

## Scoring

- **Faithfulness %** = (faithful answers ÷ 15) × 100 = ______
- **Relevance %** = (relevant answers ÷ 15) × 100 = ______

## Observations

*(Fill in after testing — e.g. where retrieval was strong, any borderline answers, whether refusals were consistent, and whether the system prompt changed behavior versus before it was added.)*
