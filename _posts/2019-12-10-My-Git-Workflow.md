---
layout: post
title: My Git Workflow
---

I worked at [Cerner](https://en.wikipedia.org/wiki/Cerner) for more than 5 years. For the last 3 and a half, I was on a team of more than 30 people, many of which were entirely new to [Git](https://en.wikipedia.org/wiki/Git) after a switch from [Subversion](https://en.wikipedia.org/wiki/Apache_Subversion). Personally, I was new to source control in general. We ran into problems constantly.
  * At first, we were using Git 1.X. Before 2.0, `git push origin` would push all branches in your local repository to the remote repository. Someone `git push -f origin`ed at least twice, overwriting our `master` and `wip` branches. Each time, leadership scrambled to find out who had pulled changes most recently so they could push those back to the remote repository.
  * Developers working on features dependent on features in progress often branched off of their teammates branches (with or without discussing it first) only to find, when they tried to update their branch, that their base branch's commit history had been modified, and/or the branch had been merged then deleted.
  * Merge conflicts were a daily part of life. A few of our files were 800-1200 lines long, so multiple developers working even just in the same module were likely to encounter conflicts every time they tried to get the most recent changes.
  * Some developers didn't attempt to retrieve changes often and, when they were ready to open a code review, would be faced with hours of work attempting to pick apart their 