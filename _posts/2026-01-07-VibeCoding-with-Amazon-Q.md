---
layout: post
title: VibeCoding with Amazon Q - Practical AI Integration for Daily Development
published: true
tags: Amazon-Q AI Claude VibeCoding productivity
excerpt_separator: <!--more-->
---

## Overview

There's a certain irony in writing about cutting-edge AI tools in 2026. Much like my previous posts about VB6 (a language that's mostly irrelevant today), there's a good chance the specific tools and techniques I'm documenting here will be outdated in a few months with how fast this industry is moving. Amazon Q will evolve, new models will emerge, and today's "best practices" will become tomorrow's legacy workflows.

That's exactly why it's worth documenting now. Just as those VB6 posts might help someone debugging ancient code today, this captures a snapshot of how I actually worked with AI in early 2026.

<!--more-->

This post documents how I integrated Amazon Q into my daily workflow over the past year. Not marketing promises or future predictions, just the practical reality of what works, what doesn't, and what I learned along the way.

## Current AI Assistant Landscape (January 2026)

After testing various AI coding assistants in my dev environments, here's my current assessment:

**S-Tier:**

- Amazon Q + Claude Sonnet 4.5
- GitHub Copilot + Claude Sonnet 4.5

**A-Tier:**

- Amazon Q + Claude Haiku 4.5
- GitHub Copilot (other models)

**B-Tier:**

- Amazon Q (Chat Mode)

**C-Tier:**

- Microsoft Copilot (Teams)
- ChatGPT Copy-Paste workflow

**D-Tier:**

- Atlassian Rovo

**F-Tier:**

- Microsoft Copilot (Outlook)

This ranking is based on actual usage patterns and will likely shift as these tools evolve. The key differentiator has been context awareness and model quality.

## What Even Is VibeCoding? 🌊

Here's my philosophy:

**Traditional Coding:**

```
🤔 Think → 💻 Code → 🧪 Test → 🐛 Debug → 😭 Cry → 🔁 Repeat
```

**VibeCoding:**

```
🤔 Think → 🌊 Vibe with AI → ✨ ??? → 🚀 Ship → 💰 Profit
```

I'm being slightly facetious. Here's my real insight: most coding isn't creative problem-solving. It's:

- Writing boilerplate
- Generating tests
- Updating documentation
- Crafting commit messages
- Refactoring similar patterns

These tasks are important, but they're also soul-crushing. This is where AI shines. ⚡

## Understanding the Two Modes 🎮

Amazon Q operates in two distinct modes, and I've learned when to use each:

### 🛡️ Chat Mode (Safe Mode)

- Read-only access to your codebase
- Ideal for code review and exploration
- Safe for late-night debugging sessions
- Cannot modify files

### ⚔️ Agent Mode (YOLO Mode)

- Full filesystem access within your project
- Can create, modify, and delete files
- Requires additional permissions for files outside project scope
- Use when you need actual code changes, not just analysis

**My Best Practice:** I default to Chat Mode for exploration and code review. I only switch to Agent Mode when I need to modify files. And I consider Chat Mode mandatory for any work after 3am (sometimes). 🌙

## My Workflow Improvements 🔧

### 1. Commit Message Generation 📝

**The Problem:**
Poor commit messages make git history useless for debugging and code archaeology.

**Before:**

```bash
git commit -m "fix stuff"
git commit -m "updates"
git commit -m "YOLO"
```

**After:**
Prompt Amazon Q: "Generate commit message for staged changes"

```bash
git commit -m "feat(auth): implement JWT refresh with rate limiting

- Add token refresh endpoint with 15min window
- Update client to handle 401s automatically
- Add comprehensive test coverage for edge cases
- Fix memory leak in token validation

TICKET-123"
```

**Result:** My git history became a useful tool for understanding code evolution and debugging production issues.

### 2. Branch Comparison and Review 🔍

**The Problem:**
Before opening a pull request, I need to understand what changed and potential impacts.

**The Approach:**
Prompt: "Scan all my changes and tell me what changed"

**What Amazon Q Analyzes:**

- Files added, modified, or deleted
- Breaking changes and API modifications
- Security implications
- Test coverage impact
- Performance considerations

**Result:** Catch potential issues before code review. I've identified several would-be production bugs using this workflow.

### 3. Unit Test Generation 🧪

**The Problem:**
Comprehensive test coverage requires thinking through edge cases, which I found time-consuming and easy to miss.

**My Approach:**
I prompt: "Generate comprehensive unit tests for this function, think of any edge cases and implement them"

**Example - Simple Division Function:**

```javascript
function divide(a, b) {
  return a / b;
}
```

**Amazon Q identifies:**

- Division by zero → `Infinity`
- Zero by zero → `NaN`
- String coercion: `divide("10", "2")` → `5`
- Boolean and Array edge cases
- Type validation scenarios

**Time Impact:** 30 minutes → 2 minutes for comprehensive test suites ⏱

**Note:** JavaScript's type coercion creates numerous edge cases that are easy to overlook manually. (JavaScript has more edge cases than a dodecahedron)

### 4. README and Documentation 📚

**The Problem:**
Documentation often becomes outdated or incomplete during rapid development.

**Before:**

```markdown
# My Project

This is my project. It does stuff.

## Installation

Run it somehow.
```

**After Amazon Q:**

```markdown
# 🚀 My Project

[![Build Status](https://img.shields.io/badge/build-passing-brightgreen)]

## ✨ Features

- Real-time data processing
- RESTful API with authentication
- Docker containerization

## 🛠️ Installation

`npm install my-project`

## 📚 API Documentation

See [docs/API.md](docs/API.md)

## 🤝 Contributing

See [CONTRIBUTING.md](CONTRIBUTING.md)
```

**Result:** Professional documentation that accurately reflects the current state of my project. My README goes from "meh" to "this dev knows their stuff!"

## My Advanced Techniques 🚀

### Multi-IDE Workflow 💻

**My Setup:**
I run multiple IDE instances with separate Amazon Q conversations:

- IDE 1: Main feature development
- IDE 2: Test generation and validation
- IDE 3: Documentation and research

**Why This Works:**

- Parallel AI conversations (no waiting for responses)
- Context isolation (each conversation maintains focus)
- Specialized workflows for different tasks

**My Pro Tip:** I monitor the status indicator (the little dot 🔵) on chat tabs to track which conversations are processing.

### Cross-Filesystem References 🔗

**The Capability:**
Amazon Q can reference code from different projects:

```
"Apply the error handling pattern from
~/other-project/src/utils/errors.ts
to this new service"
```

**My Use Cases:**

- Maintaining consistent patterns across my microservices
- Applying solutions from my previous projects
- Enforcing company coding standards
- Cross-team knowledge sharing

### Project Rules 📋

**The Feature:**
Create `.amazonq/rules/` directory with markdown files defining your patterns:

```markdown
# builder-pattern.md

- All builders return strongly typed results
- Use builder option classes for configuration
- Always validate null/undefined inputs

# error-handling.md

- Use appropriate error codes and messages
- Validate input before processing
- Log errors with context and stack traces
```

**Result:** Amazon Q follows my team's specific patterns instead of generic best practices. Teaching AI my team's secret sauce!

## Limitations and Pain Points I've Encountered 😅

### 1. Context Window Limits

Each conversation tab operates independently with its own context window. Large codebases can hit these limits quickly, requiring me to carefully manage context.

### 2. Specificity Requirements

Vague prompts produce vague results.

**Ineffective:**

- "Fix this"

**Effective:**

- "Fix the email regex validation on line 45 to allow plus addressing (user+tag@domain.com)"

### 3. Task Size Impact

- Small, focused tasks: High accuracy
- Large, complex tasks: Compounding errors

I've learned to break large refactoring into smaller, verifiable steps.

### 4. Hallucinations 🤯

Amazon Q occasionally:

- Invents APIs that don't exist
- Provides confident but incorrect information
- Creates methods without proper implementation
- "Trust me bro" energy 💯

**My Mitigation Strategies:**

- I always verify generated code
- I use `@file` references for explicit context
- I test immediately after implementation
- I ask the AI to explain its assumptions

### 5. Security Considerations ⚠️

Amazon Q uses your system credentials. If you have production access, so does Q. Exercise caution with:

- Production database connections
- Credential-containing files
- Security-sensitive operations

Always review changes before execution in sensitive environments. What permissions you have, Amazon Q has!

## When I Use (and Don't Use) AI Assistance 🎯

### ❌ I Don't Use For:

- Blindly accepting changes without review
- Security-sensitive code without manual verification
- Fundamental architecture changes without planning
- Production database migrations
- Any operation where you don't understand the output

### ✅ Perfect For:

- Boilerplate generation
- Code explanation and documentation
- Refactoring existing code with clear requirements
- Test generation
- Documentation updates
- Commit message generation

My key principle: AI should amplify my expertise, not replace my judgment.

## My Implementation Levels 📊

**Level 1: Basic Usage**
Simple prompts like "Generate this function"

**Level 2: Context-Aware Integration**
Using `@file` and `@folder` references: "Using @src/models/ create a new user test"

**Level 3: Advanced Workflows**
Multi-IDE setups, cross-project references, custom project rules

I find most developers are happy at Level 2 for daily work.

## My Summary 📈

After one year of integrating Amazon Q into my workflow, my results are measurable:

- Improved commit message quality ✅
- Earlier bug detection through automated reviews 🐛
- More comprehensive test coverage 🧪
- Consistent documentation 📝
- Faster feature delivery 🚀

For me, this isn't about replacing developers or achieving some mythical "10x" status. It's about using available tools to handle repetitive tasks more efficiently, freeing up my time for actual problem-solving.

The AI assistant landscape will continue evolving. Tools that are cutting-edge today may be obsolete tomorrow (much like the VB6 code I used to maintain). My fundamental principle remains: understand your tools, use them appropriately, and always verify the output.

And on that note about "always verify the output" - I've been thinking about writing up my homelab experiences with AI assistants, particularly the pitfalls I've encountered with GitHub Copilot. Let's just say there's an interesting story about `rm -rf`, my NAS, and a lesson in blindly trusting autocomplete suggestions that I'm still not fully over. Maybe if there's interest (or if I feel like reliving the pain), I'll document that adventure too.

---

## Additional Links

- [Blog](https://mitchscobell.github.io/)
- [LinkedIn](https://linkedin.com/in/mitchscobell)
- [My Website](https://mitchscobell.com/)
