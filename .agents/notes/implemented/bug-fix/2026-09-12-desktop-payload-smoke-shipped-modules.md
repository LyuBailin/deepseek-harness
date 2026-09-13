# Agent Note: Desktop payload smoke checks only shipped native modules

Status: implemented

English | [中文](2026-09-12-desktop-payload-smoke-shipped-modules.zh.md)

## Problem

`prepare:dsh` runs the bundled-Node [payload smoke](../../../../apps/desktop/tests/fixtures/runtime-payload-smoke.mjs) against the filtered `resources/dsh` tree before the Host smoke and the final inventory check. The fixture required `fs-ext` to prove that a native binding survived the copy policy, but no package in this workspace depends on `fs-ext`: the JSONL writer uses `@deepseek-ai/node-addon-system` and koffi, the prepared package set carries no `fs-ext` entry, and `pnpm-lock.yaml` lists none. `requireRuntime('fs-ext')` therefore resolves nothing, and Desktop packaging aborts at `prepare:dsh` with `Cannot find module 'fs-ext'` for every platform and Node release. The copy policy held two `fs-ext` rules and `runtime-file-policy.spec.ts` pinned their fixtures, so the same absent package also owned exclusions that no runtime tree can trigger. No CI workflow runs the Desktop packaging commands, so release packaging is the first consumer to reach this fixture.

## Decision

The payload smoke exercises the native and asset payloads the runtime tree carries: koffi's packaged binding, sharp's libvips binary, `turndown` with its GFM plugin, and a spawned node-pty terminal. Its result line reports those four checks. The fixture imports no `fs-ext`.

`desktopRuntimeFileExclusion` holds no `fs-ext` rule, and `runtime-file-policy.spec.ts` keeps the requirement that a package-specific rule matches only inside a `node_modules` container, asserted through a scoped package name outside one.

The profile `allowBuilds` entry for `fs-ext` in `apps/desktop/src/project-manager.ts` remains. It approves install scripts for dependencies resolved into `$DSH_HOME/profiles/desktop`, which admits third-party plugins, and it is not a statement that the bundled tree ships `fs-ext`.

## Alternatives considered

**Vendor `fs-ext` into the runtime tree to keep the check.** The check would then assert a payload the product deliberately left, and the tree would carry a NAN addon that compiles per Node ABI on consumer installs — the build requirement the JSONL writer's move to `@deepseek-ai/node-addon-system` removed.

**Check `@deepseek-ai/node-addon-system` instead.** Its `./flock` face rejects Windows with `ERR_FLOCK_UNSUPPORTED_PLATFORM` and resolves a platform package that only Linux and macOS publish, and its `./landlock-run` face is Linux-only. The Windows target this smoke gates can exercise neither.

**Skip the check when the module is absent.** A conditional skip also hides a module that is absent because the copy policy or an incomplete installation dropped it, which is the failure the smoke detects.

## Consequences

Desktop packaging reaches the payload smoke and the Host smoke on Windows and macOS. Coverage of shipped payloads is unchanged: the removed check asserted a binding absent from the package set, and the four remaining checks still cover a packaged FFI binding, a packaged image library, packaged HTML assets, and a packaged ConPTY terminal.

Every remaining copy-policy rule names a package the runtime tree can contain. The payload smoke's checks run only in release packaging, so a removed dependency leaves this fixture stale until the next packaging run reaches it.
