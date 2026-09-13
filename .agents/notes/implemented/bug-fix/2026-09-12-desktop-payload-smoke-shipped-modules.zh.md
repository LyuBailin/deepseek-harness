# Agent Note: 桌面产物 smoke 只校验实际交付的原生模块

Status: implemented

[English](2026-09-12-desktop-payload-smoke-shipped-modules.md) | 中文

## Problem

`prepare:dsh` 在 Host smoke 与最终清单校验之前，用内置 Node 对过滤后的 `resources/dsh` 树运行[产物 smoke](../../../../apps/desktop/tests/fixtures/runtime-payload-smoke.mjs)。该 fixture 曾用 `fs-ext` 证明原生绑定通过了复制规则，但本工作区没有任何包依赖 `fs-ext`：JSONL 写入方使用 `@deepseek-ai/node-addon-system` 与 koffi，准备好的包集合中不含 `fs-ext` 条目，`pnpm-lock.yaml` 中也没有记录。因此 `requireRuntime('fs-ext')` 解析不到任何模块，桌面打包在所有平台和 Node 版本上都会以 `Cannot find module 'fs-ext'` 停在 `prepare:dsh`。复制规则中保留了两条 `fs-ext` 规则，`runtime-file-policy.spec.ts` 也为其固定了用例，于是同一个已不存在的包还占有着任何运行时树都无法触发的排除项。没有任何 CI workflow 运行桌面打包命令，发布打包是该 fixture 的第一个消费者。

## Decision

产物 smoke 校验运行时树真正携带的原生与资源产物：koffi 的打包绑定、sharp 的 libvips 二进制、`turndown` 及其 GFM 插件，以及启动出的 node-pty 终端。其结果行报告这四项检查。该 fixture 不导入 `fs-ext`。

`desktopRuntimeFileExclusion` 不含 `fs-ext` 规则，`runtime-file-policy.spec.ts` 继续要求包专用规则只在 `node_modules` 容器内匹配，并改用容器外的作用域包名来断言这一点。

`apps/desktop/src/project-manager.ts` 中 profile 的 `fs-ext` `allowBuilds` 条目保留。它批准的是解析进 `$DSH_HOME/profiles/desktop` 的依赖的安装脚本，而该目录接纳第三方插件；它并不表示打包树携带 `fs-ext`。

## Alternatives considered

**把 `fs-ext` 置入运行时树以保留该检查。** 这样检查断言的将是产品刻意不再携带的产物，而且运行时树会带上一个在用户安装时按 Node ABI 编译的 NAN addon —— 正是 JSONL 写入方改用 `@deepseek-ai/node-addon-system` 所消除的构建要求。

**改检查 `@deepseek-ai/node-addon-system`。** 它的 `./flock` 面在 Windows 上以 `ERR_FLOCK_UNSUPPORTED_PLATFORM` 拒绝，且需要只有 Linux 和 macOS 发布的平台包；`./landlock-run` 面仅限 Linux。本 smoke 所把关的 Windows 目标两者都无法执行。

**模块缺失时跳过该检查。** 条件跳过同样会掩盖因复制规则或不完整安装而缺失的模块，而这正是该 smoke 要发现的故障。

## Consequences

桌面打包在 Windows 与 macOS 上都能走到产物 smoke 与 Host smoke。对已交付产物的覆盖不变：被删除的检查断言的是包集合中不存在的绑定，其余四项检查仍覆盖打包的 FFI 绑定、打包的图像库、打包的 HTML 资源和打包的 ConPTY 终端。

复制规则的每一条剩余规则都指向运行时树可能包含的包。产物 smoke 的检查只在发布打包时运行，因此被移除的依赖会让该 fixture 一直陈旧，直到下一次打包运行触达它。
