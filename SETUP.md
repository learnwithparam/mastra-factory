# Setting this Factory up

The server runs on this machine. Postgres and Redis run in containers beside it. Agent sessions run
in local sandboxes. Nothing here is deployed.

```bash
make factory-doctor   # what is missing, and what to do about it
make factory-app      # create the GitHub App, one click
make factory-up       # containers, then the server, on http://localhost:4111
make factory-down     # stop everything, leaving no listener behind
```

## What has to be true before a session

`make factory-doctor` checks all of it and names the fix for anything that is not.

| | Why |
|---|---|
| The project is generated | `bunx create-factory@0.1.18 mastra --no-platform` from the grouping folder |
| Postgres and Redis are up | Factory's storage and its event bus |
| A model provider key | The agents need one. Anthropic or OpenRouter |
| A credential encryption key | Without it, saved credentials are stored as plaintext and the server says so |
| Six GitHub App credentials | Issues in, pull requests out |
| The server answers | `http://localhost:4111` |
| Someone is signed in | See below |
| Six issues open on the repository | `make lab-reset` |

## Authentication cannot be turned off

`MASTRACODE_AUTH_DISABLED=1` stops the auth routes being mounted, so `/auth/me` returns the server's
HTML page instead of JSON, and the interface spins on a loading marker forever. It looks like a
broken build and it is a supported configuration being used outside what it supports.

There are two working choices:

- **Mastra platform sign-in**, which is the default when nothing else is configured. One browser
  sign-in, and the session persists. The server still runs here; only identity is theirs.
- **WorkOS**, self-managed, by setting `WORKOS_API_KEY` and `WORKOS_CLIENT_ID`. Everything stays on
  your own infrastructure at the cost of another account to hold.

## Webhooks reach a machine the internet cannot see

GitHub cannot post to `localhost`. Issues will not arrive on their own without a tunnel.

`MASTRACODE_GITHUB_RECONCILE_ENABLED` sweeps merged pull requests, and that is all it sweeps: it is
not a substitute for issue intake. Run a webhook relay while a session is live, pointed at
`http://localhost:4111/web/github/webhook`.

## Secrets

Everything sensitive lives in `~/.config/lwp-secrets/factory.env`, which `make factory-up` sources.
None of it is in this repository, none of it is passed on a command line, and `make factory-doctor`
reports only whether a value is present.
