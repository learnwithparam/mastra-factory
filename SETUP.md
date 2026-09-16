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

## Self-hosted does not mean no sign-in

Everything here runs on this machine: the server, its Postgres, its Redis, and the sandboxes agent
sessions work in. Nothing is provisioned by anybody else. That is what Mastra's own documentation
means by fully self-hosted, and its phrasing is exact: *configure your own authentication, storage
and sandbox providers*. Authentication is on that list rather than absent from it.

`auth: null` is documented and it works, for the API. The web application refuses it and says so:
*this server has no authentication provider configured*. Reading the shipped bundle explains why. It
asks `/auth/me` and branches on the status: 404 means authentication is off, 401 or 403 means it is
on and nobody is signed in, anything else is parsed as JSON.

With `auth: null` that route is never mounted, so the request reaches the single-page-app catch-all,
which answers 200 with HTML. The application parses HTML as JSON, throws, and renders a loading
marker forever, which is indistinguishable from a broken build.

`src/mastra/index.ts` mounts the 404 the application is looking for when auth is disabled. That does
not switch authentication off for the interface, which needs a provider either way. It replaces a
hang with a sentence naming the problem, and that is worth having whichever provider you choose.

So the self-hosted choice is an identity provider you own:

- **WorkOS**, by setting `WORKOS_API_KEY` and `WORKOS_CLIENT_ID`. Your account, your users, and
  Mastra is not in the request path at all.
- **Mastra platform sign-in**, the default when nothing else is set. One browser sign-in. The server
  still runs here and only identity is theirs, which is the part you would be giving up.

## Webhooks reach a machine the internet cannot see

GitHub cannot post to `localhost`, so the webhook at `/web/github/webhook` never fires here.

The GitHub integration also carries a reconcile sweep, and it has a `reconcileIssues` path as well
as a `reconcilePullRequests` one, so polling may cover issue intake on its own. That is read from
the shipped code and not yet watched happening, which is the difference between likely and known.
`MASTRACODE_GITHUB_RECONCILE_INTERVAL_MS` is set to a minute here so a session does not wait five.

Until it has been watched: run a webhook relay while a session is live, pointed at
`http://localhost:4111/web/github/webhook`. On a host with a real domain, neither is necessary.

## Secrets

Everything sensitive lives in `~/.config/lwp-secrets/factory.env`, which `make factory-up` sources.
None of it is in this repository, none of it is passed on a command line, and `make factory-doctor`
reports only whether a value is present.
