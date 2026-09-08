# Claude Code `settings.json` — full field reference

Verified against the official settings reference (fetched Sept 8, 2026) and the
published JSON Schema at `https://json.schemastore.org/claude-code-settings.json`.

There is no single "all fields" file in the docs — the reference now lists ~180 keys,
many of them managed-only or mutually exclusive. Below: the files, a maximal annotated
example of every key with a verified type, then a complete index of the rest.

---

## 1. Files, scope, precedence

| File | Scope label | Purpose |
|---|---|---|
| `~/.claude/settings.json` | `User` | your baseline, all projects |
| `<repo>/.claude/settings.json` | `Project` | team-shared, commit it |
| `<repo>/.claude/settings.local.json` | `Local` | personal overrides, gitignore it |
| OS policy path | `Managed` | deployed by your org |
| `~/.claude.json` | `Global config` | machine state, **not** a settings file |

Precedence: **managed → local → project → user**. Arrays in some keys
(`claudeMdExcludes`, `availableModels`) merge across layers rather than replace.

On Windows the user file is `%USERPROFILE%\.claude\settings.json`.

`~/.claude.json` holds a separate, small key set (`autoConnectIde`,
`autoInstallIdeExtension`, `diffTool`, `externalEditorContext`) plus OAuth session,
user/local MCP servers, per-project trust, and caches. Claude Code manages it; don't
hand-edit it.

---

## 2. Maximal annotated example

Strip the `//` comments before use — settings files are strict JSON (no comments,
no trailing commas). Values shown are illustrative, not recommended defaults.

```jsonc
{
  "$schema": "https://json.schemastore.org/claude-code-settings.json",

  // ---------- Model and responses ----------
  "model": "opus",                          // alias or full ID, e.g. "claude-opus-5"
  "availableModels": ["opus", "sonnet"],    // allowlist; merged+deduped across layers
  "enforceAvailableModels": true,           // make /model "Default" resolve inside the allowlist (v2.1.175+)
  "fallbackModel": ["claude-sonnet-5", "claude-haiku-4-5"],  // tried in order on overload
  "modelOverrides": {                       // Anthropic ID -> provider ID
    "claude-opus-4-6": "arn:aws:bedrock:us-east-2:123456789012:application-inference-profile/opus-prod"
  },
  "advisorModel": "opus",                   // "fable" | "opus" | "sonnet" | full ID; unset = advisor off
  "effortLevel": "high",                    // "low" | "medium" | "high" | "xhigh"
  "alwaysThinkingEnabled": false,           // false turns extended thinking off; true is a no-op
  "fastMode": false,
  "fastModePerSessionOptIn": true,          // force /fast every session
  "language": "english",                    // e.g. "japanese", "spanish"
  "outputStyle": "default",                 // named output style
  "promptCacheTtl": "5m",                   // main conversation cache lifetime
  "subagentPromptCacheTtl": "5m",           // subagents and side requests
  "showThinkingSummaries": true,
  "ultracode": false,                       // auto-plan a workflow per substantive task
  "switchModelsOnFlag": "ask",              // behavior when a safety classifier flags a request

  // ---------- Permissions ----------
  "permissions": {
    "defaultMode": "default",
    // "default" | "manual" (alias) | "acceptEdits" | "plan" | "auto"
    // | "dontAsk" | "delegate" | "bypassPermissions"
    "allow": [
      "Bash(git add:*)",
      "Bash(npm run test:*)",
      "Edit(/src/**/*.ts)",
      "mcp__github__search_repositories"
    ],
    "ask": ["Bash(git commit:*)", "Bash(gh pr create:*)"],
    "deny": ["Read(./.env)", "Read(./secrets/**)", "Bash(rm:*)", "Bash(curl:*)"],
    "additionalDirectories": ["~/projects/shared-lib"],
    "blockReadsOutsideWorkingDirectories": true,   // enforce in every permission mode
    "disableBypassPermissionsMode": "disable",     // literal string "disable"
    "disableAutoMode": "disable"                   // literal string "disable"
  },
  "autoMode": {                             // extra rules for the auto-mode classifier
    "classifyAllShell": true
  },
  "useAutoModeDuringPlan": true,
  "skipAutoPermissionPrompt": false,
  "skipDangerousModePermissionPrompt": false,      // written automatically when you accept the warning

  // ---------- Sandbox (macOS, Linux, WSL2) ----------
  "sandbox": {
    "enabled": true,
    "autoAllowBashIfSandboxed": true,       // no permission prompt for sandboxed commands
    "allowUnsandboxedCommands": true,       // allow the retry-outside-sandbox escape hatch
    "failIfUnavailable": true,              // refuse to start rather than run unsandboxed
    "excludedCommands": ["docker"],
    "allowAppleEvents": false,
    "enableWeakerNestedSandbox": false,
    "enableWeakerNetworkIsolation": false,
    "ignoreViolations": ["/private/var/**"],
    "ripgrep": "/usr/local/bin/rg",
    "bwrapPath": "/usr/bin/bwrap",          // managed only
    "socatPath": "/usr/bin/socat",          // managed only
    "filesystem": {
      "disabled": false,
      "allowRead": ["~/.cache/**"],
      "denyRead": ["~/.ssh/**"],
      "allowWrite": ["/tmp/**"],
      "denyWrite": ["/etc/**"],
      "allowManagedReadPathsOnly": false    // managed only
    },
    "network": {
      "allowedDomains": ["registry.npmjs.org", "*.github.com"],
      "deniedDomains": ["telemetry.example.com"],
      "strictAllowlist": true,              // deny instead of prompt
      "allowLocalBinding": true,
      "allowMachLookup": false,
      "allowAllUnixSockets": false,
      "allowUnixSockets": ["/var/run/docker.sock"],
      "httpProxyPort": 8080,
      "socksProxyPort": 1080,
      "tlsTerminate": false,
      "allowManagedDomainsOnly": false      // managed only
    },
    "credentials": {
      "allowPlaintextInject": false,
      "sigv4": "fail",
      "envVars": { "AWS_SECRET_ACCESS_KEY": "mask" },
      "files": { "~/.aws/credentials": "mask" },
      "awsPairs": [{ "accessKeyId": "MY_KEY_ID", "secretAccessKey": "MY_SECRET" }]
    }
  },

  // ---------- Hooks and automation ----------
  "hooks": {
    "PreToolUse": [
      {
        "matcher": "Bash",
        "hooks": [
          {
            "type": "command",
            "command": "./scripts/audit.sh",
            "args": ["--json"],              // exec form: no shell interpretation
            "shell": "bash",                 // "bash" | "powershell"
            "timeout": 30,                   // seconds
            "async": false,
            "asyncRewake": false,
            "if": "Bash(git *)",             // permission-rule filter, tool events only
            "statusMessage": "Auditing command"
          }
        ]
      }
    ],
    "PostToolUse": [
      {
        "matcher": "Edit|Write",
        "hooks": [{ "type": "command", "command": "npx prettier --write $CLAUDE_PROJECT_DIR" }]
      }
    ],
    "UserPromptSubmit": [
      {
        "hooks": [
          {
            "type": "prompt",              // LLM-evaluated hook
            "prompt": "Reject if $ARGUMENTS contains a production credential.",
            "model": "haiku",
            "timeout": 30,
            "continueOnBlock": false
          }
        ]
      }
    ],
    "SessionEnd": [
      {
        "hooks": [
          {
            "type": "http",
            "url": "https://hooks.example.com/claude",
            "headers": { "Authorization": "Bearer $CI_TOKEN" },
            "allowedEnvVars": ["CI_TOKEN"],
            "timeout": 10
          }
        ]
      }
    ],
    "Stop": [
      {
        "hooks": [
          {
            "type": "agent",               // multi-turn verification hook
            "prompt": "Verify the test suite was run and passed.",
            "timeout": 60
          }
        ]
      }
    ]
  },
  // Fifth hook type, usable anywhere above:
  //   { "type": "mcp_tool", "server": "github", "tool": "create_issue",
  //     "input": { "path": "${tool_input.file_path}" } }
  "disableAllHooks": false,                 // also kills statusLine + fileSuggestion commands
  "allowedHttpHookUrls": ["https://hooks.example.com/*"],
  "httpHookAllowedEnvVars": ["CI_TOKEN"],
  "disableWorkflows": false,
  "enableWorkflows": true,
  "workflowKeywordTriggerEnabled": true,
  "workflowSizeGuideline": 3,

  // ---------- Memory and context ----------
  "env": {                                  // applies to sessions and subprocesses
    "ANTHROPIC_MODEL": "claude-opus-5",
    "BASH_DEFAULT_TIMEOUT_MS": "120000",
    "MAX_MCP_OUTPUT_TOKENS": "25000",
    "DISABLE_TELEMETRY": "1"
  },
  "autoCompactEnabled": true,
  "autoCompactWindow": 0.85,
  "autoMemoryEnabled": true,
  "autoMemoryDirectory": "~/.claude/memory",
  "claudeMdExcludes": ["**/monorepo/other-team/CLAUDE.md"],
  "bashOutputMaxChars": 30000,
  "taskOutputMaxChars": 32000,
  "fileCheckpointingEnabled": true,
  "plansDirectory": "./plans",
  "skillListingBudgetFraction": 0.01,
  "skillListingMaxDescChars": 200,

  // ---------- MCP ----------
  "enableAllProjectMcpServers": false,
  "enabledMcpjsonServers": ["github", "memory"],
  "disabledMcpjsonServers": ["scratch"],
  "allowedMcpServers": ["github", "linear"],
  "deniedMcpServers": ["https://untrusted.example.com/mcp"],
  "disableClaudeAiConnectors": false,

  // ---------- Plugins and skills ----------
  "enabledPlugins": { "my-plugin@my-marketplace": true },
  "extraKnownMarketplaces": {
    "my-marketplace": { "source": { "source": "github", "repo": "org/marketplace" } }
  },
  "pluginConfigs": {},
  "skillOverrides": { "some-skill": "hidden" },
  "disableBundledSkills": false,
  "disableSkillShellExecution": false,
  "syncClaudeAiSkills": true,

  // ---------- Git and attribution ----------
  "attribution": {
    "commit": "",                           // empty string hides the commit trailer
    "pr": "",                               // empty string hides the PR line
    "sessionUrl": false                     // omit the claude.ai session link
  },
  "includeGitInstructions": true,
  "prUrlTemplate": "https://git.internal/{owner}/{repo}/pull/{number}",
  "respectGitignore": true,

  // ---------- Agents, sessions, worktrees ----------
  "agent": "reviewer",                      // start every session as a named subagent
  "disableAgentView": false,
  "teammateMode": "compact",
  "crossSessionInbound": "allow",           // "allow" | "notify" | "refuse"
  "isolatePeerMachines": true,
  "processWrapper": "/opt/corp/launcher",   // macOS/Linux, user or managed only
  "worktree": {
    "baseRef": "remote",                    // remote default branch vs local HEAD
    "bgIsolation": true,
    "sparsePaths": ["src", "package.json"],
    "symlinkDirectories": ["node_modules", ".venv"]
  },

  // ---------- Interface and terminal ----------
  "theme": "dark",
  "tui": "fullscreen",                      // fullscreen vs classic renderer
  "viewMode": "default",                    // "default" | "verbose" | "focus"
  "verbose": false,                         // viewMode wins if both are set
  "editorMode": "vim",
  "vimInsertModeRemaps": { "jj": "Escape" },
  "defaultShell": "bash",                   // shell for your `!` commands
  "respondToBashCommands": true,
  "timeZone": "Asia/Jerusalem",
  "timeFormat": "24h",                      // 12h / 24h / UTC / strftime pattern
  "statusLine": { "type": "command", "command": "~/.claude/statusline.sh", "padding": 0 },
  "subagentStatusLine": { "type": "command", "command": "~/.claude/subagent-line.sh" },
  "fileSuggestion": { "type": "command", "command": "~/.claude/files.sh" },
  "footerLinksRegexes": ["ENG-[0-9]+"],
  "spellcheck": true,
  "spinnerTipsEnabled": true,
  "spinnerTipsOverride": ["Check the plan before you accept it"],
  "spinnerVerbs": ["Compiling", "Untangling"],
  "showTurnDuration": true,
  "showClearContextOnPlanAccept": true,
  "promptSuggestionEnabled": true,
  "emojiCompletionEnabled": true,
  "syntaxHighlightingDisabled": false,
  "terminalProgressBarEnabled": true,
  "terminalTitleFromRename": true,
  "autoScrollEnabled": true,
  "wheelScrollAccelerationEnabled": true,
  "prefersReducedMotion": false,
  "axScreenReader": false,
  "voice": { "enabled": true, "mode": "hold" },
  "voiceEnabled": true,                     // older single-key form
  "askUserQuestionTimeout": 60000,
  "autoContinueAtUsageLimit": true,
  "dialogExpiry": 300000,
  "companyAnnouncements": ["Reminder: no prod creds in prompts"],

  // ---------- Remote, desktop, notifications ----------
  "remoteControlAtStartup": false,
  "disableRemoteControl": false,
  "agentPushNotifEnabled": true,
  "inputNeededNotifEnabled": true,
  "preferredNotifChannel": "terminal_bell",
  "awaySummaryEnabled": true,
  "enableArtifact": true,                   // false in any file wins; no file re-enables
  "disableDeepLinkRegistration": false,
  "remote": { "defaultEnvironmentId": "env_abc123" },
  "sshConfigs": [{ "name": "build-box", "host": "build.internal", "user": "tamir" }],

  // ---------- Authentication and providers ----------
  "apiKeyHelper": "/bin/generate_temp_api_key.sh",
  "awsAuthRefresh": "aws sso login --profile myprofile",
  "awsCredentialExport": "/bin/generate_aws_grant.sh",
  "gcpAuthRefresh": "gcloud auth application-default login",
  "otelHeadersHelper": "/bin/otel_headers.sh",
  "forceLoginMethod": "claudeai",           // "claudeai" | "console" | gateway
  "forceLoginOrgUUID": "00000000-0000-0000-0000-000000000000",

  // ---------- Updates, privacy, telemetry ----------
  "autoUpdatesChannel": "stable",           // "stable" | "latest"
  "minimumVersion": "2.1.200",
  "cleanupPeriodDays": 30,                  // minimum 1
  "desktopSessionCleanupPeriodDays": 30,
  "feedbackSurveyRate": 0.05,               // 0–1
  "feedbackDrafts": true,
  "skipWebFetchPreflight": false
}
```

---

## 3. Managed-settings-only keys

Valid only in the org-deployed managed file; ignored elsewhere.

**Lockdown / policy**
`allowManagedHooksOnly`, `allowManagedMcpServersOnly`, `allowManagedPermissionRulesOnly`,
`allowAllClaudeAiMcps`, `disableSideloadFlags`, `forceRemoteSettingsRefresh`,
`managedSourcesBehavior`, `parentSettingsBehavior`, `wslInheritsWindowsSettings`,
`requiredMinimumVersion`, `requiredMaximumVersion`

**Policy helper**
`policyHelper` → `.path`, `.timeoutMs`, `.refreshIntervalMs`

**Plugins / marketplaces**
`allowedChannelPlugins`, `blockedMarketplaces`, `strictKnownMarketplaces`,
`pluginSuggestionMarketplaces`, `pluginTrustMessage`, `channelsEnabled`,
`disableCommandPluginSources`,
`strictPluginOnlyCustomization` → `.agents`, `.hooks`, `.mcp`, `.skills`

**Desktop / browser**
`browserExternalPageTools`, `disableBrowserExternalNavigation`,
`disableMobileSimulatorTools`, `disableDesktopLocalSessions`, `sshHostAllowlist`

**Other**
`claudeMd` (org-wide CLAUDE.md injection), `modelPricing`, `forceLoginGatewayUrl`,
`sandbox.bwrapPath`, `sandbox.socatPath`, `sandbox.filesystem.allowManagedReadPathsOnly`,
`sandbox.network.allowManagedDomainsOnly`

---

## 4. `~/.claude.json` keys (global config, not settings.json)

`autoConnectIde`, `autoInstallIdeExtension`, `diffTool`, `externalEditorContext`.
Also present but removed/no-op: `permissionExplainerEnabled` (removed v2.1.257),
`teammateDefaultModel` (removed v2.1.234).

---

## 5. Deprecated

| Key | Replacement |
|---|---|
| `includeCoAuthoredBy` | `attribution` |
| `disableArtifact` | `enableArtifact` |
| `keybindingFlavor` | none — no effect, readline conventions always apply |

---

## 6. Remaining keys not shown above

Documented, `Any file` scope unless noted; check the reference for exact value shape:
`modelSettings` (per-model saved effort; written by `/effort`),
`modelPicker` (user or managed), `disableAutoMode`, `disableWorkflows`,
`autoMode`, `useAutoModeDuringPlan`.

---

## 7. Sources

- Settings reference (every key, type, default): https://code.claude.com/docs/en/settings-reference
- Settings overview and precedence: https://code.claude.com/docs/en/settings
- JSON Schema (IDE autocomplete): https://json.schemastore.org/claude-code-settings.json
- Env vars: https://code.claude.com/docs/en/env-vars
- Hooks: https://code.claude.com/docs/en/hooks

The schema is community-maintained on SchemaStore and can lag a release or two;
the docs reference is authoritative. Add `$schema` to your file to get autocomplete
and validation in VS Code and JetBrains.
