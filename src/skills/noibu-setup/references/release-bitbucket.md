# Release reporting: Bitbucket Pipelines (Step 6)

Works on **Bitbucket Cloud** only (bitbucket.org). Bitbucket Data Center and Server have no Pipelines: if that's what they run, stop and say so.

You do every step. The user's part: signing in if the browser isn't (section 0), one click to confirm the details (section 2), the merge and first pipeline run, and turning on 2-step verification if Bitbucket asks for it (an account security setting you never change). Nothing else.

**Prerequisites (check them yourself):** Claude in Chrome (or the built-in browser) signed in to bitbucket.org, then write access to the repo, in that order (section 0); the repo has at least one commit and a default branch. Work in the browser session. Don't use the Bitbucket API, app passwords, or the git CLI for the change.

## 0. Sign-in, then access (before anything else)

Check these in this order, before asking the user anything. Never judge access while signed out: signed out, a public repo shows "Access level: Read" to everyone, which says nothing about the user's real access.

1. **Signed in?** Open `https://bitbucket.org/account/settings/` in the browser. Any signed-in user can open it; signed out, Bitbucket sends you to the Atlassian login page → the sign-in check-in below. Don't use a repo's `/admin` pages for this: they only open for repo admins, so a signed-in user with write access would look signed out.
2. **Write access?** Only once signed in: open the repo (`https://bitbucket.org/<workspace>/<repo>`) and read the access level it shows. Write or Admin → go on. Read → no write access (Blockers).

**Sign-in check-in:** leave the login page open (if Claude's browser panel is hidden, ask them to bring it up first) and ask one `AskUserQuestion`: "Sign in to Bitbucket in the browser window I've opened." Options "I've signed in" and "Skip for now". "I've signed in" → repeat item 1. "Skip for now" → record "Add the Noibu release step to <workspace/repo>" as a to-do for the user and go to Step 7. Never type their credentials.

## 1. Domain

Use the domain UUID from Step 2 (the UUID, not the `numId`). Only if missing: resolve with `noibu_get_domain` / `noibu_list_domains`, asking which if several. Never ask the user to look it up.

## 2. Confirm the details (one click)

Work out sensible defaults instead of asking: the repo from Step 5; the trigger branch = the repo's default branch (`main` if you can't tell); the component = `frontend` (or what the business context or repo name suggests, e.g. `checkout`); a pull request. Then one `AskUserQuestion`:
- Say why first, in one line of chat: "Next, Noibu should know whenever your site gets an update. Then, if errors spike or sales dip, you can see straight away whether an update caused it."
- header: "Updates"
- question: "To turn this on, I'll add one small file to <repo>. It's a one-time change your team can review before it goes in. After that, every update your team ships is logged in Noibu automatically."
- options: "Sounds good" ("Set it up") and "Change something" ("Which branch, the label, or skip the review")
- Keep the words the user reads plain: no "push", "main", "pipeline", "webhook", "release event" or "pull request" in the message or the question. Name the branch only if it isn't the repo's default.

"Sounds good" → go ahead with no more questions. "Change something" → ask only for what they want changed, in one message, then go ahead. (If they ask: every update to the branch is logged as a successful release; Noibu doesn't check that the deploy itself succeeded.)

If, under "Change something", they want deployment-based reporting instead of every push, clarify which deployment pipeline first.

## 3. Build and validate the pipeline

File: `bitbucket-pipelines.yml` at the repo root. A repo can have only **one** of these, so this is often an edit.

- File doesn't exist → create it.
- File exists → keep everything in it. Add the Noibu step to the pipeline for the selected branch (`pipelines > branches > <branch>`), after any existing build or deploy steps.
- **Gotcha:** if the file only has a `default` pipeline, adding a branch pipeline for that branch stops `default` from running there. Copy the existing `default` steps into the new branch pipeline so current behaviour is unchanged, then append the Noibu step.
- A Noibu release step already exists → inspect it; don't duplicate reporting or overwrite unrelated work.

Configure:

- Trigger: push to the selected branch.
- POST to `https://webhook.noibu.com/release_webhook/<DOMAIN_UUID>`.
- JSON payload (all strings):
  - `component`: the confirmed component name
  - `title`: `"$BITBUCKET_REPO_FULL_NAME release $BITBUCKET_BUILD_NUMBER"`
  - `description`: the head commit message (`git log -1 --pretty=%B`)
  - `status`: `"success"`
  - `version`: `$BITBUCKET_COMMIT`
  - `release_time`: `date -u +%Y-%m-%dT%H:%M:%SZ`
- Build the JSON with `jq -n --arg ...`. Never interpolate the commit message into the JSON string or into executable shell code; pass it through a variable.
- `curl --fail --max-time 30` so HTTP errors fail the step with a bounded timeout.
- Use an image with git, curl and jq, or install what's missing in the script. Check; don't assume.
- Keep the step minimal. Write script commands as YAML block scalars (`- |`) so colons and quotes don't break the YAML. Name the step "Report release to Noibu".

Validate the YAML with a parser, test the script with a commit message full of quotes, backticks and `$( )`, and review the diff. Don't paste the full YAML into chat.

## 4. Make the change in the Bitbucket UI

- Edit via **Source → bitbucket-pipelines.yml → Edit**, or create via **Source → … → Add file**.
- The web editor auto-indents typed YAML. Insert content in a way that preserves indentation (e.g. clipboard paste, after confirming the clipboard really holds the YAML). Check the indentation in the editor afterwards.
- Commit message: "Add Noibu release event webhook".
- PR → tick **Create a pull request for this change** to commit to a new branch, and create the PR against the default branch. Leave it unmerged.
- Direct commit (only if explicitly chosen) → commit to the default branch, subject to repo protections.
- Re-read the saved file (raw view of the branch) and confirm it matches the validated YAML exactly (e.g. compare checksums).

## 5. Merge, enable Pipelines, verify

1. Share the link in one line ("Here's the one-time change for your team to review: <PR link>"), then ask with `AskUserQuestion`: "Add the one-time Noibu change to <repo> now?" options "Add it now" ("I'll merge it for you") / "My team will review it" ("Merge it when you're ready").
   - **Add it now** → merge the PR in the browser, subject to repo protections. A required approval you can't give → say who needs to approve, record it for Step 8, and continue with Step 7.
   - **My team will review it** → continue with Step 7 now; re-check in Step 8.
2. Once merged, open the repo's **Pipelines** page in the browser. ("Enable Pipelines" under Repository settings → Pipelines → Settings stays greyed out until a valid file is on the default branch; that's expected.)
   - Bitbucket asks for **2-step verification** (first pipeline on this account) → one line: "Bitbucket needs 2-step verification on your account before it'll run pipelines. Turn it on in the window I've opened." Open the prompt's page for them, then one `AskUserQuestion` with "It's turned on" / "Not now" ("Add it to the to-do list"). Never change it yourself. "Not now" → record "Turn on Bitbucket 2-step verification and run the first Noibu pipeline" as a to-do for the user.
   - Then ask "Run it once now to check updates reach Noibu?" ("Yes, run it" / "Not now"). On yes, click **Run initial pipeline** (or **Run pipeline**), select the trigger branch, and run it.
   - Not a repo admin and Pipelines can't be enabled → say who needs to do it and record it for Step 8.
3. Watch the run until "Report release to Noibu" finishes.
   - Green → call `noibu_list_releases` for the domain UUID and confirm an entry with the component, status success, title `<workspace>/<repo> release <build number>` and the commit SHA as version. Report it in one line.
   - Red → open the step log, diagnose (wrong UUID or URL, indentation, missing tool in the image), fix through the same PR flow, and re-run.
4. Don't claim delivery unless `noibu_list_releases` shows it. Record the PR link and release state in the ledger.

## Blockers

- Not signed in to Bitbucket in the browser → section 0's sign-in check-in. Never type their credentials.
- Browser control unavailable (in Claude Code: Claude in Chrome isn't connected), no write access, Data Center/Server, or a repository policy blocks the change → stop this step, say exactly what's blocking and who can fix it, record it for Step 8, and move on. Don't guess or bypass protections.

Diagnose and fix these yourself:

| Symptom | Likely cause | What you do |
|---|---|---|
| "Enable Pipelines" greyed out | No valid `bitbucket-pipelines.yml` on the default branch yet | Get the PR merged (section 5), check again |
| Run never starts | File not on trigger branch, or initial pipeline not run | Confirm the merge, then run the initial pipeline on the trigger branch |
| Other pipelines stopped on that branch | File had only `default`; a branch pipeline replaces it | Copy the `default` steps into the branch pipeline |
| Step fails with an HTTP error | Wrong domain UUID or mistyped URL | Fix the UUID and URL in the file |
| Messy indentation | Editor auto-indented typed text | Re-insert by pasting and re-verify the saved file |
