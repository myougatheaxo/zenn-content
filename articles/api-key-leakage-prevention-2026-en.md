---
title: "5 Practical Techniques to Prevent API Key Leakage (with Claude Code Auto-Check)"
emoji: "🔐"
type: "tech"
topics: ["ClaudeCode", "Security", "Python", "Git", "DevOps"]
published: false
---

## Why API Key Leaks Happen

Thousands of API keys are accidentally committed to GitHub every day. The causes are almost always simple:

- Hardcoded a key and ran `git add .`
- Misconfigured `.gitignore`
- Included a key in logs or error messages
- Shared an `.env` file along with source code

Leaked keys are discovered by crawlers within minutes and exploited immediately. With AWS keys, there are reported cases of six-figure bills arriving within hours.

## Technique 1: Manage Environment Variables with .env Files

Never hardcode keys in source code.

```python
# Bad
import openai
openai.api_key = "sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx"

# Good
import os
from dotenv import load_dotenv

load_dotenv()
openai.api_key = os.getenv("OPENAI_API_KEY")
```

Example `.env` file:

```
OPENAI_API_KEY=sk-xxxxxxxxxxxxxxxxxxxxxxxxxxxx
DATABASE_URL=postgresql://user:pass@localhost/mydb
AWS_ACCESS_KEY_ID=AKIAIOSFODNN7EXAMPLE
```

## Technique 2: Configure .gitignore Correctly

Always set up `.gitignore` when creating a project.

```gitignore
# Environment files
.env
.env.local
.env.*.local
*.env

# Credential files
credentials.json
token.json
*_token.txt
*.pem
*.key

# IDE / OS files
.DS_Store
Thumbs.db
.idea/
.vscode/settings.json
```

Just adding a file to `.gitignore` is not enough if it was already committed:

```bash
# Remove a tracked file from git's index
git rm --cached .env
git commit -m "Remove .env from tracking"
```

## Technique 3: Automatically Block with git-secrets

[git-secrets](https://github.com/awslabs/git-secrets) detects key patterns at commit time and blocks the commit.

```bash
# Install (macOS)
brew install git-secrets

# Set up in repository
git secrets --install
git secrets --register-aws  # Register AWS patterns

# Add custom patterns
git secrets --add 'sk-[a-zA-Z0-9]{48}'          # OpenAI
git secrets --add 'xoxb-[0-9]+-[0-9]+-[a-zA-Z0-9]+'  # Slack
```

When you try to commit a key by mistake:

```
[ERROR] Matched one or more prohibited patterns
Possible mitigations:
- Mark false positives as allowed using: git config --add secrets.allowed ...
```

## Technique 4: Detect Unknown Keys with Entropy Analysis

Pattern-independent detection using Shannon entropy. Random strings (i.e., keys) have high entropy values.

```python
import math
import re

def shannon_entropy(s):
    freq = {}
    for c in s:
        freq[c] = freq.get(c, 0) + 1
    entropy = 0
    for count in freq.values():
        p = count / len(s)
        entropy -= p * math.log2(p)
    return entropy

def scan_file(filepath):
    with open(filepath, 'r', errors='ignore') as f:
        for i, line in enumerate(f, 1):
            tokens = re.findall(r'[A-Za-z0-9+/]{20,}', line)
            for token in tokens:
                entropy = shannon_entropy(token)
                if entropy > 4.5:  # threshold for high entropy
                    print(f"L{i}: High entropy string detected: {token[:10]}...")
```

## Technique 5: Automate with the /secret-scanner Skill

Claude Code's `/secret-scanner` skill lets you scan all files before committing.

```bash
claude /secret-scanner src/
```

Skill config (`.claude/commands/secret-scanner.md`):

```markdown
Run a secret detection scan with the following rules.

## Detection Targets
1. API key patterns (OpenAI, AWS, GitHub, Slack, etc.)
2. Long strings with Shannon entropy above 4.5
3. Verify .env files are not git-tracked
4. Hardcoded passwords (password=, passwd=, secret=)

## Output Format
- Severity: HIGH / MEDIUM / LOW
- filename:line_number
- What was detected
- Recommended action
```

CI/CD integration example (GitHub Actions):

```yaml
name: Secret Scan
on: [push, pull_request]
jobs:
  scan:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v3
      - name: Run secret scan
        run: |
          pip install truffleHog
          truffleHog --regex --entropy=True .
```

## Emergency Response if a Key Leaks

If a leak occurs, act in this order:

1. **Revoke the key immediately** — delete it from the API provider's dashboard
2. **Issue a new key** — store it in a secrets manager
3. **Remove from history** — use `git filter-branch` or BFG Repo Cleaner
4. **Check impact** — review access logs for unauthorized usage

```bash
# Completely remove a file from git history with BFG
brew install bfg
bfg --delete-files .env
git reflog expire --expire=now --all && git gc --prune=now --aggressive
git push --force
```

## Summary

| Technique | Effect | Difficulty |
|-----------|--------|------------|
| .env + dotenv | Prevents hardcoding | Low |
| .gitignore setup | Prevents accidental commits | Low |
| git-secrets | Auto-blocks at commit | Medium |
| Entropy analysis | Catches unknown patterns | Medium |
| /secret-scanner | CI/CD integration | Low |

90% of API key leaks come from basic mistakes. Simply enforcing `.env` and `.gitignore` eliminates most risks.

---

> **Security Pack** — A 3-skill set: `/secret-scanner`, `/security-audit`, `/deps-check`. Covers API key detection through OWASP compliance checks.
>
> 👉 [note.com/myougatheaxo](https://note.com/myougatheaxo)
