# git_manual

## 6. Version Control (Git)

### Git’s data model
- Snapshots

Git models the history of a collection of files and folders within some top-level directory as a series of snapshots. In Git terminology, a file is called a “blob”, and it’s just a bunch of bytes. A directory is called a “tree”, and it maps names to blobs or trees (so directories can contain other directories). A snapshot is the top-level tree that is being tracked. For example, we might have a tree as follows:
```
. <root> (tree)
├── foo (tree)
│   └── bar.txt (blob, contents = "hello world")
├── baz.txt (blob, contents = "git is wonderful")
```
The top-level tree contains two elements, a tree “foo” (that itself contains one element, a blob “bar.txt”), and a blob “baz.txt”.

- Modeling history: relating snapshots

How should a version control system relate snapshots? One simple model would be to have a linear history. A history would be a list of snapshots in time-order. For many reasons, Git doesn’t use a simple model like this.

In Git, a history is a directed acyclic graph (DAG) of snapshots. That may sound like a fancy math word, but don’t be intimidated. All this means is that each snapshot in Git refers to a set of “parents”, the snapshots that preceded it. It’s a set of parents rather than a single parent (as would be the case in a linear history) because a snapshot might descend from multiple parents, for example, due to combining (merging) two parallel branches of development.

Git calls these snapshots “commit”s. Visualizing a commit history might look something like this:
```
o <-- o <-- o <-- o
            ^
             \
              --- o <-- o
```
In the ASCII art above, the os correspond to individual commits (snapshots). The arrows point to the parent of each commit (it’s a “comes before” relation, not “comes after”). After the third commit, the history branches into two separate branches. This might correspond to, for example, two separate features being developed in parallel, independently from each other. In the future, these branches may be merged to create a new snapshot that incorporates both of the features, producing a new history that looks like this, with the newly created merge commit shown in bold:

```
o <-- o <-- o <-- o <---- o
            ^            /
             \          v
              --- o <-- o
```
Commits in Git are immutable. This doesn’t mean that mistakes can’t be corrected, however; it’s just that “edits” to the commit history are actually creating entirely new commits, and references (see below) are updated to point to the new ones.

NOTE: Commits are immutable i.e. “edits” to the commit history are actually creating entirely new commits, and references are updated to point to the new ones.


- Data model, as pseudocode
It may be instructive to see Git’s data model written down in pseudocode:
```
// a file is a bunch of bytes
type blob = array<byte>

// a directory contains named files and directories
type tree = map<string, tree | file>

// a commit has parents, metadata, and the top-level tree
type commit = struct {
    parent: array<commit>
    author: string
    message: string
    snapshot: tree
}
```
It’s a clean, simple model of history.

Objects and content-addressing

An “object” is a blob, tree, or commit:

`type object = blob | tree | commit`

In Git data store, all objects are content-addressed by their [SHA-1](https://en.wikipedia.org/wiki/SHA-1) hash.

```
objects = map<string, object>

def store(object):
    id = sha1(object)
    objects[id] = object

def load(id):
    return objects[id]
```
Blobs, trees, and commits are unified in this way: they are all objects. When they reference other objects, they don’t actually contain them in their on-disk representation, but have a reference to them by their hash.

For example, the tree for the example directory structure above (visualized using `git cat-file -p 698281bc680d1995c5f4caaf3359721a5a58d48d)`, looks like this:
```
100644 blob 4448adbf7ecd394f42ae135bbeed9676e894af85    baz.txt
040000 tree c68d233a33c5c06e0340e4c224f0afca87c8ce87    foo
```

The tree itself contains pointers to its contents, `baz.txt` (a blob) and `foo` (a tree). If we look at the contents addressed by the hash corresponding to `baz.txt` with `git cat-file -p 4448adbf7ecd394f42ae135bbeed9676e894af85`, we get the following:

`git is wonderful`

- References
Now, all snapshots can be identified by their SHA-1 hashes. That’s inconvenient, because humans aren’t good at remembering strings of 40 hexadecimal characters.

Git’s solution to this problem is human-readable names for SHA-1 hashes, called “references”. References are pointers to commits. Unlike objects, which are immutable, references are mutable (can be updated to point to a new commit). For example, the master reference usually points to the latest commit in the main branch of development.
```
references = map<string, string>

def update_reference(name, id):
    references[name] = id

def read_reference(name):
    return references[name]

def load_reference(name_or_id):
    if name_or_id in references:
        return load(references[name_or_id])
    else:
        return load(name_or_id)
```
With this, Git can use human-readable names like `“master”` to refer to a particular snapshot in the history, instead of a long hexadecimal string.

One detail is that we often want a notion of “where we currently are” in the history, so that when we take a new snapshot, we know what it is relative to (how we set the `parents` field of the commit). In Git, that “where we currently are” is a special reference called `“HEAD”`.

 `HEAD` refers to the last snapshot.

- Repositories

Finally, we can define what (roughly) is a Git repository: it is the data `objects` and `references`.

On disk, all Git stores are objects and references: that’s all there is to Git’s data model. All `git` commands map to some manipulation of the commit DAG by adding objects and adding/updating references.

Whenever you’re typing in any command, think about what manipulation the command is making to the underlying graph data structure. Conversely, if you’re trying to make a particular kind of change to the commit DAG, e.g. “discard uncommitted changes and make the ‘master’ ref point to commit `5d83f9e`”, there’s probably a command to do it (e.g. in this case, `git checkout master;` and then`git reset --hard 5d83f9e`).

- Staging area

This is another concept that’s orthogonal to the data model, but it’s a part of the interface to create commits.

One way you might imagine implementing snapshotting as described above is to have a “create snapshot” command that creates a new snapshot based on the current state of the working directory. Some version control tools work like this, but not Git. We want clean snapshots, and it might not always be ideal to make a snapshot from the current state. For example, imagine a scenario where you’ve implemented two separate features, and you want to create two separate commits, where the first introduces the first feature, and the next introduces the second feature. Or imagine a scenario where you have debugging print statements added all over your code, along with a bugfix; you want to commit the bugfix while discarding all the print statements.

Git accommodates such scenarios by allowing you to specify which modifications should be included in the next snapshot through a mechanism called the “staging area”.

- Git command-line interface
To avoid duplicating information, we’re not going to explain the commands below in detail. See the highly recommended [Pro Git](https://git-scm.com/book/en/v2) for more information, or watch the lecture video.

- Basics

* `git help <command>`: get help for a git command
* `git init`: creates a new git repo, with data stored in the .git directory
* `git status`: tells you what’s going on
* `git add <filename>`: adds files to staging area
* `git add -a` or `--all`: add all the files in the staging area
* `git add -u` or `git add –update` : It add all the modified and deleted files but not any untracked files and it does this for the entire tree.

* `git add :/` add everithing from the top level, down of the repository
*  `git commit -m`: creates a new commit
	*  Write good [commit messages](https://tbaggery.com/2008/04/19/a-note-about-git-commit-messages.html)!
	*  Even more reasons to write [good commit messages](https://cbea.ms/git-commit/)!
* `git commit -a`: commits all the changes that were to files that are already being tracked
*  `git commit --fixup=<commit id to be fixed>`: Creates a commit that automatically marks itself as a "fixup" of an earlier commit.
The `<commit id>` is the hash of the commit that you want to fix.
When combined with `git rebase --autosquash`, the fixup commit is automatically squashed into the original one during a rebase, cleaning up the history.
* `git cat-file -p <#commit-hash>`: print out the contents of this commit. it works with tree hash or blob hash as well.
* `git log`: shows a flattened log of history
* `git log --stat`: log which files changed
* `git log --patch`: log with diff
* `git log --all --graph --decorate`: visualizes history as a DAG
* `git log --all --graph --decorate --oneline`: shows a more compact rapresentation of the DAG
* `git diff <filename>`: show changes you made relative to the staging area with respect to HEAD
* `git diff <revision> <other-revision> <filename>`: shows differences in a file between snapshots with respect to other-revision
* `git diff <revision> <filename>`: shows differences in a file between snapshots with respect to HEAD
* `git checkout <revision>`: updates HEAD and current branch. get the commit hash of a previous commit, and git checkout change the working diractory stato to how it was at the previous commit. in other word move the HEAD reference on the commit indicated by the hash. if I want come back to the last commit I will do `git checkout <last-commit-hash>` or in alternatif `git checkout master`. git checkout actually changes the contents of your working directory so it can be a somewhat dangerous command if you are not careful about it.
* `git reflog`: Shows a log of changes made to HEAD, which includes commits, checkouts, and resets. Helps recover lost commits and track what happened at each step, even if a commit was "undone."

- Branching and merging
* `git branch`: shows branches. list all the branches
* `git branch -vv`:  extra verbose and print extra information
* `git branch <name>`: creates a branch. so now there is a new reference call <name> that point at the new branch called <name>.
* `git checkout -b <name>`: creates a branch and switches to it
same as `git branch <name>; git checkout <name>`
* `git merge <revision>`: merges the <revision> branch into current branch.
* `git merge --continue`: after a git merge stops due to conflicts when you resolve the conflict you can conclude the merge by running this command. See [how to resolve conflict](https://git-scm.com/docs/git-merge/2.27.0#_how_to_resolve_conflicts) 
* `git mergetool`: use a fancy tool to help resolve merge conflicts
* `git rebase`: rebase set of patches onto a new base
* `git rebase -i <ref>`:Opens an interactive rebase session for commits back to the specified reference `<ref>`.Allows editing, reordering, squashing, or dropping commits, which is useful for cleaning up commit history.
* `git rebase --autosquash`: Reorders commits marked with `fixup!` or `squash!` prefixes to be squashed automatically with the target commit.Typically used in conjunction with` git commit --fixup=<commit>` or `--squash=<commit>` to "fix" or "add" changes to a previous commit.Streamlines the rebase process, automatically combining the fixup or squash commits with their target commits.
* `git switch <branch-name>`: Switches to an existing branch `<branch-name>`. Replaces `git checkout <branch-name>` for branch switching.
* `git switch -c <branch-name>`:Creates a new branch and switches to it immediately. Combines the functionality of creating and checking out a branch in one command.

- Remotes
* `git remote`: list remotes
* `git remote -v` or 
`--verbose`: verbose list remotes
* `git remote add <name> <url>`: add a remote
* `git remote add origin git@github.com:<user>/<repo>.git`: add a remote with the name `origin` that normally is the name of the remote if you are only using one remote
* `git push <remote> <local branch>:<remote branch>`: send the changes from the locally objects to the remote, and update remote reference. this command creates a new branch called `<remote branch>` or updates the branch on the remote with the name specified, and set it tio the contents of the `<local branch>`.
* `git push origin master:master`
* `git branch --set-upstream-to=<remote>/<remote branch>`: set up correspondence between local and remote branch. set the upstream remote for the branch that's currently checked out. after that I can push the commits in the easiest way launching only `git push` command.
* `git push`: seding changes from local copy of the repository to the remote
* `git fetch`: retrieve objects/references from a remote repository. git fetch get informaton about the remote repository becuase there maybe update on the remote. if there are apdates we can execute `git marge` in order to update the local "master" at the same commit of the master in the remote repository.
* `git pull`: basically is the same of doing `git fetch`; `git merge`. git pull merge origin master branch into our local master branch
* `git clone <remote-url> <local-destination-folder>`: download repository from remote to a specific local folder. if you con't specify a  local folder the repo will be cloned in the current directory.
* `git clone --shallow`: clone the remote repo with just the last snapshot in order to reduce the dimension and increase the download velocity.

- Undo
* `git commit --amend`: edit a commit’s contents/message
* `git reset HEAD <file>`: unstage a file
* `git reset --mixed HEAD~1`: Resets the current branch to the state it was in before the last commit (HEAD~1).
`-mixed` reset keeps the changes in your working directory (unstaged) but clears them from the staging area (index).
Useful if you realize the last commit was incorrect but want to make changes to it without losing the work.
* `git checkout -- <file>`: discard changes. thorows away the changes that there are in the <file> and set the contents of <file> back to the way it was in the commit that HEAD points to.
* `git marge --abort`: cancel a merge that had conflicts
* `git clean -n`:Performs a dry run to show files that would be removed by `git clean`, without actually deleting them.
Useful for previewing what would be deleted by a `git clean -f` command.
* `git clean -n -d`: performs a dry run and also shows untracked directories that would be removed. Useful if you want to see the effect of cleaning out both files and directories.
* `git clean -f -d`: Forces deletion of untracked files and directories. This command removes all untracked files and directories, clearing the working directory of clutter.
* `git rebase -i <ref>`: run the interactive rebase back to
* `git branch -d <branch-name>`:Deletes the branch `<branch-name>`.
Works only if the branch has been merged; otherwise, it will warn about potential data loss.

- Advanced Git
* `git config`: Git is [highly customizable](https://git-scm.com/docs/git-config). you can edit the `~/.gitconfig` file in the home folder
* `git clone --depth=1`: shallow clone, without entire version history
* `git add -p`: interactive staging pieces of files for it commit
* `git rebase -i`: interactive rebasing
* `git blame`: show who last edited which line
* `git stash`: if there are some changes uncommit and I temporarily want to put them away, so I can execute `git stash` that remove modifications to working directory. the modification are not deleted, it's saved somewhere and if I do `git stash pop` it will undo the stash ad update the files at the changes I made
* `git bisect`: binary search history (e.g. for regressions)
* `.gitignore`: [specify](https://git-scm.com/docs/gitignore) intentionally untracked files to ignore


-[How to Undo a Git Add?](https://stackoverflow.com/questions/348170/how-do-i-undo-git-add-before-commit)
  To undo `git add` before a commit, run `git reset <file>` or `git reset` to unstage all changes.
-[How to Untrack a File That Has Been Committed](https://gist.github.com/mcrd25/1b2718054b65f4720fa582c6a4032217)
It is IMPORTANT to note that you MUST COMMIT ALL PREVIOUS CHANGES BEFOE CONTINUING

1. Remove everything from Local Repo git rm -r --cached . or if you are untracking a specific file(s) git rm -r --cached name_of_file
2. Add back all untracked files you need git add . or git add name_of_file_with_fix
3. Commit Changes git commit -m "untrack name_of_file fix"
4. Push changes to remote repo git push


- Miscellaneous
[How do I update or sync a forked repository on GitHub?](https://stackoverflow.com/questions/7244321/how-do-i-update-or-sync-a-forked-repository-on-github)
[Here](https://docs.github.com/en/pull-requests/collaborating-with-pull-requests/working-with-forks/syncing-a-fork) is GitHub's official document on Syncing a fork:

- Shell integration: it’s super handy to have a Git status as part of your shell prompt ([zsh](https://github.com/olivierverdier/zsh-git-prompt), [bash](https://github.com/magicmonty/bash-git-prompt)). Often included in frameworks like [Oh My Zsh](https://github.com/ohmyzsh/ohmyzsh).
- Workflows: practices to follow when working on big projects (and there are many different approaches):
[A successful Git branching model](https://nvie.com/posts/a-successful-git-branching-model/)
[GitFlow considered harmful](https://www.endoflineblog.com/gitflow-considered-harmful)
[Gitflow workflow](https://www.atlassian.com/git/tutorials/comparing-workflows/gitflow-workflow) **

[trunk based development](https://trunkbaseddevelopment.com)

Resources
[git toutorial - Atlassian ](https://www.atlassian.com/git/tutorials)
[Trunk-based development](https://dora.dev/capabilities/trunk-based-development/) **
[Feature Branch - Martin Fowler](https://martinfowler.com/bliki/FeatureBranch.html) **
[On DVCS, continuous integration, and feature branches](https://continuousdelivery.com/2011/07/on-dvcs-continuous-integration-and-feature-branches/) **

[Test automation](https://dora.dev/capabilities/test-automation/)
[Pro Git](https://git-scm.com/book/en/v2) is highly recommended reading. Going through Chapters 1–5 should teach you most of what you need to use Git proficiently, now that you understand the data model. The later chapters have some interesting, advanced material.
[Oh Shit, Git!?!](https://ohshitgit.com) is a short guide on how to recover from some common Git mistakes.
[Git for Computer Scientists](https://eagain.net/articles/git-for-computer-scientists/) is a short explanation of Git’s data model, with less pseudocode and more fancy diagrams than these lecture notes.
[Git from the Bottom Up](https://jwiegley.github.io/git-from-the-bottom-up/) is a detailed explanation of Git’s implementation details beyond just the data model, for the curious.
[How to explain git in simple words](https://xosh.org/explain-git-in-simple-words/)
[Learn Git Branching](https://learngitbranching.js.org/?locale=it_IT) is a browser-based game that teaches you Git.
[Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/)
[Commit Often, Perfect Later, Publish Once: Git Best Practices](https://sethrobertson.github.io/GitBestPractices/)

#### Set Up and Initialization

Check your Git version with the following command, which will also confirm that Git is installed:
* `git --version`

 Git allows you to configure a number of settings that will apply to all the repositories on your local machine. For instance, configure a username that Git will use to credit you with any changes you make to a local repository:
* `git config --global user.name “firstname lastname”`

Configure an email address to be associated with each history marker:
* `git config --global user.email “valid-email” `

Configure your preferred text editor as well:
* `git config --global core.editor “nano”`
  
You can initialize your current working directory as a Git repository with init:
* `git init`

To copy an existing Git repository hosted remotely, you’ll use git clone with the repo’s URL or server location (in the latter case you will use ssh):
* `git clone https://www.github.com/username/repo-name`

Show your current Git directory’s remote repository:
* `git remote`

For more verbose output, use the -v flag:
* `git remote -v`

Add the Git upstream, which can be a URL or can be hosted on a server (in the latter case, connect with ssh):
* `git remote add upstream https://www.github.com/username/repo-name`

Staging

When you’ve modified a file and have marked it to go in your next commit, it is considered to be a staged file.
Check the status of your Git repository, including files added that are not staged, and files that are staged:

* `git status` To stage modified files, use the add command, which you can run multiple times before a commit. If you make subsequent changes that you want to include in the next commit, you must run add again.

You can specify the specific file with add:
* `git add my_script.py`

With . you can add all files in the current directory, including files that begin with a .:
* `git add .`

If you would like to add all files in a current directory as well as files in subdirectories, you can use the -all or -A flag:
* `git add -A`

You can remove a file from staging while retaining changes within your working directory with reset:
* `git reset my_script.py`

#### Committing

Once you have staged your updates, you are ready to commit them, which will record changes you have made to the repository.
To commit staged files, you’ll run the commit command with your meaningful commit message so that you can track commits:
* `git commit -m "Commit message"`

You can condense staging all tracked files by committing them in one step:
* `git commit -am "Commit message"`

If you need to modify your commit message, you can do so with the --amend flag:
* `git commit --amend -m "New commit message"`

#### Branches

A branch in Git is a movable pointer to one of the commits in the repository, it allows you to isolate work and manage feature development and integrations. You can learn more about branches by reading the Git documentation.

List all current branches with the branch command. An asterisk (*) will appear next to your currently active branch:

* `git branch`

Create a new branch. You will remain on your currently active branch until you switch to the new one:
* `git branch new-branch`
  
Switch to any existing branch and check it out into your current working directory:
* `git checkout another-branch`

You can consolidate the creation and checkout of a new branch by using the -b flag:
* `git checkout -b new-branch`
  
Rename your branch name:
* `git branch -m current-branch-name new-branch-name`

Merge the specified branch’s history into the one you’re currently working in:
* `git merge branch-name`

Abort the merge, in case there are conflicts:
* `git merge --abort`

  You can also select a particular commit to merge with cherry-pick with the string that references the specific commit:
* `git cherry-pick f7649d0`

 When you have merged a branch and no longer need the branch, you can delete it:
* `git branch -d branch-name`

If you have not merged a branch to main, but are sure you want to delete it, you can force delete a branch:
* `git branch -D branch-name` Collaborate and Update

To download changes from another repository, such as the remote upstream, you’ll use fetch:
* `git fetch upstream`

Merge the fetched commits. Note that some repositories may use master instead of main:
* `git merge upstream/main`
  
 Push or transmit your local branch commits to the remote repository branch:
* `git push origin main` Fetch and merge any commits from the tracking remote branch:

* `git pull` Inspecting

Display the commit history for the currently active branch:

* `git log`
Show the commits that changed a particular file. This follows the file regardless of file renaming:

* `git log --follow my_script.py`
Show the commits that are on one branch and not on the other. This will show commits on a-branch that are not on b-branch:

* `git log a-branch..b-branch`
Look at reference logs (reflog) to see when the tips of branches and other references were last updated within the repository:

* `git reflog`
Show any object in Git via its commit string or hash in a more human-readable format:

* `git show de754f5`

#### Show Changes

The `git diff` command shows changes between commits, branches, and more. You can read more fully about it through the Git documentation.

Compare modified files that are on the staging area:
* `git diff --staged`

Display the diff of what is in a-branch but is not in b-branch:
* `git diff a-branch..b-branch`

Show the diff between two specific commits:
* `git diff 61ce3e6..e221d9c`

Track path changes by deleting a file from your project and stage this removal for commit:
* `git rm file`

Or change an existing file path and then stage the move:
* `git mv existing-path new-path`

Check the commit log to see if any paths have been moved:
* `git log --stat -M`

#### Stashing
Sometimes you’ll find that you made changes to some code, but before you finish you have to begin working on something else. You’re not quite ready to commit the changes you have made so far, but you don’t want to lose your work. The git stash command will allow you to save your local modifications and revert back to the working directory that is in line with the most recent HEAD commit.

Stash your current work:
* `git stash`

See what you currently have stashed:
* `git stash list`

Your stashes will be named `stash@{0}`, `stash@{1}`, and so on.
Show information about a particular stash:

* `git stash show stash@{0}`

To bring the files in a current stash out of the stash while still retaining the stash, use apply:
* `git stash apply stash@{0}`

If you want to bring files out of a stash, and no longer need the stash, use pop:
* `git stash pop stash@{0}`

If you no longer need the files saved in a particular stash, you can drop the stash:
* `git stash drop stash@{0}`

If you have multiple stashes saved and no longer need to use any of them, you can use clear to remove them:
* `git stash clear`

#### Ignoring Files

If you want to keep files in your local Git directory, but do not want to commit them to the project, you can add these files to your .gitignore file so that they do not cause conflicts.

Use a text editor such as nano to add files to the `.gitignore` file:
* `nano .gitignore`

To see examples of .gitignore files, you can look at GitHub’s .gitignore template repo.


#### Rebasing

A rebase allows us to move branches around by changing the commit that they are based on. With rebasing, you can squash or reword commits.

You can start a rebase by either calling the number of commits you have made that you want to rebase (5 in the case below):
* `git rebase -i HEAD~5`

Alternatively, you can rebase based on a particular commit string or hash:
* `git rebase -i 074a4e5`

Once you have squashed or reworded commits, you can complete the rebase of your branch on top of the latest version of the project’s upstream code. Note that some repositories may use master instead of main:

* `git rebase upstream/main`
To learn more about rebasing and updating, you can read How To Rebase and Update a Pull Request, which is also applicable to any type of commit.

#### Reverting and Resetting

You can revert back the changes that you made on a given commit by using revert. Your working tree will need to be clean in order for this to be achieved:
* `git revert 1fc6665`

Sometimes, including after a rebase, you need to reset your working tree. You can reset to a particular commit, and delete all changes, with the following command:

* `git reset --hard 1fc6665`
To force push your last known non-conflicting commit to the origin repository, you’ll need to use --force:

Warning: Force pushing to the main (sometimes master) branch is often frowned upon unless there is a really important reason for doing it. Use sparingly when working on your own repositories, and work to avoid this when you’re collaborating.
* `git push --force origin main`

To remove local untracked files and subdirectories from the Git directory for a clean working branch, you can use git clean:
* `git clean -f -d`

If you need to modify your local repository so that it looks like the current upstream main branch (that is, there are too many conflicts), you can perform a hard reset:

Note: Performing this command will make your local repository look exactly like the upstream. Any commits you have made but that were not pulled into the upstream will be destroyed.
git reset --hard upstream/main

#### Syncing a fork

The Setup

Before you can sync, you need to add a remote that points to the upstream repository. You may have done this when you originally forked.

Tip: Syncing your fork only updates your local copy of the repository; it does not update your repository on GitHub.

```
$ git remote -v
# List the current remotes
origin  https://github.com/user/repo.git (fetch)
origin  https://github.com/user/repo.git (push)

$ git remote add upstream https://github.com/otheruser/repo.git
# Set a new remote

$ git remote -v
# Verify new remote
origin    https://github.com/user/repo.git (fetch)
origin    https://github.com/user/repo.git (push)
upstream  https://github.com/otheruser/repo.git (fetch)
upstream  https://github.com/otheruser/repo.git (push)
```
Syncing

There are two steps required to sync your repository with the upstream: first you must fetch from the remote, then you must merge the desired branch into your local branch.

Fetching

Fetching from the remote repository will bring in its branches and their respective commits. These are stored in your local repository under special branches.

```
$ git fetch upstream
# Grab the upstream remote's branches
remote: Counting objects: 75, done.
remote: Compressing objects: 100% (53/53), done.
remote: Total 62 (delta 27), reused 44 (delta 9)
Unpacking objects: 100% (62/62), done.
From https://github.com/otheruser/repo
 * [new branch]      master     -> upstream/master
```
We now have the upstream's master branch stored in a local branch, upstream/master
```
$ git branch -va
# List all local and remote-tracking branches
* master                  a422352 My local commit
  remotes/origin/HEAD     -> origin/master
  remotes/origin/master   a422352 My local commit
  remotes/upstream/master 5fdff0f Some upstream commit
```
Merging

Now that we have fetched the upstream repository, we want to merge its changes into our local branch. This will bring that branch into sync with the upstream, without losing our local changes.
```
$ git checkout master
# Check out our local master branch
Switched to branch 'master'

$ git merge upstream/master
# Merge upstream's master into our own
Updating a422352..5fdff0f
Fast-forward
 README                    |    9 -------
 README.md                 |    7 ++++++
 2 files changed, 7 insertions(+), 9 deletions(-)
 delete mode 100644 README
 create mode 100644 README.md
```
If your local branch didn't have any unique commits, git will instead perform a "fast-forward":
```
$ git merge upstream/master
Updating 34e91da..16c56ad
Fast-forward
 README.md                 |    5 +++--
 1 file changed, 3 insertions(+), 2 deletions(-)
```
Tip: If you want to update your repository on GitHub, follow the instructions [here](https://docs.github.com/en/get-started/using-git/pushing-commits-to-a-remote-repository#pushing-a-branch)

##### Adding locally hosted code to GitHub
```
git init                                                         // init git local at your project
git add .                                  			 // stage all current files in your project
git commit -m "Fist commit"              			 // commit all staged files with message
git remote add origin git@github.com:<your_git_repo_ulr>.git  	 // link your git local to git online
git remote -v                      				 // verify whether a new remote is created
git status                               			 // verify the branch you are standing, as usually is "master"
git push origin master              				 // push your commit to reposity
```
[see also](https://superuser.com/questions/1412078/bring-a-local-folder-to-remote-git-repo)

Note: Use git remote set-url origin followed by the HTTP link, not the .git link. The .git link requires SSH access, while using the HTTP link avoids this requirement. Be sure to use the base HTTP link, excluding /tree/branch-name.

For more details, see:
[Git error: "Please make sure you have the correct access rights and the repository exists"](https://careerkarma.com/blog/git-please-make-sure-you-have-the-correct-access-rights/)



[Bring a local folder to remote git repo](https://superuser.com/questions/1412078/bring-a-local-folder-to-remote-git-repo)

[Push Branch to Another Branch](https://devconnected.com/how-to-push-git-branch-to-remote/)


[How To Use Git: A Reference Guide](https://www.digitalocean.com/community/cheatsheets/how-to-use-git-a-reference-guide#git-cheat-sheet)

[GitHub Training Kit](https://training.github.com)

[github-git-cheat-sheet](https://training.github.com/downloads/it/github-git-cheat-sheet/)

[How to write a good commit message](https://dev.to/chrissiemhrk/git-commit-message-5e21)













