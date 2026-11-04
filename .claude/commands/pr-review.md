---
allowed-tools: Read, Glob, Grep, Bash(gh issue view:*), Bash(gh search:*), Bash(gh issue list:*), Bash(gh pr comment:*), Bash(gh pr diff:*), Bash(gh pr view:*), Bash(gh pr list:*), mcp__github_inline_comment__create_inline_comment
description: Review a pull request for bugs, CLAUDE.md compliance and security, on every push
---

<!--
Adapted from the code-review plugin in anthropics/claude-plugins-official (Apache License 2.0,
plugins/code-review/commands/code-review.md). Changes: it reviews again on every push instead of stopping
once Claude has commented, skips findings it has already posted, reports what later commits fixed, and adds
a security red-team agent that follows the "Security" section of CLAUDE.md.
-->

Provide a code review for the given pull request. Arguments: `$ARGUMENTS`. The arguments name the pull
request, may include `--comment`, and may include `prior-comments=<path>`: a JSON file of the inline review
comments already on the pull request.

**Agent assumptions (applies to all agents and subagents):**
- All tools are functional and will work without error. Do not test tools or make exploratory calls. Make sure this is clear to every subagent that is launched.
- Only call a tool if it is required to complete the task. Every tool call should have a clear purpose.

To do this, follow these steps precisely:

1. Launch a haiku agent to check if any of the following are true:
   - The pull request is closed
   - The pull request is a draft
   - The pull request does not need code review (e.g. automated PR, trivial change that is obviously correct)

   If any condition is true, stop and do not proceed. Do **not** stop because Claude has already commented:
   every push gets a fresh review.

Note: Still review Claude generated PR's.

2. Launch a haiku agent to collect what Claude has already said on this pull request: Claude's comments
   from `gh pr view <PR> --comments`, and, if a `prior-comments=` file was given, the inline comments in it
   whose author is Claude (`claude[bot]` or `claude`). Return each earlier finding as file, line and a
   one-line description, and the commit SHA of the most recent earlier review summary if there is one.

3. Launch a haiku agent to return a list of file paths (not their contents) for all relevant CLAUDE.md files including:
   - The root CLAUDE.md file, if it exists
   - Any CLAUDE.md files in directories containing files modified by the pull request

4. Launch a sonnet agent to view the pull request and return a summary of the changes, including what
   changed since the most recent earlier review (if step 2 found one).

5. Launch 5 agents in parallel to independently review the changes. Each agent should return the list of issues, where each issue includes a description and the reason it was flagged (e.g. "CLAUDE.md adherence", "bug", "security"). The agents should do the following:

   Agents 1 + 2: CLAUDE.md compliance sonnet agents
   Audit changes for CLAUDE.md compliance in parallel. Note: When evaluating CLAUDE.md compliance for a file, you should only consider CLAUDE.md files that share a file path with the file or parents.

   Agent 3: Opus bug agent (parallel subagent with agent 4)
   Scan for obvious bugs. Focus only on the diff itself without reading extra context. Flag only significant bugs; ignore nitpicks and likely false positives. Do not flag issues that you cannot validate without looking at context outside of the git diff.

   Agent 4: Opus bug agent (parallel subagent with agent 3)
   Look for problems that exist in the introduced code. This could be security issues, incorrect logic, etc. Only look for issues that fall within the changed code.

   Agent 5: Opus security red-team agent
   Attack the change as described in the "Security" section of the root CLAUDE.md, at code level (the
   changed lines) and architecture level (what the change does to trust boundaries, data flows and the
   threat model, including code it makes reachable). Read the repository beyond the diff wherever an
   attack path needs it. Each issue must name the attacker, the steps, what they gain, the file and lines,
   a severity (Critical, High, Medium or Low) and a fix. Mark anything that depends on context it can't
   verify "needs human judgement" rather than dropping it.

   **CRITICAL: We only want HIGH SIGNAL issues.** For agents 1 to 4, flag issues where:
   - The code will fail to compile or parse (syntax errors, type errors, missing imports, unresolved references)
   - The code will definitely produce wrong results regardless of inputs (clear logic errors)
   - Clear, unambiguous CLAUDE.md violations where you can quote the exact rule being broken

   Do NOT flag:
   - Code style or quality concerns
   - Potential issues that depend on specific inputs or state (except security issues from agent 5 that come with a concrete attack)
   - Subjective suggestions or improvements

   If you are not certain an issue is real, do not flag it. False positives erode trust and waste reviewer time. For agent 5, "certain" means the attack path holds up when traced through the code.

   In addition to the above, each subagent should be told the PR title and description. This will help provide context regarding the author's intent.

6. For each issue found in the previous step by agents 3, 4 and 5, launch parallel subagents to validate the issue. These subagents should get the PR title and description along with a description of the issue. The agent's job is to review the issue to validate that the stated issue is truly an issue with high confidence. For example, if an issue such as "variable is not defined" was flagged, the subagent's job would be to validate that is actually true in the code. Another example would be CLAUDE.md issues. The agent should validate that the CLAUDE.md rule that was violated is scoped for this file and is actually violated. For security issues, the agent should trace the attack step by step through the code and confirm each step holds; keep "needs human judgement" issues if the attack holds apart from the context it names. Use Opus subagents for bugs, logic and security issues, and sonnet agents for CLAUDE.md violations.

7. Filter out any issues that were not validated in step 6. Then compare the rest with the earlier findings from step 2: drop any issue that is already posted (same problem at the same place), and note which earlier findings the current code has fixed. This gives the list of new high signal issues.

8. Output a summary of the review findings to the terminal:
   - If new issues were found, list each issue with a brief description.
   - List earlier findings that are now fixed, if any.
   - If no new issues were found, state: "No new issues found. Checked for bugs, CLAUDE.md compliance and security."

   If `--comment` argument was NOT provided, stop here. Do not post any GitHub comments.

   If `--comment` argument IS provided, post one summary comment using `gh pr comment`, in the format below, then continue to step 9 if there are new issues.

9. Create a list of all comments that you plan on leaving. This is only for you to make sure you are comfortable with the comments. Do not post this list anywhere.

10. Post inline comments for each new issue using `mcp__github_inline_comment__create_inline_comment` with `confirmed: true`. For each comment:
   - Start security findings with "Security · <severity>:"
   - Provide a brief description of the issue
   - For small, self-contained fixes, include a committable suggestion block
   - For larger fixes (6+ lines, structural changes, or changes spanning multiple locations), describe the issue and suggested fix without a suggestion block
   - Never post a committable suggestion UNLESS committing the suggestion fixes the issue entirely. If follow up steps are required, do not leave a committable suggestion.

   **IMPORTANT: Only post ONE comment per unique issue. Do not post duplicate comments, and do not re-post issues already on the pull request.**

Use this list when evaluating issues in Steps 5 and 6 (these are false positives, do NOT flag):

- Pre-existing issues (unless this change makes a security issue reachable or worse)
- Something that appears to be a bug but is actually correct
- Pedantic nitpicks that a senior engineer would not flag
- Issues that a linter will catch (do not run the linter to verify)
- General code quality concerns (e.g., lack of test coverage) unless explicitly required in CLAUDE.md
- Issues mentioned in CLAUDE.md but explicitly silenced in the code (e.g., via a lint ignore comment)

Notes:

- Use gh CLI to interact with GitHub (e.g., fetch pull requests, create comments). Do not use web fetch.
- Create a todo list before starting.
- You must cite and link each issue in inline comments (e.g., if referring to a CLAUDE.md, include a link to it).
- The summary comment has this format, with `<sha>` the full SHA of the pull request's head commit:

---

## Code review

Reviewed at `<sha>`. <"No new issues found." or "N new issues, posted inline."> Checked for bugs, CLAUDE.md compliance and security.

<If any earlier findings are now fixed: "Fixed since the last review:" and a bullet per finding.>

---

- When linking to code in inline comments, follow the following format precisely, otherwise the Markdown preview won't render correctly: https://github.com/anthropics/claude-code/blob/c21d3c10bc8e898b7ac1a2d745bdc9bc4e423afe/package.json#L10-L15
  - Requires full git sha
  - You must provide the full sha. Commands like `https://github.com/owner/repo/blob/$(git rev-parse HEAD)/foo/bar` will not work, since your comment will be directly rendered in Markdown.
  - Repo name must match the repo you're code reviewing
  - # sign after the file name
  - Line range format is L[start]-L[end]
  - Provide at least 1 line of context before and after, centered on the line you are commenting about (eg. if you are commenting about lines 5-6, you should link to `L4-7`)
