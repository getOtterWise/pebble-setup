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
- run: pebble run -- vendor/bin/phpunit
- run: pebble report -f clover                 # writes build/logs/clover.xml
```

The action installs the prebuilt `pebble.so` for the job's PHP, enables it in
the PHP's ini scan directory, puts `pebble` on PATH and exports `PEBBLE_OUTPUT` and
`PEBBLE_GLUE`. The files to cover come from the PHPUnit configuration, as
php-code-coverage reads them: the include and exclude lists of `<source>`
(PHPUnit 10+), `<coverage>` (9.3+) or `<filter><whitelist>`. The `directory`
and `exclude` inputs override them. `pebble run` hooks the PHPUnit extension in through a generated
bootstrap script (PHPUnit 10+) or a copy of the configuration (PHPUnit 9), so
`phpunit.xml` stays unchanged. It works with PHPUnit 9 to 12, paratest,
`php artisan test --parallel`, Pest, and Symfony's phpunit-bridge.

The releases here are the coverage build. Fork mode, where the application
boots once per process and each test runs in a forked copy, is not part of
it: it runs on the pebble hosted runner. `pebble run --fork` with this build
says so and runs the tests as PHPUnit does.

## Report

`pebble report` reads the streams from `$PEBBLE_OUTPUT`, which the action
exports. Without `-f` it prints a text summary and writes no file. Pass
`-f clover` or `-f lcov` for a file an uploader can read:

| Command                        | Writes                    |
|--------------------------------|---------------------------|
| `pebble report`                 | text summary on stdout    |
| `pebble report -f clover`       | `build/logs/clover.xml`   |
| `pebble report -f lcov`         | `build/logs/lcov.info`    |
| `pebble report -f clover -o x`  | `x` (`-` for stdout)      |

`build/logs/` is where most uploaders look by default, so they need no
`--file` argument.

## Prebuilt assets

Each release of this repository carries `pebble.so` for PHP 8.1 to 8.5 (NTS,
Linux x86_64 and macOS arm64), the `pebble` binary for those platforms, and the
PHP glue, all without fork mode. The extension source is not published, so a PHP build without a
prebuilt asset (ZTS, debug, Linux arm64, Alpine) fails with a message that
says so. Organizations with access to the private source can set
`repository: getOtterWise/pebble` and a `token` to get the from-source
fallback.

## Inputs

| Input        | Default              | Meaning |
|--------------|----------------------|---------|
| `directory`  | from the configuration | Comma-separated directories and files to cover, relative to the workspace or absolute. Default: the include list of `phpunit.xml`. |
| `exclude`    | from the configuration | Comma-separated paths to exclude, relative to the workspace. Default: the exclude list of `phpunit.xml`. |
| `configuration` | `phpunit.xml`, `.xml.dist`, `.dist.xml` | The PHPUnit configuration to read those lists from, relative to the workspace. |
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
distro packages the image does not carry, a few seconds each. `services`
starts the listed servers on 127.0.0.1 (MySQL root/root, PostgreSQL
postgres/postgres, Redis without a password) with the databases, and
`parallel: N` adds `<name>_test_1` to `<name>_test_N` for each of them,
which is what Laravel's `--parallel` expects.

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

## License

MIT, see `LICENSE`.
