# Migrate To Git (from SVN) Experiments

ProjectX is currently in svn.  It has long history, dating back to November 1, 2011, with 6000+ commits....for just the ProjectX application code, not including the svn projects for the build system.


I recently took on the task to see how we can migrate the application code to git.  This is part of a larger effort to migrate ProjectX to Centos7. (Centos6 is going end of life this month).

After discussions with Bob, we agreed that we should follow the steps in the Atlassian guide:
https://www.atlassian.com/git/tutorials/migrating-overview

After reading this boiled down to using a Java jar tool to do the author scraping and [git-svn](https://git-scm.com/docs/git-svn) to migrate svn.  The jar file is also used for clearing branches if/when you chose to drop svn support (git-svn shows how to use git and svn in parallel).

I did not have luck with the jar file to extract the authors data, and eventually found this [workaround](https://bitbucket.org/atlassian/svn-migration-scripts/issues/4/svn-migration-scriptsjar-authors-fails).  The problem was with using the jar for svn+ssh access to svn.  I used the following command line on an svn branch (newproject) and hand edited the remaining values for authors.txt:

<pre>
svn log --quiet | grep "^r" | awk '{print $3}' | sort | uniq > authors.txt
</pre>

authors.txt:
<pre>
betty = Betty Jo <betty.jo@oldco.com>
dang = dang <dang>
bobj = Bob Jones <bob.jones@oldco.com>
ian = ian <ian>
jason = jason <jason>
jgersch = jgersch <jgersch>
rperez = rperez <rperez>
trevor = trevor <trevor>
wade = wade smith <wade.smith@oldco.com>
</pre>

## Summary of Migration Attempts

### Commit as New Repo
As a baseline, these steps were used to create a git repo from the svn newproject branch.  Not idea since it loses all history, but I wanted a baseline size.  85MB.

<pre>
git init
git add -A
git commit -m "initial commit"
</pre>

### Migrate Single Branch with History
svn-git has a method to get a single branch with all the history, [link](https://gist.github.com/trodrigues/1023167). The command line is below.  This produced a single git repo of 6.4GB, with all the history.

<pre>
time git svn clone --authors-file=authors.txt -T branches/newproject svn+ssh://joesomebody@192.168.127.42/repos/products/dnsManager newproject
</pre>
Snippet of history:
<pre>
Author: Bob Jones <Bob.Jones@oldco.com>
Date:   Fri Nov 6 17:52:00 2020 +0000

    branching newproject (1.19 dev branch) from the prevproject 1.18.1 D tag point r73249
    
    git-svn-id: svn+ssh://192.168.127.42/repos/products/dnsManager/branches/newproject@73487 48e3370b-ab0b-0410-a26a-c6b7e01d70ed

commit 63800184b13b7a6cd2a828d53acfd4977eb5cb24
Author: Bob Jones <Bob.Jones@oldco.com>
Date:   Tue Sep 8 21:33:58 2020 +0000

    Updated build directory to match the latest ISO ID
    
    git-svn-id: svn+ssh://192.168.127.42/repos/products/dnsManager/branches/prevproject@73249 48e3370b-ab0b-0410-a26a-c6b7e01d70ed
    ...
</pre>

### Full Migration
I attempted the full git-svn migration.  It ran for 16 hours, produced 23GB, but failed on an unknown author.  I will re-attempt at some later date.
<pre>
time git svn clone --authors-file=authors.txt  svn+ssh://joesomebody@192.168.127.42/repos/products/dnsManager
</pre>

## How git-svn allows continued access to svn
See this [link](https://git-scm.com/docs/git-svn), specifically look for fetch and dcommit.  I have not tried this yet.

## Attempts to Reduce Size
I wanted to see if we could reduce the size of the git-svn migration of just newproject (recall, 6.4GB with all history).

### Limit history
I found this [link](https://passingcuriosity.com/2017/truncating-git-history/) showing how to reduce history.  The steps worked on a copy of the single-branch migration (newproject2) but did not reduce the size of the .git folder.

### Repack git
I found another link showing how to repack the git history, [link](https://stackoverflow.com/questions/5613345/how-to-shrink-the-git-folder).  Again I tried this on a copy of the single-branch migration (newproject3), but this didn't work.  In fact, it increased the size to 7.2GB.

### How to inspect largest files in git history
I tried to push the single-svn-branch migration to our local Enterprise github, and it failed because there were several files > 100MB (error) and several > 50MB (warning).

I found this [link](https://stackoverflow.com/questions/10622179/how-to-find-identify-large-commits-in-git-history) that shows how to list specific files:
<pre>
git rev-list --objects --all \
| git cat-file --batch-check='%(objecttype) %(objectname) %(objectsize) %(rest)' \
| sed -n 's/^blob //p' \
| sort --numeric-sort --key=2 \
| cut -c 1-12,41- \
| $(command -v gnumfmt || echo numfmt) --field=2 --to=iec-i --suffix=B --padding=7 --round=nearest
</pre>
This works great to see large files.

### How to remove specific files permanently
This [link](https://docs.github.com/en/free-pro-team@latest/github/authenticating-to-github/removing-sensitive-data-from-a-repository) shows how to permanently remove specific files:
<pre>
git filter-branch --force --index-filter \
  "git rm --cached --ignore-unmatch PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA" \
  --prune-empty --tag-name-filter cat -- --all
</pre>

**NOTE** The PATH-TO-YOUR-FILE-WITH-SENSITIVE-DATA parameter can be a directory, like vm/\*.  However, this will only match a single directory level.


### How newproject git was reduced
The original single svn branch was 6.4GB.  Using the "git rev-list" line above, we could see many files and subdirectories that were deleted.  Then using line above to permanently remove files, the following were removed:

<pre>
  --ignore-unmatch vm/*.*"
  --ignore-unmatch lib/*"
  --ignore-unmatch lib/*/*"
  --ignore-unmatch tests/*"
  --ignore-unmatch tests/*"
  --ignore-unmatch builder/builds/*"
  --ignore-unmatch builder/builds/*.bin"
  --ignore-unmatch builds/*.gz"
  --ignore-unmatch builds/*"
  --ignore-unmatch new_tests/*"
  --ignore-unmatch new_tests/*/*"
  --ignore-unmatch builder/src/inactive_distros/*"
  --ignore-unmatch builder/src/inactive_distros/*"
  --ignore-unmatch builder/src/distros/CentOS6*"
  --ignore-unmatch builder/src/distros/CentOS6*.rpm"
  --ignore-unmatch old_files"
  --ignore-unmatch old_files/*"
  --ignore-unmatch old_files/*/*"
  --ignore-unmatch deps/*.deb"
  --ignore-unmatch deps/*.deb"
</pre>

**FINAL TRICK**  After removing all those files, the total size of the local directory was even larger that 6.4G.  The "Checklist" section of [this](https://git-scm.com/docs/git-filter-branch) document explains why.  Using a local "git clone":
<pre>
git clone file:///home/joesomebody/GitMigration/newproject01 newproject02
</pre>
The size of newproject02 was only 108MB.

Then to commit, we needed to copy the newproject00/.git/config file to newproject02.


# Migrate To Git (from TFS) Thoughts

Recently another project has the need to transition from Microsoft TFS (Teams Foundations S...).  From
some breif research, it appears TFS has some build in migration to git that this team may have used
for migrating other projects.  Appearently they also ran into size limits in the destination Git system (Bitbucket).

Also from that search, I found the following links:<br>
- https://www.methodsandtools.com/tools/gittfs.php
- https://blog.cellenza.com/devops/migrating-from-tfvc-to-git/

This seems like a very similar process to git-svn above.  The key issue with this type of migration is
the removal of large binary files, even those "deleted".