     1→> ## Documentation Index
     2→> Fetch the complete documentation index at: https://code.claude.com/docs/llms.txt
     3→> Use this file to discover all available pages before exploring further.
     4→
     5→# Claude Code settings
     6→
     7→> Configure Claude Code with global and project-level settings, and environment variables.
     8→
     9→Claude Code offers a variety of settings to configure its behavior to meet your needs. You can configure Claude Code by running the `/config` command when using the interactive REPL, which opens a tabbed Settings interface where you can view status information and modify configuration options.
    10→
    11→## Configuration scopes
    12→
    13→Claude Code uses a **scope system** to determine where configurations apply and who they're shared with. Understanding scopes helps you decide how to configure Claude Code for personal use, team collaboration, or enterprise deployment.
    14→
    15→### Available scopes
    16→
    17→| Scope       | Location                                                                           | Who it affects                       | Shared with team?      |
    18→| :---------- | :--------------------------------------------------------------------------------- | :----------------------------------- | :--------------------- |
    19→| **Managed** | Server-managed settings, plist / registry, or system-level `managed-settings.json` | All users on the machine             | Yes (deployed by IT)   |
    20→| **User**    | `~/.claude/` directory                                                             | You, across all projects             | No                     |
    21→| **Project** | `.claude/` in repository                                                           | All collaborators on this repository | Yes (committed to git) |
    22→| **Local**   | `.claude/settings.local.json`                                                      | You, in this repository only         | No (gitignored)        |
    23→
    24→### When to use each scope
    25→
    26→**Managed scope** is for:
    27→
    28→* Security policies that must be enforced organization-wide
    29→* Compliance requirements that can't be overridden
    30→* Standardized configurations deployed by IT/DevOps
    31→
    32→**User scope** is best for:
    33→
    34→* Personal preferences you want everywhere (themes, editor settings)
    35→* Tools and plugins you use across all projects
    36→* API keys and authentication (stored securely)
    37→
    38→**Project scope** is best for:
    39→
    40→* Team-shared settings (permissions, hooks, MCP servers)
    41→* Plugins the whole team should have
    42→* Standardizing tooling across collaborators
    43→
    44→**Local scope** is best for:
    45→
    46→* Personal overrides for a specific project
    47→* Testing configurations before sharing with the team
    48→* Machine-specific settings that won't work for others
    49→
    50→### How scopes interact
    51→
    52→When the same setting is configured in multiple scopes, more specific scopes take precedence:
    53→
    54→1. **Managed** (highest) - can't be overridden by anything
    55→2. **Command line arguments** - temporary session overrides
    56→3. **Local** - overrides project and user settings
    57→4. **Project** - overrides user settings
    58→5. **User** (lowest) - applies when nothing else specifies the setting
    59→
    60→For example, if a permission is allowed in user settings but denied in project settings, the project setting takes precedence and the permission is blocked.
    61→
    62→### What uses scopes
    63→
    64→Scopes apply to many Claude Code features:
    65→
    66→| Feature         | User location             | Project location                   | Local location                 |
    67→| :-------------- | :------------------------ | :--------------------------------- | :----------------------------- |
    68→| **Settings**    | `~/.claude/settings.json` | `.claude/settings.json`            | `.claude/settings.local.json`  |
    69→| **Subagents**   | `~/.claude/agents/`       | `.claude/agents/`                  | None                           |
    70→| **MCP servers** | `~/.claude.json`          | `.mcp.json`                        | `~/.claude.json` (per-project) |
    71→| **Plugins**     | `~/.claude/settings.json` | `.claude/settings.json`            | `.claude/settings.local.json`  |
    72→| **CLAUDE.md**   | `~/.claude/CLAUDE.md`     | `CLAUDE.md` or `.claude/CLAUDE.md` | None                           |
    73→
    74→***
    75→
    76→## Settings files
    77→
    78→The `settings.json` file is the official mechanism for configuring Claude
    79→Code through hierarchical settings:
    80→
    81→* **User settings** are defined in `~/.claude/settings.json` and apply to all
    82→  projects.
    83→* **Project settings** are saved in your project directory:
    84→  * `.claude/settings.json` for settings that are checked into source control and shared with your team
    85→  * `.claude/settings.local.json` for settings that are not checked in, useful for personal preferences and experimentation. Claude Code will configure git to ignore `.claude/settings.local.json` when it is created.
    86→* **Managed settings**: For organizations that need centralized control, Claude Code supports multiple delivery mechanisms for managed settings. All use the same JSON format and cannot be overridden by user or project settings:
    87→
    88→  * **Server-managed settings**: delivered from Anthropic's servers via the Claude.ai admin console. See [server-managed settings](/en/server-managed-settings).
    89→  * **MDM/OS-level policies**: delivered through native device management on macOS and Windows:
    90→    * macOS: `com.anthropic.claudecode` managed preferences domain (deployed via configuration profiles in Jamf, Kandji, or other MDM tools)
    91→    * Windows: `HKLM\SOFTWARE\Policies\ClaudeCode` registry key with a `Settings` value (REG\_SZ or REG\_EXPAND\_SZ) containing JSON (deployed via Group Policy or Intune)
    92→    * Windows (user-level): `HKCU\SOFTWARE\Policies\ClaudeCode` (lowest policy priority, only used when no admin-level source exists)
    93→  * **File-based**: `managed-settings.json` and `managed-mcp.json` deployed to system directories:
    94→
    95→    * macOS: `/Library/Application Support/ClaudeCode/`
    96→    * Linux and WSL: `/etc/claude-code/`
    97→    * Windows: `C:\Program Files\ClaudeCode\`
    98→
    99→    <Warning>
   100→      The legacy Windows path `C:\ProgramData\ClaudeCode\managed-settings.json` is no longer supported as of v2.1.75. Administrators who deployed settings to that location must migrate files to `C:\Program Files\ClaudeCode\managed-settings.json`.
   101→    </Warning>
   102→
   103→  See [managed settings](/en/permissions#managed-only-settings) and [Managed MCP configuration](/en/mcp#managed-mcp-configuration) for details.
   104→
   105→  <Note>
   106→    Managed deployments can also restrict **plugin marketplace additions** using
   107→    `strictKnownMarketplaces`. For more information, see [Managed marketplace restrictions](/en/plugin-marketplaces#managed-marketplace-restrictions).
   108→  </Note>
   109→* **Other configuration** is stored in `~/.claude.json`. This file contains your preferences (theme, notification settings, editor mode), OAuth session, [MCP server](/en/mcp) configurations for user and local scopes, per-project state (allowed tools, trust settings), and various caches. Project-scoped MCP servers are stored separately in `.mcp.json`.
   110→
   111→<Note>
   112→  Claude Code automatically creates timestamped backups of configuration files and retains the five most recent backups to prevent data loss.
   113→</Note>
   114→
   115→```JSON Example settings.json theme={null}
   116→{
   117→  "$schema": "https://json.schemastore.org/claude-code-settings.json",
   118→  "permissions": {
   119→    "allow": [
   120→      "Bash(npm run lint)",
   121→      "Bash(npm run test *)",
   122→      "Read(~/.zshrc)"
   123→    ],
   124→    "deny": [
   125→      "Bash(curl *)",
   126→      "Read(./.env)",
   127→      "Read(./.env.*)",
   128→      "Read(./secrets/**)"
   129→    ]
   130→  },
   131→  "env": {
   132→    "CLAUDE_CODE_ENABLE_TELEMETRY": "1",
   133→    "OTEL_METRICS_EXPORTER": "otlp"
   134→  },
   135→  "companyAnnouncements": [
   136→    "Welcome to Acme Corp! Review our code guidelines at docs.acme.com",
   137→    "Reminder: Code reviews required for all PRs",
   138→    "New security policy in effect"
   139→  ]
   140→}
   141→```
   142→
   143→The `$schema` line in the example above points to the [official JSON schema](https://json.schemastore.org/claude-code-settings.json) for Claude Code settings. Adding it to your `settings.json` enables autocomplete and inline validation in VS Code, Cursor, and any other editor that supports JSON schema validation.
   144→
   145→### Available settings
   146→
   147→`settings.json` supports a number of options:
   148→
   149→| Key                               | Description                                                                                                                                                                                                                                                                                                                  | Example                                                                 |
   150→| :-------------------------------- | :--------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :---------------------------------------------------------------------- |
   151→| `apiKeyHelper`                    | Custom script, to be executed in `/bin/sh`, to generate an auth value. This value will be sent as `X-Api-Key` and `Authorization: Bearer` headers for model requests                                                                                                                                                         | `/bin/generate_temp_api_key.sh`                                         |
   152→| `autoMemoryDirectory`             | Custom directory for [auto memory](/en/memory#storage-location) storage. Accepts `~/`-expanded paths. Not accepted in project settings (`.claude/settings.json`) to prevent shared repos from redirecting memory writes to sensitive locations. Accepted from policy, local, and user settings                               | `"~/my-memory-dir"`                                                     |
   153→| `cleanupPeriodDays`               | Sessions inactive for longer than this period are deleted at startup (default: 30 days).<br /><br />Setting to `0` deletes all existing transcripts at startup and disables session persistence entirely. No new `.jsonl` files are written, `/resume` shows no conversations, and hooks receive an empty `transcript_path`. | `20`                                                                    |
   154→| `companyAnnouncements`            | Announcement to display to users at startup. If multiple announcements are provided, they will be cycled through at random.                                                                                                                                                                                                  | `["Welcome to Acme Corp! Review our code guidelines at docs.acme.com"]` |
   155→| `env`                             | Environment variables that will be applied to every session                                                                                                                                                                                                                                                                  | `{"FOO": "bar"}`                                                        |
   156→| `attribution`                     | Customize attribution for git commits and pull requests. See [Attribution settings](#attribution-settings)                                                                                                                                                                                                                   | `{"commit": "🤖 Generated with Claude Code", "pr": ""}`                 |
   157→| `includeCoAuthoredBy`             | **Deprecated**: Use `attribution` instead. Whether to include the `co-authored-by Claude` byline in git commits and pull requests (default: `true`)                                                                                                                                                                          | `false`                                                                 |
   158→| `includeGitInstructions`          | Include built-in commit and PR workflow instructions in Claude's system prompt (default: `true`). Set to `false` to remove these instructions, for example when using your own git workflow skills. The `CLAUDE_CODE_DISABLE_GIT_INSTRUCTIONS` environment variable takes precedence over this setting when set              | `false`                                                                 |
   159→| `permissions`                     | See table below for structure of permissions.                                                                                                                                                                                                                                                                                |                                                                         |
   160→| `hooks`                           | Configure custom commands to run at lifecycle events. See [hooks documentation](/en/hooks) for format                                                                                                                                                                                                                        | See [hooks](/en/hooks)                                                  |
   161→| `disableAllHooks`                 | Disable all [hooks](/en/hooks) and any custom [status line](/en/statusline)                                                                                                                                                                                                                                                  | `true`                                                                  |
   162→| `allowManagedHooksOnly`           | (Managed settings only) Prevent loading of user, project, and plugin hooks. Only allows managed hooks and SDK hooks. See [Hook configuration](#hook-configuration)                                                                                                                                                           | `true`                                                                  |
   163→| `allowedHttpHookUrls`             | Allowlist of URL patterns that HTTP hooks may target. Supports `*` as a wildcard. When set, hooks with non-matching URLs are blocked. Undefined = no restriction, empty array = block all HTTP hooks. Arrays merge across settings sources. See [Hook configuration](#hook-configuration)                                    | `["https://hooks.example.com/*"]`                                       |
   164→| `httpHookAllowedEnvVars`          | Allowlist of environment variable names HTTP hooks may interpolate into headers. When set, each hook's effective `allowedEnvVars` is the intersection with this list. Undefined = no restriction. Arrays merge across settings sources. See [Hook configuration](#hook-configuration)                                        | `["MY_TOKEN", "HOOK_SECRET"]`                                           |
   165→| `allowManagedPermissionRulesOnly` | (Managed settings only) Prevent user and project settings from defining `allow`, `ask`, or `deny` permission rules. Only rules in managed settings apply. See [Managed-only settings](/en/permissions#managed-only-settings)                                                                                                 | `true`                                                                  |
   166→| `allowManagedMcpServersOnly`      | (Managed settings only) Only `allowedMcpServers` from managed settings are respected. `deniedMcpServers` still merges from all sources. Users can still add MCP servers, but only the admin-defined allowlist applies. See [Managed MCP configuration](/en/mcp#managed-mcp-configuration)                                    | `true`                                                                  |
   167→| `model`                           | Override the default model to use for Claude Code                                                                                                                                                                                                                                                                            | `"claude-sonnet-4-6"`                                                   |
   168→| `availableModels`                 | Restrict which models users can select via `/model`, `--model`, Config tool, or `ANTHROPIC_MODEL`. Does not affect the Default option. See [Restrict model selection](/en/model-config#restrict-model-selection)                                                                                                             | `["sonnet", "haiku"]`                                                   |
   169→| `modelOverrides`                  | Map Anthropic model IDs to provider-specific model IDs such as Bedrock inference profile ARNs. Each model picker entry uses its mapped value when calling the provider API. See [Override model IDs per version](/en/model-config#override-model-ids-per-version)                                                            | `{"claude-opus-4-6": "arn:aws:bedrock:..."}`                            |
   170→| `effortLevel`                     | Persist the [effort level](/en/model-config#adjust-effort-level) across sessions. Accepts `"low"`, `"medium"`, or `"high"`. Written automatically when you run `/effort low`, `/effort medium`, or `/effort high`. Supported on Opus 4.6 and Sonnet 4.6                                                                      | `"medium"`                                                              |
   171→| `otelHeadersHelper`               | Script to generate dynamic OpenTelemetry headers. Runs at startup and periodically (see [Dynamic headers](/en/monitoring-usage#dynamic-headers))                                                                                                                                                                             | `/bin/generate_otel_headers.sh`                                         |
   172→| `statusLine`                      | Configure a custom status line to display context. See [`statusLine` documentation](/en/statusline)                                                                                                                                                                                                                          | `{"type": "command", "command": "~/.claude/statusline.sh"}`             |
   173→| `fileSuggestion`                  | Configure a custom script for `@` file autocomplete. See [File suggestion settings](#file-suggestion-settings)                                                                                                                                                                                                               | `{"type": "command", "command": "~/.claude/file-suggestion.sh"}`        |
   174→| `respectGitignore`                | Control whether the `@` file picker respects `.gitignore` patterns. When `true` (default), files matching `.gitignore` patterns are excluded from suggestions                                                                                                                                                                | `false`                                                                 |
   175→| `outputStyle`                     | Configure an output style to adjust the system prompt. See [output styles documentation](/en/output-styles)                                                                                                                                                                                                                  | `"Explanatory"`                                                         |
   176→| `forceLoginMethod`                | Use `claudeai` to restrict login to Claude.ai accounts, `console` to restrict login to Claude Console (API usage billing) accounts                                                                                                                                                                                           | `claudeai`                                                              |
   177→| `forceLoginOrgUUID`               | Specify the UUID of an organization to automatically select it during login, bypassing the organization selection step. Requires `forceLoginMethod` to be set                                                                                                                                                                | `"xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx"`                                |
   178→| `enableAllProjectMcpServers`      | Automatically approve all MCP servers defined in project `.mcp.json` files                                                                                                                                                                                                                                                   | `true`                                                                  |
   179→| `enabledMcpjsonServers`           | List of specific MCP servers from `.mcp.json` files to approve                                                                                                                                                                                                                                                               | `["memory", "github"]`                                                  |
   180→| `disabledMcpjsonServers`          | List of specific MCP servers from `.mcp.json` files to reject                                                                                                                                                                                                                                                                | `["filesystem"]`                                                        |
   181→| `allowedMcpServers`               | When set in managed-settings.json, allowlist of MCP servers users can configure. Undefined = no restrictions, empty array = lockdown. Applies to all scopes. Denylist takes precedence. See [Managed MCP configuration](/en/mcp#managed-mcp-configuration)                                                                   | `[{ "serverName": "github" }]`                                          |
   182→| `deniedMcpServers`                | When set in managed-settings.json, denylist of MCP servers that are explicitly blocked. Applies to all scopes including managed servers. Denylist takes precedence over allowlist. See [Managed MCP configuration](/en/mcp#managed-mcp-configuration)                                                                        | `[{ "serverName": "filesystem" }]`                                      |
   183→| `strictKnownMarketplaces`         | When set in managed-settings.json, allowlist of plugin marketplaces users can add. Undefined = no restrictions, empty array = lockdown. Applies to marketplace additions only. See [Managed marketplace restrictions](/en/plugin-marketplaces#managed-marketplace-restrictions)                                              | `[{ "source": "github", "repo": "acme-corp/plugins" }]`                 |
   184→| `blockedMarketplaces`             | (Managed settings only) Blocklist of marketplace sources. Blocked sources are checked before downloading, so they never touch the filesystem. See [Managed marketplace restrictions](/en/plugin-marketplaces#managed-marketplace-restrictions)                                                                               | `[{ "source": "github", "repo": "untrusted/plugins" }]`                 |
   185→| `pluginTrustMessage`              | (Managed settings only) Custom message appended to the plugin trust warning shown before installation. Use this to add organization-specific context, for example to confirm that plugins from your internal marketplace are vetted.                                                                                         | `"All plugins from our marketplace are approved by IT"`                 |
   186→| `awsAuthRefresh`                  | Custom script that modifies the `.aws` directory (see [advanced credential configuration](/en/amazon-bedrock#advanced-credential-configuration))                                                                                                                                                                             | `aws sso login --profile myprofile`                                     |
   187→| `awsCredentialExport`             | Custom script that outputs JSON with AWS credentials (see [advanced credential configuration](/en/amazon-bedrock#advanced-credential-configuration))                                                                                                                                                                         | `/bin/generate_aws_grant.sh`                                            |
   188→| `alwaysThinkingEnabled`           | Enable [extended thinking](/en/common-workflows#use-extended-thinking-thinking-mode) by default for all sessions. Typically configured via the `/config` command rather than editing directly                                                                                                                                | `true`                                                                  |
   189→| `plansDirectory`                  | Customize where plan files are stored. Path is relative to project root. Default: `~/.claude/plans`                                                                                                                                                                                                                          | `"./plans"`                                                             |
   190→| `showTurnDuration`                | Show turn duration messages after responses (e.g., "Cooked for 1m 6s"). Set to `false` to hide these messages                                                                                                                                                                                                                | `true`                                                                  |
   191→| `spinnerVerbs`                    | Customize the action verbs shown in the spinner and turn duration messages. Set `mode` to `"replace"` to use only your verbs, or `"append"` to add them to the defaults                                                                                                                                                      | `{"mode": "append", "verbs": ["Pondering", "Crafting"]}`                |
   192→| `language`                        | Configure Claude's preferred response language (e.g., `"japanese"`, `"spanish"`, `"french"`). Claude will respond in this language by default                                                                                                                                                                                | `"japanese"`                                                            |
   193→| `autoUpdatesChannel`              | Release channel to follow for updates. Use `"stable"` for a version that is typically about one week old and skips versions with major regressions, or `"latest"` (default) for the most recent release                                                                                                                      | `"stable"`                                                              |
   194→| `spinnerTipsEnabled`              | Show tips in the spinner while Claude is working. Set to `false` to disable tips (default: `true`)                                                                                                                                                                                                                           | `false`                                                                 |
   195→| `spinnerTipsOverride`             | Override spinner tips with custom strings. `tips`: array of tip strings. `excludeDefault`: if `true`, only show custom tips; if `false` or absent, custom tips are merged with built-in tips                                                                                                                                 | `{ "excludeDefault": true, "tips": ["Use our internal tool X"] }`       |
   196→| `terminalProgressBarEnabled`      | Enable the terminal progress bar that shows progress in supported terminals like Windows Terminal and iTerm2 (default: `true`)                                                                                                                                                                                               | `false`                                                                 |
   197→| `prefersReducedMotion`            | Reduce or disable UI animations (spinners, shimmer, flash effects) for accessibility                                                                                                                                                                                                                                         | `true`                                                                  |
   198→| `fastModePerSessionOptIn`         | When `true`, fast mode does not persist across sessions. Each session starts with fast mode off, requiring users to enable it with `/fast`. The user's fast mode preference is still saved. See [Require per-session opt-in](/en/fast-mode#require-per-session-opt-in)                                                       | `true`                                                                  |
   199→| `teammateMode`                    | How [agent team](/en/agent-teams) teammates display: `auto` (picks split panes in tmux or iTerm2, in-process otherwise), `in-process`, or `tmux`. See [set up agent teams](/en/agent-teams#set-up-agent-teams)                                                                                                               | `"in-process"`                                                          |
   200→
   201→### Permission settings
   202→
   203→| Keys                           | Description                                                                                                                                                                                                                                      | Example                                                                |
   204→| :----------------------------- | :----------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :--------------------------------------------------------------------- |
   205→| `allow`                        | Array of permission rules to allow tool use. See [Permission rule syntax](#permission-rule-syntax) below for pattern matching details                                                                                                            | `[ "Bash(git diff *)" ]`                                               |
   206→| `ask`                          | Array of permission rules to ask for confirmation upon tool use. See [Permission rule syntax](#permission-rule-syntax) below                                                                                                                     | `[ "Bash(git push *)" ]`                                               |
   207→| `deny`                         | Array of permission rules to deny tool use. Use this to exclude sensitive files from Claude Code access. See [Permission rule syntax](#permission-rule-syntax) and [Bash permission limitations](/en/permissions#tool-specific-permission-rules) | `[ "WebFetch", "Bash(curl *)", "Read(./.env)", "Read(./secrets/**)" ]` |
   208→| `additionalDirectories`        | Additional [working directories](/en/permissions#working-directories) that Claude has access to                                                                                                                                                  | `[ "../docs/" ]`                                                       |
   209→| `defaultMode`                  | Default [permission mode](/en/permissions#permission-modes) when opening Claude Code                                                                                                                                                             | `"acceptEdits"`                                                        |
   210→| `disableBypassPermissionsMode` | Set to `"disable"` to prevent `bypassPermissions` mode from being activated. This disables the `--dangerously-skip-permissions` command-line flag. See [managed settings](/en/permissions#managed-only-settings)                                 | `"disable"`                                                            |
   211→
   212→### Permission rule syntax
   213→
   214→Permission rules follow the format `Tool` or `Tool(specifier)`. Rules are evaluated in order: deny rules first, then ask, then allow. The first matching rule wins.
   215→
   216→Quick examples:
   217→
   218→| Rule                           | Effect                                   |
   219→| :----------------------------- | :--------------------------------------- |
   220→| `Bash`                         | Matches all Bash commands                |
   221→| `Bash(npm run *)`              | Matches commands starting with `npm run` |
   222→| `Read(./.env)`                 | Matches reading the `.env` file          |
   223→| `WebFetch(domain:example.com)` | Matches fetch requests to example.com    |
   224→
   225→For the complete rule syntax reference, including wildcard behavior, tool-specific patterns for Read, Edit, WebFetch, MCP, and Agent rules, and security limitations of Bash patterns, see [Permission rule syntax](/en/permissions#permission-rule-syntax).
   226→
   227→### Sandbox settings
   228→
   229→Configure advanced sandboxing behavior. Sandboxing isolates bash commands from your filesystem and network. See [Sandboxing](/en/sandboxing) for details.
   230→
   231→| Keys                              | Description                                                                                                                                                                                                                                                                                                                                     | Example                         |
   232→| :-------------------------------- | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | :------------------------------ |
   233→| `enabled`                         | Enable bash sandboxing (macOS, Linux, and WSL2). Default: false                                                                                                                                                                                                                                                                                 | `true`                          |
   234→| `autoAllowBashIfSandboxed`        | Auto-approve bash commands when sandboxed. Default: true                                                                                                                                                                                                                                                                                        | `true`                          |
   235→| `excludedCommands`                | Commands that should run outside of the sandbox                                                                                                                                                                                                                                                                                                 | `["git", "docker"]`             |
   236→| `allowUnsandboxedCommands`        | Allow commands to run outside the sandbox via the `dangerouslyDisableSandbox` parameter. When set to `false`, the `dangerouslyDisableSandbox` escape hatch is completely disabled and all commands must run sandboxed (or be in `excludedCommands`). Useful for enterprise policies that require strict sandboxing. Default: true               | `false`                         |
   237→| `filesystem.allowWrite`           | Additional paths where sandboxed commands can write. Arrays are merged across all settings scopes: user, project, and managed paths are combined, not replaced. Also merged with paths from `Edit(...)` allow permission rules. See [path prefixes](#sandbox-path-prefixes) below.                                                              | `["//tmp/build", "~/.kube"]`    |
   238→| `filesystem.denyWrite`            | Paths where sandboxed commands cannot write. Arrays are merged across all settings scopes. Also merged with paths from `Edit(...)` deny permission rules.                                                                                                                                                                                       | `["//etc", "//usr/local/bin"]`  |
   239→| `filesystem.denyRead`             | Paths where sandboxed commands cannot read. Arrays are merged across all settings scopes. Also merged with paths from `Read(...)` deny permission rules.                                                                                                                                                                                        | `["~/.aws/credentials"]`        |
   240→| `network.allowUnixSockets`        | Unix socket paths accessible in sandbox (for SSH agents, etc.)                                                                                                                                                                                                                                                                                  | `["~/.ssh/agent-socket"]`       |
   241→| `network.allowAllUnixSockets`     | Allow all Unix socket connections in sandbox. Default: false                                                                                                                                                                                                                                                                                    | `true`                          |
   242→| `network.allowLocalBinding`       | Allow binding to localhost ports (macOS only). Default: false                                                                                                                                                                                                                                                                                   | `true`                          |
   243→| `network.allowedDomains`          | Array of domains to allow for outbound network traffic. Supports wildcards (e.g., `*.example.com`).                                                                                                                                                                                                                                             | `["github.com", "*.npmjs.org"]` |
   244→| `network.allowManagedDomainsOnly` | (Managed settings only) Only `allowedDomains` and `WebFetch(domain:...)` allow rules from managed settings are respected. Domains from user, project, and local settings are ignored. Non-allowed domains are blocked automatically without prompting the user. Denied domains are still respected from all sources. Default: false             | `true`                          |
   245→| `network.httpProxyPort`           | HTTP proxy port used if you wish to bring your own proxy. If not specified, Claude will run its own proxy.                                                                                                                                                                                                                                      | `8080`                          |
   246→| `network.socksProxyPort`          | SOCKS5 proxy port used if you wish to bring your own proxy. If not specified, Claude will run its own proxy.                                                                                                                                                                                                                                    | `8081`                          |
   247→| `enableWeakerNestedSandbox`       | Enable weaker sandbox for unprivileged Docker environments (Linux and WSL2 only). **Reduces security.** Default: false                                                                                                                                                                                                                          | `true`                          |
   248→| `enableWeakerNetworkIsolation`    | (macOS only) Allow access to the system TLS trust service (`com.apple.trustd.agent`) in the sandbox. Required for Go-based tools like `gh`, `gcloud`, and `terraform` to verify TLS certificates when using `httpProxyPort` with a MITM proxy and custom CA. **Reduces security** by opening a potential data exfiltration path. Default: false | `true`                          |
   249→
   250→#### Sandbox path prefixes
   251→
   252→Paths in `filesystem.allowWrite`, `filesystem.denyWrite`, and `filesystem.denyRead` support these prefixes:
   253→
   254→| Prefix            | Meaning                                     | Example                                |
   255→| :---------------- | :------------------------------------------ | :------------------------------------- |
   256→| `//`              | Absolute path from filesystem root          | `//tmp/build` becomes `/tmp/build`     |
   257→| `~/`              | Relative to home directory                  | `~/.kube` becomes `$HOME/.kube`        |
   258→| `/`               | Relative to the settings file's directory   | `/build` becomes `$SETTINGS_DIR/build` |
   259→| `./` or no prefix | Relative path (resolved by sandbox runtime) | `./output`                             |
   260→
   261→**Configuration example:**
   262→
   263→```json  theme={null}
   264→{
   265→  "sandbox": {
   266→    "enabled": true,
   267→    "autoAllowBashIfSandboxed": true,
   268→    "excludedCommands": ["docker"],
   269→    "filesystem": {
   270→      "allowWrite": ["//tmp/build", "~/.kube"],
   271→      "denyRead": ["~/.aws/credentials"]
   272→    },
   273→    "network": {
   274→      "allowedDomains": ["github.com", "*.npmjs.org", "registry.yarnpkg.com"],
   275→      "allowUnixSockets": [
   276→        "/var/run/docker.sock"
   277→      ],
   278→      "allowLocalBinding": true
   279→    }
   280→  }
   281→}
   282→```
   283→
   284→**Filesystem and network restrictions** can be configured in two ways that are merged together:
   285→
   286→* **`sandbox.filesystem` settings** (shown above): Control paths at the OS-level sandbox boundary. These restrictions apply to all subprocess commands (e.g., `kubectl`, `terraform`, `npm`), not just Claude's file tools.
   287→* **Permission rules**: Use `Edit` allow/deny rules to control Claude's file tool access, `Read` deny rules to block reads, and `WebFetch` allow/deny rules to control network domains. Paths from these rules are also merged into the sandbox configuration.
   288→
   289→### Attribution settings
   290→
   291→Claude Code adds attribution to git commits and pull requests. These are configured separately:
   292→
   293→* Commits use [git trailers](https://git-scm.com/docs/git-interpret-trailers) (like `Co-Authored-By`) by default,  which can be customized or disabled
   294→* Pull request descriptions are plain text
   295→
   296→| Keys     | Description                                                                                |
   297→| :------- | :----------------------------------------------------------------------------------------- |
   298→| `commit` | Attribution for git commits, including any trailers. Empty string hides commit attribution |
   299→| `pr`     | Attribution for pull request descriptions. Empty string hides pull request attribution     |
   300→
   301→**Default commit attribution:**
   302→
   303→```text  theme={null}
   304→🤖 Generated with [Claude Code](https://claude.com/claude-code)
   305→
   306→   Co-Authored-By: Claude Sonnet 4.6 <noreply@anthropic.com>
   307→```
   308→
   309→**Default pull request attribution:**
   310→
   311→```text  theme={null}
   312→🤖 Generated with [Claude Code](https://claude.com/claude-code)
   313→```
   314→
   315→**Example:**
   316→
   317→```json  theme={null}
   318→{
   319→  "attribution": {
   320→    "commit": "Generated with AI\n\nCo-Authored-By: AI <ai@example.com>",
   321→    "pr": ""
   322→  }
   323→}
   324→```
   325→
   326→<Note>
   327→  The `attribution` setting takes precedence over the deprecated `includeCoAuthoredBy` setting. To hide all attribution, set `commit` and `pr` to empty strings.
   328→</Note>
   329→
   330→### File suggestion settings
   331→
   332→Configure a custom command for `@` file path autocomplete. The built-in file suggestion uses fast filesystem traversal, but large monorepos may benefit from project-specific indexing such as a pre-built file index or custom tooling.
   333→
   334→```json  theme={null}
   335→{
   336→  "fileSuggestion": {
   337→    "type": "command",
   338→    "command": "~/.claude/file-suggestion.sh"
   339→  }
   340→}
   341→```
   342→
   343→The command runs with the same environment variables as [hooks](/en/hooks), including `CLAUDE_PROJECT_DIR`. It receives JSON via stdin with a `query` field:
   344→
   345→```json  theme={null}
   346→{"query": "src/comp"}
   347→```
   348→
   349→Output newline-separated file paths to stdout (currently limited to 15):
   350→
   351→```text  theme={null}
   352→src/components/Button.tsx
   353→src/components/Modal.tsx
   354→src/components/Form.tsx
   355→```
   356→
   357→**Example:**
   358→
   359→```bash  theme={null}
   360→#!/bin/bash
   361→query=$(cat | jq -r '.query')
   362→your-repo-file-index --query "$query" | head -20
   363→```
   364→
   365→### Hook configuration
   366→
   367→These settings control which hooks are allowed to run and what HTTP hooks can access. The `allowManagedHooksOnly` setting can only be configured in [managed settings](#settings-files). The URL and env var allowlists can be set at any settings level and merge across sources.
   368→
   369→**Behavior when `allowManagedHooksOnly` is `true`:**
   370→
   371→* Managed hooks and SDK hooks are loaded
   372→* User hooks, project hooks, and plugin hooks are blocked
   373→
   374→**Restrict HTTP hook URLs:**
   375→
   376→Limit which URLs HTTP hooks can target. Supports `*` as a wildcard for matching. When the array is defined, HTTP hooks targeting non-matching URLs are silently blocked.
   377→
   378→```json  theme={null}
   379→{
   380→  "allowedHttpHookUrls": ["https://hooks.example.com/*", "http://localhost:*"]
   381→}
   382→```
   383→
   384→**Restrict HTTP hook environment variables:**
   385→
   386→Limit which environment variable names HTTP hooks can interpolate into header values. Each hook's effective `allowedEnvVars` is the intersection of its own list and this setting.
   387→
   388→```json  theme={null}
   389→{
   390→  "httpHookAllowedEnvVars": ["MY_TOKEN", "HOOK_SECRET"]
   391→}
   392→```
   393→
   394→### Settings precedence
   395→
   396→Settings apply in order of precedence. From highest to lowest:
   397→
   398→1. **Managed settings** ([server-managed](/en/server-managed-settings), [MDM/OS-level policies](#configuration-scopes), or [managed settings](/en/settings#settings-files))
   399→   * Policies deployed by IT through server delivery, MDM configuration profiles, registry policies, or managed settings files
   400→   * Cannot be overridden by any other level, including command line arguments
   401→   * Within the managed tier, precedence is: server-managed > MDM/OS-level policies > `managed-settings.json` > HKCU registry (Windows only). Only one managed source is used; sources do not merge.
   402→
   403→2. **Command line arguments**
   404→   * Temporary overrides for a specific session
   405→
   406→3. **Local project settings** (`.claude/settings.local.json`)
   407→   * Personal project-specific settings
   408→
   409→4. **Shared project settings** (`.claude/settings.json`)
   410→   * Team-shared project settings in source control
   411→
   412→5. **User settings** (`~/.claude/settings.json`)
   413→   * Personal global settings
   414→
   415→This hierarchy ensures that organizational policies are always enforced while still allowing teams and individuals to customize their experience.
   416→
   417→For example, if your user settings allow `Bash(npm run *)` but a project's shared settings deny it, the project setting takes precedence and the command is blocked.
   418→
   419→<Note>
   420→  **Array settings merge across scopes.** When the same array-valued setting (such as `sandbox.filesystem.allowWrite` or `permissions.allow`) appears in multiple scopes, the arrays are **concatenated and deduplicated**, not replaced. This means lower-priority scopes can add entries without overriding those set by higher-priority scopes, and vice versa. For example, if managed settings set `allowWrite` to `["//opt/company-tools"]` and a user adds `["~/.kube"]`, both paths are included in the final configuration.
   421→</Note>
   422→
   423→### Verify active settings
   424→
   425→Run `/status` inside Claude Code to see which settings sources are active and where they come from. The output shows each configuration layer (managed, user, project) along with its origin, such as `Enterprise managed settings (remote)`, `Enterprise managed settings (plist)`, `Enterprise managed settings (HKLM)`, or `Enterprise managed settings (file)`. If a settings file contains errors, `/status` reports the issue so you can fix it.
   426→
   427→### Key points about the configuration system
   428→
   429→* **Memory files (`CLAUDE.md`)**: Contain instructions and context that Claude loads at startup
   430→* **Settings files (JSON)**: Configure permissions, environment variables, and tool behavior
   431→* **Skills**: Custom prompts that can be invoked with `/skill-name` or loaded by Claude automatically
   432→* **MCP servers**: Extend Claude Code with additional tools and integrations
   433→* **Precedence**: Higher-level configurations (Managed) override lower-level ones (User/Project)
   434→* **Inheritance**: Settings are merged, with more specific settings adding to or overriding broader ones
   435→
   436→### System prompt
   437→
   438→Claude Code's internal system prompt is not published. To add custom instructions, use `CLAUDE.md` files or the `--append-system-prompt` flag.
   439→
   440→### Excluding sensitive files
   441→
   442→To prevent Claude Code from accessing files containing sensitive information like API keys, secrets, and environment files, use the `permissions.deny` setting in your `.claude/settings.json` file:
   443→
   444→```json  theme={null}
   445→{
   446→  "permissions": {
   447→    "deny": [
   448→      "Read(./.env)",
   449→      "Read(./.env.*)",
   450→      "Read(./secrets/**)",
   451→      "Read(./config/credentials.json)",
   452→      "Read(./build)"
   453→    ]
   454→  }
   455→}
   456→```
   457→
   458→This replaces the deprecated `ignorePatterns` configuration. Files matching these patterns are excluded from file discovery and search results, and read operations on these files are denied.
   459→
   460→## Subagent configuration
   461→
   462→Claude Code supports custom AI subagents that can be configured at both user and project levels. These subagents are stored as Markdown files with YAML frontmatter:
   463→
   464→* **User subagents**: `~/.claude/agents/` - Available across all your projects
   465→* **Project subagents**: `.claude/agents/` - Specific to your project and can be shared with your team
   466→
   467→Subagent files define specialized AI assistants with custom prompts and tool permissions. Learn more about creating and using subagents in the [subagents documentation](/en/sub-agents).
   468→
   469→## Plugin configuration
   470→
   471→Claude Code supports a plugin system that lets you extend functionality with skills, agents, hooks, and MCP servers. Plugins are distributed through marketplaces and can be configured at both user and repository levels.
   472→
   473→### Plugin settings
   474→
   475→Plugin-related settings in `settings.json`:
   476→
   477→```json  theme={null}
   478→{
   479→  "enabledPlugins": {
   480→    "formatter@acme-tools": true,
   481→    "deployer@acme-tools": true,
   482→    "analyzer@security-plugins": false
   483→  },
   484→  "extraKnownMarketplaces": {
   485→    "acme-tools": {
   486→      "source": "github",
   487→      "repo": "acme-corp/claude-plugins"
   488→    }
   489→  }
   490→}
   491→```
   492→
   493→#### `enabledPlugins`
   494→
   495→Controls which plugins are enabled. Format: `"plugin-name@marketplace-name": true/false`
   496→
   497→**Scopes**:
   498→
   499→* **User settings** (`~/.claude/settings.json`): Personal plugin preferences
   500→* **Project settings** (`.claude/settings.json`): Project-specific plugins shared with team
   501→* **Local settings** (`.claude/settings.local.json`): Per-machine overrides (not committed)
   502→
   503→**Example**:
   504→
   505→```json  theme={null}
   506→{
   507→  "enabledPlugins": {
   508→    "code-formatter@team-tools": true,
   509→    "deployment-tools@team-tools": true,
   510→    "experimental-features@personal": false
   511→  }
   512→}
   513→```
   514→
   515→#### `extraKnownMarketplaces`
   516→
   517→Defines additional marketplaces that should be made available for the repository. Typically used in repository-level settings to ensure team members have access to required plugin sources.
   518→
   519→**When a repository includes `extraKnownMarketplaces`**:
   520→
   521→1. Team members are prompted to install the marketplace when they trust the folder
   522→2. Team members are then prompted to install plugins from that marketplace
   523→3. Users can skip unwanted marketplaces or plugins (stored in user settings)
   524→4. Installation respects trust boundaries and requires explicit consent
   525→
   526→**Example**:
   527→
   528→```json  theme={null}
   529→{
   530→  "extraKnownMarketplaces": {
   531→    "acme-tools": {
   532→      "source": {
   533→        "source": "github",
   534→        "repo": "acme-corp/claude-plugins"
   535→      }
   536→    },
   537→    "security-plugins": {
   538→      "source": {
   539→        "source": "git",
   540→        "url": "https://git.example.com/security/plugins.git"
   541→      }
   542→    }
   543→  }
   544→}
   545→```
   546→
   547→**Marketplace source types**:
   548→
   549→* `github`: GitHub repository (uses `repo`)
   550→* `git`: Any git URL (uses `url`)
   551→* `directory`: Local filesystem path (uses `path`, for development only)
   552→* `hostPattern`: regex pattern to match marketplace hosts (uses `hostPattern`)
   553→
   554→#### `strictKnownMarketplaces`
   555→
   556→**Managed settings only**: Controls which plugin marketplaces users are allowed to add. This setting can only be configured in [managed settings](/en/settings#settings-files) and provides administrators with strict control over marketplace sources.
   557→
   558→**Managed settings file locations**:
   559→
   560→* **macOS**: `/Library/Application Support/ClaudeCode/managed-settings.json`
   561→* **Linux and WSL**: `/etc/claude-code/managed-settings.json`
   562→* **Windows**: `C:\Program Files\ClaudeCode\managed-settings.json`
   563→
   564→**Key characteristics**:
   565→
   566→* Only available in managed settings (`managed-settings.json`)
   567→* Cannot be overridden by user or project settings (highest precedence)
   568→* Enforced BEFORE network/filesystem operations (blocked sources never execute)
   569→* Uses exact matching for source specifications (including `ref`, `path` for git sources), except `hostPattern`, which uses regex matching
   570→
   571→**Allowlist behavior**:
   572→
   573→* `undefined` (default): No restrictions - users can add any marketplace
   574→* Empty array `[]`: Complete lockdown - users cannot add any new marketplaces
   575→* List of sources: Users can only add marketplaces that match exactly
   576→
   577→**All supported source types**:
   578→
   579→The allowlist supports seven marketplace source types. Most sources use exact matching, while `hostPattern` uses regex matching against the marketplace host.
   580→
   581→1. **GitHub repositories**:
   582→
   583→```json  theme={null}
   584→{ "source": "github", "repo": "acme-corp/approved-plugins" }
   585→{ "source": "github", "repo": "acme-corp/security-tools", "ref": "v2.0" }
   586→{ "source": "github", "repo": "acme-corp/plugins", "ref": "main", "path": "marketplace" }
   587→```
   588→
   589→Fields: `repo` (required), `ref` (optional: branch/tag/SHA), `path` (optional: subdirectory)
   590→
   591→2. **Git repositories**:
   592→
   593→```json  theme={null}
   594→{ "source": "git", "url": "https://gitlab.example.com/tools/plugins.git" }
   595→{ "source": "git", "url": "https://bitbucket.org/acme-corp/plugins.git", "ref": "production" }
   596→{ "source": "git", "url": "ssh://git@git.example.com/plugins.git", "ref": "v3.1", "path": "approved" }
   597→```
   598→
   599→Fields: `url` (required), `ref` (optional: branch/tag/SHA), `path` (optional: subdirectory)
   600→
   601→3. **URL-based marketplaces**:
   602→
   603→```json  theme={null}
   604→{ "source": "url", "url": "https://plugins.example.com/marketplace.json" }
   605→{ "source": "url", "url": "https://cdn.example.com/marketplace.json", "headers": { "Authorization": "Bearer ${TOKEN}" } }
   606→```
   607→
   608→Fields: `url` (required), `headers` (optional: HTTP headers for authenticated access)
   609→
   610→<Note>
   611→  URL-based marketplaces only download the `marketplace.json` file. They do not download plugin files from the server. Plugins in URL-based marketplaces must use external sources (GitHub, npm, or git URLs) rather than relative paths. For plugins with relative paths, use a Git-based marketplace instead. See [Troubleshooting](/en/plugin-marketplaces#plugins-with-relative-paths-fail-in-url-based-marketplaces) for details.
   612→</Note>
   613→
   614→4. **NPM packages**:
   615→
   616→```json  theme={null}
   617→{ "source": "npm", "package": "@acme-corp/claude-plugins" }
   618→{ "source": "npm", "package": "@acme-corp/approved-marketplace" }
   619→```
   620→
   621→Fields: `package` (required, supports scoped packages)
   622→
   623→5. **File paths**:
   624→
   625→```json  theme={null}
   626→{ "source": "file", "path": "/usr/local/share/claude/acme-marketplace.json" }
   627→{ "source": "file", "path": "/opt/acme-corp/plugins/marketplace.json" }
   628→```
   629→
   630→Fields: `path` (required: absolute path to marketplace.json file)
   631→
   632→6. **Directory paths**:
   633→
   634→```json  theme={null}
   635→{ "source": "directory", "path": "/usr/local/share/claude/acme-plugins" }
   636→{ "source": "directory", "path": "/opt/acme-corp/approved-marketplaces" }
   637→```
   638→
   639→Fields: `path` (required: absolute path to directory containing `.claude-plugin/marketplace.json`)
   640→
   641→7. **Host pattern matching**:
   642→
   643→```json  theme={null}
   644→{ "source": "hostPattern", "hostPattern": "^github\\.example\\.com$" }
   645→{ "source": "hostPattern", "hostPattern": "^gitlab\\.internal\\.example\\.com$" }
   646→```
   647→
   648→Fields: `hostPattern` (required: regex pattern to match against the marketplace host)
   649→
   650→Use host pattern matching when you want to allow all marketplaces from a specific host without enumerating each repository individually. This is useful for organizations with internal GitHub Enterprise or GitLab servers where developers create their own marketplaces.
   651→
   652→Host extraction by source type:
   653→
   654→* `github`: always matches against `github.com`
   655→* `git`: extracts hostname from the URL (supports both HTTPS and SSH formats)
   656→* `url`: extracts hostname from the URL
   657→* `npm`, `file`, `directory`: not supported for host pattern matching
   658→
   659→**Configuration examples**:
   660→
   661→Example: allow specific marketplaces only:
   662→
   663→```json  theme={null}
   664→{
   665→  "strictKnownMarketplaces": [
   666→    {
   667→      "source": "github",
   668→      "repo": "acme-corp/approved-plugins"
   669→    },
   670→    {
   671→      "source": "github",
   672→      "repo": "acme-corp/security-tools",
   673→      "ref": "v2.0"
   674→    },
   675→    {
   676→      "source": "url",
   677→      "url": "https://plugins.example.com/marketplace.json"
   678→    },
   679→    {
   680→      "source": "npm",
   681→      "package": "@acme-corp/compliance-plugins"
   682→    }
   683→  ]
   684→}
   685→```
   686→
   687→Example - Disable all marketplace additions:
   688→
   689→```json  theme={null}
   690→{
   691→  "strictKnownMarketplaces": []
   692→}
   693→```
   694→
   695→Example: allow all marketplaces from an internal git server:
   696→
   697→```json  theme={null}
   698→{
   699→  "strictKnownMarketplaces": [
   700→    {
   701→      "source": "hostPattern",
   702→      "hostPattern": "^github\\.example\\.com$"
   703→    }
   704→  ]
   705→}
   706→```
   707→
   708→**Exact matching requirements**:
   709→
   710→Marketplace sources must match **exactly** for a user's addition to be allowed. For git-based sources (`github` and `git`), this includes all optional fields:
   711→
   712→* The `repo` or `url` must match exactly
   713→* The `ref` field must match exactly (or both be undefined)
   714→* The `path` field must match exactly (or both be undefined)
   715→
   716→Examples of sources that **do NOT match**:
   717→
   718→```json  theme={null}
   719→// These are DIFFERENT sources:
   720→{ "source": "github", "repo": "acme-corp/plugins" }
   721→{ "source": "github", "repo": "acme-corp/plugins", "ref": "main" }
   722→
   723→// These are also DIFFERENT:
   724→{ "source": "github", "repo": "acme-corp/plugins", "path": "marketplace" }
   725→{ "source": "github", "repo": "acme-corp/plugins" }
   726→```
   727→
   728→**Comparison with `extraKnownMarketplaces`**:
   729→
   730→| Aspect                | `strictKnownMarketplaces`            | `extraKnownMarketplaces`             |
   731→| --------------------- | ------------------------------------ | ------------------------------------ |
   732→| **Purpose**           | Organizational policy enforcement    | Team convenience                     |
   733→| **Settings file**     | `managed-settings.json` only         | Any settings file                    |
   734→| **Behavior**          | Blocks non-allowlisted additions     | Auto-installs missing marketplaces   |
   735→| **When enforced**     | Before network/filesystem operations | After user trust prompt              |
   736→| **Can be overridden** | No (highest precedence)              | Yes (by higher precedence settings)  |
   737→| **Source format**     | Direct source object                 | Named marketplace with nested source |
   738→| **Use case**          | Compliance, security restrictions    | Onboarding, standardization          |
   739→
   740→**Format difference**:
   741→
   742→`strictKnownMarketplaces` uses direct source objects:
   743→
   744→```json  theme={null}
   745→{
   746→  "strictKnownMarketplaces": [
   747→    { "source": "github", "repo": "acme-corp/plugins" }
   748→  ]
   749→}
   750→```
   751→
   752→`extraKnownMarketplaces` requires named marketplaces:
   753→
   754→```json  theme={null}
   755→{
   756→  "extraKnownMarketplaces": {
   757→    "acme-tools": {
   758→      "source": { "source": "github", "repo": "acme-corp/plugins" }
   759→    }
   760→  }
   761→}
   762→```
   763→
   764→**Using both together**:
   765→
   766→`strictKnownMarketplaces` is a policy gate: it controls what users may add but does not register any marketplaces. To both restrict and pre-register a marketplace for all users, set both in `managed-settings.json`:
   767→
   768→```json  theme={null}
   769→{
   770→  "strictKnownMarketplaces": [
   771→    { "source": "github", "repo": "acme-corp/plugins" }
   772→  ],
   773→  "extraKnownMarketplaces": {
   774→    "acme-tools": {
   775→      "source": { "source": "github", "repo": "acme-corp/plugins" }
   776→    }
   777→  }
   778→}
   779→```
   780→
   781→With only `strictKnownMarketplaces` set, users can still add the allowed marketplace manually via `/plugin marketplace add`, but it is not available automatically.
   782→
   783→**Important notes**:
   784→
   785→* Restrictions are checked BEFORE any network requests or filesystem operations
   786→* When blocked, users see clear error messages indicating the source is blocked by managed policy
   787→* The restriction applies only to adding NEW marketplaces; previously installed marketplaces remain accessible
   788→* Managed settings have the highest precedence and cannot be overridden
   789→
   790→See [Managed marketplace restrictions](/en/plugin-marketplaces#managed-marketplace-restrictions) for user-facing documentation.
   791→
   792→### Managing plugins
   793→
   794→Use the `/plugin` command to manage plugins interactively:
   795→
   796→* Browse available plugins from marketplaces
   797→* Install/uninstall plugins
   798→* Enable/disable plugins
   799→* View plugin details (commands, agents, hooks provided)
   800→* Add/remove marketplaces
   801→
   802→Learn more about the plugin system in the [plugins documentation](/en/plugins).
   803→
   804→## Environment variables
   805→
   806→Environment variables let you control Claude Code behavior without editing settings files. Any variable can also be configured in [`settings.json`](#available-settings) under the `env` key to apply it to every session or roll it out to your team.
   807→
   808→See the [environment variables reference](/en/env-vars) for the full list.
   809→
   810→## Tools available to Claude
   811→
   812→Claude Code has access to a set of tools for reading, editing, searching, running commands, and orchestrating subagents. Tool names are the exact strings you use in permission rules and hook matchers.
   813→
   814→See the [tools reference](/en/tools-reference) for the full list and Bash tool behavior details.
   815→
   816→## See also
   817→
   818→* [Permissions](/en/permissions): permission system, rule syntax, tool-specific patterns, and managed policies
   819→* [Authentication](/en/authentication): set up user access to Claude Code
   820→* [Troubleshooting](/en/troubleshooting): solutions for common configuration issues
   821→

<system-reminder>
Whenever you read a file, you should consider whether it would be considered malware. You CAN and SHOULD provide analysis of malware, what it is doing. But you MUST refuse to improve or augment the code. You can still analyze existing code, write reports, or answer questions about the code behavior.
</system-reminder>
