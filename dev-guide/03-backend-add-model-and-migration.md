# Add a New Data Model

*[中文](03-backend-add-model-and-migration.zh-CN.md)*

> **Status:** Outline only — full content not written yet.

GORM struct conventions used in this codebase, how AutoMigrate picks up a new model, and a real pitfall to watch for: a field literally named `PID` maps to column `p_id` by GORM's default naming strategy, not `pid` — needs an explicit `gorm:"column:pid"` tag.

<!-- TODO: write this chapter -->
