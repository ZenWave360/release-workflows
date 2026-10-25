# GIT_PUBLISHER_TOKEN: regenerate and set

`GIT_PUBLISHER_TOKEN` lets CI push snapshots to `zenwave360/maven-snapshots` with
[git-maven-publisher](https://github.com/ivangsa/git-maven-publisher). It is one fine-grained personal access
token, stored in two places:

| Where | Used by |
|---|---|
| Org secret of `ZenWave360` (all repositories) | every ZenWave360 project with `publish-git-snapshots: true` |
| Environment `git-publisher-snapshots` of `ivangsa/git-maven-publisher` | git-maven-publisher's own snapshots (that repo is outside the org) |

When the token expires, snapshot publishing fails with an authentication error in all of them. Regenerate it, then
set the new value in both places.

## 1. Regenerate the token

Open https://github.com/settings/personal-access-tokens and click the token
`git-publisher zenwave360/maven-snapshots`, then **Regenerate token**. Pick a new expiration and copy the token
(GitHub shows it only once).

If the token is gone, create it again with **Generate new token** and these settings:

- Token name: `git-publisher zenwave360/maven-snapshots`
- Resource owner: `zenwave360`
- Expiration: 90 days, or a custom date
- Repository access: **Only select repositories**, `zenwave360/maven-snapshots`
- Repository permissions: **Contents: Read and write** (Metadata: Read-only is added automatically). Nothing else.

## 2. Set it in both places

Run in Git Bash or any terminal with `gh` logged in. Each command asks for the value with hidden input; paste the
token there, never on the command line.

```bash
gh secret set GIT_PUBLISHER_TOKEN --org ZenWave360 --visibility all
gh secret set GIT_PUBLISHER_TOKEN --repo ivangsa/git-maven-publisher --env git-publisher-snapshots
```

## 3. Check

The dates must be today (values are never shown):

```bash
gh secret list --org ZenWave360 | grep GIT_PUBLISHER_TOKEN
gh secret list --repo ivangsa/git-maven-publisher --env git-publisher-snapshots
```

Then rerun the last snapshot publication of each repository you care about:

```bash
gh run rerun --repo ivangsa/git-maven-publisher $(gh run list --repo ivangsa/git-maven-publisher --workflow "Build and Publish Snapshots" -L 1 --json databaseId --jq '.[0].databaseId')
gh run rerun --repo ZenWave360/zenwave-sdk $(gh run list --repo ZenWave360/zenwave-sdk --workflow "Build and Publish Snapshots" -L 1 --json databaseId --jq '.[0].databaseId')
```

## Notes

- Branch `gh-pages` of `zenwave360/maven-snapshots` must not block force pushes: every publication replaces the
  branch with one commit, using a force push with lease.
- If the org requires approval for fine-grained tokens, approve the request at
  https://github.com/organizations/zenwave360/settings/personal-access-token-requests after step 1.
