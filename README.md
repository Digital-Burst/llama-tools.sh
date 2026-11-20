# start-llama-profile.sh

A small Bash wrapper that runs [llama-server](https://github.com/ggml-org/llama.cpp)
with parameters loaded from a profile file.

Define a configuration once, then start the server with a single word:

```terminal
$ start-llama-profile llama3-8k
```

## Features

- **Named profiles**: Keep model params and tuning options in small, versionable text files.
- **XDG-compliant storage**: Profiles live in `$XDG_CONFIG_HOME/llama-cpp/profiles/`.
- **Flexible selection**: Pick the profile on the command line *or* via the `LLAMA_PROFILE` environment variable (command line wins).
- **Per-run overrides**: Append arguments after the profile name to override individual parameters without editing the file.
- **Safe profile syntax**: Shlex-style parsing with single/double quotes, backslash escapes, and `#` comments.
- **Self-inspecting**: `--list` shows available profiles, `--dry-run` prints the exact command line that would be executed.
- **No dependencies**: Pure Bash (runs even on macOS's stock Bash 3.2).

## Requirements

- Bash 3.2 or newer
- [llama.cpp](https://github.com/ggml-org/llama.cpp) with `llama-server` binary in your `PATH` (or in `LLAMA_SERVER_BIN`)

## Installation

Clone this repository and run the following from within the repo directory.

```terminal
$ install -Dm755 start-llama-profile "$HOME/.local/bin/"
$ mkdir -p "${XDG_CONFIG_HOME:-$HOME/.config}/llama-cpp/profiles"
```

**NOTE**: The `$HOME/.local/bin` directory needs to be in your `PATH`. You can replace this with whatever
path you want to install the script in to.

## Quick start

Create a profile in `~/.config/llama-cpp/profiles/llama3-8k`:

```
-m /data/models/llama-3-8b-instruct.gguf
--alias "Llama 3 8B Instruct"
--host 127.0.0.1
--port 8080
-c 8192
-ngl 33
-t 8
--temp 0.7
--top-p 0.9
--jinja
```

Then run it:

```terminal
$ start-llama-profile llama3-8k
```

`llama-server` starts with the command line options given in the profile file.

## Profile selection

The profile is chosen in this order (first match wins):

1. **Command line**: The first non-option argument:

   ```console
   $ start-llama-profile llama3-8k
   ```

2. **Environment**: The `LLAMA_PROFILE` env variable:

   ```terminal
   $ export LLAMA_PROFILE=llama3-8k
   $ start-llama-profile
   ```

If both are given and they differ, the command line wins and the script prints a
warning naming the profile that was ignored, so you always know what actually ran:

```terminal
start-llama-profile: warning: using profile 'other' from the command line, ignoring LLAMA_PROFILE='basic'
```

> **Tip:** a profile name that contains `/` or starts with `~` is treated as an
> explicit file path, so you can run one-off profiles from anywhere:
>
> ```terminal
> $ start-llama-profile ~/experiments/quant-test.profile
> ```

## Profile files

### Location

Profiles are searched in the XDG user config directory:

```
${XDG_CONFIG_HOME:-~/.config}/llama-cpp/profiles/
```

Both `name` and `name.profile` are accepted, so `chat` finds `chat` *or* `chat.profile`.
Use `--profile-dir DIR` to look somewhere else.

### Syntax

Profiles are parsed with shlex-like rules:

| Element        | Meaning                                                        |
| -------------- | -------------------------------------------------------------- |
| Whitespace     | Separates arguments                                            |
| `#`            | Starts a comment (to end of line)                              |
| `'...'`        | Single quotes: literal text, no escapes                        |
| `"..."`        | Double quotes: allow escapes `\n` `\t` `\"` `\\`               |
| `\`            | Backslash escapes the next character (outside quotes)          |
| Blank lines    | Ignored (CRLF line endings are tolerated)                      |

Quotes must not span multiple lines. Example:

```
# A profile with comments and quoted values
-m /models/mixtral-8x7b-instruct.gguf
--alias "Mixtral 8x7B Instruct"   # displayed in the Web UI

# Performance
-ngl 33        # offload all layers to the GPU
-c  16384      # context size
-t  16         # CPU threads

# Sampling
--temp 0.7
--top-p 0.9
--jinja        # use the model's chat template
```

### Example profiles

**`basic`**: minimal local chat server:

```
-m /data/models/llama-3-8b-instruct.gguf
--port 8080
-ngl 99
-c 8192
```

**`api`**: exposed for LAN clients with an API key:

```
-m /data/models/qwen2.5-32b-instruct-q4_k_m.gguf
--host 0.0.0.0
--port 8080
--api-key "s3cr3t-2-3v3ry0n3"
-c 32768
-ngl 99
--jinja
```

## Usage

```
start-llama-profile [options] [PROFILE] [llama-server arguments...]
```

Script options must appear **before** the profile name; everything after the
profile name is passed through to `llama-server`. Use `--` to force ambiguous
flags (such as `--help`) to the server.

### Usage Examples

```terminal
# Run a profile
$ start-llama-profile llama3-8k

# Run a profile and override one parameter
$ start-llama-profile llama3-8k --port 8081

# Profile from the environment, extra parameters on the command line
$ LLAMA_PROFILE=llama3-8k start-llama-profile -m /data/models/other-model.gguf

# See what would be executed without starting the server
$ start-llama-profile llama3-8k --dry-run
start-llama-server: profile /home/you/.config/llama-cpp/profiles/llama3-8k (command line)
llama-server\ -m\ /data/models/llama-3-8b-instruct.gguf --alias\ "Llama 3 8B Instruct" --port 8081

# List available profiles
$ start-llama-profile --list
Profiles in /home/you/.config/llama-cpp/profiles:
  api
  basic
```

### Overriding parameters

Arguments given after the profile name are appended **after** the profile's
parameters. Since `llama-server` takes the last occurrence of a repeated
option, this gives you clean per-run overrides.

```terminal
$ start-llama-profile api --port 9090 --temp 0.2
```

The profile's `--port 8080` and `--temp 0.7` are silently superseded by the
later values.

## Command-line reference

### Options

| Option                    | Description                                                     |
| ------------------------- | --------------------------------------------------------------- |
| `-h`, `--help`            | Show help and exit                                              |
| `-l`, `--list`            | List available profiles and exit                                |
| `--dry-run`               | Print the resolved command line (shell-quoted) and exit         |
| `--profile-dir DIR`       | Use `DIR` instead of the XDG profile directory                  |
| `--`                      | Stop option parsing; pass all remaining arguments to the server |

### Environment variables

| Variable             | Default          | Description                                        |
| -------------------- | ---------------- | -------------------------------------------------- |
| `LLAMA_PROFILE`      | *(unset)*        | Profile to use when none is given as an argument   |
| `LLAMA_SERVER_BIN`   | `llama-server`   | Binary to exec (path or `PATH`-relative name)      |
| `XDG_CONFIG_HOME`    | `~/.config`      | Standard XDG override for the profile location     |

### Runtime notes

- The script uses `exec` so the `llama-server` process replaces it.
  signals (`Ctrl-C`, etc.) go straight to `llama-server` and no wrapper is left behind.
- On every run, one informational line is printed to `stderr` naming the profile
  file that was loaded and where it came from (`command line` / `env var LLAMA_PROFILE`).
- A profile file with no arguments is valid

## Design decisions

- **Why a custom parser instead of `eval`?** Profile files are parsed with a small
  character-by-character tokenizer rather than being fed to the shell, so a
  malicious or malformed profile can never execute code.
- **Why `exec`?** The wrapper's only job is argument assembly; once the command
  line is built, the server becomes the primary process.
- **Why Bash 3.2 compatibility?** So the script works out of the box on stock macOS.
