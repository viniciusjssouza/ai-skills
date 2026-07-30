---
name: pr-review
description: review Github PRs
---

You are a expert software engineer. Reviewing the PR $0. Use the Github CLI to get the PR changes and details. Print your findings here. Don't reply to the PR yet.

Follow the checklist below:

## Implementation

 - Does this code change do what it is supposed to do?
 - Can the solution be simplified?
 - Does this change add unwanted compile-time or run-time dependencies?
 - Is a framework, API, library, or service used that should not be used?
 - Could an additional framework, API, library, or service improve the solution?
 - Is the code at the right abstraction level?
 - Is the code modular enough?
 - Can a better solution be found in terms of maintainability, readability, performance, or security?
 - Does similar functionality already exist in the codebase? If yes, why isn’t it reused?
 - Are there any best practices, design patterns, or language-specific patterns that could substantially improve this code?

## Logic Errors and Bugs
 - Can you think of any use case in which the code does not behave as intended?
 - Could boundary values, missing or invalid input, concurrency, retries, timeouts, or partial failures break the code?

## Error Handling and Logging
 - Is error handling done the correct way?
 - Should any logging or debugging information be added or removed?
 - Are error messages user-friendly?
 - Are there enough log events, and are they written in a way that allows for easy debugging?
 - Do logs and error messages avoid exposing secrets or unnecessary personal data?
 
## Usability and Accessibility
 - Is the proposed solution well-designed from a usability perspective?
 - Is the API well documented?
 - Is the proposed solution (UI) accessible?
 - Is the API/UI intuitive to use?

## Testing and Testability
 - Is the code testable?
 - Have automated tests been added, or have related ones been updated to cover the change in functionality?
 - Do the existing tests reasonably cover the code change (unit/integration/system tests)?
 - Are there some test cases, input, or edge cases that should be tested in addition?
 - Do new or changed tests fail when the implementation is absent or intentionally broken?
 - For probabilistic or LLM-powered behavior, do evaluations cover representative and adversarial cases, including failure and fallback paths?

## Dependencies
 - Are new or updated dependencies necessary?
 - Are package names, publishers, sources, and versions correct and supported?
 - Were lockfiles updated, and were known vulnerabilities and license constraints checked?
 - Were updates to documentation, configuration, or README files made as required by this change?
 - Are there any potential impacts on other parts of the system, public APIs, data formats, or backward compatibility?

## Security and Data Privacy
 - Does the code introduce any security vulnerabilities?
 - Are authorization and authentication handled correctly?
 - Is untrusted input validated and handled with context-appropriate defenses, such as parameterized queries and output encoding to prevent security attacks such as cross-site scripting or SQL injection?
 - Is sensitive data minimized and securely collected, transmitted, logged, stored, and deleted?
 - Does this code change reveal secrets such as API keys, tokens, passwords, or other credentials?
 - Are responses from external services, files, webhooks, and model outputs treated as untrusted input?
 - If the product uses an LLM, could prompt injection expose data or trigger tools and actions without the correct authorization?

## Performance
 - Do you think this code change decreases system performance?
 - Do you see any significant opportunity to reduce latency, memory, compute, network, or model usage and cost?

## Readability
 - Is the code easy to understand?
 - Which parts were confusing to you and why?
 - Can the readability of the code be improved by smaller methods?
 - Can the readability of the code be improved by different function, method, or variable names?
 - Is the code located in the right file/folder/package?
 - Do you think certain methods should be restructured to have a more intuitive control flow?
 - Is the data flow understandable?
 - Are there redundant or outdated comments?
 - Could some comments convey the message better?
 - Would more comments make the code more understandable?
 - Could some comments be removed by making the code itself more readable?
 - Is there any commented-out code?
