# Bug 修复说明：space-identity 测试在特权（root）运行环境下误报 ENOENT 失败

子包 `dsh-mnemon-source-memory-spaces` 的测试「reads on when a copy cannot be removed or written, and never fails the merge for it」（tests/space-identity.spec.ts）在以 root 身份运行测试套件时必然失败——而产品代码本身行为正确。

## 证据链

1. **全量基线**（上游 HEAD `a7e3bc549c777ada56ca703099eb61a7340b1da3`，version 0.5.34）：`pnpm install --frozen-lockfile` 后，根包 typecheck ✅；根测试（构建前）3 个失败，全部是依赖 `lib/` 构建产物的用例，`pnpm run build` + workspace 构建后单独复跑全部通过（环境前置，非缺陷）；workspace 子包测试中仅本条失败：
   `plugins/dsh-mnemon-source-memory-spaces` — 1 failed | 295 passed。
2. **单文件复现**：`pnpm exec vitest run tests/space-identity.spec.ts` → 17 passed / 1 failed，报错：
   `Error: ENOENT: no such file or directory, chmod '/tmp/dsh-mnemon-space-identity-*/.dsh-space-identity-held'`，定位在 spec 第 332 行（finally 块）。
3. **缺陷陈述**：该测试用 `chmodSync(stale, 0o500)` 把残留副本目录设为只读，以模拟「副本删不掉」的场景，再断言实现不因此失败。最后在 `finally` 里无条件 `chmodSync(stale, 0o700)` 恢复权限以便清理。

## 根因

`finally` 恢复权限的前提假设是「目录一定还在」——但目录是否被删除正是本测试的被测变量。以 root（uid 0）运行的进程会绕过 0o500 权限位，实现按设计真的把残留副本删掉了；此时 `stale` 已不存在，无条件 chmod 对不存在的路径抛 ENOENT，测试在断言全部通过之后、于清理阶段误报失败。换言之：该测试只能在非特权运行器上通过，在 root 容器（插件开发常用环境）里上游套件恒红。

## 改动文件

- `plugins/dsh-mnemon-source-memory-spaces/tests/space-identity.spec.ts`（+4/−1）：`finally` 里先 `if (existsSync(stale))` 再 chmod，并加注释说明缘由（`existsSync` 本文件已有导入）。非特权环境的原行为一字不变：目录还在时照常恢复 0o700，afterEach 的递归清理不受影响。

## 复验结果（修复后）

- `pnpm exec vitest run tests/space-identity.spec.ts` ✅ 18 passed（修复前 17 passed / 1 failed）。
- 子包全量 `pnpm run test` ✅ 26 文件 passed / 2 skipped，296 tests passed / 2 skipped，exit 0（修复前 1 failed / 295 passed）。
- 改动仅限该测试文件，未触碰任何产品代码；根包 typecheck 与其余子包结果与基线一致。

## 版本与出处

根 package.json：0.5.34 → 0.5.35（patch +1），`repository.url` 改指本仓 https://github.com/lhg-dsh-plugins/lhg-dsh-mnemon；其余字段、子包版本与依赖锁定均未动。出处见 UPSTREAM.md，LICENSE 原样保留。
