# awsutils

Two single-file AWS CLI tools: `awscost` reports this month's spend by service,
`awsdashboard` prints a console sign-in URL for a role. No install, no
dependencies to manage — each is one `chmod +x` file.

## What it is

Each tool is a standalone Python script in the [clouddevbox] idiom: on every run
it builds a throwaway venv (`boto3` + `bullet`) under `~/.<toolname>`, re-execs
itself into it, and removes it on exit. Nothing persists between runs, and
nothing is ever installed into your system Python. If `boto3` and `bullet`
happen to be importable already, the bootstrap is skipped entirely
(`python3 -m pip install --user boto3 bullet` makes runs near-instant).

Profile handling is the same in both: a single configured AWS profile is used
as-is, more than one opens a [bullet] picker. The picker needs a tty, so scripts
pass `--profile` explicitly.

`awsdashboard` supersedes [pyawsah](https://github.com/GonzaloAlvarez/pyawsah)
(archived).

## awscost

```sh
awscost                          # current month, this profile, by service
awscost --month 2026-08          # a specific month
awscost --min 0                  # show everything, not just >= $0.10
awscost --profile bedrock        # skip the profile picker
```

```
September 2026 - profile bedrock (account 247412831519), USD
  Amazon Elastic Compute Cloud - Compute       14.90
  EC2 - Other                                   9.31
  Amazon Virtual Private Cloud                  1.56
  Amazon Route 53                               1.00
  --------------------------------------------------
  TOTAL                                        26.76
  (9 service(s) below 0.10 omitted, 0.22 together)
```

1. Reads Cost Explorer: `UnblendedCost`, `MONTHLY` granularity, grouped by
   `SERVICE`, paginated.
2. The default period is the running month, with an exclusive end of *tomorrow*
   so today's partial usage counts. `--month YYYY-MM` reports a past month in
   full.
3. Services below `--min` (default `$0.10`) are hidden, with a one-line note of
   how many and how much. Credits and refunds are negative and always shown,
   whatever `--min` says.
4. `TOTAL` is the sum of every service, including the hidden ones.

Cost Explorer lives at a single global endpoint, so the region is pinned to
`us-east-1` regardless of the profile's own region.

## awsdashboard

```sh
awsdashboard                                   # pick a role -> print the URL
awsdashboard --open                            # ...and open it in a browser
awsdashboard --role AdminConsole               # skip the role picker
awsdashboard --all-roles                       # offer every role in the account
awsdashboard --federation                      # sign in as the profile itself
awsdashboard --duration 43200 --federation     # a 12h console session
```

1. Resolves the profile, then lists every role in the account
   (`iam:ListRoles`, paginated) minus the service-linked ones.
2. Filters that list by each role's trust policy, keeping the ones that name
   you, the account root, or `*`. A caller that is itself an assumed role is
   compared as its underlying role ARN, so role chaining works.
3. Assumes the role you pick and exchanges the temporary credentials for a
   sign-in token at `signin.aws.amazon.com/federation`, then prints the
   `Action=login` URL. The URL lands on your profile region's console home;
   `--destination` overrides it.
4. `--federation` skips roles entirely and calls `sts:GetFederationToken`,
   signing you in as the profile's own identity. This needs long-term IAM user
   credentials, and it is the only path where a session longer than the role's
   `MaxSessionDuration` is possible (up to the console's 12h ceiling).

Every call it makes is free.

## Install

Via [gear] — typing the command installs it on first use:

```sh
awscost
awsdashboard
```

Or directly:

```sh
curl -fsSL https://raw.githubusercontent.com/GonzaloAlvarez/awsutils/main/awscost \
    -o ~/bin/awscost && chmod +x ~/bin/awscost
curl -fsSL https://raw.githubusercontent.com/GonzaloAlvarez/awsutils/main/awsdashboard \
    -o ~/bin/awsdashboard && chmod +x ~/bin/awsdashboard
```

Needs `python3` with `venv` available (`python3-venv` on Debian) and a
configured profile in `~/.aws`. Publishing is pushing to `main`; the version
lives in each script's `VERSION` constant.

## Caveats

- **`awscost` costs money to run**: one `ce:GetCostAndUsage` request per run,
  billed at **$0.01**. Don't loop it or put it in a shell prompt. Cost Explorer
  also has to be enabled once in the billing console by the payer account.
- **The trust-policy filter is advisory.** It does not evaluate `Condition`
  blocks and cannot see your own identity-based permissions, so an MFA-gated
  role may be offered and then fail at `AssumeRole` (with a message saying so).
  When the filter matches nothing, every role is offered instead —
  `--all-roles` forces that.
- **A role under a path, or in another account, needs its full ARN** passed to
  `--role`; a bare name is resolved at this account's root path. Roles chosen
  from the picker always use their real ARN.
- **The printed URL is a live credential.** Anyone holding it is signed in as
  that role until it expires. Don't paste it into chat or redirect it to a file.
- **Session length**: the console caps a federated session at 12h, and the role
  path is further capped by the role's `MaxSessionDuration` (1h by default), so
  `--duration` is clamped with a note.
- GovCloud and China partitions use different sign-in endpoints and are not
  supported.
- `awsdashboard --version` and `--help` deliberately answer before the venv
  bootstrap, so they stay instant.

## License

MIT. See LICENSE.

[clouddevbox]: https://github.com/GonzaloAlvarez/cn-cli-devbox
[bullet]: https://github.com/bchao1/bullet
[gear]: https://github.com/GonzaloAlvarez/gear
