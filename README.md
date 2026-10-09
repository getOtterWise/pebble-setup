# pebble-setup

GitHub Action that installs [pebble](https://github.com/getOtterWise/pebble)
on the job's PHP: the native coverage tracer extension, the `pebble` reporter
binary, and the PHPUnit glue. Works with setup-php (use `coverage: none`) on
GitHub-hosted and self-hosted Linux and macOS runners.

This repository is a thin installer. The code lives in the private
`getOtterWise/pebble` repository. Its release workflow copies the action here
and publishes the prebuilt assets as releases of this repository, so no token
is needed. Do not edit the files here.

## Usage

```yaml
- uses: shivammathur/setup-php@v2
  with: { php-version: '8.3', coverage: none }
- uses: getOtterWise/pebble-setup@v1         # covers what phpunit.xml lists
- run: pebble run -- vendor/bin/phpunit --coverage-clover=build/logs/clover.xml
```

The action installs the prebuilt `pebble.so` for the job's PHP, enables it in
the PHP's ini scan directory, puts `pebble` on PATH and exports `PEBBLE_OUTPUT` and
`PEBBLE_GLUE`. The files to cover come from the PHPUnit configuration, as
php-code-coverage reads them: the include and exclude lists of `<source>`
(PHPUnit 10+), `<coverage>` (9.3+) or `<filter><whitelist>`. The `directory`
and `exclude` inputs override them. `pebble run` hooks the PHPUnit extension in through a generated
bootstrap script (PHPUnit 10+) or a copy of the configuration (PHPUnit 9), so
`phpunit.xml` stays unchanged. It works with PHPUnit 9 to 13, paratest,
`php artisan test --parallel`, Pest, and Symfony's phpunit-bridge.

The releases here are the coverage build. Fork mode, where the application
boots once per process and each test runs in a forked copy, is not part of
it: it runs on the pebble hosted runner. `pebble run --fork` with this build
says so and runs the tests as PHPUnit does.

## Report

`pebble run` takes PHPUnit's `--coverage-clover`, `--coverage-cobertura`,
`--coverage-html` and `--coverage-text` out of the command and writes those reports from the streams after the tests, so
PHPUnit does not look for a coverage driver. `pebble report` writes the same
reports as a step of its own. It reads the streams from `$PEBBLE_OUTPUT`,
which the action exports. Without `-f` it prints a text summary and writes no file. Pass
`-f clover`, `-f cobertura` or `-f lcov` for a file an uploader can read, or
`-f html` for pages to read yourself:

| Command                        | Writes                    |
|--------------------------------|---------------------------|
| `pebble report`                 | text summary on stdout    |
| `pebble report -f clover`       | `build/logs/clover.xml`   |
| `pebble report -f lcov`         | `build/logs/lcov.info`    |
| `pebble report -f cobertura`    | `build/logs/cobertura.xml` |
| `pebble report -f html`         | `build/coverage/` (`index.html` and a page per file) |
| `pebble report -f clover -o x`  | `x` (`-` for stdout)      |

`build/logs/` is where most uploaders look by default, so they need no
`--file` argument.

### Branch coverage

`pebble run --branch-coverage` also records branches as php-code-coverage
counts them with Xdebug and `--path-coverage`: the Clover report then has
`conditionals` and `coveredconditionals`, the Cobertura report a `branch-rate`
per method, class and package and `branches-covered`/`branches-valid` for the
run, the text summary a `Branches:` line. Line coverage stays the same. It
costs 3 to 11 percent on top of line coverage. Without `pebble run`, set
`PEBBLE_BRANCHES=1` in the job's environment: the extension reads it when PHP
starts. `--path-coverage` (`PEBBLE_PATHS=1`) records paths too, as Xdebug
does for php-code-coverage: the text summary gets a `Paths:` line, and the
CRAP index uses the share of paths taken. The Clover report then also has
`paths` and `coveredpaths` on its metrics and method lines, and the
Cobertura report `path-rate`, `paths-covered` and `paths-valid`; neither
format defines a field for paths, so these are attributes of pebble's own. On CPU-bound code it costs about
twice what branches cost; on an application suite the two are alike.

```yaml
- run: pebble run --branch-coverage -- vendor/bin/phpunit --coverage-cobertura build/logs/cobertura.xml
```

## Prebuilt assets

Each release of this repository carries `pebble.so` for PHP 8.1 to 8.5 (NTS,
Linux x86_64 and macOS arm64), the `pebble` binary for those platforms, and the
PHP glue, all without fork mode. The extension source is not published, so a PHP build without a
prebuilt asset (ZTS, debug, Linux arm64) fails with a message that
says so. The action ships glibc builds only. Organizations with access to the private source can set
`repository: getOtterWise/pebble` and a `token` to get the from-source
fallback.

## Inputs

| Input        | Default              | Meaning |
|--------------|----------------------|---------|
| `directory`  | from the configuration | Comma-separated directories and files to cover, relative to the workspace or absolute. Default: the include list of `phpunit.xml`. |
| `exclude`    | from the configuration | Comma-separated paths to exclude, relative to the workspace. Default: the exclude list of `phpunit.xml`. |
| `configuration` | `phpunit.xml`, `phpunit.xml.dist`, `phpunit.dist.xml` | The PHPUnit configuration to read those lists from, relative to the workspace. |
| `output`     | `.pebble/runs`        | Directory for the coverage streams, exported as `PEBBLE_OUTPUT`. |
| `version`    | `latest`             | pebble release tag to install. |
| `repository` | `getOtterWise/pebble-setup` | Repository that publishes the releases. |
| `token`      | `github.token`       | Only needed when that repository is private: a fine-grained PAT with Contents: read. |

Outputs: `version`, `extension` (path of the installed `pebble.so`), `binary`
(path of the `pebble` binary).

## On a pebble runner image

When the runner already has pebble preinstalled (the tracer loaded in every
PHP, `pebble` on PATH), the action downloads nothing: it only writes the job's
directory and exclude settings and exports the same environment. The same two
workflow lines therefore work on GitHub-hosted runners and on pebble runners;
no token is needed there. On such a runner two more actions from this
repository replace `setup-php` and the `services:` block:

```yaml
- uses: getOtterWise/pebble-setup/php@v1
  with: { php-version: '8.3', extensions: 'mailparse' }   # extensions: only what the image lacks
- uses: getOtterWise/pebble-setup/services@v1
  with: { services: 'mysql:8.4, redis', databases: 'app, app_tenant', parallel: 4 }
```

`php` switches the preinstalled PHP in under a second; `extensions` adds
distro packages the image does not carry, a few seconds each, and reads
setup-php's list as it is, with `ini-values` and `coverage` too. `jit: tracing` (or `function`)
switches the JIT on for the job; it is off by default. Measure it first: it
broke tests in laravel/framework and snipe-it that pass without it. `services`
starts the listed servers on 127.0.0.1 (MySQL root/root, PostgreSQL
postgres/postgres, Redis without a password) with the databases, and
`parallel: N` adds `<name>_test_1` to `<name>_test_N` for each of them,
which is what Laravel's `--parallel` expects. `user` and `password` give the
job the account its `services:` block had (`user: root` and no `password`
for `MYSQL_ALLOW_EMPTY_PASSWORD`), and `ports: mysql=33306` the port it
mapped. `services` also takes `valkey`, `memcached` and `meilisearch`. A colon picks
a version: `mysql:8.4` (the default), `8.0`, `9`, `26` and `5.7` (amd64);
`mariadb:12.3` (the default), `11.8`, `11.4` and `10.11`; `postgres:16` (the
default) and `14` to `19`. MySQL and MariaDB share the socket, so a job runs
one of them.

A third one keeps what the next job should not build again:

```yaml
- uses: getOtterWise/pebble-setup/cache@v1
  with:
    key: vendor-8.3-${{ hashFiles('composer.lock') }}
    restore-keys: vendor-8.3-
- run: composer install --no-interaction --prefer-dist
- uses: getOtterWise/pebble-setup/cache/save@v1
  with:
    key: vendor-8.3-${{ hashFiles('composer.lock') }}
    path: vendor
```

The runner reaches a cache service over the network with a credential minted
for that job alone, so no two jobs share a directory. The scope is not the
job's to choose: a branch reads its own entries and then the default branch's,
and a job of a fork pull request writes a scope of its own and never the
branch's. An entry is immutable, so put what the content depends on in the
key. Restore takes no path, because the entry holds the paths its save named.
Nothing here fails a build: no cache service, no entry, or an entry whose
digest does not match, and the job carries on and installs what it needs.

### npm and the asset build

A front end gives two more entries, and they are a different shape from
`vendor`. `composer install` corrects a near miss in place, so a prefix key
helps it. `npm ci` deletes `node_modules` before it installs, so a near miss
buys nothing: keep the directory itself and skip the step on a hit, which
`cache-hit` reports (`true` for the exact key only). `cache-matched-key` is the
key that was restored, which differs from `key` after a `restore-keys` match.

```yaml
- uses: actions/setup-node@v7
  with: { node-version: '22', package-manager-cache: false }

- uses: getOtterWise/pebble-setup/cache@v1
  id: npm
  with:
    key: node22-${{ hashFiles('package-lock.json') }}
- run: npm ci
  if: steps.npm.outputs.cache-hit != 'true'
- uses: getOtterWise/pebble-setup/cache/save@v1
  if: steps.npm.outputs.cache-hit != 'true'
  with:
    key: node22-${{ hashFiles('package-lock.json') }}
    path: node_modules

- uses: getOtterWise/pebble-setup/cache@v1
  id: assets
  with:
    key: assets-${{ hashFiles('package-lock.json', 'vite.config.js', 'resources/**') }}
- run: npm run build
  if: steps.assets.outputs.cache-hit != 'true'
- uses: getOtterWise/pebble-setup/cache/save@v1
  if: steps.assets.outputs.cache-hit != 'true'
  with:
    key: assets-${{ hashFiles('package-lock.json', 'vite.config.js', 'resources/**') }}
    path: public/build
```

Four things decide whether this is correct:

- **The Node major belongs in the key.** A compiled native module built for
  one major does not load in the next. On the runner image `setup-node` finds
  20, 22 and 24 in the tool cache and downloads nothing for a version such as
  `22` or `22.x`; an alias such as `lts/*` is resolved online first, and
  another major is downloaded as on any runner.
- **`setup-node` from v5 on caches by itself** when `package.json` has a
  `packageManager` field, in GitHub's cache service. With `node_modules` in
  this cache that is a second cache doing the same job, so
  `package-manager-cache: false` turns it off. v5 and later also run on Node
  24; v4 declares Node 20, which a current runner runs on 24 anyway and says
  so in a notice on every job.
- **The asset key names everything the build reads**, not only the lock file.
  Then a commit that touches no front end file skips `npm ci` and `npm run
  build` both, and a commit that changes one line of CSS builds again.
- **Save on a miss only.** An entry is immutable, so a second save of the same
  key is refused, but the job packs the archive before it hears that.

A path may start with `~/`, for a package manager's own download cache
(`~/.npm`, `~/.cache/ms-playwright`). That is the other shape: keep the
downloads, and run the install every time.

### PHPStan and Larastan

PHPStan keeps a result cache in its `tmpDir` (by default the system temporary
directory's `phpstan/`) and checks it by itself: a file that changed, or a
file it depends on, is analysed again, and a new configuration or version
throws the cache away. So a result cache from an older commit is safe to
restore, and it is the third shape: a key per commit, the newest one by prefix.

```yaml
- uses: getOtterWise/pebble-setup/cache@v1
  with:
    key: phpstan-${{ github.sha }}
    restore-keys: phpstan-
- run: vendor/bin/phpstan analyse --memory-limit=2G
- uses: getOtterWise/pebble-setup/cache/save@v1
  if: always()
  with:
    key: phpstan-${{ github.sha }}
    path: /tmp/phpstan
```

`if: always()` keeps the cache when PHPStan reports errors, which is when the
next run needs it most. Every commit saves an entry of its own, since an entry
is immutable; a branch restores its newest one and then the default branch's,
and the ones nothing reads any more are evicted. Larastan is PHPStan with an
extension, so the same steps apply. A `tmpDir` set in `phpstan.neon` is the
path to save instead.

## Notes

- Do not enable another coverage driver in setup-php: `coverage: none` is the
  setting. A second driver costs exactly the speed pebble is installed for, and
  Xdebug conflicts with it outright.
- opcache must be off in the coverage process: the tracer cannot instrument
  the scripts opcache shares, and the JIT runs past its line handlers. The
  action writes `opcache.enable_cli=0` into an ini file that sorts after
  setup-php's `99-pecl.ini`, and fails when something still turns it on.
  `pebble run` switches it off for the processes it starts as well, so a
  runner that keeps opcache on for the rest of the job still records. Do not
  pass `-d opcache.enable_cli=1` to the test command.
- `pebble report` reports the lines php-code-coverage would, using the
  project's own copy of it through `php`; `--lines engine` gives the
  tracer's own lines instead, whose totals differ by design.
- Tests marked `#[RunInSeparateProcess]` are recorded when the command runs
  through `pebble run`. PHPUnit starts a process of its own for them that
  registers no extensions; the bootstrap `pebble run` generates hooks that
  process too. Starting PHPUnit directly with an `<extensions>` entry needs
  `\Pebble\Isolation::install();` in the project's bootstrap file instead, or
  those tests carry no lines and nothing warns.
- `container:` jobs must run the action inside the container.
- Some pebble runners have no Docker daemon. On such a runner, Docker container
  actions (an action with a Dockerfile, or `docker://`), `services:` blocks and
  `container:` jobs do not run; the `services` action above starts the servers
  instead, and a call to `docker` prints that. When `docker` works on the
  runner, all three run. On a GitHub-hosted runner all of them work as usual.

## License

MIT, see `LICENSE`.
