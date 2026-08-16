# Contributing

AzurPilot-Web uses a small trunk-based workflow. `main` is the stable integration line, not a scratch branch.

## Workflow

1. Create or reference an Issue/Stage with a clear scope and Definition of Done.
2. Branch from current `main` using a short-lived topic branch.
3. Open a Draft PR after the first meaningful commit.
4. Run the checks appropriate to the change and record only checks that were actually executed.
5. Resolve actionable review findings and update the PR description to match the final diff.
6. Merge only after acceptance; squash is the default for ordinary Stage/feature PRs.
7. Delete the topic branch after merge.

Do not create a permanent `develop`, `development` or `staging` branch without a separate demonstrated need.

## Branch names

Use a clear prefix:

```text
feat/<topic>
fix/<topic>
docs/<topic>
chore/<topic>
refactor/<topic>
```

## Commits

Use concise Conventional-Commit-style messages, for example:

```text
docs(architecture): define frontend repository boundary
chore(repo): establish pull request governance
feat(settings): add preference editor
```

Avoid vague messages such as `fix`, `update`, `final`, `again`, `test` or `wip` as final commits.

## Main branch rules

Normal development must not push directly to `main`. The repository governance target is:

- pull request required before merge;
- conversation resolution required;
- linear history required;
- force pushes blocked;
- branch deletion blocked;
- no phantom required status check before a stable CI check actually exists.

A second human approval is not required by default for this solo repository. PR review, automated checks when present, and explicit owner acceptance provide the normal gate.

## Merge policy

- squash merge: default;
- rebase merge: allowed when preserving individual commits is useful;
- merge commits: disabled for `main`;
- update branch: enabled when repository settings support it.

## Cross-repository changes

A PR that depends on `AzurPilot-private-Ru` must explicitly state the backend impact and whether the backend change already exists, is coordinated separately, or is deferred.

Do not copy privileged backend logic into this repository to avoid a cross-repo change.

## Security and public-repository hygiene

Never commit:

- real public IPs or private deployment details;
- router credentials;
- API/GitHub/MCP tokens;
- cookies, authorization headers or session data;
- private keys/certificates;
- `.env` contents;
- real webhook URLs;
- unnecessary personal filesystem paths or device identifiers.

Use placeholders such as `<user-domain>`, `<public-ip>` and `127.0.0.1:<backend-port>` in documentation.
