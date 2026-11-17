# AI Coding Secrets: The SAFE Checklist

Used an AI coding tool (Claude Code, Cursor, and similar) and pushed your project to GitHub? Spend 10 minutes on this checklist.

**S**can your repo · **A**ct on anything found · **F**ix your setup · **E**nable protection

> From the reel by [@talhasepooch](https://instagram.com/talhasepooch). This is a practical checklist, not professional security advice. Check the sources below yourself.

---

## What happened

On 7 November 2026, the security company Apiiro reported that a convenience option in Claude Code can leave passwords in a file that ends up on public GitHub.

- When you choose **"Yes, and don't ask again"** for a command, Claude Code saves that exact command as an allow rule in `.claude/settings.local.json`.
- If the command had a password or token typed into it, the password is saved in plain text.
- If that file is committed and the repo is public, anyone can read it.

Numbers from the report (as relayed by [The Hacker News](https://thehackernews.com/2026/11/claude-codes-dont-ask-again-file.html) and [iTnews](https://www.itnews.com.au/news/dont-ask-again-in-anthropics-claude-code-leaks-developer-secrets-630203)):

| Finding | Count |
| --- | --- |
| Public copies of the file found | 104,185 across 103,617 repos |
| Copies containing at least one secret | 1,122 (1,379 distinct secrets) |
| Database passwords | 379 |
| AWS keys | 23 |
| Total incl. old file versions, shared settings and saved chats | 2,539 secrets in 1,994 repos |

Apiiro did not test whether the credentials still work. One notified company confirmed its key was live.

**Anthropic's response** (Boris Cherny, Claude Code team, on Hacker News): the file is meant to stay local, Claude Code adds it to Git's ignore list when it creates it, Claude Code has historically not scrubbed secrets, and the team is "looking into what we can do to prevent this." The Hacker News notes that this protection is only a global Git exclude on one computer, so it does not cover a file created by hand, copied from another project, or written on another machine. Either way, adding the file to your own repo's `.gitignore` is safe and takes a minute.

Original report: `apiiro.com/blog/coding-agents-leaking-secrets-github-dont-ask-again` (link as cited by iTnews).

---

## S: Scan your repo

Run these in your repo folder (Git Bash, macOS or Linux terminal).

**1. Is the file tracked right now?**
```bash
git ls-files | grep -E "\.claude|\.env"
```
PowerShell: `git ls-files | Select-String "\.claude|\.env"`

If you see `.claude/settings.local.json` or `.env`, git is tracking it.

**2. Was it ever committed in the past?**
```bash
git log --all --oneline -- .claude/settings.local.json
```
Any output means it was in your history, even if you deleted it later.

**3. Does it contain secrets?**
```bash
git log --all -p -- .claude/settings.local.json | grep -i -E "pass|token|key|secret|-p"
```

**4. Run a scanner (optional but good).** Try [gitleaks](https://github.com/gitleaks/gitleaks) or [TruffleHog](https://github.com/trufflesecurity/trufflehog). Neither is perfect: in Apiiro's test, gitleaks flagged 564 and TruffleHog 643 of the 1,122 files, and 215 were missed by both, mostly passwords typed inside commands.

---

## A: Act on anything you found

**5. Change the password or key first.** Do this before anything else. Create a new password in your database or service dashboard and replace the old one.

> Why: your git history keeps the old value forever and anyone can have copied it. Deleting a file does not undo that. Only changing the password does.

---

## F: Fix your setup

**6. Create or edit `.gitignore`** in the top folder of your repo (where the `.git` folder is):
```gitignore
.env
.env.*
!.env.example
.claude/settings.local.json
```
> Why: `.gitignore` tells git to never track these files. `!.env.example` is an exception so the example file can be committed (step 8).

**7. Stop tracking the file if git already has it.**
```bash
git rm --cached .claude/settings.local.json
git add .gitignore
git commit -m "stop tracking local settings"
```
> Why: `.gitignore` only affects files git is not tracking yet. `--cached` removes the file from git but keeps it on your computer.

**8. Put secrets in a `.env` file.** Create `.env` in the same folder:
```env
DB_USER=admin
DB_PASS=your_real_password_here
```
Then create `.env.example` with empty values and commit it:
```env
DB_USER=
DB_PASS=
```
> Why: your real password lives only on your computer. The example file shows others which variables they need, without the values.

**9. Use variables, never typed passwords.**

| Where | How |
| --- | --- |
| Terminal (Git Bash) | `set -a; source .env; set +a` then `mysql -u admin -p"$DB_PASS"` |
| Node.js | `process.env.DB_PASS` (with the `dotenv` package) |
| Python | `os.environ["DB_PASS"]` |

> Why: now the command text contains `$DB_PASS`, not the password. If an AI tool saves the command, it saves the variable name.

**10. When an AI tool asks for approval, approve a pattern, not one exact command.** For example `Bash(npm run:*)`.
> Why: a pattern has no password inside it.

---

## E: Enable protection

**11. Turn on GitHub secret scanning and push protection.**
Repo → Settings → Code security → enable **Secret scanning** and **Push protection**.
> Why: GitHub can warn you, or block the push, when it sees something that looks like a key.

**12. Treat agent folders like `.env`.** Other AI tools keep folders too (`.cursor/`, `.gemini/`). Check what is inside before you commit.

---

## Verify

```bash
git check-ignore -v .env
git status
```
The first command should print the `.gitignore` line that covers `.env`. `git status` should not list `.env` or `.claude/settings.local.json`.

---

## Quick FAQ

**Is my repo affected if it is private?** The leak needs a public repo. But a private repo can become public later, and collaborators can still read the file. Follow the checklist anyway.

**I deleted the file. Am I safe?** No. It is still in your git history. Change the password (step 5). Wiping history with tools like `git filter-repo` or BFG is optional and does not replace changing the password.

**Does this only happen with Claude Code?** The report is about Claude Code's settings file. The same habit (typing secrets into commands or app settings) is risky with any tool.

**Other apps from AI builders.** A separate report by Claroc says many apps built with Grok Build were open to anyone, because the publish screen suggested "Anyone with the link". SpaceXSI says apps are private by default, so treat this as a disputed claim, and check the sharing setting before publishing any AI-built app. ([TNW](https://thenextweb.com/news/grok-build-apps-exposed-claroc-spacexsi), [Claroc](https://blog.claroc.io/ghostshares-grok-build-apps/))

---

## Sources

- [The Hacker News: Claude Code's "Don't Ask Again" file exposes passwords and keys](https://thehackernews.com/2026/11/claude-codes-dont-ask-again-file.html)
- [iTnews: "Don't ask again" in Anthropic's Claude Code leaks developer secrets](https://www.itnews.com.au/news/dont-ask-again-in-anthropics-claude-code-leaks-developer-secrets-630203)
- [Qodo: Claude Code's "Don't Ask Again" left secrets in 1,122 public GitHub files](https://www.qodo.ai/blog/claude-code-dont-ask-again-secrets/)
- [Cybernews: Claude Code secrets leak](https://cybernews.com/security/ai-agents-leaking-secrets-on-github/)
- [TNW: Grok Build apps exposed (Claroc)](https://thenextweb.com/news/grok-build-apps-exposed-claroc-spacexsi)

Found a mistake or something outdated? Open an issue or message [@talhasepooch](https://instagram.com/talhasepooch).
