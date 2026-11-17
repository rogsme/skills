---
name: combine-retros
description: "Merge every open retro pull request into one combined pull request, reconcile the overlaps, and address its review."
license: MIT
disable-model-invocation: true
metadata:
  rogs-skills/contract: "1"
  opencode/autoinvoke: "false"
---

Fold the open retro pull requests into one, so their changes to the same files land together and once. Each source pull request is merged in with a merge commit at the head you record, so its head reaches the base when the combined pull request lands and the forge marks it merged too. Nothing here changes a source branch.

## Steps

1. List the candidates. With the forge CLI that `docs/agents/forge.md` names (or the one the "create-pr" skill would discover), list the open pull requests by the current forge user whose head branch starts with `retro-` or whose title starts with `Retro:`, following every page. Show each one's number, title, branch, head SHA, review state and opening date. Leave out a combined pull request (`retro-combined-` branch) unless the user names it. Ask the user to confirm the set, with the harness's structured question tool (`AskUserQuestion` in Claude Code, `question` in OpenCode), and restate the confirmed set.

2. Create the worktree `retro-combined-<YYYY-MM-DD>` as [Retro branches and pull requests](#retro-branches-and-pull-requests) says. The subject is `combined <YYYY-MM-DD>`.

3. Merge each pull request in the order they were opened. Fetch its head, record the SHA, and run `git merge --no-ff -m "Merge PR #<number> (<branch>) into the combined retro branch" <sha>`. When a merge stops on conflicts, resolve each one from what both sides meant: keep one copy of a rule both added, keep both parts of guidance that adds different things to the same place, and ask the user about any hunk where the two sides decide behavior differently. Stage only resolved paths, conclude with `git commit --no-edit`, and record each resolution for the body.

4. Reconcile. For every file two or more pull requests changed, read the combined result against the base and look for guidance that now repeats or contradicts itself, a command one side renamed that the other still calls, and counts or lists one side extended. Fix them in one commit through the "commit" skill. Run the full check `## Verification` in `docs/agents/workflow.md` names, or the checks the source pull requests ran, and fix what fails.

5. Recheck the set, since a session can open or push a retro pull request while this one runs. List the candidates again and ask about each new one; merge it as in step 3 when the user adds it. Fetch each source head again: merge a head that moved, with the same message, and record its new SHA. Repeat step 4 for what the new merges changed.

6. Publish, following [Retro branches and pull requests](#retro-branches-and-pull-requests), with a short summary of the combined changes as the title's `<title>`. The body adds a `## Combined pull requests` table (number, branch, head merged, its review state), an `## Overlaps` list of the resolutions from steps 3 to 5, what the source pull requests left out of this one, and the date the user confirmed the set. Count the table's rows against the merged set before you hand it over.

7. Comment on each source pull request with the forge's `comment` operation: "Combined into #<number> at head `<sha>`. It merges when #<number> does." Close none of them: the forge marks each one merged when the combined pull request lands, as long as its head has not moved. Then address the combined pull request's review as [Retro branches and pull requests](#retro-branches-and-pull-requests) says.

8. Offer cleanup. List the local worktrees whose branch starts with `retro-`, whose branch the fetched base already contains (`git merge-base --is-ancestor <branch> <remote>/<base>`), and whose status is clean, each with its `git worktree remove <path>` and `git branch -d <branch>` commands. Remove them only when the user says yes. Name the retro worktrees you keep and why: not merged, or uncommitted changes.

9. Report the worktree path, the pull request's URL and head, the merged set with each head, the resolutions, the review result, and the worktrees removed or kept.

## Retro branches and pull requests

One retro pull request lives in one worktree whose branch name and directory name are the same. Create it in the `Usual location:` that `## Worktrees` in `docs/agents/workflow.md` gives, with the name in place of the ticket id, or beside the current worktree when no such file exists. Add `-2`, `-3` and so on to the name when that branch or directory already exists. Fetch the base (`## Forge` in `docs/agents/forge.md`, or the remote's default branch) and run `git worktree add -b <name> <path> <remote>/<base>`. Prepare it the way the project prepares a new worktree (its README, `AGENTS.md`, or the `## Worktrees` and `## Verification` notes) before running a check. Work only there: give every shell command the worktree's path (`git -C <path>`, `cd <path> && ...`) and every file edit an absolute path, since a harness can reset the shell's directory between calls.

Commit by calling the Skill tool with "commit", with the explicit paths and the project's `## Commits` rules. Publish by calling the Skill tool with "create-pr", handing over the base, the title `Retro: <subject> - <title>`, and a body that follows the project's template and adds the sections the steps above name. The first line of its `## Related work` section is `No ticket: the <subject> retrospective. <subject> is its subject, not a ticket this pull request delivers.` Tell create-pr that a title rule built on ticket ids applies to pull requests that deliver tickets.

A pull request delivers no ticket when the first line of its `## Related work` section starts with `No ticket`. The ticket ids in its title and body then name its subject, not tickets it delivers: load no ticket, write nothing to the tracker, and build the requirements from the pull request body.

Then, in a project set up with `/setup-rogs`, wait with the `ci` operation of `docs/agents/forge.md` for the checks on the pull request's head. When the `CI review` row of `docs/agents/workflow.md` is on, the CI reviewer has now posted: call the Skill tool with "address-pr-review" from the worktree. Otherwise the pull request waits for its reviewer: say so, and name the worktree to run `/address-pr-review` from. Do this for each pull request the run opened.
