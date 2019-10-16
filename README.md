<h1 align="center">
  <img src="assets/logo.svg" height="250" width="250" alt="Autorebase logo"/>
  <p>Autorebase</p>
</h1>

Autorebase aims to make the Rebase Workflow enjoyable and keep `master` always green.

Autorebase is a GitHub App, based on [Probot](https://probot.github.io/), which automatically [rebases and merges](https://help.github.com/articles/about-merge-methods-on-github/#rebasing-and-merging-your-commits) pull requests.

It integrates especially well in repositories with branch protection set up to enforce up-to-date status checks.

# Maintenance update

Focus has shifted to the development of [Autosquash](https://github.com/marketplace/actions/autosquash), the successor of Autorebase.

Indeed:

- The app can now be implemented more simply as a [GitHub Action](https://github.com/features/actions). With a GitHub Action, the Git rebase operation can now be done directly in the CLI instead of doing the complex REST calls made by [`github-rebase`](https://www.npmjs.com/package/github-rebase).
- [GitHub is more suited for squash and merge than rebase and merge](https://github.com/tibdex/autosquash/tree/2aac7f4dbd29190dfaa3f8ef05ec0ed615f35f73#why-squash-and-merge).

# Usage

1.  :closed_lock_with_key: [recommended] Protect the branches on which pull requests will be made, such as `master`. In particular, it's best to [enable required status checks](https://help.github.com/articles/enabling-required-status-checks/) with the "Require branches to be up to date before merging" option.
2.  :label: When you're ready to hand over a pull request to Autorebase, simply [add the `autorebase` label to it](https://help.github.com/articles/creating-a-label/).
3.  :sparkles: That's it! Pull requests with the `autorebase` label will then be rebased when their base branch moved forward ([`mergeable_state === "behind"`](https://developer.github.com/v4/enum/mergestatestatus/#behind)) and "rebased and merged" once all the branch protections are respected ([`mergeable_state === "clean"`](https://developer.github.com/v4/enum/mergestatestatus/#clean)).

## One-time `/rebase` command

Autorebase also supports one-time rebase commands: a collaborator with [write permission](https://developer.github.com/v3/repos/collaborators/#review-a-users-permission-level) on a repository can post a `/rebase` comment on a pull request to rebase it once.

_Note_: This feature is a convenient way to easily integrate upstream changes such as CI config edits or bug-fixes but it shouldn't be abused as rebasing rewrites the Git history and thus makes collaborating on a pull request harder. See [this discussion](https://github.com/tibdex/autorebase/issues/41) for more details.

## Demo

### Rebasing a pull request with out-of-date status checks

![Rebasing a pull request with out-of-date status checks](./assets/out-of-date.gif)

This pull request has two commits and targets the `master` branch. The `master` branch has a required status check that must be up to date before merging a pull request on it. After adding the `autorebase` label, Autorebase automatically rebases the pull request on top of `master`. The pull request's branch is now up to date with `master`. As soon as a new successful status check is added to the pull request's last commit, Autorebase merges the pull request.

---

### Autosquashing a suggested commit

![Autosquashing a suggested commit](./assets/suggested-change.gif)

Again, this pull request has two commits and targets the `master` branch. Here, a reviewer made a [suggested change](https://help.github.com/articles/incorporating-feedback-in-your-pull-request/#applying-a-suggested-change) on its first commit. We accept this suggestion and name the resulting commit "fixup! Addition". Where "Addition" is the subject of the first commit. After adding the `autorebase` label, Autorebase automatically rebases the pull request on top of `master`, [autosquashing](https://git-scm.com/docs/git-rebase#git-rebase---autosquash) the suggested commit with the first one. Autorebase will then merge the pull request. We can see that two commits (and not three!) have landed on `master` and that the diff of the "Addition" commit is the suggested one.

_Note_: We could also have named the fixup commit with the first commit's entire or abbreviated SHA. Meaning that "fixup! 00fac8d" would have worked too.

# FAQ

## How Does It Work?

Autorebase relies on [`github-rebase`](https://www.npmjs.com/package/github-rebase) to perform all the required Git operations directly through the GitHub REST API instead of having to clone repositories on a server and executing Git CLI commands.

`github-rebase` is the :old_key: to being able to run Autorebase as a stateless, easy to maintain, and cheap to operate, GitHub App!

## Which Permissions & Webhooks Is Autorebase Using?

### Permissions

- **Repository contents** _[read & write]_: because the rebasing process requires creating commits and manipulating branches.
- **Issues** _[read & write]_: to search for pull requests to rebase or merge and add to manipulate labels on pull requests.
- **Pull requests** _[read & write]_: to merge pull requests.
- **Checks** & **Commit statuses** _[read-only]_: to know whether the status checks are green or not.

### Webhooks

- **Issue Comment**: to detect one-time `/rebase` commands.
- **Pull request**: to detect when the `autorebase` label is added/removed and when a pull request is closed.
  Indeed, closing a pull request by merging its changes on its base branch may require rebasing other pull requests based on the same branch since they would now be outdated.

  _Note:_ Instead of listening to the [`pull_request.closed`](https://developer.github.com/v3/activity/events/types/#pullrequestevent) webhook, Autorebase could listen to [`push`](https://developer.github.com/v3/activity/events/types/#pushevent) instead.
  It would allow it to react when a commit was pushed to `master` without going through a pull request.
  However, Autorebase would then receive much more events, especially since the rebasing process itself triggers many `push` events.
  Thus, to prevent pull requests with the `autorebase` label to get stuck behind their base branch, try not to push commits to these base branches without going through pull requests.

- **Pull request review**: because it can change the mergeable state of pull requests.
- **Check run** & **Status**: to know when the status checks turn green.

## Why Recommend Up-to-Date Status Checks?

To "keep `master` always green".

The goal is to never merge a pull request that could threaten the stability of the base branch test suite.

Green status checks are not enough to offer this guarantee. They must be [up-to-date](https://help.github.com/articles/types-of-required-status-checks/) to ensure that the pull request was tested against the latest code on the base branch. Otherwise, you're exposed to ["semantic conflicts"](https://bors.tech/essay/2017/02/02/pitch/).

## Why Rebasing Instead of Squashing/Merging?

### Squashing

Good pull requests are made of multiple small and atomic commits. You loose some useful information when squashing them in a single big commit. Decreasing the granularity of commits on `master` also makes tools such as [`git blame`](https://git-scm.com/docs/git-blame) and [`git bisect`](https://git-scm.com/docs/git-bisect) less powerful.

### Merging

Merge commits are often seen as undesirable clutter:

- They make the Git graph much more complex and arguably harder to use.
- They are often poorly named, such as "Merge #1337 into master", repeating what's already obvious.

See also this [Bitbucket article](https://www.atlassian.com/git/articles/git-team-workflows-merge-or-rebase) for a more in-depth comparison between the merge and rebase workflows. It lists traceability as the main advantage of the merge workflow. However, on GitHub, even when pull requests are "rebased and merged" (actually merged with the [`--ff-only`](https://git-scm.com/docs/git-merge#git-merge---ff-only) option), you can still, when looking at a commit on `master` in the GitHub web app, find out which pull request introduced it.

### Enforcing Rebase Merging

If you're convinced that rebasing is the best option, you can easily [enforce it as the only allowed method to merge pull requests on your repository](https://help.github.com/articles/configuring-commit-rebasing-for-pull-requests/).

### Autosquashing

Autorebase has built-in [autosquashing](https://git-scm.com/docs/git-rebase#git-rebase---autosquash) support. It will come in handy to automatically fixup/squash these commits added on pull requests after a reviewer requested changes.

## Why Not Clicking on the “Update Branch” Button Provided by GitHub Instead?

Because it creates merge commits and thus exacerbates the issue explained just above.

## When Not to Use Autorebase?

Rebasing rewrites the Git history so it's best not to do it on pull requests where several developers are collaborating and pushing commits to the head branch.

## Why Is Autorebase Removing Its Own Label before Rebasing and Then Adding It Back?

Autorebase can receive multiple webhooks for the same pull request in a short period of time.
Letting these invocations try to concurrently rebase that pull request is unnecessary since [only one attempt would succeed anyway](https://www.npmjs.com/package/github-rebase#atomicity).
To prevent this from happening, we use the `autorebase` label as a lock.
Before Autorebase starts rebasing a pull request, it will acquire the lock by removing the label.
Other concurrent Autorebase invocations won't be able to do the same thing because the GitHub REST API prevents removing a label from a pull request that doesn't have it.

## How Does Autorebase Compare with the Alternatives?

| Name                                                                               | No Merge Commits   | Stateless (no Database Required) | Ensure Up-to-Date Status Checks Without Manual Intervention / Test on Latest Before Merging | Comments                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                 |
| ---------------------------------------------------------------------------------- | ------------------ | -------------------------------- | ------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------ |
| [Bors](https://github.com/apps/bors)                                               | :x:                | :x:                              | :white_check_mark:                                                                          | Bors provides a more sophisticated rebasing strategy. It tries to batch pull requests together and see if the build is still passing on the "agglomerated pull request" before merging the corresponding pull requests. Bors might be better for projects with long/expensive CI builds and a high rate of incoming pull requests, since the time to run the builds is, in the best case, logarithmic with the number of pull requests.                                                                                                                                                                                                                                  |
| [automerge](https://github.com/apps/automerge)                                     | :x:                | :white_check_mark:               | :x:                                                                                         |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |
| [Refined GitHub browser extension](https://github.com/sindresorhus/refined-github) | :x:                | :white_check_mark:               | :x:                                                                                         | Refined GitHub has an [option to wait for checks when merging a pull request](https://github.com/sindresorhus/refined-github#highlights) if you don't mind having to keep your browser tab opened waiting for the status checks to be green before merging your pull requests.                                                                                                                                                                                                                                                                                                                                                                                           |
| [TravisCI](https://travis-ci.com/)                                                 | :white_check_mark: | :white_check_mark:               | :x:                                                                                         | TravisCI goes halfway in the good direction: ["Rather than build the commits that have been pushed to the branch the pull request is from, we build the merge between the source branch and the upstream branch."](https://docs.travis-ci.com/user/pull-requests/#How-Pull-Requests-are-Built) but [they don't trigger a new build when the upstream/base branch move forward](https://github.com/travis-ci/travis-ci/issues/1620) so you still need to rebase your pull requests manually. Besides, it ties you to a specific CI provider since CircleCI, for instance, doesn't do the same "building the pull request from the merge commit provided by GitHub" trick. |
| Autorebase                                                                         | :white_check_mark: | :white_check_mark:               | :white_check_mark:                                                                          |                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                                          |

## Why the :panda_face: Logo?

Because Autorebase loves eating branches :bamboo:!
