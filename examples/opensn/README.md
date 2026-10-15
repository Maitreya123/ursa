# Run OpenSn parameter studies with AutoOpenSn

In this tutorial, URSA's execution agent runs parameter studies with
[OpenSn](https://github.com/Open-Sn/opensn), an open-source neutral-particle
transport code, through [AutoOpenSn](https://github.com/Maitreya123/AutoOpenSn).
AutoOpenSn is an MCP server whose tools render and run OpenSn input scripts from
templates verified against OpenSn's own tutorials and regression tests.

The split of work is the point of the example. URSA's agent understands the
question, writes a short study *spec*, and explains the results. AutoOpenSn
checks the spec against each template's declared parameter ranges and runs
only verified scripts. The agent never writes simulation input, and no language
model runs inside the AutoOpenSn server.

## What you will do

1. Install this example's environment, which includes AutoOpenSn.
2. Connect URSA to AutoOpenSn's MCP server.
3. Ask the execution agent how much the GMRES convergence tolerance changes the
   answer on the 1D Reed problem.
4. Check its results against the expected table below.

No OpenSn installation is needed for this example. When no cluster is
configured, AutoOpenSn runs the two one-dimensional scenarios in its own
built-in solver, which reproduces OpenSn's results for them.

## Prerequisites

- [uv](https://docs.astral.sh/uv/)
- git, so that uv can install AutoOpenSn from GitHub
- An API key for the inference provider URSA will use

## Install the example environment

Run every command from the [`examples/opensn`](.) directory.

```bash
uv sync
uv run autoopensn version
```

The second command prints AutoOpenSn's version and the OpenSn commit its
templates are pinned to. This example uses the URSA checkout two directories
above it, and installs AutoOpenSn at the commit it was tested against.

If this is your first URSA session, follow the
[configuration overview](../../docs/configuration/index.md) to configure your
LLM credentials.

## Connect URSA to AutoOpenSn

The [`ursa.yaml`](ursa.yaml) file in this directory adds one MCP server:

```yaml
mcp_servers:
  autoopensn:
    transport: stdio
    command: autoopensn
    args: [mcp, --runner, auto, --run-root, runs, --cache-path, runs/cache.db]
```

URSA starts `autoopensn mcp` itself and attaches its five tools to agents that
use tools:

| Tool | What it does |
| --- | --- |
| `list_scenarios` | Lists the 41 scenarios, which are the OpenSn tutorials |
| `describe_scenario` | One scenario's parameters, allowed ranges, and an example spec |
| `validate_spec` | Checks a spec and lists the runs it would make, running nothing |
| `run_study` | Starts a validated study in the background and returns a job id |
| `get_results` | Reports progress, and the results table when the study is done |

Confirm that URSA loaded the configuration:

```bash
uv run ursa --config ursa.yaml --print-config
```

Look for `mcp_servers.autoopensn` in the output. See the
[MCP configuration guide](../../docs/configuration/mcp.md) for other options.

## Run a study

Start the TUI:

```bash
uv run ursa --config ursa.yaml
```

Enter `/agents` and confirm that the `#execute` tab lists tools marked
`(MCP: autoopensn)`. Press `Esc`, then enter:

```text
#execute On the 1D Reed problem, how much does the GMRES convergence tolerance
change the answer? Compare tolerances 1e-4 and 1e-6 against a 1e-10 reference.
Report the results and say which engine produced the numbers.
```

The agent should find the `reed_1d` scenario, describe it, validate a spec
that sweeps `l_abs_tol` over 1e-4 and 1e-6 with a 1e-10 reference case, start
the study, and poll until it finishes. Expect a table like this:

| Case | Average flux | Relative difference from the reference | Sweeps |
| --- | --- | --- | --- |
| `l_abs_tol=1.0e-4` | 0.9588199124 | 2.69e-6 | 6 |
| `l_abs_tol=1.0e-6` | 0.9588224823 | 1.42e-8 | 8 |
| reference, 1e-10 | 0.9588224959 | 0 | 11 |

The answer should also say which engine produced the numbers. Without a
cluster, it is AutoOpenSn's built-in 1D solver, and the job's `note` says so,
for example "AutoOpenSn's own 1D solver. Not OpenSn." For this problem the
built-in solver and real OpenSn agree to every digit shown.

Each study's run directories, and a record of the job, are written under
`runs/` in this directory.

## Run real OpenSn on a cluster

With `--runner auto`, AutoOpenSn first tries to reach a cluster node over SSH,
by default through the `ssh` host alias `class01`, and falls back to the
built-in solver when it cannot. Scenarios other than the two 1D ones need real
OpenSn; without a cluster their runs are reported as unavailable.

To run on a cluster node where OpenSn is built at the pinned commit with its
Python module, name your SSH host and require it:

```yaml
    args: [mcp, --runner, remote, --host, my-cluster-node, --run-root, runs, --cache-path, runs/cache.db]
```

AutoOpenSn's cluster settings, such as which environment modules to load, are
currently set for Texas A&M's Orchard cluster. See
[running OpenSn](https://github.com/Maitreya123/AutoOpenSn/blob/main/docs/running_opensn.md)
and the
[MCP server documentation](https://github.com/Maitreya123/AutoOpenSn/blob/main/docs/mcp_server.md)
for the setup.

## Troubleshooting

- If no `(MCP: autoopensn)` tools appear, recheck the resolved configuration and
  make sure you started URSA with `uv run`, which puts the `autoopensn` command
  on `PATH`.
- If `validate_spec` keeps rejecting a spec, read its `errors`. They name the
  field and the allowed range or section, for example
  `l_abs_tol: 5.0 is above the declared maximum 0.01`.
- If a study fails, `get_results` returns the reason. The job's own log is at
  `runs/.mcp_jobs/<job_id>/worker.log`.
- Some OpenAI-compatible gateways reject the `null` content that LangChain sends
  with a message containing only tool calls, which fails every tool-using agent
  on its second turn with HTTP 422. AutoOpenSn's documentation describes a
  workaround for one such gateway.

Remove the `runs/` directory to delete all study results and the run cache.
