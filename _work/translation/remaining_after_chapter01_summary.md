# Remaining Translation Export

Summary of `remaining_after_chapter01.tsv`, which contains all remaining `CRES.DLL` strings outside `chapter01_priority.tsv`.

## Export Metadata

- **Source**: `C:\Games\No One Lives Forever 2\_work\analysis\CRES_modernizer_strings.csv`
- **Total CRES rows**: 6,355
- **Excluded IDs (already in `chapter01_priority.tsv`)**: 1,228
- **Exported rows**: 5,127

## Category Counts

| Category | Rows |
| --- | ---: |
| `dialogue` | 1,854 |
| `menu` | 951 |
| `intel` | 922 |
| `other` | 410 |
| `reward` | 291 |
| `equipment` | 228 |
| `mission` | 221 |
| `mission_objective` | 202 |
| `server_admin` | 46 |
| `mission_failure` | 2 |
| **Total** | **5,127** |

## Editing Rules

- Translate only the `zh` column.
- Keep format placeholders such as `%s`, `%d`, `%1!d!`, and delimiters like `@` intact.
- Keep angle brackets (`<` and `>`) around angle-bracket labels.
- Line breaks in `english` are escaped as `\n`; use `\n` in `zh` whenever a manual line break is needed.
- `server_admin` / SCMD rows are low priority for single-player testing.
