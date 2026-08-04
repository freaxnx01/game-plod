# Changelog

All notable changes to this project are documented here, following
[Keep a Changelog](https://keepachangelog.com) and
[Semantic Versioning](https://semver.org).

## [Unreleased]

### Added

- Guided tutorial level teaching all 8 skills in sequence, reached via a new
  TUTORIAL button on the ready screen — untimed, with no fail state (#4).
- Tutorial step guidance: a prompt banner for the current step, skill tiles
  locked to the skills taught so far, and a step that only advances once the
  step's skill has actually been used *and* a plod has cleared the matching
  obstacle (#4).
- Tutorial completion screen ("YOU'RE READY!") with a BACK TO MENU button that
  returns to Moss Hollow (#4).

### Fixed

- A digger or miner that tunnels out through the bottom of the world now dies
  instead of pacing the bottom row forever. Previously such a plod never
  resolved, so Moss Hollow could not reach its win/lose screen until the timer
  ran out.
- Tutorial obstacles can no longer be bypassed without the skill they teach: the
  digger riser and the bomber wall gained overhang lips (climber-flagged plods
  used to climb straight over them), and the blocker fork's wrong branch is now a
  genuinely bottomless chasm rather than a long drop floater-flagged plods
  survived (#4).
- Losing every plod during the tutorial restarts it instead of leaving the level
  parked in the play phase with no visible way forward (#4).
- The countdown timer and the "1·1 MOSS HOLLOW" level chip are hidden during the
  tutorial, which is untimed and is not Moss Hollow (#4).
- Tutorial skills now unlock cumulatively instead of only the current step's
  skill being assignable: a plod left behind at an earlier obstacle can still be
  given the skill that clears it after the tutorial has moved on. Previously such
  a straggler was locked out of that skill for good and paced in front of the
  obstacle with no way to recover (#4).

## [0.1.0] - 2026-07-18

### Added

- Initial versioned release of game-plod.
- In-game version badge sourced from `version.js`.
