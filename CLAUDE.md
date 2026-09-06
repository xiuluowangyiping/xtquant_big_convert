# CLAUDE.md

Notes for working in this repo. Everything here cost real time to learn; none of
it is obvious from the code.

## What this is

A Python RPC bridge exposing Big QMT (大 QMT) trading APIs to external programs,
plus a MiniQMT-compatible client layer. `passorder` / `get_trade_detail_data`
are globals injected into QMT's own Python process, which is why the bridge
exists: MiniQMT's `XtQuantServer` channel returns `connect() == -1` here.

Two sides, and it matters which one you are editing:

- **Server** — runs *inside* QMT: `src/bigqmt_signal_trader_strategy.py`,
  `src/bigqmt_signal_trader_redis_rpc_runtime.py`, `src/bigqmt_signal_trader/`
- **Client** — runs in the user's own program: `src/bigqmt_signal_trader/xtquant_compat.py`,
  `src/xtquant/` (a shim standing in for the real MiniQMT package)

## Hard constraints

**`get_trade_detail_data` returns EMPTY off the main strategy thread.** This is
the single most important fact in the project. It kills any "background worker
polls orders/positions" design. Order and query RPCs run on the adjust (main)
thread — see `ORDER_METHODS` / `LISTENER_DEFERRED_METHODS` in `redis_rpc.py`.
Never move them to a background thread, and never let `rpc_background_threads`
default to True in any config or template.

**QMT ships Python 3.6** (`bin.x64/python36.dll`). Server-side code must avoid
3.7+ syntax. Notably, module-level `__getattr__` (PEP 562) is 3.7+ and does
nothing there — f-strings are fine (3.6+), walrus and dataclasses are not.

**No star imports anywhere in `src/`.** The single-file builds exec each module
inside a `def _mod_N():` body, where `from X import *` is a SyntaxError. Pinned
by `tests/bigqmt_signal_trader/test_qmt_sandbox_loading.py`, which is the only
place in the suite that compiles a module inside a function.

**QMT enforces an import whitelist.** `socket` has been rejected in the field,
including indirectly (`logging_setup` → `logging.handlers` → `socket`). AST
scanning for direct imports is not enough. Keep setup-time tooling
(`init_config`, which imports `subprocess`/`getpass`) out of anything that runs
in the sandbox.

**Big QMT loads its OWN `xtquant`, not `src/xtquant/`.** The winning module
inside the sandbox is `bin.x64/Lib/site-packages/xtquant/`. Its `xtconstant`
defines **91** names; this repo's shim defines **538**. No value disagrees --
447 names are simply absent, `ACCOUNT_TYPE_DICT` among them. Reading one at
import time passes every test here and, on the live terminal, raises
`AttributeError` inside `build_app`, so `init` dies and every order/position
query answers `order_gateway is not configured` -- while `ping` still works and
the adjust loop still ticks, so the bridge looks healthy from outside.

Use `getattr(_xtconstant, NAME, default)` for anything beyond the plain
constants. Pinned by `tests/bigqmt_signal_trader/test_qmt_bundled_xtconstant.py`
against a snapshot in `tests/data/`. Pre-flight any server-side change with the
terminal's own copy first on `sys.path`:

```python
sys.path.insert(0, r"D:\...QMT...\bin.x64\Lib\site-packages")  # wins for xtquant
sys.path.insert(1, os.path.join(REPO, "src"))
import bigqmt_signal_trader.adapter_factory   # imports everything server-side
```

**The QMT sandbox never executes `bigqmt_signal_trader/__init__.py`.** Its
loader builds an empty module for the root package and returns it, because the
eager exports there trip QMT's import allowlist:

```python
# QMT native allowlist rejects the root package eager exports.
if name == "bigqmt_signal_trader":
    return module
```

So anything defined in `__init__.py` does not exist inside QMT — it works
everywhere you test and is missing in the one place that matters. The version
stamp shipped there first and came back from the live terminal as
`AttributeError: module 'bigqmt_signal_trader' has no attribute
'deployment_report'`. Put it in a submodule and import
`from bigqmt_signal_trader.version import ...`, never off the root package.
Pinned by `tests/test_version_stamp.py`.

**QMT's order/deal callbacks run on a C++ thread** entered via
`PyGILState_Ensure`. The first exec of a not-yet-imported module on that thread
fails in the C layer *without setting a Python exception* — it surfaces as
`SystemError: error return without exception set`. Import at module load, never
inside a callback.

**Futures symbols are case-sensitive; stock codes are not.** `normalize_stock_code`
keeps the caller's symbol verbatim when the suffix is one of `.SF` `.DF` `.IF`
`.ZF` `.INE` `.GF` (#58) and normalizes only the suffix, because big QMT has
`cu2610.SF` and does not have `CU2610.SF`. So **never call `.upper()` on a code
outside that function.** `quote_subscription_manager` did, and
`subscribe_whole_quote(["cu2610.SF"])` delivered exactly one frame and then
nothing (#95). That one frame is the initial snapshot, which takes the
case-preserving `get_full_tick` path -- so the symptom reads as "the
subscription dies after one tick" rather than "the code is wrong", and
`CF701.ZF` kept working throughout only because `.upper()` is a no-op on it,
which made the whole thing look like a 郑商所-vs-上期所 difference. Bare exchange
tokens (`SH` / `sz` / `if`) do still uppercase; that is the only case that
should.

**The client shim answers plausibly for things it does not do.** `src/xtquant/`
covers 538 names and some of them return a shape instead of an answer:
`create_sector` is a silent no-op, `get_sector_list` returns a hardcoded
fallback list, `get_stock_type` answered `0` for every code. Reading the shim is
not evidence that a feature works -- that is how I gave a wrong answer on #130
before #143 was opened to track the real gap. Check that the call reaches big
QMT.

**Compare path components, not substrings, when telling the two `xtquant`s
apart.** This repo's own directory is `xtquant_big_convert`, so a
`"/xtquant" in path` test excludes the entire repo and reports a reassuring
clean result.

**A QMT write function's return value tells you the request went out, not
that it worked.** Four issues in one day, all the same shape:

| | native return | truth |
|---|---|---|
| #148 | cancel returned false | the order was cancelled (50 -> 54 in 67ms) |
| #151 | cancel returned true | the order did not exist |
| #152 | submit settled with no id | the order was live at the broker |
| #142 | create_sector returned None | nothing was created, and none was expected to be |

So **read the result back before reporting success**, and when the read-back
disagrees, say so loudly. A false "it failed" costs someone a look at the 委托
list; a silent "it worked" costs them a duplicate order or a sector that never
existed. The sector family (#143) and the submit settlement (#152) both work
this way now.

Two things the wording has to get right when it does fail: say whether the
operation **is live** (a caller who retries on failure double-orders), and do
not reuse a neighbouring message that means the opposite -- "order not found
in system" leads with QMT's 模拟 run mode, which is the wrong place to look
when the order is actually at the broker.

## Nothing merges or ships without both gates

Before merging a PR or cutting a release, both of these, not either:

1. **The full suite passes** — and the collection count still matches the file
   count (see below).
2. **It was exercised against the running QMT terminal and behaved.** Green
   unit tests are not evidence on their own: they encode the premise the code
   was written from, so when the premise is wrong they pass anyway.

PR #132 is what earned this rule. 872 tests green, six new compat wrappers, all
six "already supported server-side". Live, five of them raise `needs native
xtdata SDK quote service` and `get_stock_type` returns **0 for every code** —
stock, ETF, bond, option, and every code format tried. Its test asserted only
that the parameter name was forwarded, never that the answer meant anything.
Merging on green would have replaced a loud `AttributeError` with a silent
wrong classification, which is the worse of the two.

**Verify the meaning, not the shape.** Confirming a response is well-formed
proves nothing about what it says. The README star-history image returned HTTP
200 and 60KB of valid SVG; the SVG's text said GitHub had restricted access to
the star data. PR #132's test confirmed the parameter was forwarded, never that
the answer was right. Read the content.

**A failed operation looks exactly like one that never ran.** Log rotation was
firing and raising `WinError 32`, and because the rotation is attempted inside
`emit`, a failure aborts the write -- so the file stopped growing, which I read
as "the rotation path was never triggered". Before concluding a code path is
dead, go looking for the error it would have raised.

Client-side changes can be probed from a temp worktree against the live bridge
without touching the deployment. Server-side changes need deploy **and** a
strategy restart first — otherwise the probe is measuring the old build.

## Testing

```bash
python -m pytest tests/ -q
```

Two habits worth keeping:

- **Check collection count, not just pass count.** A merged PR once left 27 test
  modules silently uncollectable; `-q` output looks identical.
  ```bash
  find tests -name "test_*.py" | wc -l
  python -m pytest tests/ --collect-only -q | grep -oE "^tests[^:]*\.py" | sort -u | wc -l
  ```
- **Verify a new test fails against the pre-fix code** (`git stash`, run, pop).
  Tests written from the same wrong premise as the code pass while the premise is
  wrong — that is exactly how PR #88's mis-mapped order types stayed green.

## Deploying to the live terminal

Server-side code goes to `D:\国金证券QMT交易端_lemo\python\`. Sync the package
directories **and the top-level modules** — `bigqmt_signal_trader_redis_rpc_runtime.py`
is a top-level file and is easy to miss. Never overwrite
`bigqmt_signal_trader_local_config.py` or `..._client_config.py`; they hold the
account id and credentials.

QMT keeps strategy modules in `sys.modules` across editor re-runs, so **a deploy
does nothing until the strategy is reloaded or restarted**. Since 0.3.8
`xt_trader.reload_deployment()` does it without a restart: it purges every
`bigqmt_signal_trader.*` module, re-points the references the strategy module
bound at import time, and re-runs `init()`. Poll `reload_status()` -- it reports
`ok`, `modules_purged` and the version stamp before and after. Verified by
deploying a changed `version.py` and watching `ping` move to it and back
(~0.8s per reload).

Two things it cannot refresh, so these still need a real restart:
`bigqmt_signal_trader_strategy.py` and `BIGQMT_REDIS_DRYRUN.py` -- QMT execs
those, and a module cannot reload the one it is running in.

The reload waits for the transport's response queue to drain before
`reset_app()`. Do not shorten that: the ZMQ reply is sent by the ROUTER thread
at the top of its own loop, after a `recv_multipart` that blocks up to RCVTIMEO
(1s), and that thread needs the GIL the adjust thread is holding. Tearing down
first loses the reply to `reload_deployment` itself, and from the client that
looks exactly like a reload that killed the bridge.

Since 0.2.16 the startup log says which build actually loaded, and
`xtdata.get_deployment_info()` answers the same over RPC — check that before
debugging a fix that "did not work". `xt_trader.sync_deployment()` does the copy
without a hardcoded path. After a reload or a restart, check
`userdata/log/XtClient_FormulaOutput_YYYYMMDD.log` (not `userdata_mini/`, which
is a stale MiniQMT subprocess) — the reqid suffix increments on a fresh
instance, and `[bigqmt_reload]` lines report each reload.

Pre-flight every server-side change by importing the whole module set with the
terminal's own `xtquant` first on `sys.path` (see the bundled-xtquant note
above). That catches the class of breakage that only shows up after a restart,
when the strategy is already down.

**When probing the deployed code, put your temp directory ahead of the QMT
directory on `sys.path`.** The QMT directory contains a real
`bigqmt_signal_trader_local_config.py`; getting the order wrong makes every
probe silently read the live config instead of your fixture, which looks exactly
like the fix not working.

## Logging

`logging.getLogger("bigqmt")` lives in logging's own registry, which nothing
purges -- while the entry drops every `bigqmt_signal_trader` module from
`sys.modules` on each start, and `reload_deployment` does the same. So a
module-level `_initialized` flag cannot guard `_setup()`: it resets, the logger
does not, and handlers accumulate one per start (#139 -- one line written 16
times, and the same `ping breakdown` 373 times in QMT's panel). `_setup`
detaches **and closes** the existing handlers; removing without closing leaves
the file handle open, which is the failure below.

The log file name carries a per-process tag (`BIGQMT_LOG_NAME` then
`BIGQMT_ACCOUNT_ID` then `bigqmt-pid<N>.log`) because two bridges -- one live
account, one simulated -- both fell back to a single `bigqmt.log` (#144). Two
OS handles on one file means `TimedRotatingFileHandler` can never rename it on
Windows, every write raises, rotation never succeeds, and `backupCount` pruning
therefore never runs, so the log grows without bound. The rotator now tolerates
a failed rename for anyone who pins one name deliberately.

Your own probe scripts are a second process: they take a handle on the
deployment's log file unless you set `BIGQMT_LOG_NAME`. Seven `WinError 32`
lines I spent a while attributing to the bug were my own probe.

## The optional redis

Redis is optional on the zmq / mysql / shm transports, and two things about that
were wrong until 0.3.12.

**"Configured" is not "reachable".** redis-py builds its client lazily and does
not dial until the first command, so a stale redis block yields a
healthy-looking client that times out on every publish. `_exec_event_sink`
treated "we built a client" as "redis works" and preferred it over the zmq push
channel that was already running fine -- losing every order/trade callback while
printing a traceback per event (#145). Publishing now falls back to the push
channel, demotes redis after `_EXEC_REDIS_FAILURE_LIMIT` failures, and throttles
the traceback.

**Every `if not redis_config: return None` guard was dead code**, because
`configure_runtime` emitted the block unconditionally from module defaults --
`config["redis"]` was never empty, so "I have no redis" was not expressible. I
told a reporter to remove the block, which was impossible. `redis_enabled=False`
now empties it and the existing guards do the rest; `transport="redis"`
overrides the switch, since there the bridge itself needs redis and honouring it
would break the RPC rather than the optional extras.

## Releasing

Version in **two** files -- `pyproject.toml` AND
`src/bigqmt_signal_trader/version.py` -- then the `CHANGELOG.md` entry, then
tag and push. `tests/test_version_stamp.py` pins the two version strings
together, so a half-bump fails the suite rather than shipping a build that
misreports itself; that stamp is what `ping` and `get_deployment_info()`
answer with, and the whole point of it is telling a deployed tree apart from
the package it came from.

**Rename the `## [Unreleased]` heading, do not just add under it.** 0.2.7
shipped -- tag, PyPI, GitHub release -- with its CHANGELOG section still
titled `## [Unreleased]`, and it sat that way for twelve releases: a second
`## [Unreleased]` buried mid-file, so its entries were invisible in the
release they belonged to. Check there is exactly one before and none after,
and check **both spellings** -- entries land under `## [未发布]` as often as
`## [Unreleased]`, so a grep for only the English one reports zero while the
section is sitting right there at the top of the file.

**Re-sync the no-redis fork on every release that touched the zmq transport.**
`bigqmt_no_redis/zmq_transport.py` is a **hand-maintained** self-contained copy
of `src/bigqmt_signal_trader/transports/zmq_transport.py` with the three package
imports inlined (`decode_text`, `encode_rpc_request_payload`,
`decode_rpc_request_payload`, and the `RpcTransport`/`TransportError`/
`TransportTimeout` base) and redis service discovery stripped, so it can load
where `import redis` is rejected. There is **no generator** — nothing re-derives
it — so it silently drifts: it sat at the Jul-29 shape while the source got #177
(wake pipe) and #193 (per-thread DEALER), which means the pure-zmq / no-redis
deployments — the ones that most need low latency — shipped the OLD transport.
The no-redis single-file build (`build_no_redis_single_file_flat.py`) overrides
the source transport with this file, so a stale fork ships stale. On any change
to the source transport: regenerate this file (current source + the same
de-redis transformation), confirm it still imports with **no** redis on the path
and carries the new code (grep the new symbols), then rebuild the no-redis
single file. Same applies to any other hand-copied module under
`bigqmt_no_redis/`.

**Write release notes to a file and use `--notes-file`.** Backticks in an inline
`--body` are command substitution and get silently eaten — this has corrupted
release notes and an issue comment. Same for `gh issue comment`.

**`gh release create` uploads assets after creating the release, and deletes the
release if an upload fails.** If it times out and moves to the background, do
NOT upload manually — the duplicate returns HTTP 422, the background create
rolls back, and the whole release disappears while the tag stays. Create the
release with no assets, then `gh release upload` one file at a time. Verify
afterwards that both artifacts are attached and that the notes still render.

**Backslashes do not survive a heredoc in this environment, even a quoted one.**
A Python probe written with `<<'PYEOF'` loses `\n` and `\\n` alike and prints a
literal `n`. Build them with `chr(92)`, or write the file with a dedicated tool.
This has cost time on four separate days.

## Working with contributed PRs

Several arrive AI-generated with passing tests. Two things have needed catching:

- **Constants asserted as literals.** Check every value against
  `src/xtquant/xtconstant.py` by name. PR #88 mapped `33`/`34` as special credit
  financing; they are stock-option operations (`40`/`41` are the credit ones),
  and its tests encoded the same mistake.
- **Contributors' own live settings in templates.** One PR carried a real account
  id plus `rpc_allow_order_methods=True`. Diff config blocks against
  `src/bigqmt_signal_trader_local_config.example.py` before merging.

Fork PRs cannot be pushed to. Merge, then land corrections as a follow-up PR
from a branch in this repo.

## Reporting

Say what was verified and what was not. Several fixes here can only be confirmed
by a reporter with a credit account, a restricted broker sandbox, or live fills.
Record those as known limitations in the release notes rather than implying
coverage — and when a limitation is later resolved by real evidence, say so
explicitly in the next release.
