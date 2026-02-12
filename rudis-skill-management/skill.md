# Rudi's Skill Management Skill

Manage Cortex Code skills consistently: create, promote, and share.

## Trigger

Use this skill when:
- User wants to create a new skill
- User wants to promote a skill from project-local to global
- User wants to share a skill on GitHub
- User asks "what skills do I have?"

---

## Directory Structure

```
~/cortex-skills/                      # Git repo - source of truth for shared skills
├── skill-name/
│   └── skill.md
└── README.md

~/.cortex/skills/                     # Global skills (symlinks to ~/cortex-skills/)
├── skill-name -> ~/cortex-skills/skill-name

/project/.cortex/skills/              # Project-local skills (not shared)
├── project-specific-skill/
│   └── skill.md
```

---

## Skill Lifecycle

| Stage | Location | Visibility |
|-------|----------|------------|
| **Project-local** | `/project/.cortex/skills/` | This project only |
| **Global** | `~/.cortex/skills/` (symlink) | All projects |
| **Shared** | `~/cortex-skills/` + GitHub | Anyone with repo access |

---

## Operations

### Create a New Project-Local Skill

```bash
mkdir -p /path/to/project/.cortex/skills/skill-name
# Create skill.md with skill content
```

### Promote Local → Global + Shared

When user says "promote this skill" or "make this skill global":

```bash
# 1. Move to cortex-skills repo
mv /project/.cortex/skills/skill-name ~/cortex-skills/

# 2. Symlink to global location
ln -s ~/cortex-skills/skill-name ~/.cortex/skills/skill-name

# 3. Commit and push
cd ~/cortex-skills
git add skill-name
git commit -m "Add skill-name skill"
git push
```

### Check Skill Status

When user asks "what skills do I have?":

```bash
# Global skills (in ~/.cortex/skills/)
echo "=== Global Skills ===" && ls -la ~/.cortex/skills/

# Shared skills (in ~/cortex-skills/)
echo "=== Shared Skills (GitHub) ===" && ls ~/cortex-skills/

# Project-local skills (if in a project)
echo "=== Project-Local Skills ===" && ls .cortex/skills/ 2>/dev/null || echo "None"
```

### Update a Skill

Since global skills are symlinked from `~/cortex-skills/`, just edit and push:

```bash
# Edit the skill
# (make changes to ~/cortex-skills/skill-name/skill.md)

# Commit and push
cd ~/cortex-skills
git add -A
git commit -m "Update skill-name: description of changes"
git push
```

### Remove a Skill from Global

```bash
# Remove symlink (keeps source in ~/cortex-skills/)
rm ~/.cortex/skills/skill-name

# Or delete entirely
rm -rf ~/cortex-skills/skill-name
rm ~/.cortex/skills/skill-name
cd ~/cortex-skills && git add -A && git commit -m "Remove skill-name" && git push
```

---

## GitHub Repository

- **Repo:** https://github.com/sfc-gh-rleibbrandt/cortex-skills
- **Visibility:** Private (change to public to share with others)

### Share with Others

1. Make repo public: `gh repo edit cortex-skills --visibility public`
2. Others can clone: `git clone https://github.com/sfc-gh-rleibbrandt/cortex-skills`
3. They symlink skills they want: `ln -s /path/to/cortex-skills/skill-name ~/.cortex/skills/`

---

## Naming Conventions

- Skill directory: `kebab-case` (e.g., `workload-year-in-review`)
- Main file: Always `skill.md`
- Prefix personal skills with context if needed (e.g., `rudis-skill-management`)
