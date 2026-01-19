---
name: design-doc-to-issues
description: Convert design documents into GitHub issues through an iterative workflow
---

# Design Doc to GitHub Issues Skill

## Overview
This skill converts design documents into GitHub issues through an iterative, user-guided workflow. It creates a temporary markdown file with proposed issue titles, allows for user review and modification, then expands them into full issue templates.

## Usage

This skill is invoked when the user asks to convert a design document into GitHub issues. The user might say things like:
- "Convert this design doc to GitHub issues"
- "Create tickets from the design doc"
- "Break down [design-doc-name] into implementation tasks"

If the user doesn't specify which design doc, prompt them to specify the path.

## Workflow

### Step 1: Generate Issue Titles
1. Read the specified design document
2. Create `TEMP_ISSUES.md` in the current directory
3. Analyze the design doc structure and extract implementation tasks
4. Generate one or more tickets per section as appropriate
5. Insert proposed ticket titles as a numbered list in TEMP_ISSUES.md
6. Inform the user that titles are ready for review

**User Action:** Review the titles, modify them directly in the file, add new ones, or provide feedback for improvements.

### Step 2: Clarify Rationale
After initial review:

1. Ask the user if there are any items they'd like clarification on
2. User provides the item numbers they want explained
3. For each requested item, provide:
   - The rationale for why it was included
   - A reference to the specific location in the design doc (e.g., "protocol/l2-contract-upgrades.md:179-186")
   - A brief explanation based on the design doc content

**User Action:** Review clarifications and continue modifying titles if needed.

### Step 3: Expand to Full Issue Templates
Once the user approves the titles:

1. Convert each bullet point into the full issue template format:
```markdown
# [Title]

## Body

### Description
[1-2 sentences or [TODO]]

### References
[1-2 sentences or [TODO]]

### Acceptance Criteria / Definition of Done
[1-2 sentences or [TODO]]

---
```

2. Fill in each section with relevant context from the design doc (max 1-2 sentences per section)
3. Use `[TODO]` for any section where content is unclear or unavailable
4. Do not make up or infer information that isn't clearly stated in the design doc

### Step 4: Create GitHub Issues
After user reviews the expanded content:

1. Ask the user for the target repository in the format `<owner>/<repo_name>` (the issues will likely be created in a different repo than the design docs)
2. Ask the user if they want to apply any labels to the issues (e.g., "L2CM Upgrades")
3. Ask the user if they want to add a prefix to all issue titles (e.g., "[L2CM Upgrades]")
4. Parse TEMP_ISSUES.md to extract each issue
5. Create GitHub issues using `gh issue create --repo <owner>/<repo_name> --label "<label>"` with the specified prefix prepended to each title
6. Report success and provide links to created issues

### Step 5: Create Parent Issue (Optional)
After all issues are created:

1. Ask the user if they want to create a parent issue that tracks all the created sub-issues
2. If yes:
   - Ask for the parent issue title (suggest using the design doc name or a descriptive project title)
   - Create a parent issue with a description that includes:
     - Link to the design doc
     - A bulleted list of all created sub-issues with links
   - Use GitHub's task list syntax (`- [ ] #issue-number Description`) to make sub-issues checkable
3. Report the parent issue URL

### Step 6: Next Steps and Clean up.
1. Remind the user that they have only created issues for the implementation work, and that there are 
   other steps required, both before and after, to satisfy the SDLC. This includes but is not limited 
   to: writing specs, devnet testing, auditing, superchain-ops tasks, governance considerations, 
   upgrade planning etc. They should refer to https://devdocs.optimism.io/pm/ for more information.
2. Optionally clean up TEMP_ISSUES.md

## Guidelines

### Breaking Down Design Docs
- Use a mixed approach: combination of sections and specific implementation tasks
- Generally create one or more tickets per major section
- Focus on actionable implementation tasks, not documentation structure
- Group related small tasks together; split large tasks into multiple tickets

### Writing Issue Content
- Keep descriptions concise: 1-2 sentences maximum per section
- Reference specific sections or line numbers from the design doc
- Use `[TODO]` rather than fabricating information
- Include links back to the design doc in References section
- Make acceptance criteria specific and testable when possible

## Notes
- The skill creates TEMP_ISSUES.md in the current working directory
- Users maintain full control over the iterative process
- The workflow encourages collaboration and refinement before creating actual issues
- Issues are typically created in a different repository than the design docs repo, so always ask for the target `<owner>/<repo_name>`
