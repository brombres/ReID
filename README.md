# ReID
Command line tool for replacing identifiers within source code and adjusting folder hierarchies.

About     | Current Release
----------|-----------------------
Version   | 2.0.4
Date      | February 14, 2023
Platforms | Windows, macOS, Linux
License   | [MIT License](LICENSE)
Author    | Brom Bresenham

## Installation
1. Install [morlock.sh](https://morlock.sh)
2. `morlock install brombres/reid`

## Example
Here is a sample project `Alpha` with its file structure:
![](Images/AlphaFiles.png)

And its content:
![](Images/AlphaContent.png)

From outside the `Alpha` folder we run `reid --partial alpha beta "Alpha/**"` (the quotes are necessary for ReID's special wildcard `**`). The screenshot does not include the necessary `--partial` as the screenshot was taken when partial matches were allowed by default.
![](Images/ReID.png)

After we confirm the preview of changes, there is the new `Beta` project file structure:
![](Images/BetaFiles.png)

and the `Beta` file content:
![](Images/BetaContent.png)

## Usage

    reid [options] OldID NewID           [wildcard_filepaths] [--exclude] [...]
    reid old/folder/path new/folder/path [wildcard_filepaths] [--exclude] [...]
    reid old$pattern new$pattern         [wildcard_filepaths] [--exclude] [...]
    reid ReIDChanges.txt

## Options

- `reid OldID NewID [wildcard_filepaths]`<br>
  Accepts IDs in any of the following capitalization styles and automatically
  generates the other style variations for ID replacement:

        OldID  -> NewID
        oldID  -> newID
        Old_ID -> New_ID
        old_id -> new_id
        OLD_ID -> NEW_ID
        oldid  -> newid

  Using the given input filepath(s) (or `**` - all files and subfolders in
  the current folder), ReID then generates and displays a list of changes it
  will make to the content of the files and the filepath names. The user can
  then proceed with the changes immediately or save them to an intermediate
  file for editing (e.g. `ReIDChanges.txt`), then running
  `reid ReIDChanges.txt`.

- `reid old/folder/path new/folder/path [wildcard_filepaths]`<br>
  When slashes are present in the search term, ReID replaces only exact
  matches, which are most often folder paths.

- `reid old$pattern new$pattern [wildcard_filepaths]`<br>
    Uses Rogue's String.replacing_pattern() syntax. Use single quotes to prevent
    '$' being escaped by the shell. Examples:

        reid '$ and $' '$ & $'        # Alice and Bob -> Alice & Bob
        reid '$ and $' '$(1) & $(0)'  # Alice and Bob -> Bob & Alice

- `reid ReIDChanges.txt`<br>
  When ReID is run, the previewed changes are saved to a file, and the file
  is edited to further adjust the changes, then run `reid <filename>` to
  read and then apply those edited changes.

- `--content`, `-c`<br>
    Applies replacements to file content only, not to filenames or folder paths.
    If neither `--content` nor `--filepaths` are specified then both are implied.

- `--exact`, `-e`<br>
    The search term is used exactly without added automatic variations.

- `--exclude=<filepattern>`, `-x <filepattern>`<br>
    Any files matching the given pattern are skipped.

- `--exclude [...]`<br>
    Files matching any wildcard patterns coming after `--exclude` are skipped.
    For example:

        --exclude "**/*.backup" "Build/**"

    Note: binary files as well as the default change description output file,
    `ReIDChanges.txt` are excluded by default.

- `--filepaths`<br>
    Applies replacements to filenames and folder paths only, not to file content.
    If neither `--content` nor `--filepaths` are specified then both are implied.

- `--help`, `-h`, `-?`<br>
    Display this help text.

- `--ignore=<pattern>`, `-i <pattern>`<br>
    Ignores any portions of lines that match the given pattern. Can specify
    multiple times. Example: --ignore="#*" -i "//*"

- `--grep`, `-g`<br>
    Search for the given IDs or patterns without making changes.
    Only one ID or pattern should be supplied instead of the usual two.
    Matching lines can be saved, edited, and applied using ReID.

- `--package`<br>
    Treats dots in the old ID as Java-style package path separators and renames
    both package "dot names" and folder paths accordingly. For example,
    'reid --package abc.def abc.ghi' performs content replacement 'abc.def' -> 'abc.ghi'
    and folder reorganization 'abc/def/xyz' -> 'abc/ghi/xyz'.

- `--partial`, `-p`<br>
    Allows partial matches within standard identifiers that start with a letter
    or underscore and end with an alpha-numeric character or underscore. For
    example, `reid --partial time date` would replace `timer` with `dater`.
    Note: pattern-based replacements always allow partial matches.

- `--plain`<br>
    Disables styled text such as bold and underline in the preview of changes.

- `--save`, `-s`<br>
    Always answers 's' (save) to the "Make changes now?" prompt.

- `--yes`, `-y`<br>
    Always answers 'y' (yes) to the "Make changes now?" prompt.

- [*wildcard_filepath*]<br>
    Use "quoted/wildcard/filepaths" to ensure that nonstandard wildcards `**`
    and `**/` are recognized. Note: `?` can be used to match a single character.

        *.rogue      # matches files in the current folder ending with ".rogue"
        **/*.rogue   # matches files in subfolders or in the current folder.
        */**/*.rogue # matches files in subfolders but not in the current folder

