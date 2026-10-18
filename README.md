<p align="center">
  <img src="claude_animation.gif" alt="Claude Job Search Assistant" width="200">
</p>

# AI Job Search

A local job-application workspace for [Claude Code](https://claude.com/claude-code). Fork it, fill in a candidate profile, and Claude evaluates postings, tailors a LaTeX CV, writes a cover letter, and can prepare interview notes.

The application workflow (profile, fit evaluation, drafting, review) is language- and country-agnostic. Job discovery in this repo is aimed at Denmark: a web-search skill with Danish site queries, plus four portal skills (Jobindex, Jobnet, Akademikernes Jobbank, Jobdanmark) whose command-line tools are only partly present. See [Danish job portal tools](#danish-job-portal-tools).

## What it does

You work inside Claude Code, in this repository. Slash commands and skills are markdown instructions. They tell Claude which files to read, what to write, and which checks to run.

```
/setup                 /scrape                         /apply <url or text>
  |                      |                                |
  v                      v                                v
Fill the profile     Web-search the queries          Score the posting
from documents,      in search-queries.md            against the profile
a CV, or an          Deduplicate and rate            (optional salary lookup)
interview            high / medium / low
  |                      |                                |
  v                      v                                v
Profile files        Table of new matches            Draft CV + cover letter
and search queries   Pick a number for a             Reviewer critiques
                     full fit evaluation             Revise, compile, inspect
```

`/expand`, `/upskill`, and `/reset` sit beside that path. Interview notes come from the `job-application-assistant` skill and from `07-interview-prep.md`. `/apply` finishes when the CV and cover letter are ready.

The instructions encode a scoring framework, forward-looking cover-letter framing, and an optional salary benchmark from data you supply.

## Key features

- **Three ways to build a profile.** `/setup` reads `documents/`, imports one CV, or walks through an interview. Re-running the documents path proposes additions and asks before it replaces anything that conflicts.
- **Fit evaluation before drafting.** Five dimensions (skills, experience, behaviour and culture, location, career alignment) plus an optional salary lookup. `/apply` stops after the score unless you ask it to continue.
- **Drafter and reviewer.** `/apply` writes the CV and cover letter, then sends both, with the posting, to a second agent that researches the company and returns edits. The drafter applies those edits and checks company-specific claims again before keeping them.
- **Compiled PDF check.** The CV is built with `lualatex` and must be exactly two pages. The cover letter is built with `xelatex` and must be exactly one page, with the signature on that page. Orphaned job titles and mismatched bullet fonts are treated as failures and edited until the PDF is clean.
- **Relevance-weighted cutting.** When a CV runs past two pages, lines are scored by relevance to the posting, whether the point appears elsewhere, and whether the cover letter depends on them. The lowest-scoring line goes first.
- **Danish job search.** `/scrape` runs the queries in `search-queries.md`, skips jobs already seen or already tracked, and sorts what remains by a quick fit rating.
- **Competency expansion.** `/expand` reads the documents folder and public URLs already in the profile (GitHub, portfolio, Kaggle, Google Scholar, and similar), looks up syllabi for named courses, and appends new skills only after you confirm them.
- **Learning plan.** `/upskill` compares the profile with the tracker, or with one posting URL, and writes a gap heatmap plus a study plan whose links come from web search.
- **Reset with a typed confirmation.** `/reset` lists what it will clear and waits until you type `RESET`.

## Tech stack

| Piece | Role |
| --- | --- |
| [Claude Code](https://claude.com/claude-code) | Runs the slash commands, skills, and the optional research agent |
| LaTeX (`lualatex`, `xelatex`) | CV via [moderncv](https://ctan.org/pkg/moderncv) (banking style); cover letter via `cover.cls` and `fontspec` |
| Python 3 (standard library) | `salary_lookup.py`. `openpyxl` is needed only for the Excel converter |
| TypeScript on [Bun](https://bun.sh) | Portal CLIs (`@bunli/core`, `zod`, and `node-html-parser` on three of the four). The command modules are not in this tree; see below |
| Markdown, CSV, JSON | Profile, search queries, application tracker, salary data, scrape state |

No environment variables are read anywhere in this repository.

## Repository layout

```
ai-job-search/
├── CLAUDE.md                     # Candidate profile, workflow rules, verification checklist
├── SETUP.md                      # Install notes, salary setup, LaTeX troubleshooting
├── .claude/
│   ├── commands/
│   │   ├── setup.md              # /setup
│   │   ├── apply.md              # /apply
│   │   ├── expand.md             # /expand
│   │   └── reset.md              # /reset
│   ├── skills/
│   │   ├── job-application-assistant/
│   │   │   ├── SKILL.md
│   │   │   ├── 01-candidate-profile.md
│   │   │   ├── 02-behavioral-profile.md
│   │   │   ├── 03-writing-style.md
│   │   │   ├── 04-job-evaluation.md
│   │   │   ├── 05-cv-templates.md
│   │   │   ├── 06-cover-letter-templates.md
│   │   │   └── 07-interview-prep.md
│   │   ├── job-scraper/          # /scrape skill and search-queries.md
│   │   └── upskill/              # /upskill skill
│   ├── agents/
│   │   └── gemini-research-expert.md
│   └── settings.local.json       # Claude Code permission allow-list
├── .agents/skills/               # Danish portal skills (see caveat below)
│   ├── jobindex-search/
│   ├── jobbank-search/           # Akademikernes Jobbank
│   ├── jobdanmark-search/
│   └── jobnet-search/
├── cv/main_example.tex           # moderncv master template
├── cover_letters/
│   ├── cover.cls
│   └── OpenFonts/fonts/          # Lato and Raleway
├── documents/                    # Source material for /setup and /expand
├── job_scraper/                  # seen_jobs.json is created here on first scrape
├── job_search_tracker.csv        # Header row for applications
├── upskill/                      # Markdown reports from /upskill
├── salary_lookup.py
└── tools/
    ├── convert_salary_excel.py
    └── README_SALARY_TOOL.md
```

`.claude/commands/` holds four slash commands: `/setup`, `/apply`, `/expand`, and `/reset`. `/scrape` and `/upskill` are skills. Their `SKILL.md` files list those phrases as the trigger.

`gemini-research-expert` is an optional agent. Its instructions run research through `gemini -p "..."`, which needs the Gemini CLI on your `PATH`. `/apply` and `/scrape` leave this agent unused.

`.gitignore` lists `.agents/` (the comment says "agent usage logs"). The skill trees are already tracked, so they stay in the clone. A new untracked file under `.agents/` is ignored unless you force-add it.

## Prerequisites

- [Claude Code](https://claude.com/claude-code). `SETUP.md` installs it with `npm install -g @anthropic-ai/claude-code`. You need an Anthropic API key or a Claude subscription. See the [Claude Code docs](https://docs.anthropic.com/en/docs/claude-code).
- A LaTeX distribution with `lualatex`, `xelatex`, `moderncv`, and `fontspec`: [TeX Live](https://tug.org/texlive/) or [MiKTeX](https://miktex.org/). On Debian or Ubuntu, `texlive-full` covers this. The CV is compiled with `lualatex` because `pdflatex` often fails on current MiKTeX with `fontawesome5` font-expansion errors. The cover letter needs `xelatex` because `cover.cls` loads `fontspec`.
- Python 3, only if you want salary lookup or the Excel converter. `salary_lookup.py` uses the standard library. `SETUP.md` asks for 3.10 or newer.
- [Bun](https://bun.sh), only if you are working on the portal CLIs. `/scrape` uses web search.

## Setup

### 1. Fork and clone

```bash
gh repo fork MadsLorentzen/ai-job-search --clone
cd ai-job-search
```

Or fork on GitHub and clone your fork.

### 2. Start Claude Code and build your profile

```bash
claude
```

Then, inside Claude Code:

```text
/setup
```

`/setup` scans `documents/` first and offers three paths.

| Path | When to use it | What happens |
| --- | --- | --- |
| A. Documents folder | You have a CV, LinkedIn export, diplomas, references, or past applications | Reads `documents/`, checks dates and titles across files, and proposes additions. Conflicts are resolved one by one. Safe to re-run. |
| B. Single CV | You have one CV to paste or `@`-mention | Extracts the structured fields and asks for the gaps |
| C. Interview | You are starting from scratch | Walks through identity, education, experience, skills, behaviour, goals, references, and search queries |

`/setup --section <name>` jumps to one interview section. `/setup --section search` rewrites the search queries without rebuilding the whole profile, and it suggests role titles your history might also fit.

Files that end up populated. Numbered files live in `.claude/skills/job-application-assistant/`:

| File | Contents |
| --- | --- |
| `CLAUDE.md` | Profile summary plus the workflow rules (the rules stay) |
| `01-candidate-profile.md` | Education, experience, skills, publications, awards, references |
| `02-behavioral-profile.md` | Assessment or answers turned into traits |
| `04-job-evaluation.md` | Your strong, moderate, and weak skill areas, and career goals |
| `05-cv-templates.md` | Role-specific profile statements |
| `07-interview-prep.md` | STAR examples (Path A leaves stubs for you to finish) |
| `cv/main_example.tex` | Name, contact details, and recent entries in the LaTeX template |
| `.claude/skills/job-scraper/search-queries.md` | Site queries and commute tiers for `/scrape` |

Path A can also append observed patterns to `03-writing-style.md` and `06-cover-letter-templates.md`. The existing rules in those two files stay as they are.

Drop source files in before Path A. Layout and accepted formats are in `documents/README.md`:

```text
documents/
├── cv/                 # PDF or .tex
├── linkedin/           # LinkedIn "Save to PDF"
├── diplomas/           # PDF
├── references/         # PDF, .txt, or .md
└── applications/
    └── <company>_<role>/
        ├── job_posting.md
        ├── cover_letter.tex
        ├── cv_draft.tex
        └── outcome.md
```

Path A reads PDF, LaTeX, Markdown, and plain text. Export a `.docx` file or a scanned image to a text PDF before you add it.

`SETUP.md` still describes the earlier two-path flow (CV import, then interview). The three-path flow above is the one in `.claude/commands/setup.md`. `SETUP.md` is still the right place for the install commands, the salary file, and LaTeX troubleshooting.

### 3. Optional: salary data

`/apply` runs `python salary_lookup.py "<Company>" --json` when `salary_data.json` exists in the repo root, and adds `--city` when the posting names one. If the file is missing, the script exits with an error and the workflow skips the benchmark.

Create the file by hand, or convert a spreadsheet:

```bash
pip install openpyxl
python tools/convert_salary_excel.py path/to/salary-data.xlsx \
  --source "My Salary Data 2025" \
  --baseline 100 \
  --baseline-desc "Index 100 = median salary"
```

The converter looks for a Company or Firma column, an optional City or By column, and pairs count and index columns when the headers match. Flags: `--output`, `--source`, `--baseline`, `--baseline-desc`. The JSON shape, including a hand-written example, is in `tools/README_SALARY_TOOL.md`. `salary_data.json` is gitignored.

## Configuration

Nothing in the Python scripts, the TypeScript helpers, or the Claude instructions reads environment variables. There are no API keys to set for the pieces that ship here. The portal helpers call public endpoints.

Configuration is the markdown and data files. Numbered files live in `.claude/skills/job-application-assistant/`:

| File | What you change |
| --- | --- |
| `CLAUDE.md` | Name, education, experience, skills, goals, deal-breakers |
| `.claude/skills/job-application-assistant/01-candidate-profile.md` | Structured CV data |
| `02-behavioral-profile.md` | Behavioural assessment or self-description |
| `03-writing-style.md` | Tone rules (leave these unless you mean to change the house style) |
| `04-job-evaluation.md` | Score bands, skill areas, motivation filters |
| `05-cv-templates.md` | Profile-statement templates and LaTeX tailoring rules |
| `06-cover-letter-templates.md` | Letter structure |
| `07-interview-prep.md` | STAR examples and interview questions |
| `.claude/skills/job-scraper/search-queries.md` | Queries, sites, and commute tiers |
| `salary_data.json` | Your salary table, created locally and gitignored |
| `job_search_tracker.csv` | Applications you choose to track |
| `documents/` | Source files for `/setup` and `/expand` |

`.claude/settings.local.json` allow-lists the `job-application-assistant` skill and `python`, `python3`, `curl`, and `bun` shell calls. It holds no secrets.

### Files Git ignores

`.gitignore` excludes generated and personal files, including:

- `salary_data.json`
- `job_scraper/seen_jobs.json` and `job_scraper/*.md`
- `cv/main_*.tex` except `cv/main_example.tex`, and `cover_letters/cover_*.tex` / `Cover_*.tex`
- compiled PDFs and LaTeX aux files
- `upskill/*.md`
- the contents of `documents/cv`, `documents/linkedin`, `documents/diplomas`, `documents/references`, and `documents/applications` (the `README` and `.gitkeep` files stay)
- `node_modules/` and `bun.lock`

`job_search_tracker.csv` is listed in `.gitignore` and is also already tracked (header row only). Git continues to see edits to a tracked file, so filled-in rows can still be committed. Treat the sheet as personal.

## Running it locally

Open the repo in Claude Code (`claude` from the repository root) and use the commands below. You can also ask in plain language. The `job-application-assistant` skill answers "evaluate this posting", "write a CV for [company]", "write a cover letter", and "help me prepare for an interview" as single steps.

### Search

```text
/scrape
/scrape data science
/scrape broad
```

The skill reads `job_scraper/seen_jobs.json` (it creates `{"seen": {}}` if the file is missing), `job_search_tracker.csv`, and `search-queries.md`. It runs the top three query categories through web search. `broad` runs every category. A focus phrase prefers the matching category and adds a few queries for that phrase.

For each promising hit it fetches the page and keeps title, company, location, date, URL, a short requirements note, and the deadline. It skips anything already in `seen_jobs.json` or already listed in the tracker, and anything outside the commute tiers or past its deadline. New jobs are labelled high, medium, or low match and shown in a table. Choosing a row number starts the `job-application-assistant` workflow: fit evaluation, then a CV and cover letter if you agree. That skill's full workflow also drafts interview notes from `07-interview-prep.md`. Applying can append a tracker row. `/apply` adds a reviewer pass and the PDF inspection, and it finishes when the letter and CV are presented.

The template queries target `jobindex.dk` and `linkedin.com/jobs`. The site list in the same file also names `karriere.dk`, `jobfinder.dk`, and `akademikernes.dk`. Placeholder tokens such as `[YOUR_CITY]` stay until `/setup` replaces them. The date filter keeps postings from the last 14 days.

### Apply

```text
/apply https://www.jobindex.dk/jobannonce/h0000000/example-role
```

If a portal blocks the fetch, paste the posting:

```text
/apply
<paste the full job description>
```

`/apply` then:

1. Reads the URL, or uses the pasted text. It records company, role, department, location, and whether the posting is Danish or English.
2. Scores the posting with `04-job-evaluation.md` and `01-candidate-profile.md`. Salary lookup runs only when `salary_data.json` is present. You are asked whether to continue.
3. Writes `cv/main_<company>.tex` (always English, moderncv banking, aimed at two pages) and `cover_letters/cover_<company>_<role>.tex` (the posting's language, `cover.cls`, aimed at one page). A named recipient is used when the posting has one. Otherwise the letter uses "Dear Hiring Manager" or the equivalent in the posting's language. Mentions of agentic coding name Claude Code.
4. Sends the drafts and the posting to a reviewer agent. The reviewer researches the company and returns exact text replacements plus notes on missed keywords, company angles, passive phrasing, and tone. It is instructed to stay inside the real profile.
5. Applies the edits. Company claims are checked with a fresh fetch or search before they are kept.
6. Compiles both documents and reads the PDFs. It repeats until the CV is exactly two pages, with no job title stranded at the bottom of a page, and the cover letter is exactly one page, signature included, bullets in Raleway Medium. Build artefacts (`.aux`, `.log`, `.out`) are deleted afterwards.
7. Prints the verification checklist from `CLAUDE.md` and a short list of tailoring decisions.

Compile the same way yourself:

```bash
cd cv && lualatex -interaction=nonstopmode main_<company>.tex
cd ../cover_letters && xelatex -interaction=nonstopmode cover_<company>_<role>.tex
```

The example CV compiles with `lualatex main_example.tex` from `cv/`. Fonts for the letter are already in `cover_letters/OpenFonts/fonts/` (Lato and Raleway). Compile from `cover_letters/` so `cover.cls` resolves `OpenFonts/fonts/`. `cover_letters/OpenFonts/cover.cls` is the same class with font paths relative to the `OpenFonts` directory and slightly wider margins. The workflow uses `cover_letters/cover.cls`.

### Expand, upskill, reset

```text
/expand
/upskill
/upskill https://example.com/job/123
/reset profile
/reset documents
/reset all
```

`/expand` only appends. It scans `documents/cv`, `documents/linkedin`, `documents/diplomas`, and `documents/references`, then any GitHub username or other URL already stored in `01-candidate-profile.md`. Named courses and certifications get a web lookup of the syllabus as well as an inferred competency list. You choose `all`, `review`, or `skip` before anything is written. New lines go to `01-candidate-profile.md` and `02-behavioral-profile.md`, each with a source tag. Inferred behavioural notes are labelled as inferred.

`/upskill` with no URL reads every row of `job_search_tracker.csv`. In that mode a gap's score sums `(100 - fit_rating) / 100` across the jobs that mention the skill, so a weaker fit counts for more. A URL fetches that single posting and uses only that posting. The report is a heatmap (critical, high, medium, low) and a learning plan for the critical and high gaps. Medium gaps join the plan when fewer than five gaps are found in total. Each planned gap gets two or three resources taken from web search, plus a suggested study order. Aggregate reports go to `upskill/report-YYYY-MM-DD.md`. A single posting goes to `upskill/report-YYYY-MM-DD-<company>-<role>.md`. A later aggregate report diffs against the previous one. Report files are gitignored.

`/reset profile` replaces `01-candidate-profile.md` and `02-behavioral-profile.md` with blank templates, clears profile statements in `05-cv-templates.md`, and clears STAR examples in `07-interview-prep.md`. It leaves `03-writing-style.md`, `04-job-evaluation.md`, `06-cover-letter-templates.md`, `CLAUDE.md`, and `cv/main_example.tex` as they are. `/reset documents` deletes files you added under the `documents/` subfolders and keeps `documents/README.md`. Both wait until you type `RESET`.

### Salary lookup

```bash
python salary_lookup.py "Novo Nordisk"
python salary_lookup.py "Ørsted" --city "Fredericia"
python salary_lookup.py "COWI" --json
python salary_lookup.py --list-all
```

Matching folds case, strips legal suffixes such as A/S and ApS, and treats æ, ø, and å as their ASCII spellings. A city filter drops rows whose city does not contain the text you passed.

### Tracker columns

`job_search_tracker.csv` header:

```text
date,company,sector,role,role_type,channel,status,contact_person,fit_rating,notes,cv_file,cover_letter_file,source
```

`/scrape` reads it to skip roles you have already recorded, and can append a row when you decide to apply. `/upskill` in aggregate mode reads `role`, `sector`, `notes`, `fit_rating`, and `source`.

## Danish job portal tools

Four skills under `.agents/skills/` document live search of Danish boards. Each has a `SKILL.md`, a `cli/README.md`, a Bun `package.json`, `src/cli.ts`, `src/helpers.ts`, and `cli/tests/helpers.ts`. Jobindex and Jobbank also have a `url-reference.md`. No authentication is implemented. The helpers retry HTTP 429 and 5xx responses.

| Skill | Board | Commands described in the skill | What the committed TypeScript actually contains |
| --- | --- | --- | --- |
| `jobindex-search` | [jobindex.dk](https://www.jobindex.dk/) | `search`, `detail` | JSON fetch against `https://www.jobindex.dk` and HTML fetch for a listing page |
| `jobbank-search` | [jobbank.dk](https://jobbank.dk) (Akademikernes Jobbank) | `search`, `detail` | RSS fetch of `https://jobbank.dk/job/rss` and description parsing |
| `jobdanmark-search` | [jobdanmark.dk](https://jobdanmark.dk) | `search`, `detail`, `categories`, `autocomplete`, `locations` | GET and POST helpers for `https://jobdanmark.dk` |
| `jobnet-search` | [jobnet.dk](https://jobnet.dk) (STAR) | `search`, `detail`, `occupations`, `suggestions` | JSON fetch against `https://jobnet.dk/bff` |

Each `src/cli.ts` imports those commands from `./commands/*.js`. That `commands/` directory is not in the repository, so `bun run src/cli.ts` cannot start. `bun install` only installs the packages named in `package.json` (`@bunli/core`, `@bunli/utils`, `zod`, and, except for Jobnet, `node-html-parser`).

The skill files invoke the CLIs as `bun run skills/<name>/cli/src/cli.ts`. In this repository those files live under `.agents/skills/<name>/cli/`.

`/scrape` uses web search and page fetch, as described above. Per-flag documentation (Jobindex `jobage` and `sort`, Jobbank type and region codes, Jobdanmark categories, Jobnet region and ESCO filters) lives in each `SKILL.md` and `cli/README.md`. Those docs describe the intended CLI. They become a running interface once the command modules are added.

Install dependencies, if you are extending the tools:

```bash
for tool in jobbank-search jobdanmark-search jobindex-search jobnet-search; do
  (cd .agents/skills/$tool/cli && bun install)
done
```

Remember the `.gitignore` rule for `.agents/` before you expect new files there to show up in `git status`.

## Tests

Each portal `package.json` defines:

```bash
bun test --timeout 30000
bunx tsc --noEmit
```

`cli/tests/helpers.ts` spawns `src/cli.ts` and parses JSON from its stdout. The repository contains that helper and no `*.test.ts` files, so `bun test` has no cases to run. `typescript` is a dev dependency from `bun install`. A typecheck stops on the missing `./commands/*.js` imports. The Python tools have no automated tests. Run `salary_lookup.py` and `tools/convert_salary_excel.py` directly.

## Customising the templates

The CV class is moderncv, banking style, blue. The letter class is `cover.cls` (Lato for the body defaults, Raleway for the letter text). Replace either, then update `05-cv-templates.md` and `06-cover-letter-templates.md` so later `/apply` runs follow the new structure.

If you are outside Denmark, point `search-queries.md` at the boards you use, or add a portal skill beside the four Danish ones. `/apply` reads the profile, the posting, and the LaTeX templates.

## Tips

Detail in the profile is what makes a tailored letter possible. For each role, name the projects, the tools, and the outcome. "Built a churn model in Python with scikit-learn" gives the drafter a concrete line to reframe. A bare "Python" leaves it with a label.

During `/setup`, say what kind of work you want more of and what you want to avoid. Those answers feed the motivation filter in `04-job-evaluation.md` and the role suggestions in the search section. The search interview will also propose adjacent titles (for example a technical consultant or team-lead search) from skills you have already described.

## Acknowledgements

- [Mikkel Krogholm](https://github.com/mikkelkrogsholm) ([skills repo](https://github.com/mikkelkrogsholm/skills)) for the job-search CLI skills
- Built with [Claude Code](https://claude.com/claude-code) by [Anthropic](https://anthropic.com)

## Licence

MIT. Copyright (c) 2026 Mads Lorentzen. See `LICENSE`.
