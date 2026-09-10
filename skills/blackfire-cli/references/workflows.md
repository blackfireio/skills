# Blackfire CLI workflows

Read [../SKILL.md](../SKILL.md) first for the command list and the conventions.

## 1. Find the environment

```bash
blackfire env:list --json
```

Pick the `uuid`, or a name exact enough to match one row. An organization environment carries an `organization` field and a personal one does not, which is the quickest way to tell them apart, and it matters: the personal agent is deprecated and its profiles cannot be listed, so prefer an organization environment for anything you intend to find again.

## 2. List recent profiles

```bash
blackfire profile:list --env <env> --limit 5 --json
blackfire profile:list --env <env> --transaction "GET /checkout" --from "2026-09-01" --to "2026-09-07" --json
blackfire profile:list --env <env> --cursor <next_cursor> --json
```

Each row carries `uuid` and, in `name`, the transaction. The numbers sit one level down: `data.envelope` for `wt`, `mu`, `ct` and the `nw*` byte counts, `context` for `{method, path, uri, status_code}`. Read `data.important_metrics.sql_queries` before anything else, because its `{ct, wt}` pair is the cheapest N+1 signal there is: a row reading `ct: 202` names its own problem, and you have not opened a profile yet.

The cursor already encodes the time window and the transaction filter, so drop `--from`, `--to` and `--transaction` when you pass it. For a regression, list the same transaction in a window before the change and one after, then read a profile from each: there is no comparison command.

## 3. Find the slowest SQL query

```bash
blackfire profile:sql <profile-uuid> --limit 10 --json
blackfire profile:graph <profile-uuid> --focus "PDOStatement::execute" --depth 2 --json
```

`profile:sql` lists queries slowest first and merges identical query text across drivers.

Profiles record function names, not file and line numbers. To reach the application code, focus the graph on the driver call (`PDOStatement::execute`, `mysqli_query`, `Doctrine\DBAL\Connection::executeQuery`) and follow its callers upward: the first caller belonging to the application is the code to change.

Two things split one function into several nodes. A driver call gets one node per query text it kept as an argument, so expect more than one row under the focus. In `--json` that argument is its own `arg` field next to `name`, and a third field `id` carries the pair already joined; only the Markdown table concatenates them for display. `--focus` matches either `name` or `name (arg)`, so quote the argument when you mean one specific query. Recursion gets one node per nesting level with an `@n` suffix (`App\Tree::walk`, `App\Tree::walk@1`, …), so focusing on the bare name reaches the outermost call only; raise `--depth` and the `@n` nodes appear among its callees, one row per level.

## 4. Explain why a request is slow

```bash
blackfire profile:show <profile-uuid> --json
blackfire profile:graph <profile-uuid> --sort exclusive-wt --limit 10
blackfire profile:graph <profile-uuid> --focus "<function>" --depth 1 --sort exclusive-wt --json
blackfire profile:timeline <profile-uuid> --json
```

`profile:show` gives the envelope in microseconds and bytes, plus the top metrics such as SQL or HTTP time.

The second call is how you learn which function to focus on: `--sort` and `--limit` shape a Markdown top ten instead of the whole graph. `--sort` takes four values, and each answers a different question. `inclusive-wt`, the default, ranks by subtree and points at the entry point. `exclusive-wt` ranks by a function's own time and points at the culprit. `calls` ranks by call count, which is how an N+1 announces itself. `memory` ranks by memory, which is the one to reach for when the complaint is memory rather than time. Take the name from that table into `--focus`.

That shaping applies to the Markdown table and to the `--focus` view only. With `--json` and no `--focus`, `--sort` and `--limit` change nothing and you get the whole graph, so a tools-only host cannot produce this top ten and picks its target from `profile_sql`, from the `profile_show` metrics or from `profile_timeline` instead.

The timeline shows in which order things ran, which is what you need when the cost is spread over many small calls.

## 5. Profile something now

```bash
blackfire curl --env <env> --json https://example.com/checkout
blackfire run --env <env> --json php bin/console app:import
```

Put every Blackfire flag before the URL or the program: both commands stop parsing at the first positional argument and hand the rest to curl or to the program, so a trailing `--json` would reach them instead of Blackfire. A misspelled Blackfire flag is worse than ignored: it becomes the first positional, so `run --environment prod php script.php` tries to execute `--environment` and reports that, not a flag error. The output is the new profile; its `id` is the profile UUID, take it into workflow 3 or 4. Add `--title "<why>"` so it can be found again.

`curl` needs a Blackfire Agent and a probe on the target, and `blackfire curl --ping <url>` tells you whether a URL is profilable at all. `run` is the exception: it does the daemon's job inside its own process, points the probe of the program it starts at itself whatever `blackfire.agent_socket` says, and uploads the profile directly, so it needs the probe in that runtime and network access to the API, and no daemon.

When a profile is *missing* rather than slow, do not reach for `run` again: see [troubleshooting.md](troubleshooting.md).

## Calling these as MCP tools

The tools are the read commands with underscores: `env_list`, `profile_list`, `profile_show`, `profile_sql`, `profile_graph`, `profile_timeline`, `profile_subprofiles`. An error is the same `{"code","message","hint"}` object carried as a failed tool result, since a tool call has no stderr.

Their output is close to what `--json` prints but not identical, and the difference decides how you parse it. The text is that JSON minus its trailing newline. Beyond that the tools split in two. `env_list`, `profile_list`, `profile_sql` and `profile_graph` *with* a `focus` are typed: they carry the CLI's shaped value, and also repeat it as compact `structuredContent`. `profile_show`, `profile_timeline`, `profile_subprofiles` and `profile_graph` *without* a `focus` are raw: they carry the verbatim API body and no `structuredContent` at all. So a field you found in the shaped output of one is not promised in the raw body of another.

The argument names are not the command line's: the profile UUID is a `profile_uuid` property rather than a positional, and flags drop their dashes (`env`, `limit`, `focus`, `depth`, `sort`, `from`, `to`, `transaction`, `cursor`). Read the schemas the server advertises rather than deriving names from the examples above, because `blackfire help` is out of reach from a host with no shell. No tool declares its arguments `required`, so a missing one comes back as the same error object rather than as a schema violation.

One thing about how the server was launched leaks into every call: `blackfire mcp --env <env>`, or `BLACKFIRE_ENV` in its environment, makes that environment a process-level default and pins the `profile_*` tools to it, which defeats the resolution from the profile UUID that those tools otherwise do. Launched without it, a profile UUID from any reachable environment resolves on its own. That is the better default, and worth checking first when a `profile_*` tool reports a profile it should be able to see as absent.
