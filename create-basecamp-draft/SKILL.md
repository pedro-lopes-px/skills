---
name: create-basecamp-draft
description: Convert a plain-text meeting summary into a Basecamp Message Board draft with native person mentions, then open the draft in the browser for manual editing and publishing.
---

# Create Basecamp Draft

Create an unpublished Basecamp Message Board draft from a plain-text meeting summary.

The user must manually review and publish the draft in Basecamp. Never publish it automatically.

## Required workflow

1. Obtain the meeting summary from:
    * Text included in the user's prompt.
    * A local `.txt` or `.md` file supplied by the user.
    * A file path supplied by the user.

2. Identify the target Basecamp project.
    * If the user supplied a project name or ID, use it.
    * If no project was supplied, ask the user which project to use.
    * Do not guess between multiple similarly named projects.

3. Verify Basecamp authentication:

    ```bash
    basecamp auth status
    ```

    If authentication fails, stop and explain that the user must authenticate the CLI first. Do not attempt to handle credentials, passwords, or OAuth tokens manually.

4. Fetch people who can be mentioned in the project, and resolve the author (yourself):

    ```bash
    basecamp people pingable --in "<PROJECT>" --json
    basecamp people show me --json
    ```

    The `pingable` list never includes the authenticated user, because Basecamp does not let you ping yourself. `people show me` returns the author's own `name` and `attachable_sgid` — keep both so the author can still be mentioned.

5. Match names in the summary against the returned Basecamp people.

    Matching rules:

    * Match exact full names first.
    * Match configured aliases only when unambiguous.
    * Do not resolve a first name when multiple project members share it.
    * Do not tag companies, products, features, or names appearing only in URLs or email addresses.
    * Do not invent people who are not present in the Basecamp people response.
    * Never silently choose between ambiguous matches.
    * **Author fallback:** If a name has no match in `pingable`, check whether it is the author (from `people show me`) — the author is absent from `pingable` precisely because they cannot ping themselves. If it matches, mention them using the author's `attachable_sgid` from `people show me`. If for any reason no sgid is available, fall back to plain `@Name` text rather than dropping the reference.
    * Otherwise, preserve names that cannot be resolved as ordinary text.

6. Before creating the draft, show the user a concise confirmation:

    ```text
    People to mention:
    ✓ Nathalie Leroy
    ✓ David Alvarez-Debrot
    ⚠ Alex — ambiguous: Alex Johnson / Alex Silva
    ```

    Include unresolved or ambiguous names. Ask the user to confirm before creating the draft.

7. Identify the client company name whenever the summary indicates a client is present (e.g. an attendee group labelled "Client", or an external company in the discussion).

    * **Always confirm the client's company name with the user before creating the draft** — ask explicitly (e.g. "What's the client's company name?"), even when you can infer it from Basecamp people profiles or the summary. Offer your best inferred guess in the question, but wait for the user's answer.
    * "Client name" means the **company / organisation name** (e.g. `icCube`), not a person's name.
    * Carry the confirmed company name into step 9 so it replaces both the "Client" group label and every generic "client" reference in the body.
    * If no client is present in the summary, skip this step.

8. Convert confirmed people into deterministic Basecamp mention syntax:

    ```markdown
    [@Full Name](mention:PERSON_ATTACHABLE_SGID)
    ```

    Use the person's `attachable_sgid` from the `people pingable` response (or from `people show me` for the author, per the step 5 author fallback).

    * **Mention every occurrence, everywhere in the body.** If a confirmed person's name appears anywhere in the body — not just in Attendees or Action Items, but in the TL;DR, Topics of Discussion, notes, or any other section — replace that name with their mention. A first name that resolves unambiguously to a confirmed person (e.g. "David" → David Alvarez) counts as an occurrence and must be mentioned too.
    * Apply the same matching safeguards as step 5: do not mention an ambiguous or unresolved first name, and do not tag names that only appear in URLs or email addresses.
    * **Exception — Action Items headings (step 10):** where a name already serves as the owner heading, do not additionally mention that same person inside that section's own action bullets.

9. Treat the message as **client-facing communication**. Preserve the substance of the summary, but shape it as a clean external post.

    * Do not rewrite or summarise the substantive content unless the user explicitly asks.
    * Only replace confirmed person references with Basecamp mention syntax — but replace **all** of them, in every section of the body, per step 8.
    * Preserve headings, lists, paragraphs, links, and line breaks of the substantive content.
    * Do not modify the source file.
    * Do not add mentions to the message title unless explicitly requested.
    * **Strip internal-only / meta scaffolding.** Remove sections written for the person publishing rather than the client — e.g. "Optional internal Slack blurb", "Open questions / ambiguities", drafting notes, placeholder reminders, or any aside about the transcript itself. The published message must read as plain client-facing communication with no editorial notes.
    * **Label internal attendees as "Pixelmatters"**, not "Internal" (or similar).
    * **Never use the generic word "Client" in the output.** Label the client group with the client's company name (see step 7), and replace every other generic "client" reference in the body with that company name. Examples: `Client: Nathalie, David` → `icCube: Nathalie, David`; "from the client list" → "from the icCube list"; "unblock João on client inputs" → "unblock João on icCube inputs". Only replace generic references to the client organisation — do not touch unrelated uses of the word (e.g. "client-side rendering").

10. Format **Action Items** so each owner is named once, as a mention, and never duplicated:

    * The owner's name is the section heading, rendered as a mention (e.g. the heading `David` becomes the mention).
    * The bullets under that heading list only the actions — plain text, no mention and no repeated name prefix.
    * Do not both title a section with a name and repeat that person's mention inside its own bullets.

11. Always end the message with a sign-off, on its own two lines:

    ```text
    Thanks!
    @[author] & the Pixelmatters Team
    ```

    Render `@[author]` as a mention of the author using the `attachable_sgid` from `people show me`.

12. Determine the message title.

    * Use a title supplied by the user.
    * Otherwise derive a concise title from the meeting context and date.
    * If the title would be uncertain, ask the user before creating the draft.

13. Create the message as a draft, never as an active message. The body is easiest to pass on stdin (use `-` as the body argument) to avoid shell-escaping issues, and `--no-subscribe` avoids notifications:

    ```bash
    cat body.md | basecamp messages create \
        "<TITLE>" \
        - \
        --draft \
        --no-subscribe \
        --in "<PROJECT>" \
        --json
    ```

14. Capture the created draft's response.

    Inspect the JSON for a browser URL using fields such as:

    ```text
    app_url
    url
    html_url
    ```

    Do not assume a URL field exists without inspecting the response.

15. Open the draft in the user's browser if a URL is available:

    ```bash
    open "<DRAFT_URL>"
    ```

16. Report clearly that:

    * The draft was created.
    * It has not been published.
    * The user can edit it in Basecamp.
    * The user must manually publish it.

## Safety rules

* Never use a command that publishes the message.
* Never run `basecamp messages publish`.
* Never create the message without `--draft`.
* Never send notifications intentionally.
* Never update or overwrite a published message. (Revising a draft this skill created — still unpublished — is fine when the user asks.)
* Never modify the original transcript or summary file.
* Never use fuzzy matching when it could tag the wrong person.
* Never expose OAuth tokens or credentials.
* If the Basecamp CLI is not authenticated, stop before creating anything.
* If the project cannot be identified confidently, ask the user.
* If a person match is ambiguous, leave it untagged until the user resolves it.
* If a client is present, confirm the client's company name with the user, and never leave a generic "Client" label or generic "client" reference in the output.

## Optional aliases

If the project contains recurring name variations, use a local configuration file only when the user has explicitly provided or approved it.

Example:

```yaml
people:
    Nathalie Leroy:
        aliases:
            - Nathalie
            - Nathalie L.

    David Alvarez-Debrot:
        aliases:
            - David Alvarez
            - David
```
