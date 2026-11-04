# ADR-0002: Public GitHub repository

- **Status:** Accepted
- **Date:** 2026-11-04
- **Scope:** GitHub repository visibility. Changes only the "private GitHub repo" assumption in ADR-0001 ("Consequences for the roadmap") and roadmap milestone M0.1

## Context

ADR-0001 planned a private repository with branch protection on `main`. Branch protection is an entry condition for access level 0 (A5), because the cloud session GitHub proxy does not limit which branches can be updated.

The repository uses GitHub Free, and these two requirements conflict:

- GitHub's plan documentation lists protected branches in private repositories as a GitHub Pro feature. GitHub Free offers a full feature set on public repositories and a limited one on private repositories.
- Private repositories need GitHub Secret Protection for repository-level push protection, which M0.1 also relies on to block secrets.

## Decision

Make the repository **public** and keep GitHub Free. Enforcement on `main` takes priority over keeping the repository private.

The `protect-main` ruleset covers the default branch: pull request required, no force push, no deletion.

## Verification

On 2026-11-04, a direct push to `main` was rejected with `GH013: Changes must be made through a pull request`. This meets the first part of the M0.1 acceptance test.

The first push, which created `main`, was accepted. The rule was enforced on the next push to the existing branch, so a branch-creating push does not count as a test of this rule.

## Consequences

- **Everything pushed is public, including git history.** This covers docs, ADRs, the roadmap and notes. Deleting a file later does not remove it from history.
- **No secrets or sensitive lab details in the repo.** Internal addresses, credentials, keys, tokens and exported configs stay out. Secret scanning is a safety net, not the control. Treat any secret that reaches GitHub as compromised and rotate it.
- **Review before pushing.** Check the diff for private information in the PR, especially notes and lab documentation.
- **CI/CD decision P5.0 is affected.** GitHub advises that self-hosted runners should almost never be used with public repositories, because anyone can open a pull request and compromise the environment. The option "GitHub Actions + self-hosted runner in the lab" needs this risk addressed in the P5.0 ADR, as the runner would be inside the lab.
- **The roadmap needs an update.** M0.1 must stop saying "private".
- **The secret-blocking test in M0.1 is still open.** It must be run on the public repository.

## Revisit when

- The repo needs to hold material that cannot be public.
- The account moves to a paid plan.

Making the repository private on GitHub Free would again remove the protection this ADR relies on, so it needs a new ADR first.

## Facts verified in the official docs (2026-11-04)

Plans and features change: re-check before relying on them.

- GitHub's plans: Free has "a full feature set" on public repositories and "a limited feature set" on private ones. Pro lists "Protected branches" under tools for private repositories.
- Push protection: repository-level protection on private repositories requires GitHub Secret Protection and is off by default. User-level protection is on by default and covers pushes to public repositories.
- Secure use of Actions: "self-hosted runners should almost never be used for public repositories".
- Not confirmed: the rulesets pages checked do not state rulesets availability for private repositories on GitHub Free.

## Rejected options

- **Private repo + GitHub Pro:** meets all requirements, but adds a recurring cost for a personal lab.
- **Private repo on GitHub Free without protection:** direct pushes to `main` would remain possible, so the level 0 entry condition and the M0.1 test would fail.
- **Self-hosted Git forge (e.g. Gitea) now:** the lab does not exist yet, and cloud sessions and phone monitoring depend on GitHub (A1).
