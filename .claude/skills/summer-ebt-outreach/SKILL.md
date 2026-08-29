---
name: summer-ebt-outreach
description: Use when the user wants to research a state's Summer EBT (SUN Bucks) program from a list of URLs and/or attached files and produce plain-language, cited Q&A answers for family-facing outreach materials, using only those sources. Triggers on requests like "research Summer EBT for <state>", "pull SUN Bucks info from these links", or "answer these outreach questions about <state>'s Summer EBT program from these URLs/files".
---

# Summer EBT Outreach Research

Turn a state's Summer EBT (a.k.a. SUN Bucks) source material into short,
plain-language, cited Q&A for family-facing outreach materials — as a chat
reply and as a durable Markdown file per state.

## 1. Collect inputs

Require, before doing any research:

- **State name.** Do not guess it.
- **One or more sources** — a source is a URL and/or a file attached to
  the prompt (PDF, DOCX, plain text, etc.). At least one source of either
  kind must be present. Do not invent URLs or proceed on an attachment
  alone without a state name.

If the state name or all sources are missing, ask the user for them
before proceeding.

**Questions to answer:**
- If the user supplies their own list of questions, use exactly that list.
- Otherwise, read the default questions from
  `.claude/skills/summer-ebt-outreach/default-questions.md` (the bulleted
  lines in that file — the user can edit that file directly to change the
  defaults, without touching this file).
- If `default-questions.md` is missing or has no bullet lines, fall back
  to this list and tell the user the defaults file wasn't found:
  - What is the name of the state?
  - What is the name of the program in the state? Is it called "Summer
    EBT" or "SUN Bucks"?
  - Which programs and statuses are used for streamlined certification or
    automatic eligibility?
  - What is the age range for children to be streamline certified or
    automatically eligible?
  - What is the URL for the application?

**Multiple states:** treat one run as covering one state. If the sources
provided obviously cover more than one state, confirm with the user before
splitting the work into separate runs — don't silently merge results from
different states into one file.

## 2. Fetch / read sources

- For each **URL**: use the `WebFetch` tool. If a fetch fails with a
  transient error (network/timeout, 5xx, or similar), retry with backoff —
  up to 3 attempts total, waiting ~2s then ~4s between attempts. A
  definitive failure (e.g. 404 not found) does not need retries. If all
  attempts fail, note that URL as unreachable and move on — don't abort
  the whole run.
- For each **attached file**: read it directly (`Read` tool, or the `pdf`
  / `docx` skill for those formats as appropriate) — do not treat it as a
  URL.

## 3. Answer each question

For every question, using **only** the content gathered in step 2:

- **≤100 words** per answer.
- **Sixth-grade reading level**: short sentences, plain everyday words, no
  unexplained jargon or acronyms.
- **Cite sources inline** at the end of the answer — e.g. `(Source:
  <url>)` or `(Source: <filename>)` — citing every source actually used
  for that answer.
- If none of the provided sources answer a question, say so explicitly
  (e.g. "Not found in the sources provided.") rather than guessing.

**Guardrail — sources only, no exceptions:** Do not use `WebSearch`,
training/general knowledge, memory of other states' programs, or any
URL/file not supplied by the user in this run. Every fact in every answer
must trace back to a source fetched or read in step 2. When a question
can't be answered from those sources, say it wasn't found — never fill the
gap from outside knowledge.

## 4. Reply format

Reply in the chat with every question in order, in this exact format:

```
Q: <question text>
A: <plain language response text with citations>
```

If any source URLs failed to fetch, mention that briefly before or after
the Q&A block.

## 5. Save the output — one file per state, never overwritten

- Derive `<state-slug>` from the state name (lowercase, spaces → hyphens;
  e.g. "New York" → `new-york`).
- Look for an existing `summer-ebt-research/<state-slug>-summer-ebt-qa.md`
  (create the `summer-ebt-research/` directory if it doesn't exist).
- **File doesn't exist:** create it with a header — state name, date
  generated, and the list of sources used (URLs and/or attached
  filenames) — followed by all of this run's `Q:`/`A:` pairs.
- **File already exists:** for each question in this run, check whether a
  matching question (case/whitespace-insensitive match on the question
  text) already has an entry in the file:
  - **Already present** → leave that entry untouched. Tell the user in
    the chat reply that this question was already answered in the file,
    and show them the existing answer.
  - **Not present** → append a new `Q:`/`A:` entry to the end of the
    file, and add any newly-used sources to the header's source list if
    they aren't already listed.
- This file is **append/skip-only**: never edit or remove an existing
  entry. Tell the user the file path after saving.
- This check-then-append isn't atomic — a genuinely simultaneous write
  from two users could still race. That's an accepted, low-probability
  risk for this prototype; don't add file locking or other machinery to
  guard against it.

## 6. Confirm before sharing the file with other users

The research file lives in a shared repository other people can see. After
writing to it in step 5, ask the user whether to share this run's changes
— **in plain, non-technical language, never using words like "commit,"
"push," "repository," or "branch."** Pick the phrasing based on what
actually happened in step 5:

- **New file created (no prior file for this state):** ask something like
  *"Do you want to save this [State] research file so others can see
  it?"*
- **Existing file, new questions appended:** ask something like *"Do you
  want to update the [State] research file with these new answers so
  others can see them?"*
- **Existing file, nothing new appended (every question was already on
  file):** don't ask anything — there's nothing new to share.

If the user says yes, run (adjust the commit message to name the state
and what changed, e.g. "Add Ohio Summer EBT research" or "Update Ohio
Summer EBT research with N new answers"):

```
git add summer-ebt-research/<state-slug>-summer-ebt-qa.md
git commit -m "<Add|Update> <State> Summer EBT research"
git push
```

If `git push` fails (e.g. no upstream configured yet), fall back to
`git push -u origin <current-branch-name>`. If the push still fails for
any reason, tell the user in plain language that the file is saved on
their computer but couldn't be shared yet, and why.

If the user says no, leave the file as an uncommitted local file and say
so — don't commit or push without a yes.

## Example output file structure

```markdown
# <State> Summer EBT / SUN Bucks Research

Generated: <date>
Sources:
- <url or filename>
- <url or filename>

Q: <question>
A: <answer with citation>

Q: <question>
A: <answer with citation>
```
