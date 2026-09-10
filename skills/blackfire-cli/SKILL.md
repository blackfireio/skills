---
name: blackfire-cli
description: Profile PHP and Python applications and read Blackfire profiles from the terminal with the blackfire CLI. Use this skill whenever a task mentions Blackfire, a Blackfire profile, environment or profile UUID, or asks why a PHP or Python request, endpoint, console command or SQL query is slow, which function or query costs the most time or memory, or how to find the code behind a slow query, even when the user does not name Blackfire or profiling explicitly.
license: MIT
compatibility: Requires the blackfire CLI with client credentials (blackfire client:config, or BLACKFIRE_CLIENT_ID and BLACKFIRE_CLIENT_TOKEN). Under server credentials instead, env is ignored and env:list fails with code auth
metadata:
  author: Blackfire
---

# Blackfire CLI

Blackfire records what an application did during one request or command and stores it as a profile inside an environment. The `blackfire` CLI creates profiles (`run`, `curl`) and reads them (`env:list`, `profile:*`). The same read commands are MCP tools through `blackfire mcp`.

## Commands

| Command | Use it for | `--env` |
| --- | --- | --- |
| `env:list` | find an environment's `uuid` or `name` | not a flag here |
| `profile:list` | recent profiles of an environment, paged with `next_cursor` | required |
| `profile:show` | the envelope: `wt`, `cpu`, `io`, `mu`, `pmu`, `ct`, `nw`, plus top metrics | from the UUID |
| `profile:sql` | queries slowest first, with call count and wall time | from the UUID |
| `profile:graph` | the call graph, narrowed with `--focus` and ranked with `--sort` | from the UUID |
| `profile:timeline` | in which order things ran | from the UUID |
| `profile:subprofiles` | the sub-profiles of a distributed profile | from the UUID |
| `run` | profile a command or script now | optional, and it decides where the profile lands |
| `curl` | profile a URL now | same |
| `doctor`, `agent:events` | read the daemon's state, only when a profile is missing | no |

`--env` takes a UUID, a name, or a case-insensitive fragment of one, and it comes from `BLACKFIRE_ENV` when you leave it out. The read commands and `run`/`curl` match it differently, which is why passing the UUID is the habit that always works.

The `profile:*` read commands try the UUID, then an exact name, then a fragment, and refuse only a tie inside one of those tiers, as a usage error listing the candidates. An exact name therefore beats a longer name that merely contains it.

`run` and `curl` have no exact-name tier: a name that merely contains your value matches just as well, and two matches abort with `Ambiguous --env value: "<value>" matches several environments, please provide an UUID.` and no candidate list. So given environments `prod` and `prod-eu`, `run --env prod` fails even though `prod` names one of them exactly. Pass the UUID to `run` and `curl` whenever names overlap.

Every `profile:*` command except `profile:list` resolves the environment from the profile UUID, so leave `--env` off. It is accepted there and it overrides that resolution, which means a wrong value turns a call that would have worked into an exit 3. Pass it for one reason, when the resolution itself failed because your credentials cannot see the profile's environment:

```
No agent found for profile <uuid>, its environment is not among the agents your client credentials can access
```

That is the case the flag exists for, and the CLI's own hint points at it.

`run` and `curl` never fail for want of `--env`, they pick one and profile anyway, which is the usual reason a profile is nowhere to be found. `run` resolves `--env`, then `BLACKFIRE_SERVER_ID`, then the personal environment, then the first the credentials reach, and prints a notice on stderr naming its choice. That third step is the one that bites: the personal agent is deprecated, and a personal environment is the one place `profile:list` cannot look, so a profile that lands there is hard to find again. Every organization has an environment intended to replace it, so pass `--env` with that instead of letting the fallback choose. `curl` resolves the same way minus `BLACKFIRE_SERVER_ID`, and says nothing at all. So always pass `--env` when you know it, and read stderr when you did not.

Which file answers which question is listed under [References](#references) at the end.

## Conventions

Run `blackfire help <command>` before using a flag you have not seen in this session; the installed help describes the exact binary you are calling, so it beats any memorised example, including the ones in these files. `blackfire help --json` prints the whole command tree in one call.

Two things to expect from that manifest. It lists canonical names, so the `run`, `curl`, `doctor` and `help` you type appear there as `client:run`, `client:curl`, `doctor:check` and `self:help`. And it reports `--env` on `profile:list` as `"required": false` although the command refuses to run without it, so on that one flag trust this file instead.

Pass `--json` whenever you will parse the output: JSON goes to stdout and progress to stderr, so piping stdout is safe. Without it the commands print Markdown for a human reader.

With `--json`, read commands (`env:list`, `profile:*`) report a failure as one `{"code","message","hint"}` object on stderr; without it the same message is plain text. One case sits outside that contract: a missing positional argument is refused by the console as prose before the command runs, so tolerate a non-JSON stderr there. They exit 1 for a usage error, 2 for missing or invalid credentials, 3 when the profile or environment does not exist, 4 for an API or network problem. Apply the `hint` before retrying.

`run` and `curl` differ: they take `--json` for their result but print their errors as plain text, and `run` exits with the status of the program it profiled. Blackfire only gets to add a verdict when that program exited 0: `run` then exits 1 with `No data received from the probe, is it installed and enabled correctly?`, followed by a second line pointing at `/help/probe-not-found`, if nothing arrived. So an exit 0 from `run` means the probe did report. A non-zero is not attributable on its own: it is the program's status when the program ran and failed, but Blackfire uses it too, for a program it could not execute at all, for credentials that were refused and for a profile it could not upload. Read the printed message, it is what tells them apart.

Start narrow, because these payloads are big enough to crowd out the reasoning you wanted them for. `profile:graph --json` without `--focus` returns the whole graph, thousands of nodes on a real application. `profile:timeline --json` is heavier still, around 170 KB for one demo request. And `profile:show --json` carries a few dozen keys of which two are usually the point: `profile.data.envelope` for what the request cost, `profile.metrics` for where it went. Reach for `profile.data.envelope` rather than `profile.envelope`, which only some profiles carry. Open the timeline only for a question about ordering, once the cheaper commands have failed to explain the cost.

Exit code 2 means the client credentials are missing or wrong: ask the user to run `blackfire client:config` or to export `BLACKFIRE_CLIENT_ID` and `BLACKFIRE_CLIENT_TOKEN`. Never print, echo or paste credentials into a transcript or a file.

In Blackfire the "agent" is a daemon that forwards profiles to the servers, nothing to do with coding agents. `doctor` and `agent:events` are the only two daemon commands to run, both read only. Never run `agent:start`, `agent:config` or `agent:log-level`; hand those to the user.

## References

Read the one that matches your situation, you do not need both.

- **[references/workflows.md](references/workflows.md)** - going from a question to an answer: find the environment, list recent profiles, chase the slowest query up to the application code that issues it, explain why a request is slow, profile something now. Read it also for the `blackfire mcp` tool equivalents, whose output contract differs from the CLI's.
- **[references/troubleshooting.md](references/troubleshooting.md)** - `run` or `curl` reported success and no profile appeared. Which environment it landed in and why that one may not be listable, what `doctor` and `agent:events` can and cannot tell you, and how to prove the chain without re-running the command the user profiled.
