# MCP Server Privilege Inventory

Paste an MCP client config and get a privilege inventory: which servers launch a subprocess as your user, how mutable each code reference is, and which secrets you hand each one.

**Live demo:** https://0xelitesystem.github.io/mcp-server-privilege-inventory/

The headline is the part people skip. A stdio MCP server is a subprocess your client launches, so its blast radius is your entire user account, and the `env` block is a floor on what it can reach, never a ceiling. The tool prints that bound first and the enumerable secret union second, in that order, on purpose.

## Use

1. Open the page. Paste the contents of `claude_desktop_config.json`, a project `.mcp.json`, or any JSON with an `mcpServers` (or `servers`) object.
2. Press Inventory. Nothing leaves the tab.
3. Read top to bottom:
   - Account-level bound first: how many entries launch a local subprocess, and what that means regardless of what the config declares.
   - Enumerable secret union second: the distinct credential-bearing declarations across every server, labeled a floor.
   - Per-server table: transport, exec surface, mutability class, secret count, declared publisher.
   - Per-server detail: what was parsed out of `command` and `args`, and why the mutability class was assigned.
   - Receipts: the primary sources, quoted, with URLs.

Load example fills the box with a nine-server config that lands in nine different mutability classes, including the pair the tool exists to separate: `npx -y pkg` and `uvx pkg` look identical in a config and resolve on opposite schedules.

The tool reports properties of the JSON you pasted. It does not grade servers, score risk, or predict what any server's code will do. A config tells you who you are trusting and how mutable that trust is. It cannot tell you what the code does.

## Why this exists

Three claims get repeated about MCP configs and two of them are wrong.

Wrong: the `env` block is the server's permission set. It is not a sandbox. It is a set of extra variables handed to a process that already runs as you. The MCP transports specification describes stdio in one line: "The client launches the MCP server as a subprocess." A subprocess inherits your uid. Your home directory, your SSH keys, your credential store and your network are inside the bound whether or not a single variable is declared. So the union of declared secrets is a floor on reach, and the tool says so above the union rather than below it.

Right, and worth stating as a receipt rather than a hedge: a token sitting in your shell is not silently granted. The MCP debugging documentation states that MCP servers launched over stdio inherit only a limited subset of environment variables automatically, that the exact set is platform-dependent, and that you specify an `env` key to override the default variables or provide your own. The reference TypeScript SDK ships the actual allowlist. `DEFAULT_INHERITED_ENV_VARS` is 6 names on non-Windows platforms (HOME, LOGNAME, PATH, SHELL, TERM, USER, carrying a source comment that describes the list as inspired by the default env inheritance of sudo) and 12 names on win32. `getDefaultEnvironment()` also drops any value beginning with `()`, commented in the source as skipping functions because they are a security risk. The transport spawns with the default environment spread first and the server's declared `env` spread second, so a declared variable wins and an undeclared one is absent. One exception is handled explicitly: Claude Code expands `${VAR}` and `${VAR:-default}` in a project-scoped `.mcp.json`, so a value written that way is a declared grant sourced from your shell, and the tool labels it that way instead of treating it as a literal string.

Wrong, and this is the part that had to be rebuilt before any UI was written: the mutability taxonomy. The obvious version of this tool says a `:latest` Docker tag re-resolves on every launch and a pinned npm version is safe. Both halves are wrong, and the correction runs in the opposite direction from what people expect.

| What you wrote | When the code is chosen |
| --- | --- |
| `image@sha256:...` | Never re-resolved. Content-addressed. |
| `image:latest`, default pull policy | Once, on first pull. `docker run` defaults to `--pull=missing`. |
| `image:latest`, `--pull=always` | Every launch, by explicit policy. |
| `npx -y pkg` (no version) | Every launch. |
| `npx -y pkg@latest`, or a semver range | Every launch. |
| `npx -y pkg@1.2.3` | Top level once. The transitive tree is not pinned by your config. |
| `uvx pkg` (no version) | Once, on first invocation. Then cached. |
| `node /abs/path/server.js` | Whatever is on disk at launch. No registry, no integrity check. |
| `"url": "https://..."` | Not chosen locally at all. |

The two `npx` "every launch" rows are the ones that get stated backwards, and the `npx` and `uvx` rows are the reason a taxonomy keyed on "is it a package runner" instead of "which package runner" prints confident falsehoods. `npx -y pkg` and `uvx pkg` are the same shape in a config file and have opposite resolution schedules. Sources for every row are in the Receipts panel on the page.

Docker's CLI reference gives `--pull` the default `missing` and describes the policy this way: "The default (`missing`) is to only pull the image if it's not present in the daemon's image cache." So a `:latest` image is not churning under you. The real problem is narrower and less comforting: your config does not record which digest you are actually running, and two machines with the same config file can be running different images indefinitely.

npm goes the other way. In `libnpmexec`, the npx cache lookup treats a bare package name as a special case, with the source comment "If looking in the npx cache check if a newer version is available". A bare name, an explicit dist-tag and a semver range all fall through to a manifest lookup performed with pacote's `preferOnline` set to true, which pacote documents as "Prefer to revalidate cache entries, even when it would not be strictly necessary." Only an exact version short-circuits, by matching `spec.type === 'version' && node.pkgid === spec.raw` against the cached tree. One precision worth stating: the local project tree is consulted before the npx cache, and a bare name matches anything already installed there. A server launched by an MCP client normally has no project tree to match against, which is why the npx cache path is the one that governs. The practical result: `npx -y some-mcp-server` picks up a new upstream publish without you changing a character of your config.

uv is not the same. Its documentation says uvx will use the latest available version of the requested tool on the first invocation, and after that will use the cached version of the tool unless a different version is requested, the cache is pruned, or the cache is refreshed. Same config shape, opposite schedule.

## Privacy

Everything runs in the tab. There is no network request of any kind after the page loads, no analytics, and no storage of your config. The single file makes zero external requests: no fonts, no scripts, no images. The only thing written to `localStorage` is your theme choice, wrapped in `try`/`catch`.

Your config contains live credentials. The tool never transmits them, and its own output redacts literal secret values down to a character count and a short prefix rather than reproducing them, so a copied report is safe to paste into a ticket.

## Run locally

```
git clone https://github.com/0xelitesystem/mcp-server-privilege-inventory.git
cd mcp-server-privilege-inventory
open index.html
```

Any static server works too, but none is required. `index.html` opens from `file://`.

## Build

There is no build. One HTML file, inline `<style>`, inline script, no dependencies, no package manager, no toolchain. Edit `index.html` and reload.

## Related

Three repos read MCP configuration and they do different jobs. Use the one that matches your question.

- [mcp-server-config-builder](https://github.com/0xelitesystem/mcp-server-config-builder) builds and validates the config. Use it when the question is "is this JSON correct and will the client accept it".
- [mcp-tool-description-scanner](https://github.com/0xelitesystem/mcp-tool-description-scanner) reads the tool definitions a server returns and scans their text for prompt injection, hidden characters, tool shadowing and exfiltration-shaped parameters. Use it when the question is "what is this server telling the model".
- This repo inventories privilege across the servers you already installed. Use it when the question is "what did I hand these things, and how mutable is the code behind each one".

Also relevant: [mcp-basics-reference](https://github.com/0xelitesystem/mcp-basics-reference) for the protocol itself, and [mcp-public-servers-directory](https://github.com/0xelitesystem/mcp-public-servers-directory) for finding servers in the first place.

## License

MIT. See [LICENSE](LICENSE).

## More

Part of a catalog of single-file browser tools and plain-language references, all MIT licensed and dependency-free: [0xelitesystem.github.io](https://0xelitesystem.github.io/).
Built by [elitesystem.ai](https://elitesystem.ai).
