---
title: "Pingchilling: The greatest notification system for pseudo-SLURM clusters, powered by Discord"
date: 2025-10-04T18:00:00Z
BookToc: false
Summary: "The true story of ONE discord bot BORN to monitor jobs, FORCED to operate without a job submission system."
draft: false
---

![pingchilling](/pingch/pingch_dm.webp)
<!-- {{< figure src="/pingch/pingch_dm.webp" width=60% >}} -->

{{% hint info %}}
My entire lab thought we were running a SLURM cluster. One postdoc submitting jobs hundreds at a time. The job submission system? Appending an ampersand to the end of a command. Our cluster architecture: six computers running Ubuntu with a shared home directory. Not Ubuntu server, Ubuntu GNOME. Meet the "head node", another computer running Ubuntu GNOME, headless. This is the true story of one discord bot born to monitor jobs, but forced to operate without a job submission system. The ending will leave you wondering, "is it even a SLURM cluster?"
{{% /hint %}}

# FLURM (Fictitious Linux Utility for Resource Management)
Chemists have a history of not understanding computers. My undergraduate advisor (love the guy but this one is funny) often said that the Macs in our computer lab were all running off one 2013 Mac Pro, the trash can model. We would be running up-to-date OSX on ~2017 iMacs. Pointing to the dusty cylinder in the corner of the room, he would say, "you know all these computers are running off that?" I clicked on "About This Mac" to make sure the model was indeed the one in front of me. Some things are best left unsaid. I often worked there to do my undergraduate research, so when I was alone, I investigated the ominous cylinder. It wasn't even plugged in.

Another member of my group, at the time, downloaded a terminal app on his Android phone. He used it exclusively to `ssh` into the university supercomputer to see if his calculations had ended. When I told him about the flag that emails you when a job finishes, he seemed uninterested. I guess he just liked thumbing through a supercomputer on his phone. Some things are best left unsaid.

Flash forward to graduate school, and my entire lab is convinced we're running a cluster of some kind. I've never built one before, so I thought "I guess it is possible that 6 PCs each separated by 30ft of ethernet cable could be a cluster." After spending more time around them, I noticed they would often appropriate HPC lingo. It started with using "head node" to describe one of the computers. The only special thing about it was that it served the home directory for every computer. Then it was `ssh` to describe using the terminal on any node. You see, each of the "nodes" is the work computer for one lab member. So opening a terminal on any one of them counted as `ssh`. The inflection point was seeing that each "node" had it's own IP and hostname open to the university network. The breaking point was seeing the "job submission system": a single ampersand appended to the end of a command in `bash`.

So we have a cluster where
1. The head node is identical to all other nodes
2. Each node has a separate IP/hostname
3. If you `ssh` into one, you somehow `ssh` into the head node but it only submits jobs to the node you `ssh`ed into.
4. The job submission system is backgrounding jobs in `bash`
5. The job scheduling system is definitely real, just ask our advisor.

"Some things are best left unsaid" is a good philosophy when leaving things unsaid minimally affects people. Running 16 separate calculations, each using 20GB+ of RAM, in the background of a single computer tests the limits of that philosophy.

# If a job ends in a notify-send, does it make a sound?

Recalling the days of undergrad, where I could be emailed after jobs finished, I longed for something similar last Spring. I tried using `notify-send` after a few calculations, which didn't work. A labmate and I were working with collaborators out of state, so while `ssh`ed into one of the nodes, the only attached display was 400 miles away. I ran a few `notify-send` commands in isolation with CRITICAL flags and text reading "AHHHHHHH HELP ME" and "LET ME OUT" (a trademark of mine). I completely forgot about it until a week later when the same labmate logged back onto the computer and was met with many persistent alerts that each had to be closed individually.

That still didn't solve my problem. How would I notify myself when a command finished? I can't set up an email server because 1. I don't have the techromancy skills and 2. the computers are on a private network and reverse proxies are black magic banned by university IT. Is there some other obvious solution? Probably, but you know what's better than an email after a command finishes? A discord notification after a command finishes. Initially, I wanted to avoid relying on a cloud service, but I made an exception for discord because it was funny.

# Hold on Kitten, Daddy doesn't have sudo privs yet

Once I started looking into discord bots, the ideas started flooding in. The comedic potential was endless. Bots in discord are OP. They have access to more features than server admins. They can create custom embeds, create interactive items like select menus and buttons, create ephemeral messages (messages only one user can see in the server), set up microtransactions, and slash commands that do your bidding with a hotkey. There's probably a way to make members execute JS in their discord instance so you can run crypto miners if they don't pay those microtransactions.

I knew this was the way to go, the only problem is that the server can only run on one computer, and I want to know the status of every computer. Two technical challenges defined my approach:
1. Computers only share a single home directory, not `/proc`
2. I'm doing this on hard mode: I don't have root access to any of the computers.

I wanted to design this with other users it in mind. This, combined with lack of root access, meant I could only use tools that were already installed on the computers.

> This limitation proved needless. Note to the ambitious doer-of-things: nobody wants to use your cool tool that makes work easier. Some people take it as an insult when you say there's a better way of doing things. Don't let that stop you from making cool stuff though.

The solution I landed on was a hidden file placed in the home directory of each user. I named it `.pingchilling` for reasons that will become obvious upon hearing the name of my discord bot.
# A bot named pingchilling

{{< figure src="/pingch/bingchilling.gif" width=50% >}}

The bot was written in python using [discord.py](https://github.com/Rapptz/discord.py). It scans the home directory of every user, every few seconds, and writes any updates in that user's `.pingchilling` file to a dictionary. If a job started then finished for a given user, it would DM them. A secret file holds my API key and a map between each lab member's discord ID and their username on the computers. 
## Bash side
The `.pingchilling` file is appended to by each user upon executing my bash function. I wrote a function `pingch` that uses the PID of the last job placed in the background (`$!` in bash). The command would take the PID, grab the name of the command from `ps`, the current time, and when the job finished, it would write all those things to the hidden file. I often used it like this:
```bash
{ time python RAMMELTER.py ; } |& tee output.log & pingch &
```
Which I admit is ugly, but half of it is just to capture the output of `time`.

At this point, `pingch` ran **after** backgrounding `<command>`, which is now a child of bash. All `pingch` has access to is that PID. Therefore, `pingch` does not own the backgrounded process, and is not notified upon the completion of that command. I ran a loop that checked the PID with the `kill -0` trick. The null signal isn't a real POSIX signal (like SIGSEGV or SIGINT), but it is reserved for use by `kill`. It just checks to see if the process exists and a signal can be sent. This worked for a while, but eventually I wanted the exit status of the job, which I had no way of obtaining.

`pingch` v.2 restructured everything. I had to call `pingch` **before** `<command>` to get the exit code. This means parsing the arguments, then evaluating the arguments as a command. Calling `pingch` looks like this:
```bash
pingch python RAMMELTER.py > output.log &
```
This is a more conventional approach. Here's what the function looks like:
```bash
# Add this to your .bashrc
pingch() {
    usage="Usage: pingch [-t TITLE] <command>"
    if [[ $# -eq 0 ]]; then
        echo $usage
        return 1
    fi
    if [[ "$1" == "-t" ]]; then
        if [[ $# -lt 3 ]]; then
            echo $usage
                return 1
        fi
        title="$2"
        shift 2 # Don't evalute "-t TITLE"
    else
        title="$1"
    fi
    hname=$(hostname -s)
    # evaluate all arguments in background without expansion
    eval "{ $@ ; }&"
    PID="$!"
    echo "$PID@$hname: Started  [$title] "$(date +%F\ %T) >> ~/.pingchilling
    wait $PID
    exitcode=$? # Must be done for the return statement to work
    echo "$PID@$hname: Finished [$exitcode] "$(date +%F\ %T) >> ~/.pingchilling
    return $exitcode
}
```

We evaluate the arguments `$@` enclosed within a block `{...}`. This must be done to prevent syntax errors. For example, `eval "$@ &"` with the command `pingch command1 ; command2 &` evaluates `command1; command2 & &`, which is a syntax error. The ampersand evaluates the command in a subshell, so changes to the environment variables cannot affect the current environment. It also allows us to use `wait $PID` which also (thankfully) does not overwrite the bash variable `$?`: the exit status of the last finished command. If we had 
```bash
echo "$PID@$hname: Finished [$?] "$(date +%F\ %T) >> ~/.pingchilling
return $?
```
We would always return the exit code of `echo`, which is always 0 unless a the error indicator for stdout is set.

Other changes to this version include saving the hostname of the computer running the calculation. At this time, I was running jobs on multiple computers. If any second job started, the job before that would not ping you when finished.

In all, `.pingchilling` looks something like this:
```
1001@node5: Started [python] 2025-10-03 13:05:59
450@node3: Started [sleep 2] 2025-10-03 13:06:30
450@node3: Finished [0] 2025-10-03 13:06:32
1001@node5: Finished [0] 2025-10-03 13:40:01
```
## Discord side
In short, the discord bot reads changes to the `.pingchilling` file in every user's directory, every 10 seconds. I leave plenty of room for edge cases like multiple updates happening within 10 seconds, or overflowing the dict of currently running processes... or by creating very large strings to parse through. Now that I think about it, there are an uncountable number of ways to crash my bot if you're trying. So for ~~in~~security reasons, I won't be sharing the code.
# The rest of pingchilling
This project felt like a big waste of time at first, but given a few months of working with it, it's been an absolute game changer. I've never had to restart the server, unless all the computers go down, which has only happened a handful of times. This project was really fun to work on. To end, here's a non-exhaustive list of features:
## List running processes on each computer

![Listing Processes on the computer](/pingch/pingch_proc.webp)

This feature prints all major processes running on each of the computers. I manually go into each computer and set up a `cron` job to run every minute. It runs the following command:
```sh
ps -o pid,user,pcpu,pmem --sort=-pcpu | awk '$3 > 2.0'
```
Which gets a table of PID, user, %CPU, and %MEM usage for all processes using more than 2% CPU. That gets saved to another shared file and read by the bot on request. It's a quick and dirty solution, but it works.

This seems more useful than it is. Our lab is smaller now than it was when I made this feature, so there's always less than 100% utilization these days.

## Lising papers written by our advisor

![Here is one of my many papers where I uncover the secrets of the universe!](/pingch/pingch_paper.webp)

## Reminding us what grad school is really about

![PAPER, MATT! PAPER! 15 PAPERS END OF WEEK NATHAN](/pingch/pingch_demi.webp)


