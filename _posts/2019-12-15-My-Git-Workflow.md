---
layout: post
title: My Git Workflow
---

- [Rambling](#rambling)
- [Workflow](#workflow)
  * [Happy Path](#happy-path)
    + [Pull Changes](#pull-changes)
    + [Add your feature](#add-your-feature)
    + [Clean Up Your Commit History](#clean-up-your-commit-history)
    + [Rebase With The Base Branch](#rebase-with-the-base-branch)
  * [Dealing With Conflicts](#dealing-with-conflicts)
    + [Fixup And Rebase](#fixup-and-rebase)
    + [Branch and Cherry Pick](#branch-and-cherry-pick)
  * [Most Importantly, Relax](#most-importantly-relax)
    + [Git Reference Logs](#git-reference-logs)
    + [Recover](#recover)
  * [When All Else Fails](#when-all-else-fails)

## Rambling
I worked at [Cerner](https://en.wikipedia.org/wiki/Cerner) for more than 5 years. For the last 3 and a half, I was on a team of more than 30 people, many of which were entirely new to [Git](https://en.wikipedia.org/wiki/Git) after a switch from [Subversion](https://en.wikipedia.org/wiki/Apache_Subversion). Personally, I was new to source control in general. We ran into problems constantly.
  * At first, we were using Git 1.X. Before 2.0, `git push origin` would push all branches in your local repository to the remote repository. Before we figured out branch protection, Someone `git push -f origin`ed at least twice, overwriting our `master` and `wip` branches. Each time, leadership scrambled to find out who had pulled changes most recently so they could push those back to the remote repository.
  * Developers working on features dependent on features in progress often branched off of their teammates branches (with or without discussing it first) only to find, when they tried to update their branch, that their base branch's commit history had been modified, and/or the branch had been merged then deleted.
  * Merge conflicts were a daily part of life. A few of our files were 800-1200 lines long, so multiple developers working even just in the same module were likely to encounter conflicts every time they tried to get the most recent changes.
  * Some developers didn't attempt to retrieve changes often and, when they were ready to open a code review, would be faced with hours of work attempting to pick their changes apart from other features. Once done, they'd almost certainly be left with a healthy chunk of broken unit tests. It's also possible more changes had been merged while they were busy...
  * One contractor even chose to pick all of his changes, then delete any broken unit tests. I don't think he was allowed back in the office once we figured out where all of our missing features went.

## Workflow
Most workflows I've used attempted to follow (with varying degrees of success) the [Git flow](https://nvie.com/posts/a-successful-git-branching-model/) model. Although that post is thorough, it gives a view that may be a little overwhelming, especially to a developer who just wants a workflow that'll minimize conflicts and time spent fighting Git.

To paraphrase Git flow, there will likely be one or more branches in your repository that are considered "main" or "central". Names for these include `master`, `release`, `staging`, `develop`, `wip`, and the like. They typically represent the codebase at various stages of release. Which of these you branch off of depends on what you're trying to achieve, but hopefully you at least know which you should be using as your base branch.

### Happy Path
A quick summary of the happy path work flow should look something like this:

#### Pull Changes
Check out your base branch, pull any recent changes, and start a new branch for your work.

![Pull Changes](/images/2019-12-15-My-Git-Workflow/1.checkout-master.png)

It's important to note here that I've prefixed my branch name. `feature` is a prefix from [Git flow](https://nvie.com/posts/a-successful-git-branching-model/) that signifies from where I'm branching and what I intend to do in this branch. `GH-1` signifies that this branch is related to GitHub issue #1. If you were using something like JIRA, it would read `JIRAQUEUE-1` instead. This provides traceability back to the requirements of the feature this branch will implement. Ideally, all branches should be prefixed with at least an issue number, especially in a professional setting.

#### Add your feature
Write your tests, fix your bugs, etc. At this point, feel free to make as many commits as you'd like. Make them and push them often, especially before you leave work for the day. You never know what might happen to your local copy of the work you've done, or the computer on which it resides.

Once you think you're ready for a code review, take a look at your commit history. Personally, I prefer `git log --pretty=oneline`. Its output looks like this:

![One Line Log](/images/2019-12-15-My-Git-Workflow/2.one-line-log.png)

#### Clean Up Your Commit History
Looking at your Git log, count how many commits you've made, maybe add 1 or 2 to that number to have some context, then run `git rebase -i HEAD~X`. This will allow you to edit your commit history starting from X commits before your current commit. Depending on what the history looks like before your changes, you should see something like this:

![Interactive Rebase](/images/2019-12-15-My-Git-Workflow/3.interactive-rebase.png)

The order of commits is opposite of what you saw in `git log`. Figure out which commits you would like to merge, which are junk, and which need better messages. Replace the word `pick` with `f` (for `fixup`) to fold junk commits into ones above them. Also, feel free to replace `pick` with `r` (for `reword`) to change a commit message if it isn't very helpful.

![Interactive Rebase Selections](/images/2019-12-15-My-Git-Workflow/4.interactive-rebase-selections.png)

Save and close the text file when you're done. If you've opted to reword something, a new file will show up giving you the opportunity to reword that commit. Close the file when you're done.

Once that's done, your commit history should look much nicer:

![Clean History](/images/2019-12-15-My-Git-Workflow/5.clean-history.png)

Note that throughout this process I've been tagging all of my commit messages with `GH-1:`. Along with providing traceability like `GH-1` in the branch name does, adding issue tags in commit messages provides extra features within GitHub and even for 3rd party integrations like JIRA and Crucible.

Here's a specific example. When I open a pull request for this branch, each commit is tagged with a link that, when clicked, takes me directly to the issue to which this feature branch belongs:

![Pull Request](/images/2019-12-15-My-Git-Workflow/6.pull-request.png)

#### Rebase With The Base Branch
Before you open a code review or pull request, you need to check on master to incorporate any changes that have been made to it while you were working.

![Rebase Master](/images/2019-12-15-My-Git-Workflow/7.rebase-master.png)

Hopefully, the result should look something like that. Now, unless there have been absolutely no changes in your base branch, in order to get your changes updated in the remote repository you're going to need to run `git push -f origin/<branchName>`. This is called a "force push" and will be destructive to (that is, it will overwrite) the branch in the remote repository. If you'd prefer NOT to do that, feel free to create another branch at this point. Maybe name it something like, `feature/GH-1-Write-Git-Workflow-Post-REVIEW` and push that instead. You could even do this *before* starting the rebase instead of after it's completed.

If everything went well and there were no conflicts, go ahead and open your review/pull request. If not, read on.

### Dealing With Conflicts

#### Fixup And Rebase
Dealing with conflicts can be a hassle, especially if another person has been working in the area to which you've made changes. However, with fixup and rebase, you can minimize the time and effort needed to handle conflicts and get your code into review. The most important thing to remember is to fixup before you begin working with your base branch again. If you already have your rebase with the base branch in progress, you can abort that by running `git rebase --abort`.

When you *do* rebase, you will need to resolve any code conflicts that your code has with your base branch, one commit at a time. This can lead to you repeatedly resolving the same code conflicts over and over again. If you find yourself in this situation, stop and try to figure out what's going wrong. Don't try to trudge through too many commits worth of conflicts. Ideally, you should have to resolve conflicts once, and only once. Revisit [Clean Up Your Commit History](#Clean-Up-Your-Commit-History) for help with that.

Do your best to stick to the Happy Path strategy above and you should be fine. If you find yourself getting into an even worse mess though, abort your rebase and try Cherry Pick

#### Branch and Cherry Pick
Sometimes rebasing just... has issues. It may even be because of something out of your control or understanding, like forking commit ancestry. If you stick to fixup and rebase, you should be okay. But if you're not, try cherry-picking. Now, this is optional, but if you fixup until you have only one commit you will only have to cherry-pick once. If you have more commits, you will need to go through the cherry-pick process once for each commit.

![Clean Cherry Pick History](/images/2019-12-15-My-Git-Workflow/8.clean-cherry-pick-history.png)

Once you have your branch's number of commits minimized, take note of the commit hash ID in the Git log (b8dd946ea631a96fbaf6e3a878299ab3d1bd9633 in the picture shown above). When you have that safe, checkout master, pull any changes, and create a new branch.

![New Cherry Pick Branch](/images/2019-12-15-My-Git-Workflow/9.new-cherry-pick-branch.png)

Once that's done, run cherry-pick with your commit hash, `git cherry-pick b8dd946ea631a96fbaf6e3a878299ab3d1bd9633`

![Cherry Pick](/images/2019-12-15-My-Git-Workflow/10.cherry-pick.png)

Most likely, you won't get a nice response like that, because if you're trying to cherry-pick you're probably having trouble with conflicts. Either way, cherry-pick will guarantee that that you only have to deal with conflicts a finite and expected number of times. Good luck.

### Most Importantly, Relax

#### Git Reference Logs
Dealing with Git can get a little stressful, especially for people unfamiliar with it. Occasionally, developers will even reach a point in their Git flailing that they think their code is, "gone" which is definitely not the case. Always keep in mind that *every* action you take is recorded in a Git log. Use `git reflog` to take a look:

![Cherry Pick](/images/2019-12-15-My-Git-Workflow/11.git-reflog.png)

As you can see from this repository's history, I've been doing a lot of thrashing around trying to demonstrate and screenshot various situations.

#### Recover
All of those identifiers on the left are the SHA hashes of states that Git has recorded. Any of them can be returned to if necessary. If you find yourself lost or think that you've done something terrible, take a look at the reference logs. Grab an identifier that you think looks like the last place you knew what you were doing. To visit that state temporarily, run `git checkout <SHA>`. If you like what you see, you can create a new branch with `git checkout -b <branchName>` or force your branch back to that state by checking it out and running `git reset --hard <SHA>`. This will take your branch back to that commit and allow you to figure out a new strategy.

### When All Else Fails
If you find yourself in a place that this workflow and its recovery strategies can't handle, you've probably learned a lot about Git on the way there. You should know enough to navigate this handy flow chart from [Justin Hileman](http://justinhileman.info/article/git-pretty/) to a successful resolution:

![Cherry Pick](/images/2019-12-15-My-Git-Workflow/12.git-pretty.png)
