# Assignment 6 — Building an AI-Assisted Git Safety Net (PR Ready Check)

Part of the DevOps Micro Internship (DMI) Cohort 3 with Agentic AI

---

## Purpose

In Week 2 you built Claude Code hooks that block a dangerous action *before* it happens (`PreToolUse`), and a restricted skill that could look but not touch (`allowed-tools` without `Write`). In this assignment you will discover that Git has the exact same idea, decades older: a **pre-commit hook** that blocks a commit before it's created.

You will build both halves of a real "PR Ready" workflow:

1. A **Git hook that follows fixed rules** — scans staged changes for hardcoded secrets and oversized files and refuses the commit. No AI involved, no guessing, just a rule that gives the same answer every time.
2. A **restricted Claude Code skill** (`/pr-ready`) that reads your staged diff and drafts a Pull Request title, description, and a short list of things worth a second look — the kind of judgment a fixed rule can't make (mixed changes, missing context, unclear intent). The skill never commits, pushes, or opens the PR. You do that yourself, using its draft as a starting point.

This mirrors the Agentic Loop from Week 3's Linux triage assignment: **Gather → Analyze → Human Act → Verify**. The hook and the skill both gather and analyze; only you act.

---

# Task 1 — Create a Branch with Realistic Risk

## Goal

On your own fork of this repository (the one you've been submitting your DMI work in since onboarding), create a new branch and stage a change that a real reviewer should catch: a hardcoded-looking secret and a leftover debug statement.

### What to do

```bash
git checkout -b feature/ai-pr-ready
```

Create a file `scripts/notify.sh` (or edit any existing script) that includes a fake AWS-style key and a debug `echo`, for example:

```bash
#!/bin/bash
# demo only — fake credential for this assignment, never a real key
AWS_ACCESS_KEY_ID=AKIAABCDEFGHIJKLMNOP
echo "DEBUG: token is $AWS_ACCESS_KEY_ID"
```

Stage it with `git add`.

### Evidence

#### Screenshot 1 — `git status` showing the staged file on your new branch

![git status](screenshots/wk4-a6-s1.png)

---

### Notes

**1. Why does this assignment use an obviously fake key instead of a real one?**

It uses an obviously fake key to safely demonstrate how secret detection tools such as GitLeaks identify credentials without exposing any real AWS credentials. This allows you to practice detecting and preventing secrets from being committed to Git while avoiding any security risk, unauthorized access, or accidental credential leakage.

---

# Task 2 — Write a Real Git Pre-Commit Hook

## Goal

Create a tracked, shareable pre-commit hook that blocks a commit containing secret-like patterns or files over 1MB.

### What to do

Create `hooks/pre-commit` (tracked in the repo, not `.git/hooks/`, so teammates get it too):

```bash
#!/bin/bash
# hooks/pre-commit — blocks commits with likely secrets or oversized files
set -e

staged=$(git diff --cached --name-only --diff-filter=ACM)
blocked=0

for file in $staged; do
  if git diff --cached -- "$file" | grep -qE 'AKIA[0-9A-Z]{16}|-----BEGIN (RSA|OPENSSH|PRIVATE) KEY-----'; then
    echo "BLOCKED: possible secret in $file"
    blocked=1
  fi
  size=$(git cat-file -s "$(git rev-parse ":$file")" 2>/dev/null || echo 0)
  if [ "$size" -gt 1000000 ]; then
    echo "BLOCKED: $file is $(($size / 1000000))MB — over the 1MB limit"
    blocked=1
  fi
done

if [ "$blocked" -eq 1 ]; then
  echo "Commit rejected. Fix the issues above and try again."
  exit 1
fi
```

Point Git at the tracked hooks folder and make it executable:

```bash
chmod +x hooks/pre-commit
git config core.hooksPath hooks
```

### Evidence

#### Screenshot 2 — `hooks/pre-commit` open in VS Code showing the full script

![pre commit hook](screenshots/wk4-a6-s2.png)

---

#### Screenshot 3 — Output of `git config core.hooksPath` confirming it points to `hooks`

![git config](screenshots/wk4-a6-s3.png)

---

### Notes

**1. Why is `hooks/pre-commit` tracked in the repo instead of living only in `.git/hooks/`?**

The hooks/pre-commit script is tracked in the repository so it can be version-controlled, reviewed, and shared with everyone working on the project. Since files in .git/hooks/ are local to each Git repository and are not committed, storing the hook in the repository ensures every team member can use the same hook by installing or linking it into .git/hooks/.

---

**2. Compare this to `PreToolUse` from Week 2 Assignment 6. What does each one intercept, and what do they have in common?**

- Git pre-commit hook: Intercepts the Git commit process before a commit is created. It can validate staged files, run tests, or scan for secrets, and block the commit if checks fail.

- PreToolUse: Intercepts a tool invocation before the tool is executed, allowing validation, approval, or policy enforcement before the action runs.

What they have in common: Both act as preventive checkpoints that execute before an action occurs. They automate validation, enforce rules or security policies, and can stop the operation if the required conditions are not met.

---

# Task 3 — Prove the Hook Blocks the Risky Commit

## Goal

Attempt to commit the staged file from Task 1 and show the hook rejecting it.

### Evidence

#### Screenshot 4 — Terminal showing `git commit` rejected with the hook's "BLOCKED" message naming the exact file

![git commit](screenshots/wk4-a6-s4.png)

---

### Notes

**1. Which line in `hooks/pre-commit` matched your fake key, and why did it match?**

Line 9 in hook/pre-commit.

It matched because the the fake AWS access key, which begins with the AKIA prefix followed by 16 uppercase letters/numbers. The regular expression looks for strings that begin with AKIA followed by exactly 16 uppercase letters or numbers, which is the format used by AWS Access Key IDs.

---

**2. Could this hook have caught a poorly-named variable that stores a secret without the `AKIA` prefix? What does that tell you about the limits of a fixed rule like this?**

No. If the secret did not match one of the predefined patterns (such as an AWS key starting with AKIA or a private key header), the hook would not detect it, regardless of the variable name. This demonstrates that fixed rule-based detection has limitations—it can only identify secrets that match its configured patterns. Secrets with different formats, custom tokens, or obfuscated values may go undetected, which is why more advanced secret-scanning tools often combine pattern matching with entropy analysis and other detection techniques.

---

# Task 4 — Build the `/pr-ready` Skill

## Goal

Create a manually invoked Claude Code skill that reads your staged changes and produces a PR-readiness report and a draft PR description — without writing, committing, or pushing anything itself.

### What to do

Create `.claude/skills/pr-ready/SKILL.md` with frontmatter restricting it to read-only inspection tools:

```markdown
---
name: pr-ready
description: Reviews staged Git changes and drafts a PR title, description, and risk report. Never commits, pushes, or opens PRs.
allowed-tools: Bash, Read, Grep
disable-model-invocation: true
---

You are reviewing staged changes before a Pull Request is opened.

1. Run `git diff --cached` and `git status` to see exactly what is staged.
2. Report any of the following if present: secrets or credential-shaped
   strings, debug print/echo statements, TODO/FIXME left in code, a diff
   that mixes unrelated concerns, or a change with no corresponding notes.
3. Draft a PR title that starts with a short word like `feat:` or `fix:`
   telling the reader what kind of change this is, and a 3-5 sentence PR
   description explaining what changed and why.
4. Never run `git commit`, `git push`, or `gh pr create`. Never edit files.
   Your output is a draft for a human to review and use.
```

Run it with `/pr-ready`.

### Evidence

#### Screenshot 5 — `SKILL.md` frontmatter showing `allowed-tools: Bash, Read, Grep` (no `Write`) and `disable-model-invocation: true`

![skill.md](screenshots/wk4-a6-s5.png)

---

#### Screenshot 6 — `/pr-ready` output while the risky file is still staged, showing it flagged the secret and/or debug statement

![pr ready1](screenshots/wk4-a6-s6.png)

---

### Notes

**1. Why does `/pr-ready` have `Bash` and `Read` but not `Write`?**

/pr-ready has Bash permission so it can run commands (such as Git commands) to inspect the repository, and Read permission so it can examine the staged changes and project files. It does not have Write permission because its role is to analyze and report issues, not to modify files or change the repository. This helps ensure it performs a safe, read-only review.

---

**2. The pre-commit hook and `/pr-ready` both looked at the same staged diff. Did they flag the same things? What did one catch that the other didn't?**

No. While both examined the staged diff, they served different purposes.

- The pre-commit hook enforced specific rules, such as detecting the fake AWS key pattern and rejecting oversized files. It automatically blocked the commit when those rules were violated.
- /pr-ready performed a broader review of the staged changes, checking overall readiness for a pull request. It could identify issues such as code quality, documentation gaps, formatting, or other review concerns, but it did not modify files or block the commit by itself.

In this task, the pre-commit hook specifically caught the fake AWS-style key using its pattern-matching rule, whereas /pr-ready provided a more comprehensive review of the staged changes beyond those fixed checks.

---

# Task 5 — Fix the Issues and Re-Verify

## Goal

Remove the secret and debug statement, then prove both gates now pass clean.

### Evidence

#### Screenshot 7 — `git commit` succeeding after the fix (no BLOCKED message)

![git commit2](screenshots/wk4-a6-s7.png)

---

#### Screenshot 8 — Second `/pr-ready` run showing a clean risk report and a drafted PR title + description

![pr ready2](screenshots/wk4-a6-s8.png)

---

### Notes

**1. What exactly did you change to satisfy the pre-commit hook?**

I removed the hardcoded AWS-style key and the debug echo statement that exposed the key. This eliminated the pattern that the pre-commit hook identified as a potential secret, allowing the commit to pass the hook's security checks.

---

# Task 6 — Open the Pull Request Using the AI Draft

## Goal

Push your branch and open a real Pull Request, using `/pr-ready`'s drafted title and description as your starting point — read it critically and edit before you use it.

**Important:** Open this Pull Request with base repository set to **your own fork** — not the shared upstream `pravinmishraaws/devops-micro-internship-pravinmishra` repository. This assignment's hook and skill files are your own practice work, not a change meant for the shared class repo.

### Evidence

#### Screenshot 9 — Your Pull Request showing the base repository is your own fork, plus the title and description, with the `/pr-ready` draft visible for comparison (paste it in the PR conversation or your notes below)

![pull request](screenshots/wk4-a6-s9.png)

---

#### PR Link

https://github.com/Bummieboaxyl/devops-micro-internship-pravinmishra/pull/1

---

### Notes

**1. What, if anything, did you edit in the AI's drafted PR description before using it? Why?**

I did not use an AI-drafted PR description. I wrote the PR description myself, so no edits were needed.

---

**2. If you had blindly copy-pasted the AI's draft without reading it, what could go wrong?**

The PR description could contain inaccurate information, omit important details, or misrepresent the changes made. Reviewing it ensures the description is accurate, relevant, and reflects the actual work completed.

---

**3. Why does this PR need to target your own fork instead of the shared upstream repository?**

The PR should target my own fork because I do not need to edit the shared upstream repository. Working in a fork protects the main project from unintended changes and allows my work to be reviewed before it is merged.
---

# Task 7 — Map the Workflow to the Agentic Loop

## Goal

Explain this assignment's workflow using the same Gather → Analyze → Human Act → Verify structure from Week 3.

### Notes

**1. Which step(s) represent Gather?**

- Reviewing the task requirements.
- Inspecting the staged changes using Git.
- Running the pre-commit hook and /pr-ready to gather information about the staged files.

---

**2. Which step(s) represent Analyze?**

- The pre-commit hook analyzing the staged changes for secrets and oversized files.
- The /pr-ready skill reviewing the staged diff and assessing whether the changes are ready for a pull request.

---

**3. Which step is Human Act, and why must a human — not Claude — run `git commit`, `git push`, and open the PR?**

The Human Act step is committing the changes, pushing the branch, and opening the pull request. These actions modify the repository and publish changes, so they require human approval and responsibility to ensure the correct code is shared.

---

**4. Which step is Verify?**

The Verify step is confirming that the pre-commit hook passes, reviewing the /pr-ready output, checking the Git status, and ensuring the pull request accurately reflects the intended changes.

---

**5. In one or two sentences: why do you need *both* the fixed-rule pre-commit hook and the AI skill? Isn't one enough?**

The pre-commit hook provides fast, consistent enforcement of specific security rules, such as detecting secrets and oversized files. The AI skill complements it by performing broader contextual reviews and identifying issues that fixed rules alone may not catch.

---

# Task 8 — LinkedIn Post

## Goal

Publish a LinkedIn post summarizing what you built and what you learned about combining fixed-rule safety checks with AI-assisted review.

### Evidence

#### LinkedIn Post URL

https://www.linkedin.com/posts/oluwabunmi-olowoyeye_dmibypravinmishra-devops-git-ugcPost-7485378589601800192-zmQF/?utm_source=share&utm_medium=member_desktop&rcm=ACoAABIxKt4BWOFz-d7RRyAsVUilmny_HuUV_Iw

---

## Key Learnings

Add 3-5 bullet points on what you learned this week.

- Learned and practiced the complete GitHub collaboration workflow—forking repositories, creating branches, committing changes, syncing with upstream, and opening Pull Requests.
- Gained hands-on experience with Git fundamentals, including staging changes, writing meaningful commits, reviewing commit history, and resolving common Git workflow tasks.
- Built and tested a Git pre-commit hook to automatically detect potential secrets and oversized files before commits are created.
- Created a read-only AI-assisted /pr-ready skill that reviews staged changes and drafts Pull Request descriptions while leaving all repository actions under human control.
- Strengthened my understanding of how Git automation, AI-assisted reviews, and human decision-making work together to create a secure and efficient development workflow.

---

# Submission Instructions

- Ensure `hooks/pre-commit` and `.claude/skills/pr-ready/SKILL.md` are committed to your GitHub repository
- Add all required screenshots to your submission
- All written answers must be in your own words
- Do not use a real secret or credential anywhere in your submission — the fake key in Task 1 is intentional and must stay clearly fake
- Open your Pull Request against your own fork, not the shared upstream repository
- Push your final changes to your forked repository
- Include your PR link and LinkedIn post URL

---

## GitHub Repository URL

Paste your forked repository URL here:

`https://github.com/Bummieboaxyl/devops-micro-internship-pravinmishra.git`

---

# Completion Checklist

- [ ] Branch `feature/ai-pr-ready` created with a staged file containing a fake secret and a debug statement
- [ ] `hooks/pre-commit` created and tracked in the repo (not only in `.git/hooks/`)
- [ ] `core.hooksPath` configured to point at `hooks/`
- [ ] Pre-commit hook shown blocking the risky commit
- [ ] `.claude/skills/pr-ready/SKILL.md` created with correct `allowed-tools` (no `Write`) and `disable-model-invocation: true`
- [ ] `/pr-ready` run against the risky diff and shown flagging issues
- [ ] Risky file fixed; `git commit` succeeds cleanly
- [ ] `/pr-ready` re-run showing a clean report and drafted PR title/description
- [ ] Pull Request opened using the AI draft as a starting point, with your own fork as the base repository (not upstream), PR link included
- [ ] Agentic Loop mapping (Task 7) completed in your own words
- [ ] LinkedIn post published and URL submitted
- [ ] All required screenshots added
- [ ] GitHub repository URL provided

---

## 📌 About DMI & CloudAdvisory

DevOps Micro Internship (DMI) is a project-based DevOps program run by Pravin Mishra (The CloudAdvisory) focused on real-world execution, systems thinking, and career readiness.

It helps learners build strong DevOps foundations with hands-on experience.

---

## 📌 Resources

- 🌐 DMI Official Website: https://pravinmishra.com/dmi  
- 🎓 DevOps for Beginners (Udemy): https://www.udemy.com/course/devops-for-beginners-docker-k8s-cloud-cicd-4-projects/  
- 🎓 Agentic AI DevOps with Claude Code: https://www.udemy.com/course/ultimate-agentic-ai-devops-with-claude-code/  
- 🎓 DevOps with Claude Code: Terraform, EKS, ArgoCD & Helm: https://www.udemy.com/course/devops-with-claude-code-terraform-eks-argocd-helm/  
- ▶️ YouTube Playlist: https://www.youtube.com/playlist?list=PLFeSNDtI4Cho  
- 🔗 Pravin Mishra (LinkedIn): https://www.linkedin.com/in/pravin-mishra-aws-trainer/  
- 🏢 CloudAdvisory (LinkedIn): https://www.linkedin.com/company/thecloudadvisory/

---

*This submission is part of DevOps Micro Internship (DMI) Cohort 3 — Agentic AI Track.*
