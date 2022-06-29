---
layout: post
title:  "How to become a Linux power user"
date:   2022-03-07 8:30:00 -0600
---

This guide is for those who have touched the command line before, but are rather limited to a set of memorized incantations, or whatever they find on Q&A sites.

## Learning to fish: `man`
No, not becoming fishers of men. `man` is short for "manual", and it is the most important command in Linux to remember. Why? Because `man` will tell you what everything else does!

Just type `man` and then a command, and it will open a page of documentation right in your terminal!

Let's get real meta for a moment (no not the Facebook kind) and look at the documentation for `man` itself by running:

```
$ man man
```

### Navigating a pager (i.e. `less`)
Assuming you are on a desktop and have a terminal that supports the mouse/touchpad, you should just be able to scroll through it. If scrolling with the mouse/touchpad isn't working, you should be able to use the arrow keys to scroll up and down. And at the bottom it gives some helpful information:

```
Manual page man(1) line 6 (press h for help or q to quit)
```

You may notice that your option to type in a new command is unavailable. This is because you are viewing the output of `man` in something called a _pager_ (not the electronic device) which lets you scroll through a big wall of text and do convenient things like search. Most of the time, you will actually be looking at a specific pager called `less`. We'll come back to `less` later.

As explained in that neat help string, you can get out at any time with `q`. You can take a look at the help with `h`, but there's really only one command I use inside of `less`: the search.

#### Searching inside of `less`
All you do is type `/` followed by the pattern you want to search for. Then you press `Enter` to confirm, and then you can go 

`man`'s output can be a little intimidating at first, but here are some pointers 