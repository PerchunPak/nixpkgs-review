# nix-review

[![Build Status](https://travis-ci.org/Mic92/nix-review.svg?branch=master)](https://travis-ci.org/Mic92/nix-review)

Review pull-requests on https://github.com/NixOS/nixpkgs. 
nix-review automatically builds packages changed in the pull requests

## Features

- [ofborg](https://github.com/NixOS/ofborg) support: reuses evaluation output of CI to skip local evaluation, but
  also fallbacks if ofborg is not finished
- automatically detects target branch of pull request
- provides a `nix-shell` with all packages, that did not fail to build
- remote builder support
- allows to build a subset of packages (great for mass-rebuilds)
- allow to build nixos tests
- colorful output

## Requirements

`nix-review` depends on python 3.6 or higher and nix 2.0 or higher:

Install with:

```console
$ nix-build
./result/bin/nix-review
```

### Development Environment

For IDEs:

```console
$ nix-build -A env -o .venv
```

or just use:

```console
./bin/nix-review
```

## Usage

Change to your local nixpkgs repository checkout, i.e.:

```console
cd ~/git/nixpkgs
```

Note that your local checkout git will be not affected by `nix-review`, since it 
will use [git-worktree](https://git-scm.com/docs/git-worktree) to perform fast checkouts.

Then run `nix-review` by providing the pull request number

```console
$ nix-review pr 37242
$ git fetch --force https://github.com/NixOS/nixpkgs pull/37242/head:refs/nix-review/0
$ git worktree add /home/joerg/git/nixpkgs/.review/pr-37242 1cb9f643480612696de93fb2f2a2f3340d0e3156
Preparing /home/joerg/git/nixpkgs/.review/pr-37242 (identifier pr-37242)
Checking out files: 100% (14825/14825), done.
HEAD is now at 1cb9f643480 redis: 4.0.7 -> 4.0.8
Building in /tmp/nox-review-4ml2epyy: redis
$ nix-build --no-out-link --keep-going --max-jobs 4 --option build-use-sandbox true <nixpkgs> -A redis
/nix/store/jbp7m1gshmk8an8sb14glwijgw1chvvq-redis-4.0.8
$ nix-shell -p redis
[nix-shell:~/git/nixpkgs]$ /nix/store/jbp7m1gshmk8an8sb14glwijgw1chvvq-redis-4.0.8/bin/redis-cli --version
redis-cli 4.0.8
```

To review a local commit without pull request, use the following command:

```console
$ nix-review rev HEAD
```

Instead of `HEAD` also a commit or branch can be given.

## Remote builder:

Nix-review will pass all arguments given in `--build-arg` to `nix-build`:

```console
$ nix-review pr --build-args="--builders 'ssh://joerg@10.243.29.170'" 37244
```

This allows to parallelize builds across multiple machines.

## Github api token

In case your IP exceeds the rate limit, github will return an 403 error message.
To increase your limit first create a [personal access token](https://help.github.com/articles/creating-a-personal-access-token-for-the-command-line/).
Then use either the `--token` parameter of the `pr` subcommand or
set the `GITHUB_OAUTH_TOKEN` environment variable.

```console
$ nix-review pr --token "5ae04810f1e9f17c3297ee4c9e25f3ac1f437c26" 37244
```

## Checkout strategy (recommend for r-ryantm + cachix)

By default `nix-review pr` will merge the pull request into the pull request's
target branch (most commonly master). However at times mass-rebuilding commits
have been applied in the target branch, but not yet build by hydra. Often those
are not relevant for the current review, but will significantly increase the
local build time. For this case the `--checkout` option can specified to
override the default behavior (`merge`). By setting its value to `commit`,
`nix-review` will checkout the user's pull request branch without merging it:

```console
$ nix-review pr --checkout commit 44534
```

## Only building a subset of packages

To build only certain packages use the `--package` (or `-p`) flag.

```console
$ nix-review pr -p openjpeg -p ImageMagick 49262
```

## Running tests

NixOS tests can be run by using the `--package` feature and our `nixosTests` attribute set:

```console
$ nix-review pr -p nixosTests.ferm 47077
```

Sometimes this requires to rebuild the kernel, when it was changed in master, also that would be not important for most tests.
In those cases it can help to use the pull-request commit as a base
for review by using `--checkout commit`.

```console
$ nix-review pr --checkout commit -p nixosTests.ferm 47077
```

## Ignoring ofborg evaluations

By default, nix-review will use ofborg's evaluation result if available to
figure out what packages need to be rebuild. This can be turned off using
`--eval local`, which is useful if ofborg's evaluation result is outdated. Even
if using `--eval ofborg`, nix-review will fallback to local evaluation if
ofborg's result is not (yet) available.

## Roadmap

- [ ] Build multiple pull requests in parallel and review in serial.
- [ ] trigger ofBorg builds (write @GrahamcOfBorg build foo into pull request discussion)
- [ ] build on multiple platforms
- [ ] test backports
- [ ] show pull request description + diff during review

## Run tests

Just like `nix-review` also the tests are lightning fast:

```console
$ python3 -m unittest discover .
```

We also use python3's type hints. To check them use `mypy`:

```console
$ mypy nix_review
```

## Related projects:

- [nox-review](https://github.com/madjar/nox):
    - works but is slow as a snail: the checkout process of nox-review is slow
      since it requires multiple git fetches. Also it cannot make use of
      ofborg's evaluation
    - it only builds all packages without providing a `nix-shell` for review
- [niff](https://github.com/FRidh/niff):
    - only provides a list of packages that have changed, but does not build packages
    - also needs to evaluate changed attributes locally instead of using ofborg
