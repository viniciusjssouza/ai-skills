---
name: pr-description
description: generates PR description following the company standard template.
---

Generate a GitHub PR description, using markdown format.
The template that must be used is the following. Output the test with markdown characters so I can copy it and past into the 
Github text area. You MUST output markdown symbols like "*" for bold text and "- [x]" for check lists. Escape it if necessary to output it 
in the Claude Code CLI interface.
Be clean but concise on the description - engineers don't want to read large pieces of text.

```
### Description:
<!--
One or two line summary of what this PR does and why it is needed, followed by a list
of changes in imperative, present tense for use in the commit message or changelog. Example:

Add support for ...

* Add config property
* Change column name
* Remove ...
-->

### Related issue(s):

Fixes #

### Notes for reviewer:
<!-- Provide logs, performance numbers or screenshots of the new functionality -->

### Checklist:

- [ ] Documented (Code comments, README, etc.)
- [ ] Tested (unit, integration, etc.)

```

To get the changes, compare current branch with head of main branch.
The commit messages are also a good source of information for the description.
Be concise to facilitate the reviewer to understand the context in a few words.

As a last line of your suggestion, add also a suggestion for the title, following the conventional commits standard.

For code references, use the code markdown format like "`some code here`".
End all the list items in the summary with a ';' character.
