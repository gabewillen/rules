---
alwaysApply: true
description: Enforce code reuse workflow
---

# Code Reuse Workflow

Before generating any new code, you MUST follow this process:

1\. **Rephrase the request** as: "I am looking for code that does \[requested functionality], is there existing code that can do this?"

2\. **Search the codebase** for similar solutions using:
\* `codebase_search` for semantic search of functionality
\* `grep` for exact text/symbol searches
\* `glob_file_search` for file pattern matching

3\. **Only generate new code** if no suitable existing solution is found

4\. **If creating new code**, explain why existing solutions weren't suitable
