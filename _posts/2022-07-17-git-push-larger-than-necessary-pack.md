---
layout: post
title:  "Why Would Git Push a Larger than Necessary Pack"
date:   2022-07-17 16:08:52 -0400
categories: git git-push
---

## Git push hoping for the best
In my time pretending to be an engineer and working with git at Twitter, I've seen an interesting behavior pop up intermittently. People start complaining about git-push being slow. This particular issue becomes hard to diagnose, especially since the pandemic because we can't be certain of the quality of connection being used, and optimizations to `git-push` has always taken a back seat to all the other changes we've done to git internally.
But it has persisted long enough that it needed some deeper diving into, and the intermittent nature always fascinated me. Let's talk about the problem a little more.

## Problem
You create a topic branch off a synced up `master/main`. Your commit consists of a single line change and you proceed to `git push` your
branch and notice that the pack being pushed is much larger than what your intuition tells you it should be.

Imagine having a single line change and then uploading a pack worth 1GiB+ (in this economy!?)

## Prerequisite reading
I'll do my best to keep this easy to read but it would help the reader to skim through [how does git push work](https://kdspaul.github.io/git/git-push/2021/09/06/how-does-git-push-work.html) before reading this post.

## Let's Dive In
I’ve seen this particular problem happen with a non-trivial amount of repos. Most of these repos are active repos with plenty of topic branches. We also know that the problem comes in waves, sometimes the bigger than normal pack is still small enough that it does not raise concerns while other times it is so big that it causes failures.

Let’s create an example to help understand the problem. I’ll be using double digits to represent SHAs for ease of reading. Let’s assume we just have a single branch (`main`) and it has 10 commits [^1]:
```
main: 10 -> 09 -> 08 -> 07 -> 06 -> 05 -> 04 -> 03 -> 02 -> 01
```

We clone that repo locally and start create a branch off `main` and create 2 commits:
```
Main: 10 -> 09 -> 08 -> 07 -> 06 -> 05 -> 04 -> 03 -> 02 -> 01
Topic: a12 -> a11 -> 10 -> 09 -> 08 -> 07 -> 06 -> 05 -> 04 -> 03 -> 02 -> 01
```

When we push `topic` to the server, it will pack the objects referenced by a12 and a11 and just those objects. [^2]

#### Now, let’s assume a different scenario
This time, like before, we create a branch off `10` and add 2 commits but this time `main` was active and added several commits [^3].
```
(server)
Main: 30 -> 29 -> 28 -> 27 -> …10 ->…. 03 -> 02 -> 01

(local)
Main: 10 -> 09 -> 08 -> 07 -> 06 -> 05 -> 04 -> 03 -> 02 -> 01
topic-2 (locally): b12 -> b11 -> 10 -> 09 -> 08 -> 07 -> 06 -> 05 -> 04 -> 03 -> 02 -> 01
```

Now, here is what is going to happen when we try and push `topic-2`:
* Client asks the server what it has and it tells the client that it has `main` and it's on `30`.
* Client translates that information by marking ‘30’ uninteresting and since the client does not have `30`.
* Since the client has no idea about `30` it can't possibly mark anything as uninteresting.
* The client, knowing what it does, starts packing the objects.
* It packs `b12` and its parent is `b11`.
* It packs `b11` and its parent is `10`.
* < and so on >
* It packs `01`, it has no parent and now the pack is complete [^4].
* The pack now has 10 commits that really didn’t need to be there.

## Solution(s)

* It would be great if git gets a list of objects and queries the server to figure out what it needs to pack. Data is after all content addressable. (there's also the discussion about how would these objects be kept alive)
* The user could fetch, as long as the tips match, we can use that information to mark commits uninteresting which would help us to shrink the size of the pack that we’re pushing. This could work but in practice the miss rate for this strategy is high. Again, it’s going to depend on activity in your repo.
* We could add refs in the repo at some cadence that could serve as sync points.
** Let’s say that the repo in our example created 5 commits an hour and it took 4 hours for those 20 commits to land.
** A utility could create a branch every 30 minutes. We would have a ref that points to 1, a ref that points to 5, 10, 15 and so on.
** With all those refs, when the client asks the server what it has it would have a ref that points to 10 and we would end up pushing only b12 and b11.
* My co-worker [Hao Luo](https://www.linkedin.com/in/hao-luo-a1a48344/) pointed out that there is [new feature](https://patchwork.kernel.org/project/git/list/?series=477041&state=*) that uses similar algorithm to fetch negotiation to walk the commits to reduce pack size. Thanks to Jonathan Tan for the feature, we'll definitely be looking at this internally to harden it so it's a net positive for the repos.





###### Footnotes
[^1]: If you want to do this example at home, here is how I was creating files ```head -c 1048576 < /dev/urandom  >filename && git add filename && git commit -m filename```

[^2]: We're pushing 2MiB which is what we'd expect ``` Writing objects: 100% (6/6), 2.00 MiB | 3.79 MiB/s, done. ```

[^3]: If you're following along at home, anytime you're making changes to `main` do it from a different copy of the repo. Otherwise, you will get different results

[^4]: ``` Writing objects: 100% (36/36), 12.01 MiB | 987.00 KiB/s, done. ```

