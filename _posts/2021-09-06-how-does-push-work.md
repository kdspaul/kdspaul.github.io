---
layout: post
title:  "How Does Push Work"
date:   2021-09-06 16:08:52 -0400
categories: git git-push
---

## We’re going to add details to our intuition about how git push works.

Let’s start with a repo:
```
$ git status
On branch main
Your branch is up to date with 'origin/main'.
```

The repo only has a single ref, a single commit and a single file:
```
$ git log -1
commit 9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09 (HEAD -> main, origin/main, origin/HEAD)

$ git rev-list --all --objects
9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09
1f06f8ce5f6866c0ffcf060e8d6cfec6e92de551
a07b18b2b3da82149f9caac7ea00dcdbe8a5f831 README.md
```

Now, Let’s create a branch and add a few commit to that branch:
```
$ git checkout -b test-branch
$ echo "appending data to README.md" >> README.md
$ git commit -am 'added more data to README.md'
$ echo "file1 contents" > file1
$ git add file1
$ git commit -m 'Added file1'
```

We should have two commits now, we're ready to push:
```
$ git log --oneline
069b2ab (HEAD -> test-branch) Added file1
240e059 added more data to README.md
9804d7e (origin/main, origin/HEAD, main) Initial commit
```
---

### With the setup done, we are ready to dig into what happens behind the scenes.  

By intuition, we understand that git-push would need to create a pack with everything that we have but the server does not, namely ‘240e059’ and ‘069b2ab’.
Here is our push command:

```
$ git push -u origin test-branch
..........
```

First, git will get refs as known by the server. This happens via a transport helper invoking git-receive-pack on the server side ([transport.c:transport_push](https://github.com/git/git/blob/master/transport.c#L1266), [remote-curl.c](https://github.com/git/git/blob/master/remote-curl.c#L538)).  
If we use GIT_TRACE and GIT_TRACE_PACKET, we’ll see the following lines:

```
15:25:04.961819 pkt-line.c:80           packet:          git< # service=git-receive-pack
15:25:04.961867 pkt-line.c:80           packet:          git< 0000
15:25:04.961878 pkt-line.c:80           packet:          git< 9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09 refs/heads/main
```

That is what is known to the server. We can see that the server does not know about our newly created local branch.
git-send-pack will now call pack-objects with this information. This is the data is passed to pack-objects:

```
069b2abf7b075742a1e74464f85697b4dae662b0
^9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09
```

Pack-objects reads this information and interprets this as packing everything that `069b2ab` points to **minus** everything that `9804d7e0` points to. Pack-objects gets to work, reads the input and for each line marks objects uninteresting where it can.  

For `069b2ab`, it will read the sha, parse it and since it’s not preceded with a ‘^’, it is interesting, no more work needs to be done here.  

For the second line `^9804d7e0`, we parse it, we know it’s a commit, we mark it uninteresting. Since `9804d7e0` is uninteresting, its parent would be uninteresting as well so we mark it uninteresting as well. Let’s mark edges uninteresting as well. For example, we know `9804d7e` is uninteresting, we should be able to mark everything that’s reachable that that commit:

```
$ git cat-file -p 9804d7e0
tree 1f06f8ce5f6866c0ffcf060e8d6cfec6e92de551

$ git cat-file -p 1f06f8ce5
100644 blob a07b18b2b3da82149f9caac7ea00dcdbe8a5f831    README.md
```

So, at this point, we’ve marked `9804d7e0` (the commit), `1f06f8ce5` (tree) and `a07b18b` (blob) uninteresting.

Now, let’s traverse commits that are interesting so we can add objects that need to be sent to the server. We know `069b2ab` is interesting, we look at its parent to see if that is interesting as well.

```
$ git cat-file -p 069b2ab
tree 06fec2a9da8539e5c25730ee75a95c6d8890754f
parent 240e0591c61bc3bbaa6d4144d7040867246fd7f2
```

`240e059` is not the list of objects we had marked uninteresting before so this must be interesting. Let’s go further:

```
$ git cat-file -p 240e059
tree 7dbfcd222f09027a582d99e637cfb5cbb72d6fd0
parent 9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09
```

`9804d7e0` is familiar, it’s uninteresting so we stop our traversal at this point.

---

### We are now ready to start packing objects.
With objects marked uninteresting, we have two commits that we need to pack: `069b2ab` and `240e059`.  
For each of them, we will look at what they refer to and add them to the pack that we need to send to the server. 

Let’s start with `069b2ab`. We know, we need to add the commit itself (object_count = 1). Let’s see what the commit points to:
```
$ git cat-file -p 069b2ab
tree 06fec2a9da8539e5c25730ee75a95c6d8890754f
parent 240e0591c61bc3bbaa6d4144d7040867246fd7f2
```

We’ve found a tree, let’s parse it, add it to pack (object_count = 2) and mark it as _seen_. Let’s look at what is inside the tree:
```
$ git cat-file -p 06fec2a9
100644 blob 8a02c371eb297122078205497c505c8ac7f8630c    README.md
100644 blob 84d55c5759cf6b954e16c54527ca94af4c1bce69    file1
```

We found two blobs, and since they are not uninteresting we will add to the the pack (object_count = 4) and mark them _seen_.

Let’s look at `240e059`. We know we need to add the commit itself (object_count = 5). Let’s look at what the commit has
```
$ git cat-file -p 240e059
tree 7dbfcd222f09027a582d99e637cfb5cbb72d6fd0
parent 9804d7e0cd1880e45c92a2f04419d7d1e6ee9e09
```

As we did before, we are going to add the tree (object_count = 6) and everything that it has that is interesting and has not been seen before:
```
$ git cat-file -p 7dbfcd2
100644 blob 8a02c371eb297122078205497c505c8ac7f8630c    README.md
```
`7dbfcd2` has a single blob `8a02c37`. This blob isn’t uninteresting but we don’t add it because it’s marked _seen_.

We create a pack and that's sent over to the server:
```
Writing objects: 100% (6/6), 559 bytes | 559.00 KiB/s, done.
Total 6 (delta 0), reused 0 (delta 0)
15:25:04.995596 pkt-line.c:80           packet:          git< PACK ...
15:25:04.995914 pkt-line.c:80           packet:          git> 0000
```

I’ve glossed over a lot of stuff but hopefully this gives a bit more detail to things you already knew about how git-push does its job.
