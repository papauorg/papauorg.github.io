---
layout: post
title:  "Everyday git commands"
description: Some of my most used git commands and useful aliases for every day use.
categories: git development commandline cli
---

As a software developer I use git as my version control system almost every day. I really like git and its flexibility. Here I want to share my git aliases and the commands I use every day.

I like to use the command line a lot, as it gives you a lot of flexibility and a better understanding in what's happening compared to some GUIs. So the main work with git is done in the console. There
are a few exeptions. I am using a GUI merge tool for example. But most of the time I'm working with git in the command line.

To make things easier I use quite a few aliases. Some of which are commonly known and used by fellow devs. But maybe here are some that you didn't know, yet.

## git checkout
The alias I use for the checkout command is `co`. It is very simple and commonly used by many people. The nice thing is, this one alias
already gives a lot of benefits. Create it using:
```bash
git config --global alias.co checkout
```

### Create a branch
Usually the first time I can use this new alias is to create a new branch before starting to code. For example to create a new feature branch.
```git co -b feature/some-feature```
Note the `-b` switch which allows you to switch to a branch that gets created in the process. Short for `git branch feature/some-feature` and `git checkout feature/some-feature`.

### Undo changes
Next is to undo changes on tracked files
```bash
git co -- .
```

### Switch branches
What I really like is this little gem:
```bash
git co -
```
The `-` switches back to the previous branch. It's the `Alt`+`Tab` of branches.

Additionally I created the `coma` alias for directly switching to the master branch regardless of where I was before.
```bash
git config --global alias.coma 'checkout master'
```
{% include image.html name="git_checkout_shortcuts.png" caption="git checkout shortcuts" class="image-center" %}

## git add
I like to check the changes I made before adding them to the staging area. When I did that by using a gui tool I was annoyed by opening the compare tool, reviewing the
changes, closing the compare tool and then adding the file to the staging area. It's just too many steps and usually I have to switch to the mouse.

As an alternative there is this fine command:
`git add . -p`

{% include image.html name="git_add_patch.png" caption="git add patches" class="image-float-right" %}
The `-p` switch causes the add command to split up changes in the files into small hunks, that can be reviewed and accepted or rejected individually in the command line.
It asks for each hunk what it should do with it and you can choose if you want it in your commit by simply using the `y` or `n` key. There are some more options, but those
are the ones that are most relevant to me.

Especially for a lot of small changes in many files this is a lot faster than opening every file and adding it to the statging area. An additional advantage is of course
that you can add only parts of a file to the next commit.

See  [git add documentation](https://git-scm.com/docs/git-add){:target="_blank"} for more details.

## git rebase
I like clean history, that is why I do a rebase on my feature branches before creating my pull requests. Often the branch I rebase against is the
master branch. I created a `rema` alias, that switches to the master branch, pulls the latest commits, switches back to the previous branch and then
executes a rebase onto the master branch.
```bash
git config --global alias.rema !'git checkout master && git pull && git co - && git rebase master'
```
The git aliases that start with a ! are treated like shell commands. The above command runs under my linux system. If you want to be able
to use it with powershell, you might have to replace the `&&` with a `;`.

Sometimes it is necessary to resolve conflicts. For this I use the `git mergetool` command or the alias `mt`.
```bash
git config --global alias.mt mergetool
```

When a merge is successful the `git rebase --continue` comes to play. The rebase creates *.orig files per default. To remove them and continue
the rebase I have an alias `rc`. I know that the creation of the orig files can be surpressed, but they are quite handy sometimes. So I still
want them created but removed if I continue the rebase.
```bash
git config --global alias.rc !'git clean -f && git rebase --continue'
```
Again, use `;` instead of `&&` on windows.

## git push
When the feature is finished or ready to be reviewed it must be published to the remote repository.
The command
```bash
git push --set-upstream origin feature/branch-name
```
can be used for that, but is quite verbose. I don't want to type that much. So I use an alias `pu`:
```bash
git config --global alias.pu !'git push --set-upstream origin $(git rev-parse --abbrev-ref HEAD)'
```

### Open WebUI for creating pullrequests (Linux only)
The next step for me is usually to switch to the web view and create a pull request for my newly published branch.
To speed up this process a little bit, I use an alias `pr`. It reads the URL from the origin remote and builds
the URL to the `Create pull request` view of my provider. Because I don't want to install additional tools I tried
to solve this only with git aliases and configs. That is why this step may seem a little complicated. Additionally it
only works with a linux bash because the aliases use e.g. `sed`.

The steps required are:
1. Get the remote URL
2. Try to resolve ssh URLs to https URLs
3. Append the path for the pull request creation view to this URL
4. Replace the $branch and $base placeholders in the URL templates with the actual branch names
5. Open the URL with the systems default browser

### step 1 & 2
```bash
git config --global alias.getbaseurl !'git remote get-url origin | sed -r "s/git@(.+?):(.+?)\\.git/https\\/\\/\1\\/\\2/"'
```

### step 3
To provide paths for different providers, in my case github and gitlab. I created config entries per provider and a `default`
which can be set globally and overwritten per repository if needed.

The URLs may contain placeholders. Those will get replaced. `$base` will be replaced with `master` and `$branch` with the currently checked out branch name.

```bash
git config --global pullrequesturls.github '/compare/$base...$branch?expand=1'
git config --global pullrequesturls.gitlab '/merge_requests/new?merge_request[source_branch]=$branch&merge_request[target_branch]=$base'
git config --global pullrequesturls.default pullrequesturls.github
```
The default is the used in combination with the command that gets the base URL from step 1.
```bash
git config --global alias.getpullrequesturl !'echo "$(git getbaseurl)$(git config $(git config pullrequesturls.default))"'
```

### step 4
Here it gets tricky. We will need to replace the `$branch` and `$base` placeholders. Calling sed with variables is doable, but
I found myself in quote escaping hell while trying to create the alias command for it. So please just insert this line manually
in your git config file and please don't be mad at me.

```ini
[alias]
    replacebranches = !git getpullrequesturl | sed 's,$branch,'\"$(git rev-parse --abbrev-ref HEAD)\"'', | sed 's,$base,master,'
```

### step 5
Put it all together and create a nice short `pr` alias. `sensible-browser` uses the systems default browser to open the URL.
```bash
git config --global alias.pr !'sensible-browser $(git replacebranches)'
```

## dotfiles
There are a lot more aliases that I have configured and that I use from time to time. The above ones are especially noteworthy in my opinion.
You can have a look at my [dotfiles](https://github.com/papauorg/dotphiles/blob/master/git/gitconfig){:target="_blank"} if you want to view the other aliases and configs that I have.

By the way if you are a linux user you should have a look at dotfiles in general because it is really awesome to have your config files in a git repository.
