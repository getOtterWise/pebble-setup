# fphpcov-setup

GitHub Action that installs [pcov2](https://github.com/getOtterWise/pcov2)
on the job's PHP: the native coverage tracer extension, the `pcov2` reporter
binary, and the PHPUnit glue. Works with setup-php (use `coverage: none`) on
GitHub-hosted and self-hosted Linux and macOS runners.

This repository is a thin installer. The code lives in the private
`getOtterWise/pcov2` repository. Its release workflow copies the action here
and publishes the prebuilt assets as releases of this repository, so no token
is needed. Do not edit the files here.

## Usage

```yaml
- uses: shivammathur/setup-php@v2
  with: { php-version: '8.3', coverage: none }
- uses: getOtterWise/fphpcov-setup@v1
  with: { directory: "app,routes,helpers.php" }   # what phpunit.xml <source> lists
- run: pcov2 run -- vendor/bin/phpunit
- run: pcov2 report -f clover                 # writes build/logs/clover.xml
```

The action installs the prebuilt `pcov2.so` for the job's PHP, enables it in
the PHP's ini scan directory, puts `pcov2` on PATH and exports `PCOV2_OUTPUT` and
`PCOV2_GLUE`. `pcov2 run` hooks the PHPUnit extension in through a generated
bootstrap script (PHPUnit 10+) or a copy of the configuration (PHPUnit 9), so
`phpunit.xml` stays unchanged. It works with PHPUnit 9 to 12, paratest,
`php artisan test --parallel`, Pest, and Symfony's phpunit-bridge. For a
Laravel application it also turns fork mode on: the application boots once
per process and each test runs in a forked copy. `pcov2 run --no-fork`, or
`PCOV2_FORK=0` in the job's `env`, switches that off.

## Report

`pcov2 report` reads the streams from `$PCOV2_OUTPUT`, which the action
exports. Without `-f` it prints a text summary and writes no file. Pass
`-f clover` or `-f lcov` for a file an uploader can read:

| Command                        | Writes                    |
|--------------------------------|---------------------------|
| `pcov2 report`                 | text summary on stdout    |
| `pcov2 report -f clover`       | `build/logs/clover.xml`   |
| `pcov2 report -f lcov`         | `build/logs/lcov.info`    |
| `pcov2 report -f clover -o x`  | `x` (`-` for stdout)      |

`build/logs/` is where most uploaders look by default, so they need no
`--file` argument.

## Prebuilt assets

Each release of this repository carries `pcov2.so` for PHP 8.1 to 8.5 (NTS,
Linux x86_64 and macOS arm64), the `pcov2` binary for those platforms, and the
PHP glue. The extension source is not published, so a PHP build without a
prebuilt asset (ZTS, debug, Linux arm64, Alpine) fails with a message that
says so. Organizations with access to the private source can set
`repository: getOtterWise/pcov2` and a `token` to get the from-source
fallback.

## Inputs

| Input        | Default              | Meaning |
|--------------|----------------------|---------|
| `directory`  | required             | Comma-separated directories and files to cover, relative to the workspace or absolute. Use what `phpunit.xml` lists in `<source>` (PHPUnit 10+) or `<coverage>` (PHPUnit 9). |
| `exclude`    | empty                | Comma-separated paths to exclude, relative to the workspace. |
| `output`     | `.pcov2/runs`        | Directory for the coverage streams, exported as `PCOV2_OUTPUT`. |
| `version`    | `latest`             | pcov2 release tag to install. |
| `repository` | `getOtterWise/fphpcov-setup` | Repository that publishes the releases. |
| `token`      | `github.token`       | Only needed when that repository is private: a fine-grained PAT with Contents: read. |

Outputs: `version`, `extension` (path of the installed `pcov2.so`), `binary`
(path of the `pcov2` binary).

## On a pcov2 runner image

When the runner already has pcov2 preinstalled (the tracer loaded in every
PHP, `pcov2` on PATH), the action downloads nothing: it only writes the job's
directory and exclude settings and exports the same environment. The same two
workflow lines therefore work on GitHub-hosted runners and on pcov2 runners;
no token is needed there. On such a runner two more actions from this
repository replace `setup-php` and the `services:` block:

```yaml
- uses: getOtterWise/fphpcov-setup/php@v1
  with: { php-version: '8.3', extensions: 'mailparse' }   # extensions: only what the image lacks
- uses: getOtterWise/fphpcov-setup/services@v1
  with: { services: 'mysql:8.4, redis', databases: 'app, app_tenant', parallel: 4 }
```

`php` switches the preinstalled PHP in under a second; `extensions` adds
distro packages the image does not carry, a few seconds each. `services`
starts the listed servers on 127.0.0.1 (MySQL root/root, PostgreSQL
postgres/postgres, Redis without a password) with the databases, and
`parallel: N` adds `<name>_test_1` to `<name>_test_N` for each of them,
which is what Laravel's `--parallel` expects.

## Notes

- Do not use `coverage: pcov` or `coverage: xdebug` in setup-php. PCOV next to
  pcov2 gives the same numbers but makes the run slow again; Xdebug conflicts.
- opcache must be off in the coverage process. The action writes
  `opcache.enable_cli=0` into an ini file that sorts after setup-php's
  `99-pecl.ini`, and fails when something still turns it on. Do not pass
  `-d opcache.enable_cli=1` to the test command.
- Executable-line totals differ from php-code-coverage by design. Coverage
  percentages stay within about a point. Recalibrate thresholds once.
- `container:` jobs must run the action inside the container.

## License

MIT, see `LICENSE`.
