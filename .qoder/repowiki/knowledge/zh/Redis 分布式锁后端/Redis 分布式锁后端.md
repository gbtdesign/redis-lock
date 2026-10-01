---
kind: external_dependency
name: Redis 分布式锁后端
slug: redis
category: external_dependency
category_hints:
    - vendor_identity
    - client_constraint
scope:
    - '**'
---

### Redis
- 角色：本项目 `rlock` 包依赖的唯一外部存储，用作分布式锁的协调节点。
- 集成点：通过 `github.com/redis/go-redis/v9` 的 `redis.Cmdable` 抽象接入；加锁、解锁、续期分别以 Lua 脚本（`script/lua/lock.lua`、`unlock.lua`、`refresh.lua`）经 `EVAL` 执行，单次抢锁走 `SetNX`（即 `SET ... NX EX`）。
- 稳定约束：README 要求 Redis ≥ v7（作者仅在该版本测试过）；实现是单节点 Redis 锁，未实现 Redlock，主从切换时存在锁丢失风险（见 q6 对话结论）。
- 用法要点：value 使用 UUID 标识持有者，所有"先比对 value 再操作"的复合逻辑（解锁、续期、带重试的加锁）必须走 Lua 以保证原子性。