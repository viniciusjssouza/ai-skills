---
name: pr-briefing
description: collect all the required information to start a review of a Pull-Request / merge-request, compiling it in a easily digestable form. 
---

Collect all the information from the provided pull request using github CLI or any CLI available. Summarize the changes, providing what has changed and the "why". Check the commits and comments on the PR to look for useful information for a reviewer. 

Recommend a sequence/reading order for the changes, aiming an easy understanding of the scope.
Recommend the top files to look for or files I should skip because they are unrelevant.


Output the summary in an easy, well-structured and digestable form. Feel free to skip/omit unrelevant changes. Focus on what is important.


  Classify each PR across four dimensions. The **overall risk** is the highest single-dimension rating.

  ---

  ## Risk Levels

  | Level | Label        | Meaning                                                                          |
  |-------|--------------|----------------------------------------------------------------------------------|
  | 4     | **Critical** | Can take nodes offline, cause ISS, or introduce an exploitable vulnerability     |
  | 3     | **High**     | Can degrade throughput, cause silent data loss, or allow a security bypass       |
  | 2     | **Medium**   | Can cause incorrect behavior in edge cases, operational confusion, or partial failures |
  | 1     | **Low**      | Style, naming, documentation, or dead code only — no runtime impact              |

  ---

Evaluate dimensions like availability, security, reliability (correctness).

  ## Summary Classification Template

  Risk Classification

  ┌──────────────┬────────┬───────────┐
  │  Dimension   │ Rating │ Signal(s) │
  ├──────────────┼────────┼───────────┤
  │ Determinism  │ [1–4]  │ ...       │
  ├──────────────┼────────┼───────────┤
  │ Availability │ [1–4]  │ ...       │
  ├──────────────┼────────┼───────────┤
  │ Security     │ [1–4]  │ ...       │
  ├──────────────┼────────┼───────────┤
  │ Correctness  │ [1–4]  │ ...       │
  └──────────────┴────────┴───────────┘

   Overall: [Critical / High / Medium / Low]
