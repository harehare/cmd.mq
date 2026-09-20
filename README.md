<h1 align="center">cmd.mq</h1>

Command output parsers implemented as an [mq](https://github.com/harehare/mq) module. Turn the text printed by `ps`, `df`, `ls -l`, `git log` and friends into arrays of dicts, then filter it with mq or render it as JSON or a Markdown table.

## Features

- Parsers for 150 commands, nearly all of them declared as data (see [Supported commands](#supported-commands)), plus generic parsers for whitespace columns, aligned tables and `key: value` text
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

## Supported commands

Every function takes the command output as a string. Keys are lowercased header names, and only plainly numeric fields become numbers.

### System and processes

| Function | Command | Output |
|---|---|---|
| `ps_parse` | `ps aux`, `ps -ef` | One dict per process. `pid`, `ppid`, `cpu_percent`, `mem_percent`, `vsz`, `rss` are numbers |
| `top_parse` | `top -l 1` (macOS), `top -bn1` | The process table only |
| `pgrep_parse` | `pgrep`, `pgrep -l` | `pid`, `name` |
| `lsof_parse` | `lsof` | `SIZE/OFF` becomes `size_off`. `pid` is a number |
| `uptime_parse` | `uptime` | `time`, `uptime`, `users`, `load_1`, `load_5`, `load_15` |
| `who_parse` | `who` | `user`, `tty`, `login`, `host` |
| `w_parse` | `w` | The user table, without the uptime line |
| `id_parse` | `id` | `uid`, `user`, `gid`, `group`, `groups` (array of `{id, name}`) |
| `free_parse` | Linux `free` | One dict per row (`Mem`, `Swap`) with numeric columns |
| `vmstat_parse` | Linux `vmstat` | One dict per sample, all numbers |
| `vm_stat_parse` | macOS `vm_stat` | One dict of page counts and `page_size` |
| `uname_parse` | `uname -a` | `kernel_name`, `node_name`, `kernel_release`, `kernel_version`, `machine` |
| `sysctl_parse` | `sysctl` | One dict, values as strings |
| `time_parse` | shell `time` | `real`, `user`, `sys` in seconds |
| `launchctl_list_parse` | `launchctl list` | `pid` (None when not running), `status`, `label` |
| `systemctl_parse` | `systemctl list-units`, `list-unit-files` | The table, without the footer |
| `dmesg_parse` | `dmesg`, `dmesg -T` | `time`, `message` |
| `syslog_parse` | `journalctl`, `/var/log/syslog` | `timestamp`, `host`, `process`, `pid`, `message` |
| `date_parse` | `date` | `weekday`, `month`, `day`, `time`, `timezone`, `year` |
| `hostname_parse`, `whoami_parse` | `hostname`, `whoami` | `{hostname}`, `{user}` |
| `last_parse` | `last` | `user`, `tty`, `host`, `login`, `session` |
| `ulimit_parse` | bash `ulimit -a` | `name`, `flag`, `value` (number or `unlimited`), `unit` |
| `memory_pressure_parse` | macOS `memory_pressure` | One dict of numbers (`memory_bytes`, `pages`, `page_size`, `pages_free`, ...) |
| `pmset_batt_parse` | macOS `pmset -g batt` | `source`, `batteries` (`name`, `id`, `percent`, `state`, `remaining`, `present`) |
| declared as data only | macOS `system_profiler SPHardwareDataType` (and the other data types) | Nested dicts that follow the section headers, e.g. `d["Hardware"]["Hardware Overview"]["Chip"]`. Values stay strings |
| `csrutil_parse`, `spctl_parse` | macOS `csrutil status`, `spctl --status` | `{status}`, `{assessments}` |
| `lsns_parse`, `lslocks_parse`, `lsmem_parse`, `lslogins_parse` | `lsns`, `lslocks`, `lsmem`, `lslogins` | Lowercased columns with numeric ids |
| `loginctl_parse` | `loginctl list-sessions`, `list-users` | The table, without the footer. `uid` is a number |

### Disks and hardware

| Function | Command | Output |
|---|---|---|
| `df_parse` | `df`, `df -k`, `df -P` | `Mounted on` becomes `mounted_on`. Plain integers are numbers, `460Gi` stays a string |
| `du_parse` | `du`, `du -k` | `size`, `path` |
| `mount_parse` | `mount` | `device`, `mount_point`, `type`, `options` (array) |
| `lsblk_parse` | `lsblk` | Tree glyphs removed from `name`. `rm` and `ro` are numbers |
| `blkid_parse` | `blkid` | `device` plus lowercased attributes (`uuid`, `type`, ...) |
| `lsmod_parse` | `lsmod` | `module`, `size`, `used`, `used_by` |
| `lspci_parse` | `lspci` | `slot`, `class`, `description` |
| `lsusb_parse` | `lsusb` | `bus`, `device`, `id`, `description` |
| `diskutil_list_parse` | macOS `diskutil list` | One dict per disk with `device`, `description`, `partitions` |
| `findmnt_parse` | `findmnt` | Tree glyphs removed from `target`. Lowercased keys |
| `swapon_parse` | `swapon --show` | Lowercased keys. `prio` is a number |

### Files and text

| Function | Command | Output |
|---|---|---|
| `ls_parse` | `ls -l`, `ls -la` | `type`, `permissions`, `links`, `owner`, `group`, `size`, `modified`, `name`, `link_target` |
| `tar_parse` | `tar -tvf` (GNU and bsdtar) | `permissions`, `owner`, `size`, `modified`, `name` |
| `unzip_list_parse` | `unzip -l` | `size`, `date`, `time`, `name` |
| `file_parse` | `file` | `path`, `description` |
| `checksum_parse` | `md5`, `md5sum`, `shasum`, `sha256sum` | `hash`, `file`, and `algorithm` for the BSD format |
| `wc_parse` | `wc` | `lines`, `words`, `bytes`, `file` |
| `grep_parse` | `grep -n`, `grep -rn`, `rg` | `file`, `line`, `text` |
| `uniq_c_parse` | `uniq -c` | `count`, `value` |
| `mdls_parse` | macOS `mdls` | One dict. `(null)` becomes None |
| `otool_l_parse` | macOS `otool -L` | `path`, `compatibility_version`, `current_version` |
| `ldd_parse` | `ldd` | `name`, `path` (None when not found), `address` |
| `stat_parse` | `stat` (BSD/macOS and GNU) | One dict per file. BSD: `path`, `size`, `mode`, `owner`, `mtime`, ... GNU: `file`, `size`, `permissions`, `user`, `access`, `modify`, ... |
| `zipinfo_parse` | `zipinfo` | `permissions`, `version`, `os`, `size`, `flags`, `method`, `date`, `time`, `name` |
| `gzip_list_parse` | `gzip -l` | `compressed`, `uncompressed`, `ratio`, `uncompressed_name` |
| `hexdump_parse` | `hexdump -C` | `offset`, `bytes`, `ascii` |
| `cksum_parse` | `cksum`, `sum` | `checksum`, `size`, `file` |
| `tree_parse` | `tree` | `name`, `depth`, `path` |
| `getconf_parse` | `getconf -a` | One dict, plain numbers as numbers |
| `lsattr_parse` | `lsattr` | `attributes`, `path` |
| `getfacl_parse` | `getfacl` | `file`, `owner`, `group`, `entries` |
| `getcap_parse` | `getcap` | `path`, `capabilities` |

### Network

| Function | Command | Output |
|---|---|---|
| `ping_parse` | `ping` (macOS and Linux) | `transmitted`, `received`, `loss_percent`, `rtt_min`/`rtt_avg`/`rtt_max`/`rtt_stddev` |
| `traceroute_parse` | `traceroute` | `hop`, `host`, `ip`, `rtts` (array) |
| `ifconfig_parse` | `ifconfig` | `name`, `flags`, `mtu`, `ether`, `status`, `ipv4`, `ipv6` |
| `ip_addr_parse` | `ip addr`, `ip -br addr` | Full: `index`, `name`, `flags`, `mtu`, `state`, `mac`, `ipv4`, `ipv6`. Brief: `name`, `state`, `addresses` |
| `ip_route_parse` | `ip route` | `destination` plus `via`, `dev`, `proto`, `scope`, `src`, `metric`, `flags` |
| `ss_parse` | `ss -tuln` | `netid`, `state`, `recv_q`, `send_q`, `local_address`, `peer_address`, `process` |
| `netstat_parse` | `netstat -an`, `netstat -tulpn` | `proto`, `recv_q`, `send_q`, `local_address`, `foreign_address`, `state`, `program` |
| `arp_parse` | `arp -a` | `host`, `ip`, `mac`, `interface` |
| `dig_parse` | `dig` | `name`, `ttl`, `class`, `type`, `data` |
| `curl_headers_parse` | `curl -I`, `curl -i` | `http_version`, `status`, `reason`, `headers` (lowercased names) |
| `ssh_keygen_parse` | `ssh-keygen -l` | `bits`, `fingerprint`, `comment`, `type` |
| `route_parse` | Linux `route -n` | Lowercased columns. `metric`, `ref`, `use` are numbers |
| `host_parse` | `host` | `name`, `type` (A, AAAA, CNAME, MX, TXT, NS, PTR), `value`, `priority` |
| `nslookup_parse` | `nslookup` | `server`, `server_address`, `answers` (`name`, `addresses`) |

### Users and configuration

| Function | Command | Output |
|---|---|---|
| `passwd_parse` | `/etc/passwd`, `getent passwd` | `user`, `password`, `uid`, `gid`, `comment`, `home`, `shell` |
| `group_parse` | `/etc/group`, `getent group` | `name`, `password`, `gid`, `members` (array) |
| `hosts_parse` | `/etc/hosts` | `ip`, `hostname`, `aliases` |
| `os_release_parse` | `/etc/os-release`, `locale` | One dict, quotes removed |
| `dpkg_status_parse` | `dpkg -s`, `dpkg-query -s` | One dict per package, field names as printed |

`env`, `printenv`, `git config -l` (`KEY=value`) and `sw_vers`, `lscpu`, `hostnamectl`, `timedatectl`, `localectl`, `lsb_release`, `route -n get`, `diskutil info`, `pdfinfo`, `exiftool` (`key: value`) go through `kv_parse`. `networksetup -listallhardwareports` and `nmcli` (`connection`, `device`) go through `kv_blocks_parse` and `table_parse`.

### Git

| Function | Command | Output |
|---|---|---|
| `git_log_parse` | `git log` | `commit`, `author`, `date`, `message`, `merge`, `refs` |
| `git_log_oneline_parse` | `git log --oneline` | `commit`, `message`, `refs` |
| `git_status_parse` | `git status --porcelain` | `index`, `worktree`, `path`, `orig_path` |
| `git_branch_parse` | `git branch`, `git branch -v` | `name`, `current`, and `commit`, `message` with `-v` |
| `git_diff_numstat_parse` | `git diff --numstat` | `added`, `deleted` (None for binary), `path` |
| `git_diff_stat_parse` | `git diff --stat` | `files`, `files_changed`, `insertions`, `deletions` |
| `git_shortlog_parse` | `git shortlog -sn` | `commits`, `author`, `email` |
| `git_remote_parse` | `git remote -v` | `name`, `url`, `direction` |
| `git_stash_list_parse` | `git stash list` | `index`, `ref`, `branch`, `message` |
| `git_reflog_parse` | `git reflog` | `commit`, `ref`, `action`, `message` |
| `git_worktree_parse` | `git worktree list` | `path`, `commit`, `branch` or `state` |

### Packages

| Function | Command | Output |
|---|---|---|
| `pip_list_parse` | `pip list` | `package`, `version`, `editable_project_location` |
| `pip_freeze_parse` | `pip freeze` | `name`, `version` or `url` |
| `pip_show_parse` | `pip show` | One dict per package, field names as printed |
| `brew_list_parse` | `brew list --versions` | `name`, `versions` (array) |
| `brew_outdated_parse` | `brew outdated --verbose` | `name`, `installed_versions`, `current_version` |
| `gem_list_parse` | `gem list` | `name`, `versions` (array) |
| `cargo_install_list_parse` | `cargo install --list` | `name`, `version`, `binaries`, `source` |
| `npm_ls_parse` | `npm ls` | `name`, `version`, `depth` |
| `dpkg_list_parse` | `dpkg -l` | `status`, `name`, `version`, `architecture`, `description` |
| `apt_list_parse` | `apt list` | `name`, `suite`, `version`, `architecture`, `status` |
| `dnf_list_parse` | `dnf list installed`, `yum list installed` | `name`, `architecture`, `version`, `repo` |
| `pacman_q_parse` | `pacman -Q` | `name`, `version` |
| `conda_list_parse` | `conda list` | `name`, `version`, `build`, `channel` |
| `snap_list_parse` | `snap list`, `brew services list` | Lowercased columns. `rev` is a number |
| `helm_list_parse` | `helm list` | Tab-separated table. `revision` is a number |
| `rpm_qa_parse` | `rpm -qa` | `name`, `version`, `release`, `arch` |
| `apt_cache_madison_parse` | `apt-cache madison` | `package`, `version`, `source` |
| `apt_cache_search_parse` | `apt-cache search` | `package`, `description` |

### Declared as data only

These commands have no `*_parse` function. Reach them with `parse_output`, `run` or `parse_auto`.

| Command | Output |
|---|---|
| `system_profiler` | Nested dicts that follow the section headers |
| `plutil -p` | Nested dicts and arrays (array items are keyed `"0"`, `"1"`, ...). Values stay strings |
| `netstat -rn` | macOS: one dict per family (`Internet`, `Internet6`) with `routes`. Linux: the routing table |
| `networksetup -listnetworkserviceorder` | `order`, `service`, `hardware_port`, `device` |
| `networksetup -getinfo` | One dict of the printed fields |
| `xxd` | `offset`, `bytes`, `ascii` |
| `lipo -info` | `file`, `architectures` (array) |
| `pmset -g` | One dict of the power settings, keys normalized |
| `nvram -p` | One dict of the variables |
| `dscacheutil -q user` | One dict per record. `uid` and `gid` are numbers |
| `ip neighbour` | `address`, `dev`, `lladdr`, `flags` |
| `ethtool -i` | One dict of the driver info |
| `lscpu -e` | One dict per CPU |
| `fdisk -l` | One dict per disk with `device`, `bytes`, `model`, `label_type`, `identifier` and `partitions` (GPT layout, an MBR `Id` stays in `type`) |
| `networkctl list`, `busctl list` | Lowercased columns |
| `systemctl list-sockets`, `systemctl list-jobs` | The table, without the footer |
| `systemctl show`, `loginctl show-session`, `loginctl show-user` | One dict from the `Key=value` lines |
| `resolvectl status` | One dict per `Global` or `Link` section |
| `systemd-analyze blame` | `time`, `unit` |
| `chage -l`, `systemd-path`, `bluetoothctl show` | One dict of the printed fields |
| `apt-config dump`, `debconf-show`, `getent hosts`, `uuidparse`, `nstat`, `ionice`, `pmap` | See the definitions in `_registry()` |

### Containers

`docker ps`, `docker images`, `docker stats`, `docker volume ls`, `docker network ls` (and `podman`), `kubectl get` and `kubectl top` are aligned tables. Parse them with `table_parse`, or let `run()` and `parse_auto()` pick it.

### Keys

`parse_output` and `run` use these keys. Detection by `parse_auto` is available where the text has a distinctive header or shape.

| Kind | Keys |
|---|---|
| Single commands | `ps`, `top`, `pgrep`, `lsof`, `uptime`, `who`, `w`, `id`, `free`, `vmstat`, `vm_stat`, `uname`, `sysctl`, `time`, `launchctl`, `systemctl`, `dmesg`, `syslog`, `df`, `du`, `mount`, `lsblk`, `blkid`, `lsmod`, `lspci`, `lsusb`, `ls`, `tar`, `unzip`, `file`, `checksum`, `wc`, `grep`, `uniq`, `mdls`, `otool`, `ldd`, `ping`, `traceroute`, `ifconfig`, `ss`, `netstat`, `arp`, `dig`, `curl`, `ssh-keygen`, `passwd`, `group`, `hosts`, `os-release`, `locale`, `env`, `sw_vers`, `lscpu`, `hostnamectl`, `timedatectl`, `dpkg`, `pacman`, `lines`, `date`, `hostname`, `whoami`, `stat`, `last`, `route`, `zipinfo`, `gzip`, `hexdump`, `cksum`, `tree`, `getconf`, `csrutil`, `spctl`, `memory_pressure`, `xxd`, `lipo`, `pmset`, `nvram`, `dscacheutil`, `plutil`, `ethtool`, `fdisk`, `lsattr`, `findmnt`, `swapon`, `getfacl`, `getcap`, `host`, `nslookup`, `localectl`, `lsb_release`, `lsns`, `lslocks`, `lsmem`, `lslogins`, `rpm`, `pdfinfo`, `exiftool` |
| With a subcommand | `ip addr`, `ip route`, `git log`, `git log --oneline`, `git status`, `git branch`, `git diff --numstat`, `git diff --stat`, `git shortlog`, `git remote`, `git stash`, `git reflog`, `git worktree`, `git config`, `pip list`, `pip freeze`, `pip show`, `brew list`, `brew outdated`, `brew services`, `gem list`, `cargo install`, `npm ls`, `apt list`, `dnf list`, `conda list`, `snap list`, `helm list`, `docker ps`, `docker images`, `docker stats`, `docker volume`, `docker network`, `kubectl get`, `kubectl top`, `route get`, `diskutil list`, `diskutil info`, `pmset batt`, `netstat routes`, `networksetup -listallhardwareports`, `networksetup -listnetworkserviceorder`, `networksetup -getinfo`, `lscpu extended`, `networkctl list`, `busctl list`, `systemctl show`, `systemctl list-sockets`, `systemctl list-jobs`, `loginctl show-session`, `loginctl show-user`, `resolvectl status`, `systemd-analyze blame`, `nmcli connection`, `nmcli device`, `loginctl list-sessions`, `loginctl list-users`, `apt-cache madison`, `apt-cache search`, `dpkg status` |

`chage -l`, `systemd-path`, `bluetoothctl show` (`key: value`), `apt-config dump`, `debconf-show`, `getent hosts`, `uuidparse`, `nstat`, `ionice`, `pmap`, `system_profiler` and `ip neighbour` are declared only as definitions, so they have no `*_parse` function. Use `parse_output`, `run` or `parse_auto`.

Aliases such as `md5sum`, `shasum`, `rg`, `journalctl`, `printenv`, `pip3`, `yum`, `podman`, `ip a` and `docker container` resolve to the keys above.

## Auto-detection

`parse_auto` recognizes the output by its text alone. Each parser has a signature: a multi-line regular expression matched against a bounded input prefix. Nothing is guessed:

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

A definition is checked by the test suite. Add a sample of its output to `_sig_fixtures()` in `cmd_tests.mq`. The suite then verifies that the sample is detected as exactly this key, and that no other signature claims it. Add a test with the expected result, and a row to the tables above.

Commands that are declared only as data have no `*_parse` function. Reach them with `parse_output("nstat", text)`, `run("nstat")` or `parse_auto(text)`. The older `*_parse` functions are thin wrappers that call the same definitions.

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
