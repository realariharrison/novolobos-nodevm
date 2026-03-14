# @novolobos/nodevm — What This Is and Why We Did It

## The Short Version

`@novolobos/nodevm` is a **sandboxed JavaScript execution engine**. It lets NovoLobos run user-written code (like custom analysis scripts in the Analyst node) in a secure, isolated environment where that code can't access your filesystem, network, or other sensitive system resources.

We forked it from the original `vm2` library to own the dependency under the `@novolobos` npm scope, removing all upstream branding from the NovoLobos codebase.

---

## What Problem Does This Solve?

NovoLobos is a multi-agent workflow platform. Some of its nodes — particularly the **Analyst** (Biostatistician) and **Custom Function** nodes — allow users to write and execute JavaScript code at runtime. For example:

```javascript
// A user might write this in an Analyst node:
const pValue = $houndData.results.map(r => r.significance);
const significant = pValue.filter(p => p < 0.05);
return { significant_count: significant.length, total: pValue.length };
```

The question is: **where does this code run?**

If you just `eval()` it in Node.js, that user code has full access to everything — it could read files, make network requests, access environment variables (API keys, database credentials), or even crash the server. That's a massive security risk, especially in a health & life sciences platform handling sensitive research data.

**This is where `NodeVM` comes in.** It creates a sandboxed environment:

```javascript
import { NodeVM } from '@novolobos/nodevm'

const vm = new NodeVM({
    sandbox: {
        $houndData: { results: [...] },   // Only these variables are visible
        $protocol: { studyType: 'RCT' },
        $state: { phase: 'analysis' }
    },
    require: {
        external: false,     // Can't require external packages
        builtin: ['path'],   // Only whitelisted Node.js builtins
    }
})

// User code runs here — isolated from the host system
const result = vm.run(`
    const pValue = $houndData.results.map(r => r.significance);
    return { count: pValue.length };
`)
```

The user's code can only see `$houndData`, `$protocol`, and `$state` — nothing else. No `process.env`, no `fs`, no `require('child_process')`. It's like running code inside a locked room with only the tools you hand through the window.

---

## The History: vm2 → @flowiseai/nodevm → @novolobos/nodevm

### 1. vm2 (the original)

**Created by:** Patrik Šimek ([patriksimek/vm2](https://github.com/patriksimek/vm2))
**What it is:** A Node.js library for running untrusted JavaScript in a sandboxed environment.
**Status:** Archived/deprecated by the author (he stopped maintaining it), but still functional.

`vm2` wraps Node.js's built-in `vm` module with additional security layers:
- **Context isolation** — sandboxed code can't access the host's global scope
- **Module control** — you whitelist which `require()` calls are allowed
- **Timeout support** — kill runaway scripts after N milliseconds
- **Proxy-based protection** — prevents sandbox escape techniques

### 2. @flowiseai/nodevm (the upstream fork)

**Created by:** FlowiseAI (the project NovoLobos was forked from)
**What they did:** Took `vm2`, made minor patches, and published it under their npm scope (`@flowiseai/nodevm`) so they controlled the dependency.
**Why:** They needed a maintained version of `vm2` for their custom function nodes, and the original was abandoned.

### 3. @novolobos/nodevm (our fork)

**Created by:** You (Ari Harrison)
**What we did:** Forked the original `patriksimek/vm2` to `realariharrison/novolobos-nodevm`, changed the package name to `@novolobos/nodevm`, and published to npm.
**Why:** To remove all `@flowiseai` branding from the NovoLobos codebase. NovoLobos is its own product — it shouldn't depend on upstream's npm scope for a critical security component.

---

## Where It's Used in NovoLobos

**File:** `packages/components/src/utils.ts`

```javascript
import { NodeVM } from '@novolobos/nodevm'
```

This import is used by:
- **Custom Function nodes** — any agentflow node that executes user-written JavaScript
- **Analyst node** — when running custom statistical analysis code (beyond the 8 built-in methods)
- **Tool nodes** — custom tool implementations that include JavaScript logic

The `availableDependencies` array in the same file controls which npm packages the sandboxed code is allowed to `require()`. This is an additional security layer on top of the VM sandbox.

---

## How npm Scoped Packages Work

When you see `@novolobos/nodevm`, the `@novolobos` part is called a **scope**. Here's how it works:

```
@novolobos/nodevm
 ^          ^
 |          |
 scope      package name
```

- **Scopes** are npm organizations. You created one at npmjs.com/org/novolobos.
- **Scoped packages** are namespaced — `@novolobos/nodevm` won't conflict with `@flowiseai/nodevm` or plain `vm2`, even though they contain the same code.
- **Public vs Private:** Scoped packages default to private (paid). The `--access public` flag makes them free and publicly installable.
- **Anyone can install it:** `pnpm add @novolobos/nodevm` works for anyone, anywhere.
- **Only org members can publish:** Only people in the `@novolobos` npm org can push new versions.

---

## How to Update It

If you ever need to patch the VM (security fix, new feature):

```bash
cd ~/Documents/Github/novolobos-nodevm

# Make your changes, then bump the version
npm version patch   # 3.10.5 → 3.10.6

# Publish
npm publish --access public

# Update NovoLobos
cd ~/Documents/Github/NovoLobos
pnpm update @novolobos/nodevm
```

---

## Security Considerations

`vm2` (and by extension `@novolobos/nodevm`) was deprecated by its author partly because **no JavaScript sandbox is 100% escape-proof**. There have been CVEs where clever code could break out of the sandbox.

For NovoLobos, this is acceptable because:
1. The platform is self-hosted — the user running code is typically the same person who deployed the server
2. The `availableDependencies` whitelist adds an additional layer
3. The HIPAA/compliance use case means access is already controlled
4. The alternative (no sandboxing at all) is far worse

If you want stronger isolation in the future, consider migrating to `isolated-vm` (uses V8 isolates — true process-level isolation) or running user code in Docker containers.

---

## Summary

| Layer | What | Purpose |
|-------|------|---------|
| `vm2` (original) | Patrik Šimek's sandbox library | Run untrusted JS safely |
| `@flowiseai/nodevm` | Flowise's fork | Their branded dependency |
| `@novolobos/nodevm` | Your fork | Your branded dependency, zero upstream ties |
| `NodeVM` class | The actual API you use | Creates isolated execution contexts |
| `availableDependencies` | Whitelist in utils.ts | Controls what sandboxed code can import |

**Bottom line:** You now own every dependency in the NovoLobos stack under your own npm scope. No more `@flowiseai` anywhere.
