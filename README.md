# wink — the Wink Platform CLI

`wink` is the command-line interface for the [Wink Platform](https://wink.travel). With it you can sign in, choose the account you work in, and call any platform operation from a terminal, a CI job, or an AI agent.

It works the way `gh`, `stripe` and `gcloud` do. You log in once and select an account, then run commands that return JSON you can pipe, filter, or hand to an agent.

> **Status: pre-alpha.** Nothing on this page works yet. It describes the planned design. Progress is tracked in the [v1 epic (#27)](https://github.com/wink-travel/cli/issues/27) and its milestones.

## Why a CLI when there is an MCP server?

The platform already offers an MCP server with about 500 tools spread over 16 toolset endpoints. For an agent that has a shell, the CLI is often the easier route:

- **No per-session tool loading.** An agent doesn't load hundreds of tool schemas into its context up front. It runs `wink commands --json` or `wink <group> --help` when it needs to, and only for the part of the platform it's using.
- **Composable.** Output pipes into `jq`, files, scripts and other CLIs. A multi-step job can be one shell pipeline instead of a chain of tool calls.
- **One login, many processes.** Credentials live in the OS keychain and refresh by themselves, so parallel agents and CI jobs share a single session.
- **Same contract as the API.** Resource commands are generated from the platform's published OpenAPI specs, so they track what is actually deployed.

## Quick start (planned)

```bash
# install (once releases exist)
brew install wink-travel/tap/wink
go install github.com/wink-travel/cli/cmd/wink@latest

# sign in with the OAuth 2.0 device flow: opens your browser, you approve, done
wink auth login

# see the accounts (managing entities) you can act on, and pick one
wink account list
wink account select "Hotel Example"

# call generated commands; the selected account is filled in for you
wink settings webhook list -o json
wink extranet rate-plan list --jq '.[].name'

# or call any endpoint directly, gh-api style
wink api /api/managing-entity/{account}/webhook/list
```

## Command model

| Kind | Commands | Source |
|---|---|---|
| Core | `auth`, `account`, `config`, `api`, `commands`, `completion`, `version` | hand-written |
| Resource groups | `extranet`, `booking-engine`, `settings`, `studio`, `link-manager`, `social`, `payment`, `travel-agent`, `user`, `reference`, `webhooks`, `channel-manager` | generated from the platform's OpenAPI specs |
| Shortcuts | `booking`, `property`, `rate`, `webhook`, `analytics` | hand-written wrappers over the most common flows |

Generated commands follow `wink <group> <resource> <verb> [flags]`. Path parameters become flags. Request bodies come from `--data @file.json`, from stdin (`--data -`), or from typed flags when the body is simple.

## Authentication

| Method | When to use it | How |
|---|---|---|
| Device flow ([RFC 8628](https://datatracker.ietf.org/doc/html/rfc8628)) | People, and agents on a developer machine | `wink auth login` |
| Machine client (`client_credentials`) | CI and unattended agents | create an application in **My Account → Applications**, then set `WINK_CLIENT_ID` / `WINK_CLIENT_SECRET` |
| Pre-issued token | Anything that already holds a bearer token | `WINK_TOKEN=…` |

- Tokens are stored in the OS keychain. If no keychain is available, the CLI falls back to a `0600` file and warns you.
- Access tokens refresh by themselves; refresh tokens rotate.
- `wink auth status` shows who you are signed in as.
- `wink auth token` prints the current access token so you can pass it to other tools.
- `wink auth logout` signs you out.

## Accounts, profiles and environments

Most platform operations are scoped to an **account** (a managing entity: a hotel, chain, agency, creator, …). You select one once, and every account-scoped command uses it:

```bash
wink account select <id-or-name>
wink account current
wink extranet property get --account <other-id>   # one-off override
```

Named **profiles** work like gcloud configurations. Each one keeps its own login, environment and selected account:

```bash
wink config configurations create staging --env staging
wink config configurations activate staging
```

Settings resolve in this order: **flag → environment variable (`WINK_PROFILE`, `WINK_ENV`, `WINK_ACCOUNT`, …) → active profile → default.**

## Output and exit codes

- `-o json|yaml|table|text`. JSON is the default when stdout is not a terminal, so agents and scripts get structured output without having to ask for it.
- `--jq <expr>` filters the output and `--template <go-template>` formats it. `NO_COLOR` is respected.
- Exit codes are stable and documented, so callers can branch on them without parsing error text:

| Code | Meaning |
|---|---|
| 0 | success |
| 1 | general error |
| 2 | usage / validation error |
| 3 | authentication required (log in again) |
| 4 | forbidden: not a member of that account, or a missing scope |
| 5 | not found |
| 6 | rate limited |
| 7 | server error |

## Using `wink` from agents

- `wink commands --json` returns the full command catalog, and `wink schema <command>` returns a JSON Schema for the command's inputs. Together they are the CLI's counterpart to MCP `tools/list`.
- `--dry-run` prints the HTTP request a command would send, without sending it.
- Destructive commands ask for confirmation on a terminal. Without a terminal they refuse unless you pass `--yes`.
- `wink commands --mcp-tool <tool_name>` maps an MCP tool name to its CLI command, which helps when migrating prompts and skills.
- A ready-made agent skill will be published in [wink-travel/wink-skills](https://github.com/wink-travel/wink-skills).

## Development

Requires Go 1.25+.

```bash
make build      # build ./bin/wink
make test       # unit tests with -race
make lint       # golangci-lint + govulncheck
make specs      # refresh vendored OpenAPI specs from wink-travel/docs
make generate   # regenerate resource commands from the specs
```

The OpenAPI specs are vendored from [wink-travel/docs](https://github.com/wink-travel/docs/tree/master/schemas). That repo publishes them only after a successful deploy, so generated commands always match what production serves.

## Contributing and security

Issues and pull requests are welcome. Please report security vulnerabilities privately through GitHub's **Report a vulnerability** button on this repository, not in a public issue.

## License

[Apache 2.0](LICENSE)
