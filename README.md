<h1 align="center">cmd.mq</h1>

Command output parsers implemented as an [mq](https://github.com/harehare/mq) module. Turn the text printed by `ps`, `df`, `ls -l`, `git log` and friends into arrays of dicts, then filter it with mq or render it as JSON or a Markdown table.

## Features

- Parsers for many commands, nearly all of them declared as data (see [Command definitions](#command-definitions)), plus generic parsers for whitespace columns, aligned tables and `key: value` text
- Detects the command from the output text with `parse_auto()`, and refuses when the text is ambiguous
- `run()` executes a command and parses its output in one call
- Generic parsers keep every value as a string. Command-specific parsers convert only fields that are plainly numeric
- Parsers are data: adding a command means adding one dict to the registry (see [Adding a parser](#adding-a-parser))
- Pure mq, no dependencies on other modules

## Installation

Copy `cmd.mq` to your mq module directory, or place it anywhere and reference it with `-L`.

```sh
cp cmd.mq ~/.local/mq/config/
```

### HTTP Import (no local installation needed)

HTTP imports are disabled by default; pass `--allow-http-import` to import directly from GitHub without any local setup:

```sh
mq --allow-http-import -I null --allow-run=ps 'import "github.com/harehare/cmd.mq" | cmd::run("ps", ["aux"])'
```

## Usage

Pipe command output into a parser:

```sh
ps aux | mq -L . -I raw 'import "cmd" | cmd::ps_parse(.)' -F json
```

Or let `run()` execute the command. mq never runs commands through a shell, and needs `--allow-run`:

```sh
mq -L . -I null --allow-run=df -F json 'import "cmd" | cmd::run("df", ["-k"])'
```

## API

### Choosing a parser

| Function | Description |
|---|---|
| `parse_auto(output)` | Detects the command from the text and parses it. Raises an error when no signature matches or when several do, listing the candidates |
| `detect(output)` | Returns the keys whose signature fits the inspected input. Generic formats (`wc`, `du`, `git status`, ...) are never returned |
| `parse_output(key, output)` | Parses `output` with the parser for `key`, e.g. `"ps"`, `"git log"`, `"ip addr"`. Raises an error for an unknown key. Naming a parser is a claim about the text: when the parser has a signature the text must fit it, otherwise an error lists what `detect` finds. Blank output parses to an empty result |
| `run(cmd, args = [])` | Runs the command with `system` and parses its output. The subcommand picks the parser (`git log`, `docker ps`, `ip addr`). Requires `--allow-run` |

### Generic parsers

| Function | Input | Output |
|---|---|---|
| `columns_parse(input, headers = None)` | Whitespace-separated columns with a header line. The last column keeps the rest of the line | Array of dicts |
| `table_parse(input)` | Columns aligned under the header (two or more spaces between names), e.g. `docker ps`, `kubectl get` | Array of dicts |
| `kv_parse(input, sep = ":")` | `key: value` lines, split on the first separator | Dict |
| `kv_blocks_parse(input, sep = ":")` | Blank-line separated `key: value` blocks | Array of dicts |
| `lines_parse(input)` | Any text | Array of trimmed non-blank lines |

`columns_parse` always treats the first line as the header. Pass `headers` to replace the names.

## Command definitions

Command-specific definitions are in `_registry()` in `cmd.mq`; aliases are in `_name_aliases()` and `_key_aliases()`. Pass a key such as `"git log"` to `parse_output`, or let `run()` select it from the command and arguments. `parse_auto()` works for definitions with a distinctive signature. Parsers without a signature must be named explicitly.

The older `*_parse` functions remain available for commands that have them. Newer definitions can be reached through `parse_output` or `run` without a dedicated wrapper.

## Auto-detection

`parse_auto` recognizes the output by its text alone. Parsers eligible for auto-detection have a signature: a multi-line regular expression matched against a bounded input prefix. Nothing is guessed:

- If exactly one signature fits, that parser runs.
- If none fits, or several do, it raises an error. The message lists the candidates, and you name the parser with `parse_output`.
- Generic formats (`wc`, `du`, `git status`, `git branch`, `git log --oneline`, `git diff --numstat`, `pgrep`, `grep`, ...) have no signature and are never claimed. Name them with `parse_output` or `run`.
- Naming a parser does not skip the check. `parse_output("id", text)` raises an error when the text does not look like `id` output. Call the parser function itself (`id_parse(text)`) to bypass it.

Signatures describe the whole shape of the text, not one line that happens to appear. A single-line format is anchored at both ends (`\A` and `\z`), so `uptime` does not claim the first line of `w`. A line-oriented format such as `ls -l`, `lsusb` or `/etc/passwd` must consist only of lines of that shape, so a text that mixes two formats is refused instead of being read as one of them. Formats with a header line are anchored on it (`\AFilesystem\s+`).

The test suite holds a sample for every signature and checks that each sample is detected as exactly its own key, so a signature that overlaps another one fails the tests.

```sh
echo 'uid=501(alice) gid=20(staff) groups=20(staff)' | mq -L . -I raw -F json 'import "cmd" | cmd::parse_auto(.)'
```

```json
{
  "uid": 501,
  "user": "alice",
  "gid": 20,
  "group": "staff",
  "groups": [
    {
      "id": 20,
      "name": "staff"
    }
  ]
}
```

Headers are matched as printed, so translated output (for example a non-English `df` header) is refused. Run the command with `LC_ALL=C` when piping, e.g. `LC_ALL=C df -k | mq -I raw '...'`. `run()` cannot set the environment of the command.

## Adding a parser

Most parsers are declared as data. `_registry()` in `cmd.mq` is a list of dicts, and a small engine reads them. To support a new command, add one entry:

```mq
{"key": "swapon",
 "sig": "\\ANAME\\s+TYPE\\s+SIZE\\s+USED\\s+PRIO",
 "parse": {"type": "columns"},
 "fields": {"prio": {"type": "int"}}}
```

| Key | Meaning |
|---|---|
| `key` | The command, or the command and its subcommand (`git log`, `docker ps`). `run()` builds it from the command line, and `parse_output(key, text)` takes it |
| `sig` | A multi-line regex that says what the text must look like. Use `None` for a format that is too generic to claim, so that it is only used when named |
| `parse` | A spec dict (below), or a function for a format the engine cannot express |
| `fields` | Optional rules for the fields the spec produces |

Spec types:

| `type` | Reads | Options |
|---|---|---|
| `regex` | One dict per line from the named groups of `patterns` (the first pattern that matches wins) | `pattern`, `patterns` (each a string or `{"re": ..., "set": {constants}}`), `trim`, `single` (return one dict instead of an array), `as: "map"` (rows with `key` and `value` groups become one dict), `normalize_keys`, `keep_empty` |
| `columns` | Whitespace columns under a header line. The last column keeps the rest of the line | `headers` (replace the names), `prefix_headers` (name columns the header leaves out), `header: false` |
| `aligned` | Columns cut at the header's offsets, for values with spaces | `offsets: "dashes"` (use a `---- ----` line), `keys: "raw"` (keep the header names as printed) |
| `delimited` | Rows split on `separator` (default tab) under a header line | `separator` |
| `kv` | One dict from `key<separator>value` lines | `separator`, `blocks` (an array from blank-line separated blocks), `unquote` |
| `record` | One dict from the first match of every pattern over the whole text, or from `captures` | `patterns`, `captures`, `require` (keys that must be captured, otherwise the result is `{}`) |
| `blocks` | One dict per block, where a line matching `start` begins a block. Every block is read by `captures` | `start`, `captures`, `require` (blocks without these keys are dropped) |
| `choice` | The first of `options` whose `when` regex matches the text, each with its own `parse` and `fields`. Use it when one command prints two layouts | `options` |
| `indent` | Nested dicts from an indented listing: a line with a deeper line after it is a section, the others are `key<separator>value` | `separator` (default `:`), `normalize_keys` |
| `words` | One dict per line of words: the first word is `first`, a word in `keys` takes the next word as its value, and the other words go to a `flags` array | `first`, `keys`, `prefix_types` (words that may come before the first word and become `type`) |
| `lines` | The trimmed non-blank lines | |
| `tree` | One dict per line of a glyph tree | `pattern` (with an `indent` group), `unit` (indent width per level), `base`, `path` |

Input options that every type accepts: `replace` (`[[regex, text], ...]` on the whole text), `start_at` (drop the lines before the first match), `until_blank` (keep the first block only), `strip` (a regex removed from each line) and `skip` (regexes of lines to ignore).

`captures` is a list of `{"re": ...}` entries. A plain entry adds the groups of its first match to the dict. An entry with `"all": true, "into": "key"` collects every match into an array. Add `"values": true` to keep only the `value` group of each match, `"join": "\n"` (with `"trim": true`) to join the values into one string, and `"fields": {...}` to type the rows. This reads `ifconfig` (several `inet` lines per interface), `git log` (a message of many lines) and `diskutil list` (a table under each disk). More capture options: `"merge": true` builds one dict from the `key` and `value` groups of every match (placed under `into` when given, otherwise merged into the block), `"within": regex` limits an entry to the `value` group of another regex (and skips it when that regex does not match), `"type": "float"` turns `values` into numbers, and an entry with `"blocks": {...}` and `"into"` reads the text as nested blocks. On a `blocks` spec, `"last": true` returns only the last block as one dict, which is how the last response of `curl -i` is read.

Field rules, keyed by the field name (`"*"` matches any other field): `type` (`int` and `float` turn plain numbers into numbers and leave other text alone, `bool`, `array`, `string`), `null_if`, `trim_suffix`, `remove`, `true_values`, `false_values`, `split`, `rename`, `trim`, `squeeze` (collapse whitespace), `default` (used when the field is missing), `derive` and `keep_empty`. `derive` fills a missing field from another field: `{"op": "prefix", "from": "permissions", "map": {"d": "directory"}, "default": "other"}` maps the first character, and `{"op": "basename", "from": "path"}` takes the last path component. Empty strings are dropped from `regex` results and kept in the other types unless `keep_empty` says otherwise.

A definition is checked by the test suite. For a definition with a signature, add a sample of its output to `_sig_fixtures()` in `cmd_tests.mq`. The suite then verifies that the sample is detected as exactly this key, and that no other signature claims it. Add a test with the expected result.

Commands that are declared only as data have no `*_parse` function. Reach them with `parse_output("nstat", text)` or `run("nstat")`; `parse_auto(text)` works when the definition has a signature.

## Example

```sh
df -k | mq -L . -I raw 'import "cmd" | cmd::df_parse(.) | filter(fn(r): r["mounted_on"] == "/";)' -F json
```

Render a Markdown table:

```sh
ps aux | mq -L . -I raw 'import "cmd" | import "csv" | cmd::ps_parse(.) | csv::csv_to_markdown_table()'
```

## Limitations

- Checked against real output on macOS: `ps`, `top`, `pgrep`, `lsof`, `uptime`, `who`, `w`, `id`, `vm_stat`, `uname`, `sysctl`, `df`, `mount`, `ls`, `tar`, `unzip`, `zipinfo`, `gzip`, `file`, `checksum`, `cksum`, `grep`, `mdls`, `otool`, `hexdump`, `ifconfig`, `launchctl`, `ping`, `traceroute`, `date`, `stat`, `last`, `getconf`, `diskutil`, `pmset`, `csrutil`, `spctl`, `networksetup`, `memory_pressure`, `system_profiler`, `route get`, `netstat -rn`, `networksetup`, `xxd`, `lipo`, `pmset`, `nvram`, `dscacheutil`, `plutil`, `passwd`, `group`, `hosts`, `ssh-keygen`, `pip`, `gem`, `cargo`, `git`. The Linux-only and tool-specific parsers (`free`, `vmstat`, `ip`, `ss`, `netstat`, `route`, `lsblk`, `blkid`, `lsmod`, `lspci`, `lsusb`, `dmesg`, `syslog`, `systemctl`, `loginctl`, `findmnt`, `swapon`, `lsattr`, `getfacl`, `getcap`, `host`, `nslookup`, `nmcli`, `ethtool`, `lscpu -e`, `networkctl`, `busctl`, `systemctl show`, `loginctl show-*`, `resolvectl`, `systemd-analyze`, `fdisk`, `ldd`, `dig`, `curl`, `tree`, `dpkg`, `apt`, `apt-cache`, `dnf`, `rpm`, `pacman`, `conda`, `snap`, `helm`, `brew`, `npm`, `docker`, `kubectl`) are tested against sample output only. The GNU form of `stat` is also sample-only
- `df_parse` splits on whitespace, so a filesystem name containing spaces (e.g. `map auto_home` on macOS) shifts the columns
- `ls_parse` expects the default `ls -l` time format and skips lines it cannot parse
- `git_branch_parse` puts tracking info such as `[ahead 1]` at the start of `message` with `-v`
- Column widths in `table_parse` are counted in characters, so wide (CJK) characters misalign
- `traceroute_parse` reads only the first responder on each hop line
- Named parsers refuse text that does not fit their signature, so a variant the signature does not describe (for example `lsblk` without a `NAME` header) has to go through the parser function directly

## Compatibility

Requires [mq](https://github.com/harehare/mq) v0.5 or later.

## License

MIT
