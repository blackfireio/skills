# No profile appears

`run` or `curl` succeeded but no profile came back, or the interface shows nothing. Read [../SKILL.md](../SKILL.md) first for the conventions.

## First, prove it is really absent

```bash
blackfire profile:list --env <env> --limit 5 --json
```

An untitled profile is easy to miss in the interface. The likelier cause is that the `run` or `curl` carried no `--env`: neither command fails for that, they resolve one and profile into it, so the profile exists in an environment nobody is watching. `env:list --json` gives you the candidates, and [../SKILL.md](../SKILL.md) gives the resolution order. This is a read command against the API, so it works from anywhere.

Do not expect this step to cover the likeliest case, though, because the environment a `run` falls back to is the one you cannot list. A personal agent has no environment attached to it, and the profile list is served per region, so there is no region to ask: `profile:list` against a personal environment answers a 404 as exit 3 no matter how right the rest of the command is. Organization environments list normally.

So when the notice said the profile went to your Personal environment, recover it from what the command already told you rather than from a list: `run` and `curl` print the profile's URL on stderr, and `profile:show <uuid>` still resolves a personal environment's profile by UUID. Then fix the cause rather than the symptom. The personal agent is deprecated, which is why the interface answers `Deprecated Personal Agent. Switch to an environment instead.`, and every organization has an environment meant for exactly this. Pass it as `--env` and the profile becomes listable like any other.

## Which path was the profile on

`run` never contacts the daemon, and it may already have reported the probe through its own exit code. That only happens when the profiled program itself exited 0: `run` then exits 1 with `No data received from the probe, is it installed and enabled correctly?` and a second line pointing at `/help/probe-not-found`, if nothing arrived. When the program ran and exited non-zero, `run` hands back that status and never gets to a probe verdict, so check the probe yourself below. Do not read every non-zero `run` as the program's own failure though: Blackfire exits non-zero as well when it could not execute the program at all, when the credentials were refused and when the upload failed, and the printed message is what tells those apart. For a missing `run` profile, skip the daemon entirely and go to "Prove the chain".

For a web request or a `curl`, the daemon is in the path and the steps below apply.

## Where to run what

`doctor` and `agent:events` talk to the daemon, so they run where the daemon runs. The `php -m` and `php -i` checks run where the application ran. On Upsun that is the same container and `upsun ssh --` prefixes everything. Under docker compose it usually is not: `doctor` and `agent:events` go to `docker compose exec <agent-service>`, the PHP checks to `docker compose exec <app-service>`. From the docker host you can reach the daemon with `--socket=tcp://localhost:8307`, but only if that port is published rather than merely exposed. `--socket` belongs to the daemon-facing commands alone, so it goes after the command name (`blackfire doctor --socket=...`) and the read commands and `mcp` will not take it at all. Client credentials have to exist in whichever container runs `run`, `curl` or `profile:list`.

When the daemon and the application are not yours to operate, run nothing there yourself: hand the commands to whoever administers them and keep to the read commands against the API, which need only your own client credentials.

## The four steps

Run `doctor` before `agent:events`: on an Agent too old to know the `events` request, asking for events refreshes `last_probe_ping_at`, the field step 1 reads.

```bash
blackfire doctor --json
blackfire agent:events --json
```

1. `doctor --json` exits 1 when a check it votes on failed, with `failures` naming it, and 0 otherwise. Read that 0 narrowly: the verdict covers the credentials, the APM and continuous-profiling upload stats, and a `last_profile_error` newer than `last_profile_at`. The browser and profile sections are reported but never voted on, so exit 0 is not a clean bill of health, it only means nothing in that shorter list failed. Then read three fields inside `agent`: `last_probe_ping_at` (when a probe last reached the daemon), `last_profile_at` (when a profile last completed) and `last_profile_error`, an object `{occured_at, reason}`. That last field is narrower than its name: it records a profile the daemon finished but could not upload. A profile the daemon refused to start, for invalid credentials or a bad signature, never lands there, so an empty `last_profile_error` does not mean nothing was refused. Refusals show up in step 2. The `occured_at` and `connnection_status` spellings are the daemon's own. Do not look for `connnection_status` beside the three fields above either: it sits under `credentials.<name>`, one per credential pair, not under `agent`.

2. `agent:events --json` returns the daemon's most recent records, at info level and above whatever level it was started with, so it works on a default install that writes nothing to disk. Two hundred records is the ceiling, not a promise: the ring also caps itself at 256 KiB, so verbose records mean fewer of them. Read the newest records at the end of `events`. `Agent is not targeted` means the credentials that created the profile do not belong to the environment the daemon serves. `Request signature cannot be validated` means the probe and the daemon disagree on credentials. `The probe requested the cancellation of the profile` or `Aggregation canceled` means the request ended before the profile was complete. Records live in memory, so a restart loses them: a changed `agent_started_at` tells you that happened, and `next_seq` can be passed back as `--since`.

3. If `last_probe_ping_at` is old or missing, nothing ever reached the daemon. That field decides it alone: an empty `events` alongside it is not a second witness, since a working profile writes no record there either. Two causes, in this order. First check the probe is loaded in the runtime that actually ran. In PHP that means the right SAPI: `php -m | grep blackfire` for a console command, the fpm configuration for a web request, and the probe is routinely installed for one and not the other. In Python it is the `blackfire` package in the interpreter that served the request, `python -c 'import blackfire; print(blackfire.__version__)'`, which for a virtualenv or a container means that interpreter and not the system one. If the probe is there, compare the address it dials against the daemon's own socket, which is the `socket` row of `agent.config_variables`. In PHP the probe's address is `blackfire.agent_socket`, from `php -i | grep blackfire.agent_socket`. In Python it is `BLACKFIRE_AGENT_SOCKET` in the application's environment, falling back to `~/.blackfire.ini` and then to a platform default, `unix:///var/run/blackfire/agent.sock` on Linux. Do not use the top-level `socket` field: it only echoes the address you dialled. Compare transport and port rather than strings, because a probe pointing at `tcp://blackfire:8307` and a daemon bound on `0.0.0.0:8307` is healthy. What actually breaks: the app container cannot resolve that hostname, the daemon binds loopback inside its own container, or one side is a unix socket and the other TCP, which only works when the same file is bind-mounted into both containers.

4. If `last_profile_error` or a record names credentials or a quota, the fix is a configuration or a plan change. Report it, do not attempt it.

Both commands need a CLI and an Agent recent enough to carry them. An older Agent answers `agent:events` with exit 1 and an upgrade message: report it and rely on `doctor` alone. An older CLI accepts `doctor --json` and ignores it, printing the text report, so if stdout is not JSON it is the CLI that needs upgrading, not the setup you are diagnosing.

## Prove the chain

Do not re-run the user's command. `run` executes it for real, and the commands people profile import, migrate, send or bill. Profile something inert instead:

```bash
blackfire run --env <env> --title "pipeline check" --json php -r 'usleep(1000);'
```

The probe message and exit 1 mean the probe is not loaded in the runtime that ran, which `php -m | grep blackfire` confirms there for PHP; the probe is routinely installed for fpm and not for the CLI.

A profile `id` on stdout means the opposite: the probe in that runtime, the credentials and the API all work, and the fault is specific to the original command. Look there for a hard termination, since a signal killing the process loses the profile because the probe never gets to flush; a plain `exit()` and a fork whose child ends are not it, both still produce a profile.

If this proof works while the original `run` exited 0 and its profile is still missing, nothing was lost: the profile was created and is somewhere you have not searched, another environment or untitled in the one you did.

This proof says nothing about the daemon: `run` never contacts it, so `last_probe_ping_at`, `last_profile_at` and `next_seq` do not move for a `run`, and their stillness afterwards is expected rather than a finding.

To prove the daemon's own path, send a request through the web SAPI, `blackfire curl --env <env> --json <url>` on a page that only reads, then read `doctor` again: `last_probe_ping_at` and `last_profile_at` both advance for a profile that went through the daemon. `agent:events` does not, because a successful profile writes no record at info level, so an empty ring after a working profile is not evidence of anything.
