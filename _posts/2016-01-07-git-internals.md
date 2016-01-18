---
layout: post
title: "Git FAQs"
description: "Just some light on how Git works"
category: web
tags: [Threads, AsyncTask, Android]
subheading: Answers to some basic questions that came every time i did git commit or git pull etc.
permalink: web/git-interals
---
This post answers some of the basic questions that came every time i used Git.So finally after almost a year after  i included this task of writing about Git into my Trello board,i am doing this. So lets dive in straight.

## The Basics?
Before starting about the details,let us know that there is a Git object model and there are 4 types of objects in it. These objects are stored in `.git/objects` directory<br>
-> **blob** - Every file in Git i.e source code file or image file etc. is stored as a blob.It can be understood as just a bunch of bytes stored together.<br>
-> **tree** - Every folder corresponds to a tree in git.A tree can be thought of as a simple tree data structure which contains several nodes. In case of Git, these nodes can be blob objects or other tree objects.Blobs here are files and trees are folders<br>
-> **commit** - Every git commit that we do is also stored as an object. A git tree is actually a series of commit objects. While commit is stored as an object, other operations such as branching,pull,push etc are not stored as an object. A git commit object stores the commiter's info(name and email), the commit message, its parent commit info and the tree it is pointing to( explained in `git commit` section).<br>
-> **tag** - A tag is just to give a name to a particular commit. Tags are explained in the below sections.

### Git init
The first step of creating a git repo is `git init`. When we do this,it creates a `.git` directory whose structure is roughly as

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][git_dir]

The relevant files and folders here are<br>
**-** `index`, which is the staging area. When we add any file to a staging area, this file stores the file name(with the path of the file) and the hash value of the contents of that file along with the timestamp( explained below).<br>
**-** `objects`, which stores the four types of objects mentioned above. It creates the name of the folder and files using the content of that file. This is called Content Addressable Storage. The process of naming the file is explained below in the `git add` section.<br>
**-** `refs`, which store the references. References are actually pointers to a particular commit. Branches, remotes and tags are example of references.Unlike git objects which are immutable, references are movable.<br>
**-** `HEAD`, which stores the latest commit of the branch which you are currently on.<br>
**-** `hooks`, which are the scripts which are present locally to our system and can be configured to attach certain actions to the git operations. For example we can configure if we want to prevent commit to a particular branch or do something when a push is made to a branch.

## Tracing the operations
Now that we know about the objects in Git, lets see what happens in each of the git's operations.

### Git add
**--** When we do `git add fileName`, git calculates the 40 character SHA1 hash value of the contents of the file. Suppose the hash value of the contents comes out to be as `7bb9e552dc3b74818456099d8290018de1a9e6a5`. A file is created with its names as the last 38 characters of this value, it is placed in a directory whose name is the first two characters of this value and this file is stored in `.git/objects` directory as `.git/objects/7b/b9e552dc3b74818456099d8290018de1a9e6a5`. <br>
**--** Means two files with same contents will not be stored as two objects. Instead, only one object will be created but the `index` will contain references to both the files.<br>
**--** Now suppose, i change the contents of a file and add it to index again, then a new object will be created and and added to the index. <br>
**--** If i do `git add <folder_name>`, then all the files of the folders are converted to separate blob objects and their entry is stored in the index. Now if you have an empty folder with no files, and try to do `git add <folder_name>`, nothing will happen because here there is no file to be stored as a blob.<br>
&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][git_init]

The above diagram shows two things. 1 represents the flow that takes place when a file is added to index. 2 represents the flow when the file's contents is modified and it is added to index again. Note that a new object is formed for the same file when its contents are changed and its added to index again. Since git objects are immutable, an object here once created and stored will never be modified.

### Git commit
Now that our staging area has some blob objects, we can commit them. When we do a commit,a commit object is created and that object points to a tree which is the root node of all the other blobs and tree which represents the state of the index at that point of time. As already stated above,this object will store commiter's name,email, hash of its parent commit and hash of the tree which points to the current state of the index. If it is the first commit, it wont have any parent. If its a merge commit, it will have two parents(explained later).This diagram shows the tree structure after commits are made.

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][commit_tree]

**-** When we add the folder present in the first diagram to index and commit it, the files and folders get converted to blobs and trees respectively and the commit object points to the root tree.<br>
**-** When we add a new file to the folder `second_dir` , so the hash of `second_dir` and  `top_dir` will get changed(because its contents has changed),so objects will be created for them again. While for the rest of the objects ( `abc.txt, def.txt, efg.txt` ), the hashes will remain same, so they wont be created again this time. The new commit object can simply point to them.<br>
**-** Also, the newly formed commit object will point to the previous commit object(not shown in the diagram)because the previous one is its parent.

### Git branch
**-** A git branch is just a reference to a particular commit.A new branch is always created from a particular commit. In case we create a new branch from some other branch, it is created from the top commit of that branch.<br>
**-** When we create a new branch, a new file is created(with name as name of the branch) under `refs/heads` directory. The file contains hash value of the commit from which this branch was taken. If we make commits into this branch, then the hash value will get updated to contain the hash of the latest commit.<br>
**-** Also if we checkout this branch, then the `HEAD` file will point to this branch. For example, we created a branch `test-branch`, then `HEAD` file will contain the entry `ref: refs/heads/test-branch`

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; ![alt text][git_branch]

Lets go into the flow of the above diagram.<br>
1. Firstly, there is a commit which points to the tree which in turn holds other trees and blobs. So the `master` branch will point to this commit.<br>
2. Now we make a commit(C2) into the `master` branch. The `master` branch now points to C2.<br>
3. Then we take a branch (test-branch) from `master` branch. So the `test-branch` will now point to commit C2.<br>
4. Now we checkout the `test-branch` and make a commit(C3) on `test-branch`. So `test-branch` will now point to commit C3 and also the file `refs/heads/test-branch` will contain the hash of commit C3.<br>
5. Now we again switch to `master` branch and take out a new branch(test-branch2) from it. The `test-branch2` will point to commit C2 now. Now we make a commit(C4) into the `test-branch2` and then it points to C4.

### Git merge
Now merging of branches can happen in two ways. First is that there will be no merge commit created when we merge two branches. This is called **Fast Forward Merge**. In the other case, there is a merge commit created and we get some weird commit messages if we see it in SourceTree for example.<br><br>
**- Fast forward merge -** In the above diagram, suppose we want to merge branch `test-branch2` into `master`. Now what git will do is it will simply move the `master` pointer from commit C2 to commit C4 resulting in a Fast Forward merge and both the branches now point to commit C4. So in the tree that is formed above in the diagram, if one commit can be reached from other commit by tracing its pointer, then a FF merge is possible. So C2 and C4 can be merged using FF merge.<br><br>
**- 3 Way Merge-** But suppose we want to merge `test-branch` into `test-branch2`. Now there is no direct path between C3 and C4 that we can follow. Both of them merge at C2, which is their common ancestor.So here git follows `3-way merge` using the nearest common ancestor(C2) of two branches. Here the common ancestor is also required in the merge process because suppose there are changes at the same line in both branches (C3 and C4). Without consulting the base-branch(C2) and checking the contents at the same line in the base branch, git wont be able to automatically resolve the conflict. As here there are 3 commits to be consulted, its called a 3 way merge. Suppose Git follows a 2 way merge and doesn't consult the base-branch, then it wont be able to resolve the conflicts automatically because it doesn't whether one or both branch has changed the content on the same line.So in this case a merge commit is created and it points to both the branches which has been merged.

&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;![alt text][ff_merge]![alt text][3way_merge]

### Git rebase
Rebasing is a method to escape the unwanted merge commits that we see when we do a `3 Way Merge`.Let's see how does that work

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][git_rebase]

Considering the same graph(git branch graph)and suppose now we want to rebase `test-branch` into `test-branch2`.<br>
**-** We checkout `test-branch2` and write the command `rebase test-branch` in it.<br>
**-** Doing the above step will take all the commits(here its only C4) applied on `test-branch2` from the point `test-branch` and `test-branch2` have diverged(nearest common ancestor), and apply it one by one on `test-branch`. So after this,the tree looks like as the left image above. `test-branch2` will now point to a new commit created C4' and the commit C4 will be left dangling.<br>
**-** Now we can merge `test-branch` into `test-branch2` and it will be a `Fast Forward Merge` because the commits C3 can be directly reached from C4'. Doing a `FF merge` will take the tree structure to the point as illustrated in the right figure. This will give us a clean history without any merge commits.But ironically `rebase` is also a way to change the history as we did above in the tree.<br>
**-** The commits which are left dangling are not deleted by Git.C4 will be eft untouched, and if something goes wrong with the rebase, we can go right back to the previous state.

### Git remotes
We know that a `remote` is also a reference and it is included in the `refs\remotes` directory. And what does it points to? It points to a remote repository that may not be present in our local system.<br>
**-** When we initialize a new git repo and we also want it to be used by other people in our team for example,then we need to push it to some remote repo. This remote repo is actually called a `remote` and the default `remote` is named as `origin`.<br>
**-** So using the command `git remote add origin https://github.com/user/abc.git`, we are actually specifying the remote repo present at the given URL as our remote and naming it as `origin`.<br>
**-** Now when we push the changes using `git push origin branch`, then we are actually pushing our changes in the branch specified to the remote added in the previous step.
**-** So `origin/master` is the `master` branch present at the remote `origin`. While our local branch is just `matser`.

### Git stash
We use `git stash` when we want to save our changes without committing them. We can save all the changes that happened since the last commit by stashing them and the stash can be applied in any of the branches. Stash is just local and is never pushed to a remote.<br>
**-** Internally, a stash is also saved as a commit object. The commit object's tree records the present state of the working directory. This commit object has two parents. As obvious, the first parent is the `HEAD` commit(at the time of stashing). The second parent is the commit which records the snapshot of the staging area at that point of time.

&nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp; &nbsp;![alt text][git_stash]

**-** The stashes are saved in `refs/stash` file. But there is only one entry in that file.Actually the entry that we see belongs to the latest stash that we have done. All the other stashes are saved in the `reflogs` of that reference. `reflogs` are the reference logs that store a history of a reference. For example `stash` is a `reflog` and to get the latest stash we use the `reflog` syntax `stash@{0}` and `stash@{1}` is the one before it and so on.

### Git tags
Generally while each release, we tag the latest commit with messages like `release V1` etc. This helps us to checkout and examine the exact code that went into release in case any bugs come in that version at some point in the future.Now tags can be either `lightweight` or `annotated`.<br>
**-** A lightweight tag is just a pointer to a particular commit.It doesn't store extra information like tagger name,date etc while an annotated tag stores all this information and an annotated tag is stored properly as an object in `.git/objects` directory.<br>
**-** In any case, all the tags that are created are stored as a file in `refs/tags` directory.Each tag file in this directory stores the SHA1 hash of the commit on which the tag is applied. <br>
**-** Generally its not advised to checkout a tag and do some active development on it.Checking out a tag will put our working directory in a `DETACHED HEAD` state.

### Detached HEAD
This is an interesting state and occurs whenever the `HEAD` is not pointing to the tip of the current branch. It can happen when we checkout a commit which is not at the top of a branch. Also when we checkout a branch using its SHA1 name and not the branch name(like `git checkout 1bhd..888`), we will get a `DETACHED HEAD` state.
&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;![alt text][detached_head]

**-** In the first figure, we have checked-out `master` branch. So `master` points to the latest commit(C3) and `HEAD` points indirectly to C3.<br>
**-** Now we checkout the commit C2 and so `HEAD` will now point to C2(middle figure). At this point the `HEAD` is in a detached state as its not pointing to the tip of the branch(C3).<br>
**-** Suppose we make further commits C4 and C5 at this point. The commits will base out from C2 now. As seen in the rightmost figure, we have created an another branch of development here .Its an anonymous branch because we haven't given a name to it.<br>
**-** If we push `master` branch to `origin` now, then obviously C4 and C5 wont be pushed. If we checkout `master` again , then `HEAD` will again point to commit C3 and commits C4 and C5 will be left in a dangling state. Now since C4 and C5 doesn't have any reference from anywhere, they will be garbage collected by git next time when GC runs.

### Git reset and Git revert
Here comes this classic interview question. The difference between reset and revert.<br>

&nbsp; &nbsp; &nbsp; &nbsp;&nbsp; &nbsp; &nbsp;![alt text][git_reset]

**-** The second above below is an example of `git revert`. Suppose in the above history we want to remove the changes of `C2`. Then we will simply do `git revert C2`. This will actually create another commit(`C2'`) which will just remove the changes applied by `C2` and the commit(`C2'`) is applied at the top of the project history. It doesn't alter any other commits before or after that.<br>
**-** The third image shows `git reset C2`. This command actually removes all the commits  after the `C2` and resets the project history to `C2`. But it will actually not remove all the changes done by those commits from  the working directory.This is quite a safe operation. But if we do `git reset --hard C2`, then it becomes a dangerous operation because it will remove all the commits after `C2` and their changes from the working directory. This of course is a dangerous way to undo things because unless its desirable,it can result in permanent loss of the commits and changes from the history.


### Git checkout
Now as we have seen that Git is nothing but a tree data structure whose nodes are the files and folders of our working directory. The tree contains commits and if we want to checkout some branch or tag, Git just has to traverse the tree and give the files and folders pointed by those commits.<br>
For example, suppose we do `git checkout test-branch`. Then this flow will happen<br>
1. Git will check the entry in `refs/heads/test-branch` file. It will be pointing to a commit.<br>
2. The commit in turn will be pointing to a tree which contains in turn points to all the trees and blobs that were present in the index when the commit was done.<br>
3. Once we checkout a commit,the working directory will now have all the files and folders pointed by that commit and its lower commits. Git does this by traversing the complete tree and checking out the files. This is because the commit actually points out to the SHA1 of the blobs and the trees.<br>
4. `HEAD` will get updated to point to the `test-branch`'s' most recent commit.<br>

Once all these trees and pointers are understood, other things like `git push`,`git pull`,`git fetch` etc can be easily thought of. Git contains several other advanced features like squashing several small commits into a larger commit, changing the order of commits,rewording commits etc but they all require a separate blog for each of them.







[git_dir]: /images/git_dir.png "git directory"
[commit_tree]: /images/commit_tree.png "Commit tree"
[git_init]: /images/git_init.png "git init"
[git_branch]: /images/git_branch.png "Git branch"
[ff_merge]: /images/ff_merge.png "Fast forward merge"
[3way_merge]: /images/3way_merge.png "3way merge"
[git_rebase]: /images/git_rebase.png "git rebase"
[detached_head]: /images/detached_head.png "detached head"
[git_stash]: /images/git_stash.png "Git stash"
[git_reset]: /images/git_reset.png "Git reset"
