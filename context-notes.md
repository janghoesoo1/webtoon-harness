# Context Notes

- 2026-06-30: Repository contains `.claude/agents`, `.claude/skills`, and docs images.
- 2026-06-30: User asked to make the harness usable for Codex too. The narrow implementation is to add Codex-native entrypoints while preserving the existing Claude Code layout.
- 2026-06-30: Codex support should be a thin adapter that reads `.claude/agents` and `.claude/skills`, not a duplicate skill tree.
- 2026-06-30: User requested `origin` be changed to `https://github.com/janghoesoo1/webtoon-harness.git` and pushed. The remote was changed and `main` was already up to date before the Codex harness changes were committed.
