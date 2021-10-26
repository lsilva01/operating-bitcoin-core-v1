# Git Tutorial for Bitcoin Core

- [1. Basic Operations](#1-basic-operations)
- [2. Squashing and Rebasing](#2-squashing-and-rebasing)
- [3. Writing Good Commit messages](#3-writing-good-commit-messages  )
- [4. Creating Pull Requests](#4-creating-pull-requests)
- [5. Sources](#5-sources)

## 1. Basic Operations

### 1.1.1 Fork the Project

Go to `https://github.com/bitcoin/bitcoin` and click on the `Fork` button in the upper right corner.

This will create the `<username>/bitcoin` project.

### 1.1.2 Clone the Project

To clone the forked project, the command below can be used. This will allow you to work on the code and to open pull requests.

```bash
git clone https://github.com/<username>/bitcoin.git [<directory>]
```

The `directory` argument is optional. If no one is provided, the name of the repository will be used (in this case `bitcoin`).

If the intention is just to review or study the code, you can clone the original repository.

```bash
git clone https://github.com/bitcoin/bitcoin.git [<directory>]
```

### 1.1.3 Add Remote Repository (Upstream)

If you intend to open pull requests and contribute to the original code, you also need to add the original repository.

Otherwise, this step can be skipped.

```bash
git remote add upstream https://github.com/bitcoin/bitcoin.git
```

### 1.1.4 Create a New Branch

To propose a new feature, a new branch must be used to develop it. This can be done with the command:

```bash
git checkout -b new_branch
Switched to a new branch ‘new_branch’
```

Specifying `-b` causes a new branch to be created just like when `git-branch` is called and then checked out.

If switching to an existing branch, this parameter must be omitted.

### 1.1.5 Commiting and Publishing the Changes

After making the changes, you can check the files that have been changed with `git status`. This command is used to examine the current state of the repository and can be used to confirm a git add promotion.

The commit can be done with the following commands:

```bash
git add .
git commit -m "First commit"
git push -u origin new_branch
```

The `git add` command adds a change in the working directory to the staging area. It tells Git that you want to include updates to a particular file in the next commit. However, git add doesn't really affect the repository in any significant way—changes are not actually recorded until you run `git commit`.

If you want to add specific files to the staging area, you can use `git add <filename>`.

The `git reset` command is used to undo a git add. The `git commit` command is then used to commit a snapshot of the staging directory to the repository's commit history.

You can also use `git commit -a -m "First commit"`.Specifying `-a` automatically stages all files that git already knows about.

The `git push` is essential for a complete collaborative Git workflow. It is utilized to send the committed changes to remote repositories for collaboration.

The `-u` argument sets up the association between the current branch and the remote one (defined in the step 1.1.3). You only need to use `-u` once.

Alternatively, you can use the `–set-upstream` option that is equivalent to the `-u` option.

### 1.1.6 Create the Pull Request in GitHub

Once you push the changes to the repository, the `Compare & pull request` button will appear in GitHub in the initial page.

Open a pull request by clicking this button. This allows the repository maintainers to review the contribution. From here, they can merge it if it's good, or they can request some changes.

### 1.1.7 Review a Pull Request

GitHub exposes PRs as branches on the upstream repository with `pull/<pr_number>/head`.  So they can be retrieved with following command:

```bash
git fetch origin pull/<pr_number>/head && git checkout FETCH_HEAD
```

The review message usually contains `ACK` if the reviewers agree with the changes, `NACK` if not.

An `ACK` is usually followed by a description of how the review was done.

`Concept ACK` means the reviewer agrees with the objective of the change, but has not looked at or tested the code.

`Approach ACK` means agreement with both the objective and the approach used in the change.

`Code review ACK` means the code has been reviewed but not yet tested.

`Tested ACK` (or `tACK`) means the code was tested and the reviewer agrees with the change.

Some reviews contain a concise description of the tests performed, such as the [PR 23065 - comment](https://github.com/bitcoin/bitcoin/pull/23065#issuecomment-925224499). It is good practice for new contributors.

When giving an `ACK`, specify the commits reviewed by appending the commit hash of the HEAD commit, for example, `tACK 94b6c8d`.

### 1.1.8 Get a Specific Version (Tag)

The command `git tag` shows all existing tags.

You can switch to a specific tag with the following command, which is useful when testing a version.

```bash
git checkout tags/v0.21.0
```

## 2 Squashing and Rebasing

Keeping commits organized is a good practice. Usually after opening a PR, reviewers will make suggestions and the code will need to be changed. So, rather than adding new commits indiscriminately, `git rebase` can be used to organize these new commits in a coherent way.

### 2.1 Amending your last commit

If the `--amend` option is used, the changes will be squashed into the most recent commit.
This is the fastest way to merge two commits into one.

```bash
git commit -a --amend
```

### 2.2 Fixing up older commits

Amending only works for the most recent commit. But it's common to need to fix an older commit.

There is a tool to do this: the interactive rebase, which can be used in two ways: passing `HEAD~n` as parameters, where `n` is the last `n` commits that are about to be rebased. Or passing the commit hash as parameter.

```bash
git rebase -i HEAD~3
# or
git rebase -i 1be0581
```

This will open the default text editor with something like this:

```
pick 8d3fc77 Add test.cpp
pick 2a73a77 Change net.h
pick 0b9d0bb Add net.h

# Rebase f5f19fb..0b9d0bb onto f5f19fb (3 commands)
#
# Commands:
# p, pick <commit> = use commit
# f, fixup <commit> = like "squash", but it discards the commit's log message.
```

When you run git rebase -i, you get an editor session listing all of the commits that are being rebased and a number of options for what you can do to them. The default choice is `pick`.

`pick` maintains the commit in your history.

`reword` allows you to change a commit message, perhaps to fix a typo or add additional commentary.

`squash` merges multiple commits into one.

`fixup` is like `squash`, but discard this commit's log message.

You can reorder commits by moving them around in the file.

To merge commits, change `pick` to `fixup` for the items to be merged.

```
pick 8d3fc77 Add test.cpp
fixup 2a73a77 Change net.h
pick 0b9d0bb Add net.h
```

After saving and closing the text editor, git will remove all these commits from its history and then execute each line, one at a time. Lines with the `pick` command will be kept. Those with `fixup` or `squash` will be squashed.

### 2.3 Using git rebase --autosquash

The squashing can also be performed in a more automated manner by using the `--autosquash` option of `git rebase` in combination with the `--fixup` option of `git commit`:

```
git commit -a --fixup HEAD^
git rebase -i --autosquash HEAD~3
```
This automatically generates the rebase script with commits reordered and actions set up.

`HEAD^` refers to the first immediate parent of the tip of the current branch. `HEAD^` is short for `HEAD^1`. This is usually used on merge commits.

`HEAD ~ 3` gets the last 3 commits. This value can be changed to an arbitrary number of commits.

### 2.4 Fetching Upstream Commits

When developing new features, it is common to need to fetch new commits from `bitcoin:master` and rewrite the local master with the upstream's master.

To do this, the upstream repository must have already been configured, as shown in step 1.1.3.

The commands below execute this merge.

```bash
$ git fetch upstream
$ git rebase upstream/master
```

To push the update to the master, it may be necessary to force the push with `--force` (or `-f`).

```bash
$ git push origin master --force
```

## 3 Writing Good Commit messages

Good commit messages are one of most important aspects for a project's maintainability. They allow new contributors to retrieve the context of a change and understand _what_ has changed and _why_.

Therefore, a developer should not neglect good practices when writing commit messages. Some of them will be presented below.

### 3.1 Separate subject from body with a blank line

From the `git commit` manpage:

 Though not required, it’s a good idea to begin the commit message with a single short (less than 50 character) line summarizing the change, followed by a blank line and then a more thorough description. The text up to the first blank line in a commit message is treated as the commit title, and that title is used throughout Git.


Firstly, not every commit requires both a subject and a body. Sometimes a single line is fine, especially when the change is so simple that no further context is necessary. For example:

```
Some small improvements to release notes
```

This message can be found in the [commit 9f9ffe5](https://github.com/bitcoin/bitcoin/commit/9f9ffe5).

Nothing more needs be said; if the reader wonders what the improvements were, she can simply take a look at the change itself, i.e. use `git show` or `git diff` or `git log -p`.

If the commit message is simple, just add the `-m` option to `git commit`:

```bash
$ git commit -m "Some small improvements to release notes"
```

However, when a commit deserves a little explanation and context, it's better to write a body. For example:

```
doc: Stop nixing `-` in manual pages

The version replacement here is not working anyway, not just that but it
is actively harmful as it removes all `-` from the text. So remove that
line. See discussion in #22681.
```

Commit messages with bodies are not so easy to write with the `-m` option. It's better to write in a proper text editor.


Note that there is a blank line between the title and the body.

This entire message will be shown in the result of `git log` command.

`git log --oneline` prints out just the subject line and `git shortlog` groups commits by user, again showing just the subject line for concision.

### 3.2 Limit the subject line to 50 characters

50 characters is not a hard limit, just a rule of thumb. Keeping subject lines at this length ensures that they are readable, and forces the author to think for a moment about the most concise way to explain what’s going on.

GitHub’s UI will warn users if they exceed the 50 character limit and will truncate any subject line longer than 72 characters with an ellipsis.

So shoot for 50 characters, but consider 72 the hard limit.

Note that some of the last commits prior to v22 follow this rule:

```
$ git log --oneline

a0988140b (HEAD, tag: v22.0, origin/22.x) Merge bitcoin/bitcoin#22921: Some small improvements to release notes
9f9ffe5bb Some small improvements to release notes
f75615ebd doc: Manual pages update for 22.0 final
afbee409b build: Bump version to 22.0 final
03f142278 Merge bitcoin/bitcoin#22857: [22.x] Backports
fbf498d26 Merge bitcoin/bitcoin#22920: doc: Move 22.0 release notes from wiki
d44797241 doc: Move 22.0 release notes from wiki
303bc8a06 guix/prelude: Override VERSION with FORCE_VERSION
0640bf5c8 doc: mention bech32m/BIP350 in doc/descriptors.md
86de56776 (tag: v22.0rc3) doc: Manual pages update for rc3
c1c79f4c8 doc: Stop nixing `-` in manual pages
f95b655ba Improve doc/i2p.md regarding I2P router options/versions
59d4afc27 build: Bump version to 22.0rc3
```

### 3.3 Capitalize the subject line

This is as simple as it sounds. Begin all subject lines with a capital letter.

Note that most of the above commit messages start with a capital letter, except when prefixing the area affected by the commit. Valid areas are:

* `consensus` for changes to consensus critical code
* `doc` for changes to the documentation
* `qt` or `gui` for changes to bitcoin-qt
* `log` for changes to log messages
* `mining` for changes to the mining code
* `net` or `p2p` for changes to the peer-to-peer network code
* `refactor` for structural changes that do not change behavior
* `rpc`, `rest` or `zmq` for changes to the RPC, REST or ZMQ APIs
* `script` for changes to the scripts and tools
* `test`, `qa` or `ci` for changes to the unit tests, QA tests or CI code
* `util` or `lib` for changes to the utils or libraries
* `wallet` for changes to the wallet code
* `build` for changes to the GNU Autotools or reproducible builds

### 3.4 Do not end the subject line with a period

Trailing punctuation is unnecessary in subject lines. Besides, space is precious when you’re trying to keep them at 50 characters or less.

This can be seen in the above commit message headers.

### 3.5 Use the imperative mood in the subject line

Imperative mood just means “spoken or written as if giving a command or instruction”. A few examples:

* Remove extra \r from all.SHA256SUMS line ending
* Add a note that codesigners need to rebuild after tagging
* Make all.SHA256SUMS rather than codesigned.SHA256SUMS

The use of the imperative is important only in the subject line. This restriction can be relaxed when writing the body.

### 3.6 Wrap the body at 72 characters

Git never wraps text automatically. When writing the body of a commit message, mind its right margin, and wrap text manually.

The recommendation is to do this at 72 characters, so that Git has plenty of room to indent text while still keeping everything under 80 characters overall.

### 3.7 Use the body to explain what and why vs. how

The [commit eb0b56b](https://github.com/bitcoin/bitcoin/commit/eb0b56b19017ab5c16c745e6da39c53126924ed6) from Bitcoin Core is a great example of explaining what changed and why:

```
commit eb0b56b19017ab5c16c745e6da39c53126924ed6
Author: Pieter Wuille <pieter.wuille@gmail.com>
Date:   Fri Aug 1 22:57:55 2014 +0200

   Simplify serialize.h's exception handling

   Remove the 'state' and 'exceptmask' from serialize.h's stream
   implementations, as well as related methods.

   As exceptmask always included 'failbit', and setstate was always
   called with bits = failbit, all it did was immediately raise an
   exception. Get rid of those variables, and replace the setstate
   with direct exception throwing (which also removes some dead
   code).

   As a result, good() is never reached after a failure (there are
   only 2 calls, one of which is in tests), and can just be replaced
   by !eof().

   fail(), clear(n) and exceptions() are just never called. Delete
   them.
```

Take a look at the full diff and just think how much time the author is saving fellow and future committers by taking the time to provide this context here and now. If he didn’t, it would probably be lost forever.

The important thing here is to focus on making clear the reasons why you made the change in the first place—the way things worked before the change (and what was wrong with that), the way they work now, and why you decided to solve it the way you did.

## 4 Creating Pull Requests

### 4.1 Focus on a clear, minimal change

It is important that the PR has a clear objective. This makes the review easier and reduces the risk of changes introducing bugs.

If the PR is too long, consider breaking it down into smaller ones.

A good example of this is [PR 17728](https://github.com/bitcoin/bitcoin/pull/17728).

### 4.2 Use the PR description to explain why

The PR title and description is the first thing reviewers will see. Therefore, it is important that they present clarity and objectivity so that the reviewer can easily understand the PR motivations.

The PR title should use the prefixes presented in section 1.3.3. They are described in Bitcoin Core's contributing guide. The title of [PR 22732](https://github.com/bitcoin/bitcoin/pull/22732), for example, uses the prefix `net`.

The same recommendation for the commit description applies here. PR text should focus on _why_, not _what_. Summarizing what was done is okay, but explaining the motivations and the problem that the RP solves is crucial.

The text of [PR 22791](https://github.com/bitcoin/bitcoin/pull/22791), for example, explains the problem (how the bug was introduced) and the proposed solution.

Including step-by-step instructions on how to review and test the PR in the text is also interesting.

### 4.3 Organize the commits coherently

A PR can contain multiple commits. It is easier for the reviewer to analyze each change separately.

A good strategy for deciding the granularity of each commit is by functionality. For example, one changes the component, the other updates the documentation about this component, and another implements the tests for this component.

A good example of this strategy is [PR 23065](https://github.com/bitcoin/bitcoin/pull/23065).

```
d96b000e9 (HEAD) Make GUI UTXO lock/unlock persistent
077154fe6 Add release note for lockunspent change
719ae927d Update lockunspent tests for lock persistence
f13fc1629 Allow lockunspent to store the lock in the wallet DB
c52789365 Allow locked UTXOs to be store in the wallet database
```

The 5 commits that make up this PR are logically divided. `c52789365` implements the proposed solution. `f13fc1629` updates the RPC. `719ae927d` updates the tests. `077154fe6` adds release note and `d96b000e9` implements the same solution in GUI.

Separating commits this way is a good practice and makes it a lot easier for reviewers.

### 4.4 Create test coverage if necessary

When adding a new feature, such as a new RPC, a test should be added.

If the change occurs at the component level, unit testing is likely to be more appropriate. If it's at the resource level (something users interact with), then functional testing fits the case.

An example of a PR that does this is the [PR 14667](https://github.com/bitcoin/bitcoin/pull/14667), which added a new RPC and  functional test for it.

## 5 Sources:

[Git Rebase in Depth](https://git-rebase.io/)

[How to Contribute Pull Requests to Bitcoin Core](https://jonatack.github.io/articles/how-to-contribute-pull-requests-to-bitcoin-core)

[How to create a pull request in GitHub](https://opensource.com/article/19/7/create-pull-request-github)

[How to Review Pull Requests in Bitcoin Core](https://jonatack.github.io/articles/how-to-review-pull-requests-in-bitcoin-core)

[How to update a forked repo with git rebase](https://medium.com/@topspinj/how-to-git-rebase-into-a-forked-repo-c9f05e821c8a)

[How to Write a Git Commit Message](https://chris.beams.io/posts/git-commit/)

[Git Reference](https://git-scm.com/docs/)

[The life-changing magic of git rebase -i](https://opensource.com/article/20/4/git-rebase-i)

[Saving Changes](https://www.atlassian.com/git/tutorials/saving-changes)

[What's the difference between HEAD^ and HEAD~ in Git?](https://stackoverflow.com/a/2222920)