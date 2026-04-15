# Changelog

All notable changes to this project will be documented in this file.

## [v2.1] - 2026-04-15

### Added
- Introduced AWS Secrets scope, can be used by adding the parameter "scope: Secrets", which allows
  users to pull JSON from AWS Secrets store.
- New `matrix` output: a JSON object of all fetched parameters, keyed by parameter name.
  Accessible directly via `steps.<id>.outputs.matrix` without going through the encrypt/decrypt
  cycle. Individual values can be extracted with `fromJSON(steps.<id>.outputs.matrix).KEY_NAME`.

## [v2] - 2026-04-08

### Added
- Workflow testing with `act`

### Changed
- Allow existing AWS session for credentialed access

### Fixed
- Users attempting to log in with AWS access keys must supply both secret key & access key

## [v1.1] - 2022-11-28

### Fixed
- Use `echo >> $GITHUB_OUTPUT` instead of deprecated `::set-output` approach

## [v1] - 2022-09-16

### Added
- Initial release; action can be used to collect JSON values from SSM