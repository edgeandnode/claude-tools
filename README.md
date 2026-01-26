# claude-tools

Tools for running Claude Code autonomously in a sandboxed environment. The main
use case is letting Claude work through a list of tasks iteratively, with each
iteration running in isolation to limit potential damage from mistakes.

## Components

### `bin/loop`

An autonomous loop runner that executes Claude iteratively until tasks are
complete. Each iteration runs in a sandbox and outputs are logged for review.

```
Usage: loop [-f <prompt-file> | -s <file>...] [-n <max-iterations>]
  -f <prompt-file>    Use prompt from file
  -s <file>           Study file(s) and work on tasks (can be repeated)
  -n <max-iterations> Maximum number of iterations (default: 20)
```

The loop exits when:
- Claude creates `target/loop/stop` (signaling completion)
- Maximum iterations are reached
- Claude exits with an error

**Output files** (in `target/loop/`):
- `stop` - Signal file to halt the loop
- `progress.txt` - Log of iteration start/end times
- `iter-N.json` - JSON stream output from each iteration
- `prev/` - Previous run's iteration files (moved on new run)

### `sandbox/`

Sandboxing scripts for different operating systems. Each script takes a command
to run inside the sandbox.

**Available implementations:**

- `bubblewrap` - Linux sandbox using [bubblewrap](https://github.com/containers/bubblewrap)
  - Read-only system directories
  - Read-write access only to the current git repository
  - Works with git worktrees
  - Isolated PID namespace and resource limits

## Installation

1. Install dependencies:
   - [clox](https://github.com/edgeandnode/clox) - Claude output formatter
   - [bubblewrap](https://github.com/containers/bubblewrap) - Linux sandboxing (if on Linux)

2. Create a `sandbox` symlink on your PATH pointing to the appropriate sandbox
   script for your OS:
   ```bash
   ln -s /path/to/claude-tools/sandbox/bubblewrap ~/bin/sandbox
   ```

3. Add `bin/` to your PATH or symlink `bin/loop`:
   ```bash
   ln -s /path/to/claude-tools/bin/loop ~/bin/loop
   ```

## Usage

**Work through tasks in a plan file:**
```bash
loop -s PLAN.md
```

**Use multiple study files:**
```bash
loop -s PLAN.md -s SPEC.md
```

**Custom prompt from file:**
```bash
loop -f my-prompt.txt
```

**Limit iterations:**
```bash
loop -s PLAN.md -n 5
```

**Stop early:**
```bash
touch target/loop/stop
```

## How it works

1. The loop script builds a prompt (from file or by combining study files)
2. Each iteration runs `sandbox claude` with the prompt
3. Claude works on tasks, commits changes, and marks progress
4. When all tasks are done, Claude creates `target/loop/stop`
5. The loop detects the stop file and exits

The sandbox ensures Claude can only modify files within the current git
repository, protecting the rest of your system.
