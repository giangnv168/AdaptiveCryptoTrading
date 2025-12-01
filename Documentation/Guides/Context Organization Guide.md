# Context Organization Guide

**Last Updated:** 2025-12-01

This guide explains how to organize all documentation and context files in the Obsidian vault.

---

## 📁 Folder Structure

All documentation should be placed in the appropriate folder within the Obsidian vault:

```
obsidian-vault/
├── Documentation/
│   ├── Sessions/          # Session context and work logs
│   ├── Architecture/      # System architecture documentation
│   ├── Bots/             # Bot-specific documentation
│   ├── Feeders/          # Feeder-specific documentation
│   ├── Optuna/           # Optuna optimization documentation
│   └── Guides/           # How-to guides (like this one)
├── Daily/                # Daily notes and observations
├── Research/             # Research notes and experiments
└── Strategies/           # Trading strategy documentation
```

---

## 📝 File Naming Convention

### Session Context Files
**Location:** `Documentation/Sessions/`
**Format:** `SESSION_CONTEXT_<description>_YYYY-MM-DD.md`

**Examples:**
- `SESSION_CONTEXT_BOT1_OPTUNA_2025-12-01.md`
- `SESSION_CONTEXT_FEEDER_OPTIMIZATION_2025-12-01.md`

**What to include:**
- Work summary
- Problems solved
- Decisions made
- Next steps
- Key code changes

### Architecture Documentation
**Location:** `Documentation/Architecture/`
**Format:** `<COMPONENT>_ARCHITECTURE.md`

**Examples:**
- `FEEDER_ARCHITECTURE.md`
- `OPTUNA_SERVICE_ARCHITECTURE.md`
- `DATA_PIPELINE_ARCHITECTURE.md`

**What to include:**
- System design
- Component relationships
- Data flow
- Technical decisions

### Bot Documentation
**Location:** `Documentation/Bots/`
**Format:** `BOT<number>_<TOPIC>_<description>.md`

**Examples:**
- `BOT1_OPTUNA_DEPLOYMENT_COMPLETE.md`
- `BOT2_CONFIGURATION_GUIDE.md`
- `BOT3_TROUBLESHOOTING.md`

**What to include:**
- Bot-specific configuration
- Deployment guides
- Troubleshooting steps
- Performance analysis

### Feeder Documentation
**Location:** `Documentation/Feeders/`
**Format:** `FEEDER_<TOPIC>_<description>.md`

**Examples:**
- `FEEDER_OPTUNA_INTEGRATION_GUIDE.md`
- `FEEDER_DEPLOYMENT_CHECKLIST.md`
- `FEEDER_GPU_TROUBLESHOOTING.md`

**What to include:**
- Feeder setup and configuration
- Optimization strategies
- Performance metrics
- Issue resolution

### Optuna Documentation
**Location:** `Documentation/Optuna/`
**Format:** `OPTUNA_<TOPIC>_<description>.md`

**Examples:**
- `OPTUNA_CONFIGURATION_GUIDE.md`
- `OPTUNA_BEST_PRACTICES.md`
- `OPTUNA_INFLUXDB_SETUP.md`

**What to include:**
- Optimization workflows
- Parameter tuning guides
- Results analysis
- Integration guides

---

## ✅ Best Practices

### 1. **Always Create Files in the Vault**
```bash
# ✅ CORRECT - Create in vault
cd /home/ubuntu/freqtrade/obsidian-vault/Documentation/Sessions
vim SESSION_CONTEXT_NEW_WORK_2025-12-01.md

# ❌ WRONG - Don't create in freqtrade root
cd /home/ubuntu/freqtrade
vim SESSION_CONTEXT_NEW_WORK_2025-12-01.md
```

### 2. **Use Descriptive Names**
- ✅ `BOT1_OPTUNA_PARAMETER_OPTIMIZATION.md`
- ❌ `notes.md`
- ❌ `temp.md`

### 3. **Include Dates for Time-Sensitive Content**
- ✅ `SESSION_CONTEXT_2025-12-01.md`
- ✅ `DEPLOYMENT_COMPLETE_2025-12-01.md`

### 4. **Link Related Documents**
Use Obsidian's wiki-links to connect related documentation:

```markdown
See also:
- [[BOT1_ARCHITECTURE]]
- [[OPTUNA_CONFIGURATION_GUIDE]]
- [[SESSION_CONTEXT_2025-11-30]]
```

### 5. **Use Tags for Organization**
```markdown
#bot1 #optuna #deployment #complete
```

---

## 🔄 Workflow for Creating New Documentation

### Step 1: Determine the Type
Ask yourself: What am I documenting?
- Session work → `Documentation/Sessions/`
- System design → `Documentation/Architecture/`
- Bot-specific → `Documentation/Bots/`
- Feeder-specific → `Documentation/Feeders/`
- Optuna-specific → `Documentation/Optuna/`
- How-to guide → `Documentation/Guides/`

### Step 2: Create the File in the Correct Location
```bash
# Example: Creating a new session context
cd /home/ubuntu/freqtrade/obsidian-vault/Documentation/Sessions
vim SESSION_CONTEXT_BOT1_OPTIMIZATION_$(date +%Y-%m-%d).md
```

### Step 3: Use a Template

**Session Context Template:**
```markdown
# Session Context - [Topic] - [Date]

## Summary
Brief overview of what was accomplished.

## Tasks Completed
- [ ] Task 1
- [ ] Task 2
- [ ] Task 3

## Problems Encountered
Description of issues and how they were resolved.

## Key Decisions
Important decisions made during this session.

## Code Changes
- File: `path/to/file.py`
  - Change: Description

## Next Steps
- [ ] Next task 1
- [ ] Next task 2

## Related Documents
- [[Related Document 1]]
- [[Related Document 2]]

## Tags
#session #bot1 #optimization
```

### Step 4: Git Sync
Files are automatically synced to GitHub! But you can manually sync:
```bash
cd /home/ubuntu/freqtrade/obsidian-vault
git add .
git commit -m "Added new session context"
git push
```

---

## 📊 Quick Reference

| Content Type | Location | Example Filename |
|--------------|----------|------------------|
| Session work | `Documentation/Sessions/` | `SESSION_CONTEXT_2025-12-01.md` |
| Architecture | `Documentation/Architecture/` | `FEEDER_ARCHITECTURE.md` |
| Bot docs | `Documentation/Bots/` | `BOT1_DEPLOYMENT_GUIDE.md` |
| Feeder docs | `Documentation/Feeders/` | `FEEDER_OPTIMIZATION_RESULTS.md` |
| Optuna docs | `Documentation/Optuna/` | `OPTUNA_BEST_PRACTICES.md` |
| How-to guides | `Documentation/Guides/` | `Git_Sync_Guide.md` |
| Daily notes | `Daily/` | `2025-12-01.md` |
| Research | `Research/` | `Strategy_Backtesting_Analysis.md` |

---

## 🎯 Benefits of This Organization

1. **Everything is Synced** - All documentation automatically backed up to GitHub
2. **Easy to Find** - Organized by topic and type
3. **Cross-Device Access** - Available from any device via GitHub
4. **Version Control** - Full history of all changes
5. **Searchable** - Obsidian's powerful search across all notes
6. **Linkable** - Connect related documentation with wiki-links

---

## 💡 Tips

- **Use the search** - `Ctrl+Shift+F` to search across all notes
- **Use graph view** - Visualize connections between documents
- **Use tags** - Organize with `#tag` for easy filtering
- **Create index notes** - Summary pages linking to related docs
- **Review regularly** - Keep documentation up to date

---

**Remember:** All new documentation should be created directly in the Obsidian vault, not in the freqtrade root directory!
