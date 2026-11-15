---
layout: post
title:  "Lecture from the local System Administrator"
date:   2026-11-15 09:18:23
categories: jekyll update
katex: false
---

Every Linux user has probably seen this text before:

```
We trust you have received the usual lecture from the local System
Administrator. It usually boils down to these three things:

    #1) Respect the privacy of others.
    #2) Think before you type.
    #3) With great power comes great responsibility.
```

If you don't know what it is, it's [sudo](https://en.wikipedia.org/wiki/Sudo)'s first-run message, which I was reminded of a few months ago when I was setting up my new Linux system it popped up again.

To me, as someone who *wasn't* in the era of mainframes and terminals being a thing, it's a sort of timeless reminder (yes, i know how ironic that is) of an older time, where the "System Administrator" wasn't just someone on your home computer. Evidently I'm not the only one whose felt this, as Randall Munroe of xkcd riffs this in "Incident", with this "classic" xkcd.

![Image of xkcd 838, "Incident". The comic shows a stick figure typing sudo su on his computer, with the error message "robm is not in the sudoers file. This incident will be reported". He asks another stick figure behind him "Hey - who does sudo report these incidents to?", with the response "You know, I've never checked". The next panel shows Santa Claus on a computer with a large piece of paper behind him with a presumably naughty and nice list.](https://imgs.xkcd.com/comics/incident.png)

It's also comforting to me. Now, with depersonalized software and the corporatization of it, these little touches of humanity contrasted with distinctly non-human *software* reminds you that there were people that made this out of passion and not by another soulless machine dispensing code. A few months ago, [Louis Rossmann's campaign](https://en.wikipedia.org/wiki/Louis_Rossmann#Clippy_profile_picture) against the very symptoms (or causes?) of this rallied behind the figure of [Clippy](https://en.wikipedia.org/wiki/Office_Assistant), undoubtedly playing on this same idea.

Git's changelogs do something similar, where it "learns" a feature instead of a feature being implemented.

Some examples from [git's 2.9.0 release log](https://github.com/git/git/blob/main/Documentation/RelNotes/2.9.0.adoc):
> - "git mergetools" learned to drive ExamDiff.
> - "git pull" has been taught to pass the "--allow-unrelated-histories" option to underlying "git merge".
> - "git apply -v" learned to report paths in the patch that were skipped via --include/--exclude mechanism or being outside the current working directory.

Anyway, I found the little sudo message a comforting reminder, and it is some pretty good advice either way. I chose to keep this message to print whenever I run sudo. You can do the same by adding 
```
Defaults        lecture = always
```

to a file in `/etc/sudoers.d/`.
