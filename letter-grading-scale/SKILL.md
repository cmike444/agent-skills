---
name: letter-grading-scale
description: >
  Converts any percentage score (0–100%) to a letter grade using the standard US college
  A+/A/A− through F scale, with GPA equivalents. Use this whenever a percentage needs to be
  expressed as a letter grade, whether for trade setups, checklists, evaluations, or any
  scored assessment. Triggers on "what grade is", "convert to grade", "score to letter", or
  when any skill needs to map a percentage to a letter.
---

# Grade Scale

Source: commonly used US college grading system
([Wikipedia – Academic grading in the United States](https://en.wikipedia.org/wiki/Academic_grading_in_the_United_States))

| Score      | Grade | GPA |
|------------|-------|-----|
| 97–100%    | A+    | 4.0 |
| 93–96%     | A     | 3.9 |
| 90–92%     | A−    | 3.7 |
| 87–89%     | B+    | 3.3 |
| 83–86%     | B     | 3.0 |
| 80–82%     | B−    | 2.7 |
| 77–79%     | C+    | 2.3 |
| 73–76%     | C     | 2.0 |
| 70–72%     | C−    | 1.7 |
| 67–69%     | D+    | 1.3 |
| 63–66%     | D     | 1.0 |
| 60–62%     | D−    | 0.7 |
| 0–59%      | F     | 0.0 |

Look up the score in the table and return the corresponding grade and GPA. Show the score,
grade, and GPA together on one line: **XX% → B+ (3.3)**