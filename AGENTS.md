# Repository workflow

After implementing a requested change, run the relevant verification. If the
verification succeeds, commit the requested changes and push the current branch
to `origin` without asking for confirmation.

Do not commit or push unrelated user changes. If verification fails, report the
failure and do not push unless the user explicitly asks you to proceed.

For skill changes, start the commit message with the skill name followed by a
colon and a space, then capitalize the subject, for example:
`vml: Make snapshots opt-in`.
