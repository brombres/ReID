# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

ReID is a command-line tool for renaming identifiers across a source tree and reorganizing folder hierarchies to match. It is written in [Rogue](https://github.com/brombres/Rogue) and built with Rogo (Rogue's build tool). The entire program lives in `Source/ReID.rogue`; `Build.rogue` is the Rogo build script. There is no test suite.

A local checkout of the Rogue language repo is at `~/Projects/Rogue`. Consult it for language syntax and the standard library (e.g. `String.replacing_pattern`, `FilePattern`, `Files.sync_to`, `Console/CommandLineParser`) rather than guessing.

## Git policy

Auto-commit changes when a task is done, but never push. Pushing and releasing are done by the user (see Releasing below).

## Build and run

Rogo compiles Rogue -> C -> native executable. `roguec`, `rogo`, and `morlock` are installed at `/opt/morlock/bin`.

```
rogo                 # build (if needed) and run
rogo build           # incremental build only; output is Build/ReID-<OS> (e.g. Build/ReID-macOS)
rogo rebuild         # force full rebuild
rogo debug           # force debug build (--debug roguec flag, -O0) and run
rogo release         # force release build (-O3) and run
rogo clean           # delete Build/ and .rogo/
rogo install         # build and create a /usr/local/bin/reid launcher script
rogo link            # build and register the launcher with Morlock's binpath
rogo help            # list all rogo commands
```

To try a change manually, run the built binary directly, e.g. `Build/ReID-macOS --grep SomeID "Source/**"`. Use `--yes`/`--save` to skip the interactive prompt, and `--plain` to disable ANSI styling in output. Quote wildcard args so the shell doesn't expand `**`.

`Build/` and `.rogo/` are gitignored build outputs (the `.rogo/` folder holds the compiled build script itself). A `Local.settings` file (also not committed) can override `Build` properties such as `BUILD_MODE = debug`.

## Releasing

Version and date are defined in two places and must stay in sync: `$define VERSION`/`$define DATE` at the top of `Source/ReID.rogue`, and the table in `README.md`. Do not edit them by hand; use the build script:

```
rogo update_version 2.4   # rewrite version/date in both files
rogo commit 2.4           # update_version + `git commit -am "[v2.4]"`
rogo publish 2.4          # commit, merge develop -> master, `gh release create v2.4`
```

Work happens on `develop`; `master` is the release branch. Commit messages use a bracketed tag prefix, e.g. `[Bugfix] ...`, `[--exclude] ...`, `[v2.3]`. Morlock installs releases by scanning GitHub releases (`Morlock/reid.rogue`), so a version is only installable once published.

The usage text in `print_usage` (in `Source/ReID.rogue`) and the Options section of `README.md` are duplicates of each other; update both when changing command-line options.

## Architecture (Source/ReID.rogue)

Three classes form a pipeline: parse args -> build a replacement table -> collect per-file changes -> preview -> apply.

- **`ReID`** (entry point): parses the command line with `Console/CommandLineParser`, expands wildcard filepath args via `FilePattern` (honoring `--exclude`, which is either `--exclude=pat` or a positional group of patterns following bare `--exclude`), and then either loads a saved change list (`reid ReIDChanges.txt`) or calls `collect_changes`.
  - `collect_changes` decides *how* `OldID NewID` becomes a replacement map. Four modes: `$` in the search term -> Rogue pattern replacement (`uses_patterns = true`); `--exact` or a `/` in the term -> single literal replacement; `--package` -> dotted name plus its slash-separated folder equivalent; otherwise -> automatic case-variant generation. The case-variant logic splits the ID into words (from underscores or CamelCase boundaries) and emits lowerCamel, snake, SCREAMING_SNAKE, Capitalized_Snake, and UpperCamel forms. Insertion order into `replacements` matters because later entries with the same key overwrite earlier ones; UpperCamel is added last deliberately.
- **`Changes`**: holds the `replacements` map and everything derived from it. Key methods:
  - `replace` implements whole-identifier matching (a match is rejected if adjacent characters are identifier characters) unless `is_partial`. Pattern mode always allows partial matches.
  - `apply_replacements` and `grep` first split text around any `--ignore` pattern match and recurse on the outer parts, so ignored regions (e.g. comments) are never touched.
  - `collect_changes(filepath)` produces a `FileChanges` for a file if its path or any line changes (binary files are skipped via `is_valid_utf8`). In `--grep` mode it records matching lines instead.
  - `preview_and_apply` prints the styled preview, prompts `y/s/n` (or uses `auto_response` from `--yes`/`--save`), then either applies changes or serializes them to a change-list file.
  - `apply_changes` runs in three stages: rewrite line content, rename/relocate individual files (prompting before overwriting), then reorganize whole folders longest-path-first using `Files.sync_to` followed by deletion of the old copies and any emptied parent folders.
  - `init(file, options)` parses the change-list file format, which is also what the preview prints: `replace old -> new` / `grep pattern` header lines, then per file `file path`, optional `rename newpath`, and `NNNNNN  new line content` entries. Leading whitespace on changed lines is stripped in the file and restored from the original line on apply.
- **`FileChanges`** / **`LineChange`**: one file's pending path rename plus its line edits. `FileChanges.init` refuses renames that differ only by case (exits with a message suggesting a two-step rename), since that fails on case-insensitive filesystems.

Styled output works by keeping a parallel `styled_replacements` map whose values are wrapped in ANSI underline codes; the preview re-runs replacement with that map rather than diffing.
