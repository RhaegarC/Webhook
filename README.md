# Webhook API

A small service that listens for activity from GitHub and keeps a record of it.

## What it is

GitHub can notify an outside service whenever something happens in a repository — a pull request
is merged, a comment is left, a branch is pushed. This project is the service on the receiving end
of those notifications.

Events arrive, are confirmed to be genuine, and are written down. Later, they can be looked up
again: what happened, in which repository, when, and by whom.

Useful when you want to know what occurred, not just what the repository looks like today.

## What it does

- **Receives** notifications from GitHub.
- **Confirms** each one genuinely came from GitHub and not from someone else.
- **Keeps** a complete record, so an event can always be revisited.
- **Answers questions** about what has been received.

## Why bother

GitHub does not keep a replayable history of the notifications it sends, and it does not resend
them. If nothing is listening, or the listener fails to record an event, that event is gone.

This service exists so that it isn't.

## Status

**Design phase — not yet implemented.** The requirements have been written and reviewed; the
service itself has not been built. See [docs/PRD.md](docs/PRD.md) for the full specification.

## Scope

The first version records events and answers questions about them. Reacting to events — sending
notifications, triggering other work — is deliberately left to a later phase.

## Documentation

| Document | Contents |
|---|---|
| [docs/PRD.md](docs/PRD.md) | Full requirements specification |
| [docs/features/](docs/features/) | The work broken into six pieces, with acceptance criteria |

## License

See [LICENSE](LICENSE).
