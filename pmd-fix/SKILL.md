---
name: pmd-fix
description: run pmd command and fix code issues found by it.
---

Run [PMD](https://pmd.github.io/) analyzer and fix the issues. You don't have to fix all of them. Feel free to ask about them. The pmd command is installed. 
Just fix issues related to changes applied in this branch. To get the changes, compare the current branch with main HEAD. 

Skip rules that doesn't make sense, like:

 - adding conditionals do log statements for info, warn and error.
 - using concurrenthashmap outside concurrent code.
 - add /* default */ to default methods or constructors.

Look for the PMD's ruleset.xml file inside the project folder.
