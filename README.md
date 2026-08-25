# nft-blacklist

A Bash script that builds an [nftables](https://nftables.org/) ruleset from
public IP blacklists and drops traffic from every listed address.

Fork of [leshniak/nft-blacklist](https://github.com/leshniak/nft-blacklist),
which in turn derives from `trick77/ipset-blacklist`.

## How It Works

One run of the script does the following:

1. Download every URL in `BLACKLISTS`. Sources that answer `503` are reported and
   skipped; other failures produce a warning. A failing source never aborts the run.
2. Extract IPv4 and IPv6 addresses and CIDR ranges by regex, so list formats with
   comments or extra columns work without per-source parsing.
3. Drop entries that must never be blocked: `0.0.0.0`, loopback, RFC 1918 ranges,
   multicast, the IPv6 default route and link-local addresses.
4. Sort, deduplicate, and — if `cidr-merger` is on `PATH` — merge adjacent ranges.
   This matters: without merging, overlapping ranges from different sources make
   nftables reject the set.
5. Write a complete nftables ruleset to `RULESET_FILE`.
6. Apply it with `nft -f`, unless `APPLY_RULESET` says otherwise.

The generated ruleset creates its own `inet` table, so it does not interfere with
an existing firewall. The chain hooks in at `filter - 1`, one step ahead of the
normal filter priority, which is what keeps the blacklist in front of rules added
later by tools such as fail2ban.

Traffic on `lo`, broadcast and multicast packets, and the whitelisted source
ranges are accepted before the blacklist sets are consulted.

## Requirements

- `nftables`
- `curl`, `grep` (with `-P`), `sed`, `sort`, `wc`, `date`
- [`cidr-merger`](https://github.com/zhanhb/cidr-merger/releases) — strongly
  recommended. Without it the script warns and continues, but large lists are
  likely to be rejected by `nft`.

## Installation

```bash
install -m 0755 nft-blacklist.sh /usr/local/sbin/nft-blacklist.sh
install -d /etc/nft-blacklist /var/cache/nft-blacklist
install -m 0644 nft-blacklist.conf /etc/nft-blacklist/nft-blacklist.conf
```

The script does **not** create the directories for its output files. If they are
missing it exits with an error.

Adjust `/etc/nft-blacklist/nft-blacklist.conf`, then run it:

```bash
/usr/local/sbin/nft-blacklist.sh /etc/nft-blacklist/nft-blacklist.conf
```

## Configuration

The configuration file is sourced as shell code, so it can contain any Bash.

| Variable | Default in the shipped config | Purpose |
| --- | --- | --- |
| `BLACKLISTS` | Array of public sources | URLs to download. `file:///` paths work for local lists. |
| `RULESET_FILE` | `/var/cache/nft-blacklist/blacklist.nft` | Generated nftables ruleset. |
| `IP_BLACKLIST_FILE` | `/var/cache/nft-blacklist/ip-blacklist.list` | Intermediate IPv4 address list. |
| `IP6_BLACKLIST_FILE` | `/var/cache/nft-blacklist/ip6-blacklist.list` | Intermediate IPv6 address list. |
| `IP_WHITELIST` | RFC 1918 ranges | Comma-separated IPv4 sources always accepted. |
| `IP6_WHITELIST` | ULA and link-local | Comma-separated IPv6 sources always accepted. |
| `TABLE` | `blackhole` | Name of the dedicated nftables table. |
| `APPLY_RULESET` | `yes` | Apply the ruleset immediately after generating it. |
| `VERBOSE` | `yes` | Progress output. Set to `no` for cron jobs. |

The whitelists are inserted verbatim into an nftables set, so the comma-separated
syntax has to be kept exactly.

Sources are commented in the shipped config; uncomment or add entries to suit.
A comment after the URL documents what a list is, and commenting a line out is
the way to retire a source without losing the note:

```sh
BLACKLISTS=(
"http://www.example.com/files/mycustomblacklist.txt" # Your personal blacklist
"http://www.projecthoneypot.org/list_of_ips.php?t=d&rss=1" # Dictionary attackers
# I don't want this: "http://www.openbl.org/lists/base.txt"  # OpenBL 30 day list
)
```

Country-wide and ASN-wide blocklists can be pulled from
[ipverse](https://github.com/ipverse/rir-ip), and a large collection of
categorised lists lives in
[firehol/blocklist-ipsets](https://github.com/firehol/blocklist-ipsets).

## Persistence

The ruleset lives in kernel memory and is lost on reboot. Re-apply it at boot,
either by running the script from a systemd unit or by loading the generated file:

```bash
nft -f /var/cache/nft-blacklist/blacklist.nft
```

## Scheduled Updates

Refresh once a day. Updating more often gets your address throttled or banned by
some list providers.

Put the following into `/etc/cron.d/nft-blacklist-update`:

```text
PATH=/usr/local/sbin:/usr/local/bin:/sbin:/bin:/usr/sbin:/usr/bin
MAILTO=root
33 23 * * *      root /usr/local/sbin/nft-blacklist.sh /etc/nft-blacklist/nft-blacklist.conf
```

Set `VERBOSE=no` in the configuration so cron does not mail a progress report on
every successful run.

## Checking Effectiveness

Each address family gets its own counter:

```console
$ sudo nft list counter inet blackhole blacklist_v4
table inet blackhole {
        counter blacklist_v4 {
                packets 52 bytes 2303
        }
}

$ sudo nft list counter inet blackhole blacklist_v6
table inet blackhole {
        counter blacklist_v6 {
                packets 0 bytes 0
        }
}
```

The generated `.nft` file starts with a header recording when it was built, how
many entries it holds, and which sources produced it.

## History

Carried over from the upstream projects, with credit to the contributors:

- 12/31/2023: Add more customization options using shell variables
  ({[@henrythasler](https://github.com/henrythasler)})
- 08/26/2022: Added experimental IPv6 support and whitelists
  ([@leshniak](https://github.com/leshniak))
- 08/24/2022: Created this fork and nftables-based version
  ([@leshniak](https://github.com/leshniak))
- 10/17/2018: Added support for CIDR aggregation if `iprange` is available
- 10/17/2018: Merged Shellcheck PR from
  [@extremeshok](https://github.com/extremeshok)
- 05/10/2018: Added regex filter improvements from
  [@sbujam](https://github.com/sbujam)
- 08/15/2017: Filtering default gateway and multicast ranges
- 01/20/2017: Ignoring "Service unavailable" HTTP status code, removed
  `IGNORE_CURL_ERRORS`
- 11/04/2016: Documented how to keep fail2ban from inserting its rules above
  the blacklist when the service restarts
- 11/11/2015: Merged all suggestions from
  [@drzraf](https://github.com/drzraf)
- 10/24/2015: Moved the entire configuration into its own file
- 10/22/2015: The script belongs in `/usr/local/sbin`, not `/usr/local/bin`
