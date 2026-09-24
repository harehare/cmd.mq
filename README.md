<h1 align="center">cmd.mq</h1>

Parse command output with [mq](https://github.com/harehare/mq). The module turns text from commands such as `ps`, `df`, and `git log` into structured values.

## Install

Copy `cmd.mq` into your mq module directory, or use `-L` to point mq at this repository:

```sh
cp cmd.mq ~/.local/mq/config/
```

Requires mq v0.5 or later.

## Use

Parse the output of a known command:

```sh
ps aux | mq -L . -I raw -F json 'import "cmd" | cmd::parse_output("ps", .)'
```

Let `parse_auto()` identify output with a distinctive format:

```sh
df -k | mq -L . -I raw -F json 'import "cmd" | cmd::parse_auto(.)'
```

Run and parse a command in one call. `run()` requires mq's `--allow-run` option:

```sh
mq -L . -I null --allow-run=df -F json 'import "cmd" | cmd::run("df", ["-k"])'
```

Extract certificate details from text output:

```sh
openssl x509 -in cert.pem -noout -text | mq -L . -I raw -F json 'import "cmd" | cmd::parse_output("openssl x509 text", .)'
```

## API

- `parse_output(key, output)` parses text using a named command, such as `"git log"` or `"ip addr"`.
- `parse_auto(output)` detects a distinctive format. It raises an error if no parser matches or the result is ambiguous.
- `detect(output)` returns the matching parser keys.
- `run(cmd, args = [])` runs a command and selects its parser from the command and arguments.

For output without a command-specific parser, use `columns_parse`, `table_parse`, `kv_parse`, `kv_blocks_parse`, or `lines_parse`. Generic parsers keep values as strings. Command-specific parsers convert fields that are plainly numeric.

Definitions live in `_registry()` in `cmd.mq`. Many older `*_parse` functions are also available. Parsers without a distinctive signature require an explicit key through `parse_output()` or `run()`.

## Add a parser

Add a definition to `_registry()`. For example:

```mq
{"key": "swapon",
 "sig": "\\ANAME\\s+TYPE\\s+SIZE\\s+USED\\s+PRIO",
 "parse": {"type": "columns"},
 "fields": {"prio": {"type": "int"}}}
```

Use `sig: None` when the output is too generic for automatic detection. For a signed format, add a sample to `_sig_fixtures()` in `cmd_tests.mq` and test the parsed result. The suite checks that each sample matches only its own signature.

## Notes

- Headers are matched as printed. Set `LC_ALL=C` on the command when its output is localized.
- `df` output with spaces in filesystem names may shift columns. `ls -l` parsing expects its default time format.
- Aligned tables containing wide characters may need a command-specific parser.
- Some platform-specific commands have only been tested against sample output.
- `git status --porcelain=v2` reads line-separated output. Its `-z` form needs a different parser.

## License

MIT
