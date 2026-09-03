# workdock-autolog

Tops each workday up to 7.5h in Workdock from that day's git activity, on a nightly
schedule.

It is a **top-up**, not a logger: it reads the Workdock calendar first and posts only
the shortfall. Running it twice on the same day is a no-op, so an hour logged by hand
at midday simply means the nightly run adds less.

## What it does

At 22:00 it collects the day's commits, turns them into one time entry per ticket,
and posts them into two fixed blocks — 08:00–13:30 and 14:00–16:00 — packed from the
start of each. An entry too long for the room left in the morning is split across the
lunch gap.

It refuses to act on weekends, on future dates, on dates listed in `SKIP_DATES`, and
on days with no activity at all.

## Days that leave no trace in git

Planning, calls, review, PM — work that produces no commits would otherwise log
nothing, because commits are all the tool can see. Say what you did instead:

```sh
workdock-autolog --manual "sprint planning and roadmap review" \
                 --manual "1:30 client call about Q4 scope" \
                 --manual "backlog grooming"
```

An optional leading `H:MM` fixes that entry's length exactly; entries without one
share out whatever is left of the day's target. So above, the call books exactly 1:30
and the other two split the remaining 5:30 between them.

Your wording is used **verbatim** — manual text is never passed through the
summariser, because the point of typing it out is that these are your words.

Manual items combine with commits on a mixed day, so a morning of coding and an
afternoon of meetings comes out as both. Add `--manual-only` to ignore the day's
commits entirely. A `VEL-123 - ...` prefix files the entry under that ticket; without
one it is logged with a plain description, and no ticket id is ever invented.

## Install

```sh
git clone <this repo> ~/projects/workdock-autolog
ln -sfn ~/projects/workdock-autolog/workdock-autolog ~/.local/bin/workdock-autolog
ln -sfn ~/projects/workdock-autolog/worklog          ~/.local/bin/worklog

cp workdock.conf.example          ~/.workdock.conf          && chmod 600 ~/.workdock.conf
cp workdock-autolog.conf.example  ~/.workdock-autolog.conf  && chmod 600 ~/.workdock-autolog.conf
# edit both, then:
workdock-autolog --dry-run        # see what it would post
workdock-autolog --install        # schedule it at 22:00
```

`--install` writes `~/Library/LaunchAgents/com.workdock-autolog.daily.plist` and
loads it. `--install=18` picks a different hour. `--uninstall` removes it.

Requires `php`, `git`, `python3`; uses the `claude` CLI for entry wording if present.

## Usage

| Command | |
|---|---|
| `workdock-autolog` | top today up to 7.5h |
| `workdock-autolog --dry-run` | print what would be posted, post nothing |
| `workdock-autolog --yesterday` | ...or `--date=2026-09-02` |
| `workdock-autolog --status` | agent loaded? day's total? tail of the log |
| `workdock-autolog --show` | the activity a date would be derived from |
| `workdock-autolog --manual "..."` | log work that has no commits; repeatable |
| `workdock-autolog --manual "1:30 ..."` | ...with an exact duration |
| `workdock-autolog --manual-only` | ignore commits, use only `--manual` items |
| `workdock-autolog --force-weekend` | the only way to log a Saturday or Sunday |

Logs to `~/.local/share/workdock-autolog/`.

## Entry format

`VEL-123 - short description`, **no square brackets** (Harvest used brackets; Workdock
does not). One entry per ticket per day. Commits with no ticket collapse into a single
entry with a plain description — a ticket id is never invented.

## Two things that are easy to get wrong

**Scheduler environment.** launchd starts a job with
`PATH=/usr/bin:/bin:/usr/sbin:/sbin`. On a typical Mac that has *none* of `python3`
(homebrew), `php` (homebrew) or `claude` (`~/.local/bin`) on it, and leaves `USER`
unset — and the `claude` CLI reports "Not logged in" when `USER` is unset even though
its keychain item is readable. This silently cost `harvest-autolog` three nights of
logging. Defended twice over: the generated plist runs `/bin/bash -lc` with an
explicit `EnvironmentVariables` block, *and* the script re-prepends the paths and
derives `USER` from `id -un` itself, so a hand-edited plist cannot reintroduce it.

Reproduce a scheduler-like run before trusting a change:

```sh
env -i HOME="$HOME" PATH=/usr/bin:/bin /bin/bash -lc '~/.local/bin/workdock-autolog --dry-run'
```

**Commit dates.** Activity is matched on **author date**, not `--since`/`--until`
(which match the *committer* date). A cherry-pick or rebase rewrites the committer
date, so on a rebase-heavy repo that would credit work to the day it was moved rather
than the day it was done. Commits are deduplicated by **subject, not sha**, because
`git log --all` walks remote-tracking refs too — a cherry-picked commit otherwise
counts twice under two different shas.

## Credentials

Workdock rejects plaintext credentials — the browser encrypts them before login — so
`worklog` has to reproduce that, which is why the config holds a cipher key alongside
the password. That key comes from the app's own build and is not a secret; it is not
what protects the account.

What protects the account is the file. Treat `~/.workdock.conf` as you would any
password file: mode 600, never committed. `.gitignore` covers `*.conf`, and both
example files are placeholders only.

## `WORKDOCK_TASK_ID`

Every entry this tool creates is filed against a single Workdock task — the one your
day-to-day work belongs to, typically a project's development task. Workdock has no
public API, so the id is not discoverable from the outside; take it from the payload
the web app itself sends. Open the time-tracking page with the browser devtools
network tab recording, add an entry by hand, and read `task_id` off the `POST /timers`
request. That value goes in `WORKDOCK_TASK_ID` and every entry lands there.

Nothing instance-specific is baked into the tool: `WORKDOCK_BASE` and
`WORKDOCK_TASK_ID` are required configuration, so no URL or record id from any
particular deployment ships in this repo.

## Files

| | |
|---|---|
| `workdock-autolog` | the scheduler-facing script (Python 3) |
| `worklog` | posts and reads single entries (PHP) |
| `*.conf.example` | copy to `~`, fill in, `chmod 600` |
