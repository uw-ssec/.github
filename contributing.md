# Contributing to UW-SSEC Open-Source Projects

Thank you for your interest in contributing to the University of Washington’s **Scientific Software Engineering Center (SSEC)**.  
We welcome contributions from students, researchers, engineers, and the wider open-source community.

This guide outlines expectations for contributing code, documentation, and ideas across all repositories in the [**uw-ssec**](https://github.com/uw-ssec/) GitHub organization. Additionally, refer to [RSE Guidelines](https://rse-guidelines.readthedocs.io/en/latest/) for SSEC quick guides and practical tutorials.


> [!NOTE]
> Individual repos may define additional rules — if so, those repo-specific rules take priority.


---
## Before You Start

Before contributing, please:

- Read the repository’s **README**
- Review any repo-level **CONTRIBUTING.md**, **CODE_OF_CONDUCT**, or developer docs
- Check the **issue tracker** to see if someone is already addressing your idea
- Comment on an issue before starting work (especially for new contributors)


## Do’s
### Read the project documentation first.
Start with the README, CONTRIBUTING.md, and CODE_OF_CONDUCT. Understand the project’s structure and workflow before generating or editing code.
### Start small
Pick beginner-friendly issues or documentation fixes to learn the codebase and build rapport with maintainers.
### Communicate early and clearly
Comment on an issue before starting. If using AI for an idea or draft, still discuss implementation plans with maintainers.
### Follow the coding standards
Match the project’s existing style. If you use AI to generate code, ensure it follows the same conventions.
### Write good commit messages
Clear titles + meaningful descriptions. AI can help draft them, but make sure they accurately reflect your work.
### Test your changes thoroughly
Run existing tests. Add new tests when relevant. Validate any AI-generated code carefully—AI can create “plausible but broken” logic.
### Be respectful and patient
Maintain polite communication and openness to feedback.
### Learn from reviews
Treat reviews as mentorship. Ask clarifying questions. If AI generated some code, be prepared to explain how it works—you are responsible for it.
### Keep PRs small and focused
One logical change per PR. Makes review easier and reduces merge conflicts.
### Document your work
Update docs or comments whenever behavior changes. If AI generated docs, verify accuracy.
### Use AI for brainstorming, not autopilot coding
AI is great for:
-	learning unfamiliar APIs
-	generating code snippets
-	explaining errors
-	drafting documentation or tests
But final decisions should be yours.
### Verify everything AI produces
AI-generated code can be:
-	incorrect
-	non-idiomatic
-	insecure
-	incompatible with the project’s style
Always review, test, and adjust.
### Ensure generated code matches the project’s license and policies
Some projects restrict AI-generated contributions.
If a project explicitly says no AI-generated PRs, follow that rule.
### Keep your contributions interpretable
If you use AI to generate complex code, simplify it or annotate it so maintainers can understand it.
### Use AI ethically
Use AI as a helper, not a source of truth.

#### DO NOT:
1. paste private or sensitive code into public AI tools
2. present AI output as your own expertise without reading it
3. bypass learning fundamentals by relying on AI for every step
AI should accelerate your learning, not replace it.
4. open large PRs without discussion
Avoid huge refactors or new features unless approved.
5. ignore the issue tracker
Ensure no one else is working on the same issue.
6. break backward compatibility without approval
7. push messy commits. Clean up history before submitting.
8. take critique personally, maintain a growth mindset
9. expect immediate responses
10. circumvent tests or lint rules
11. add dependencies without approval
12. disappear after creating a PR
13. submit AI-generated code without understanding it. You must be able to explain every line you contribute.
14. use AI to auto-generate entire files or subsystems. It usually results in low-quality, unreviewable code. Don’t assume AI knows the project context.
It doesn’t know:
- the maintainers’ preferences
-	project architecture
-	historical decisions

Don’t leak private information

Never paste into AI tools any of the following:
-	secrets
-	API keys
-	unpublished research
-	proprietary code

### Don’t include unnecessary files in PR
Files like CHANGELOG are good for releases, unnecessary for small PRs. Avoid adding templates like 
ACCEPTANCE_CRITERIA_CHECK, COMMIT_SUMMARY etc.

### Don’t ignore licensing concerns
If the project prohibits AI-generated content, respect that.


### 🔀 Pull Request Guidelines


Your PR should:

1. **Reference the related issue.**
2. Include a clear description of what changed and why.
3. Contain focused commits (one conceptual change).
4. Include tests (if relevant).
5. Update documentation as needed.
6. Pass CI checks: tests, linting, and build steps.
7. Follow coding style and architecture.

---

## Examples of Good and Bad PRs

### ✔️ Good PR Example

**Title:**  
`Fix out-of-bounds index error in sonar calibration routine`

**Description:**  
This PR fixes an index error that occurs when calibration tables have fewer
entries than expected. The fix ensures the index is clamped to the valid range.

Added bounds-checking logic

Added unit test reproducing original issue

Updated calibration.md to describe expected input size
Fixes #142


**Characteristics:**  
- Small and focused  
- Includes tests  
- Explains *why* the change was made  
- Links to the issue  
- No unrelated file changes  

---

### ❌ Bad PR Example

**Title:**  
`Update with several bug fixes`

**Description:**  
Made some changes and improvements.


**Problems:**  
- No issue reference  
- Large unrelated changes  
- Mixes style changes, refactors, and new features  
- No tests  
- Breaks formatting and adds noise files  
- Reviewer cannot understand the intent  

---

## Security, Licensing, and Privacy

- Follow open source security best practices  
- Never commit secrets, tokens, or private data  
- Respect project licensing (e.g., Apache, MIT, BSD)  
- Do not copy code from incompatible licenses  
- Follow AI-usage restrictions when specified

---

# 🙌 Thank You

Your contributions help advance open, reproducible, and impactful scientific software at SSEC.  
We appreciate your time, expertise, and collaboration.

