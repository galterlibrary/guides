#Development and release workflow for Galter Website

In order to...

* Allow easy application of bug fixes outside of the release cycle, and
* Allow integration of ongoing work on the next release

##Versions

Galter Website uses a variant on [Semantic Versioning](http://semver.org). Taking a usual three-part version number as `{MAJOR}.{MINOR}.{PATCH}`, the elements are incremented in the following conditions:

* `PATCH` is incremented for out-of-cycle hotfix releases. Hotfix releases may only contain bug fixes and will usually only contain one bugfix each.
* `MINOR` is incremented for planned releases. Planned releases may contain any combination of new features, bugfixes, and internal changes.
* `MAJOR` is incremented rarely, probably only for marketing purposes.

###Prerelease specifiers

A version may also contain fourth component -- a prerelease specifier; e.g., `1.5.0.beta1`. These prerelease specifiers should follow rubygems semantics:

* They must be alphanumeric.
* Their semantic order must coincide with their lexicographical order. I.e., `alpha` comes before `beta` comes before `rc`.
* They must always indicate a prerelease. I.e., `1.5.0.whatever` comes before `1.5.0`.
* They must not include periods.

The version of any arbitrary commit on the `develop` branch should end in `.dev`. When an alpha release needs to be made off of `develop`, the version should be changed to reflect the alpha (e.g., `1.5.0.alpha1`), the release tagged, and the version changed back (e.g., to `1.5.0.dev`).

##Version control workflow

###General git/VC notes

* All developers are encouraged to use `git pull` with the `--rebase` option so as not to introduce unnecessary merge commits due to local changes. [This can be made the default](http://mislav.uniqpath.com/2013/02/merge-vs-rebase/).
* When making a large number of style (particularly whitespace) changes to a file, introduce them in a separate commit. This will make reviewing your functional changes easier.
* When doing manual merges (e.g., for hotfixes that do not require a review), use `git merge --no-ff` to ensure the creation of a merge commit.
  * `--no-ff` means "no fast-forward". It will preserve the history that these changes were made on a separate branch. While all commits should also be tagged with issue numbers, preserving the fact of the branch is a bit of additional information that can help track down where and why something was introduced.


###Commit message style

The first line of a commit message should be a summary of approximately 50 characters, plus the ticket number. E.g.:

`Correct computation of Participant#height. #1234.`

If more information is required, it should be added following a blank line and hard-wrapped at 72 characters. E.g.:

```
Correct computation of Participant#height. #1234.

There is code elsewhere which expects it to be exact. There's no
downside to making it a `Rational` instead of a float.
```

Commit messages may contain markdown markup.

[Discussion of the 50/72 convention](http://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html).

###Relationship to git-flow

The git workflow described here follows the conceptual model of [git-flow](http://nvie.com/posts/a-successful-git-branching-model/), so it may be helpful to review that document also. This workflow differs from git-flow in two ways:

* The introduction of GitHub pull request code reviews in between development
  on a topic branch and merge into the appropriate base branch.

* Some of the specific git operations used to synchronize work between branches:
** `cherry-pick` instead of `merge` for synchronizing hotfixes and post-release-branch changes
** `rebase` instead of `merge` for getting the latest `develop` changes into a topic branch

###Base branches

There are two branches which always exist: `master` and `develop`. `master` always reflects production-ready code -- code which has been deployed or will be soon. `develop` is the integration branch for features and bugfixes in the next release.

###Work branches

There are three categories of work you might do which could affect the source. Each of these categories has its own kind of branch and workflow.

####1. Features or fixes for the next release

Branch name: `{ticket_number}-{brief_synopsis}`
Branches from: `develop`
Merges back into: `develop` via pull request
Code review required.

When doing planned work for a future release, all work should be done in a topic branch. When the work is ready to be reviewed, the topic branch should be rebased over the current `develop` and then pushed to `origin`. I.e.:

```
$ git checkout develop
$ git pull --rebase
$ git checkout -b 1234-fix_foo_barness
[...]
$ git commit
[...]
$ git commit
[...]
$ git commit
# ready to submit for review
$ git checkout develop
$ git pull --rebase
$ git checkout 1234-fix_foo_barness
$ git rebase develop
# resolve any conflicts, then
$ git push origin 1234-fix_foo_barness
```

If your style of git use has produced many dozens of commits for a smallish feature, you may want to use an interactive rebase (`git rebase -i`) to combine some of them.

After your topic branch is pushed to `origin`, file a pull request for it. Be sure to include the redmine ticket number in the title of the PR. Also be careful to select the correct target branch for your PR — you want `develop`, not `master`.

If you get comments in the code review which require changes, make those changes in your topic branch and push them to origin. If there are many small changes, you may want to squash them into your original commits. If there are more substantial changes required, you may want to keep them as separate commits. Use your judgement about what would produce the most useful history for someone trying to understand what you did, six months from now.

If you get comments and push changes to your topic branch, add a comment to the PR indicating when it is ready to review again.

After the pull request is merged into `develop`, the topic branch should be deleted (this should be handled by the person who merges it in the GitHub UI). You will want to delete your local copy of the branch also:

```
$ git checkout develop
$ git fetch -p # retrieve the fact that the branch was deleted on origin
$ git branch -d 1234-fix_foo_barness
```

####2. Release preparation

Branch name: `release-{version_number}`
Branches from: `develop`
Merges into: `master`

Once `develop` has all the features and major fixes planned for the next release integrated, it's time to prepare a release. For this you need a release branch:

```
$ git checkout develop
$ git pull --rebase
$ git checkout -b release-1.77.0
$ git push origin release-1.77.0
```

Any further minor changes that are necessary to prepare the release should be made in this branch. This includes setting the version number and correcting remaining minor bugs.

In general, changes made on this branch will not be significant enough to require a code review. If a significant fix is found to be necessary, it should be built using a topic branch as described in the previous section, except that the base branch for the topic branch _must be the release branch_.

Once the release is ready for QA, a release candidate should be tagged. This involves:

* Setting & committing the RC version number. E.g., `1.77.0.rc1`.
* Tagging the release candidate version.
* Cherry-picking any applicable changes made in the release branch back to `develop`. "Applicable" is at the discretion of the release manager, but in general you will want to cherry-pick everything back except for the version number changes.

With feedback from QA and additional release testing, additional changes may be required. As they are made, create new RCs following the same procedure.

Once everyone agrees that the release is ready, prepare the release:

* Set and commit the final release number.
* Cherry-pick any remaining applicable changes to `develop`.
* Merge the release branch into `master`.
* Tag the release on `master`.
* Delete the release branch.

####3. Hotfixes

Branch name: `hotfix-{version_number}`
Branches from: `master`
Merges into: `master`

A hotfix is a bugfix that needs to be applied to production immediately, without waiting for the next release. ("Needs to be" is hard to define, but it will generally cover bugs which can cause data loss or make the application unusable for some users.)

As with release branches, changes made on this branch will not generally need a code review. If a hotfix does require a code review, the hotfix branch itself can be submitted as pull request.

Note that the `{version_number}` should be the name of the release that the hotfix will be not the release it is branched from. The hotfix branch should also include updating the version number to the expected release version number.

Once a hotfix is ready to be deployed, follow these steps:

* Merge the hotfix branch into `master`
* Tag the hotfix version
* Cherry-pick any applicable changes back to `develop`.
* Delete the hotfix branch

##Git References

* [Pro Git](http://git-scm.com/book)
* [Visual Reference](http://marklodato.github.io/visual-git-guide/index-en.html)
* [Git User's Manual](https://www.kernel.org/pub/software/scm/git/docs/user-manual.html)
