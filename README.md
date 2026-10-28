# arabic-search-normalizer

An AI skill for designing Arabic-aware search. It defines a normalisation pipeline (alef and
hamza forms, taa marbuta, alef maqsura, tatweel, diacritics, digits, Persian letter
variants), chooses between no stemming, light stemming and root stemming, plans exact,
stemmed and autocomplete fields, and builds a test table you can run against real queries.

Configuration advice for PostgreSQL, Elasticsearch/OpenSearch, SQLite and MySQL is given in
prose, so you can implement it in your own stack.

## Install

Claude Code:

    git clone https://github.com/AIX-store/skill-arabic-search-normalizer ~/.claude/skills/arabic-search-normalizer

Other agents (Codex, Cursor, Claude Desktop, claude.ai): copy `SKILL.md` into the agent's
instructions or project knowledge.

## Example prompts

- "Our product search can't find 'مستشفى' when users type 'مستشفي'. Design a fix."
- "Plan Arabic name search with autocomplete in PostgreSQL."
- "Should we use root stemming for our legal document search?"
- "Write test cases for Arabic customer deduplication."

## Licence
MIT, © 2026 CodeMeld LLC.

Listed on AiX: https://aixstore.ai/store
