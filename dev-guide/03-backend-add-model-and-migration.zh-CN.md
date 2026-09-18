# 新增数据模型

*[English](03-backend-add-model-and-migration.md)*

> **状态：** 仅目录大纲，正文尚未撰写。

本代码库里 GORM 结构体的写法约定、AutoMigrate 怎么识别新模型，以及一个真实踩过的坑：字段名如果写成全大写的 `PID`，GORM 默认命名策略会映射成列名 `p_id` 而不是 `pid`——需要显式加 `gorm:"column:pid"` 标签。

<!-- TODO: 撰写本章内容 -->
