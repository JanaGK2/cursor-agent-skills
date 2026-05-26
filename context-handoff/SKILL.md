---
name: context-handoff
description: Manage conversation length to prevent LLM degradation. Create handoff documents for long sessions, proactively warn about context limits, and ensure smooth transitions between chats.
---

# Context Handoff Skill

## Purpose

Proactively manage conversation length to prevent LLM performance degradation during long-running tasks. Based on research showing LLMs corrupt documents and make increasing errors over extended interactions (DELEGATE-52 benchmark).

## When to Trigger

Monitor for these signals that a context handoff may be beneficial:

### Quantitative Signals
- Conversation has exceeded ~15-20 substantive back-and-forth exchanges
- Multiple complex file edits have been made (5+)
- The user has been working on the same task for an extended period
- You notice yourself making errors you wouldn't normally make
- You're having difficulty tracking the full state of changes

### Qualitative Signals
- User expresses frustration with repeated mistakes
- You're asked to fix something you just implemented
- Context from early in conversation becomes relevant but feels fuzzy
- You're unsure about decisions made earlier in the thread
- The task scope has evolved significantly from the original request

## What to Do

### 1. Acknowledge the Risk
When you notice degradation signals, proactively inform the user:
> "We've been working on this for a while, and to ensure I continue providing accurate help, I recommend we start a fresh thread. I'll create a context document first."

### 2. Create Context Documentation
Before suggesting a new thread, always create a handoff document:

```markdown
# [Project Name] - Context for New Chat

**Created:** [Date]
**Purpose:** Continuation context for [brief description]

## Current State
- What has been accomplished
- What is in progress
- What remains to be done

## Key Resources
- File paths
- URLs
- Credentials/IDs (non-sensitive)

## Important Technical Details
- Architecture decisions made
- Key implementation details
- Known issues/edge cases

## Commands to Run
- How to build/run
- How to deploy

## Next Steps
- Immediate tasks
- Open questions

## Instructions for New Chat
1. Read [architecture doc] first
2. Read this context document
3. [Other specific instructions]
```

### 3. Update Project Documentation
If the project has an architecture doc or README:
- Ensure it reflects the current state
- Add version history entry
- Document any recent changes

### 4. Suggest Starting Fresh
After creating documentation:
> "I've created/updated the context documentation at [path]. You can start a new chat and share this document to continue with full context and fresh performance."

## File Locations

Context documents should be saved:
- In the project folder: `PROJECT_DIR/CONTEXT_FOR_NEW_CHAT.md`
- Or alongside architecture: `PROJECT_DIR/docs/HANDOFF_CONTEXT.md`

## Example Trigger Phrases

Use these naturally when appropriate:
- "Given the complexity of what we've covered, let me create a handoff document..."
- "To maintain quality, I recommend we checkpoint our progress..."
- "Before we continue, let me document our current state for reference..."

## Research Background

Per the DELEGATE-52 benchmark:
- Even frontier LLMs (GPT 5.4, Claude 4.6 Opus) corrupt ~25% of document content over 20 interactions
- Degradation compounds over time - short simulations underestimate severity
- Errors tend to be sparse but severe ("critical failures" cause ~80% of degradation)
- Performance in early interactions doesn't predict long-horizon performance

Starting fresh threads with good context documentation mitigates these risks.

## Integration with Other Skills

When creating context documents, reference:
- Existing ARCHITECTURE.md or README files
- Project-specific skills or rules
- Relevant external documentation URLs
