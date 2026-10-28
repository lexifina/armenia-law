# Armenian law, change by change

The laws of the Republic of Armenia as a git repository: the Constitution, 42 codes and 1342 laws from [ARLIS](https://www.arlis.am), the official legal information system of the Republic of Armenia. Each article is its own file, with one paragraph per line. Every change is a commit credited to the act that made it and dated the day the changed text took effect, at noon Yerevan time. The text is the Armenian original, which is the authoritative text.

On `main`, the record runs from 1961-03-07 to 2026-10-24 in 14423 commits, built from 14487 texts: each law's original act plus every consolidated text ARLIS made of it. A further 6 commits apply after the build day and sit on the `future` branch.

## A lawyer's questions as git commands

| Question | Command |
| --- | --- |
| What did article 4 of the Law on Copyright and Related Rights say on 1 June 2012? | `python tools/in_force_on.py 2012-06-01 law/25161 4` |
| The whole law on a date | `python tools/in_force_on.py 2012-06-01 law/25161` |
| Which act last changed each paragraph of an article? | `git blame law/25161/4.md` |
| Every change to an article, with the act behind it | `git log -p law/25161/4.md` |
| The words that changed, not whole paragraphs | `git log -p --word-diff law/25161/4.md` |
| Everything one act changed, across every law | `git log --author='ՀՕ-106-Ն' --stat` |
| Every act that changed a law, in order | `git log --format='%(trailers:key=In-force,valueonly,separator=) %an' -- law/25161` |
| What takes effect after today | `git log main..future` |
| The list of laws, with their ARLIS links | [`laws.csv`](laws.csv) |

Use `tools/in_force_on.py` for "on this date" questions rather than `git log --before`. It picks the text by the `In-force:` trailer, so it is right in any time zone and for dates before 1970. If you use raw git, write the time out in full: `git rev-list -1 --before="2012-06-01T23:59:59+04:00" main`. In PowerShell, `git show "$(git rev-list -1 --before=... main):law/25161/4.md"` needs the quotes.

Folders are `constitution/`, `code/` and `law/`, and each law's folder is named after its ARLIS number, the `N` in `arlis.am/hy/acts/N`. Inside it there is one file per article (`4.md`, `22.1.md`; a superscript article such as 327¹ becomes `327.1.md`), one per appendix (`annex-1.md`), and `title.md` for the text before the first article. An article file starts with the Part and Chapter headings above it (`##`), then the heading line (`# Հոդված 4. …`), then one line per paragraph. Each article's own history note, such as `(4-րդ հոդվածը լրաց. 30.09.13 ՀՕ-106-Ն)`, is kept as a line, exactly as ARLIS prints it.

## The dates, and which one we use

ARLIS keeps every consolidated text it made of a law and gives each one its period of application, for example `Գործում է (01.09.2026-01.11.2026)`. **Commits are dated by the first day of that period.** That date decides which text governs a case, and it is the day the changes in that text took effect. The amending act's adoption date is in the `Enacted:` trailer. In 2 commits, the text applies from a day before the act was adopted (retroactive effect). Those commits keep the in-force date and show the adoption date.

The first commit of each law is its original act (`Kind: original`), dated the day the law took effect. Laws older than ARLIS's consolidated texts may be missing some intermediate texts. Their next commit can then carry several amendments at once (see "what author means").

**Git cannot store dates before 1970.** Laws that took effect earlier (Soviet-era codes) are committed on 1970-01-01, one second apart and in order. Their real date is in the `In-force:` trailer, which `tools/in_force_on.py` reads.

**Texts that apply after the build day** are on the branch `future`, which starts at the tip of `main` and is never merged into it, so `main` never shows text that is not yet law. After a rebuild, run `git fetch --force origin future`.

## What "author" means

The author of a commit is the act that made the change, at an address that cannot exist, for example `ՀՕ-106-Ն (30.09.2013) <ho-106-n-20130930@arlis.invalid>`. The number is the act's official number (ՀՕ = law of the Republic of Armenia) and the date is its adoption date. So `git blame` names, for each paragraph, the act that last wrote it. One commit holds everything one act changed on one day, across every law it touched.

ARLIS pairs every consolidated text with the one amending act it was made for, and that act is credited (`Evidence: edition`). Where the changed article's own history note newly names that act, the evidence is `note`. Where the note instead newly names another amending act of the same law, one that has no consolidated text of its own, the change goes to the latest such act (`Evidence: note (no text of its own)`). Amending acts that ARLIS lists but never made a text for (513) are in [`data/not-incorporated.csv`](data/not-incorporated.csv). Their changes reach this record only through the next text that was made.

Evidence counts by changed unit: note 30098, edition 46429, note with no text of its own 608. In all there are 1383 original acts and 13039 amendment commits by 13027 amending acts.

Blame credits whole paragraphs. A paragraph changed by two acts names only the later one, so use `git log -p --word-diff` to see each act's words.

## What it does not show

- Sub-laws: government decisions, ministerial orders and other acts below the level of law. They are not in this build.
- Texts ARLIS has not consolidated (see above), and texts it could not serve or we could not read (3, in [`data/unreadable.csv`](data/unreadable.csv)).
- Laws whose only purpose is to amend others. They are the authors of commits here, not files. Their own transitional provisions, which say when and to which cases a change applies, are in each act at the `Source:` link of the commit.
- ARLIS's links to case law, its tables of contents, and its English and Russian translations.
- Renumbering: an article that moves to a new number appears as one file removed and another added.

ARLIS's consolidated texts are marked official or unofficial. This repository is a research aid and is not an official publication. For the authoritative text, see the official publication cited at the `Source:` link.

## How it is checked

- Before each push, `tools/in_force_on.py` is compared with ARLIS's own text for several articles and dates.
- Every commit's trailers parse with `git interpret-trailers --parse`.
- `main` has no merge commits, its commit dates never go backwards, and none of them is after the build day.
- Two builds from the same cache give the same commit hashes.

The sha256 of every downloaded text is in [`data/texts-sha256.tsv`](data/texts-sha256.tsv). See [`REPRODUCE.md`](REPRODUCE.md).

## How it is built

`tools/build/fetch.py` finds every Constitution, code and law through ARLIS's search, leaving out acts that only amend others. It then reads each law's change history, which pairs amending acts with consolidated texts, and downloads the print view of the original act and of every consolidated text. `parse_text.py` splits each text into articles and paragraphs. `build.py` compares each law's texts in date order and credits each change as above. It writes the whole history with one `git fast-import`. No AI model reads or writes any text, either in the build or in the checks.

## Licence

Source: ARLIS, the legal information system of the Republic of Armenia (www.arlis.am). Under article 4(1)(c) of the Law of the Republic of Armenia on Copyright and Related Rights (ՀՕ-142-Ն, 2006), official documents, including legal acts and their official translations, are not objects of copyright. The texts here were split into files and arranged as commits; their wording was not changed. Lexifina's own contributions (the tools, the arrangement and the commit messages) are CC0. See [`LICENSE`](LICENSE).
