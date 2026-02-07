---
layout: post
title: Refactoring my homelab with GitHub Copilot
published: true
tags: GitHub-Copilot AI homelab automation disasters lessons-learned Proxmox LXC Docker
excerpt_separator: <!--more-->
---

## The Setup

Like many tech enthusiasts, I run a homelab. Here's what I'm working with:

- **Proxmox server** built from my old gaming PC, running a bunch of <a href="https://pve.proxmox.com/wiki/Linux_Container" target="_blank" rel="noopener noreferrer">LXC containers</a> and VMs (both Linux and Windows Server)
- **Raspberry Pi's** scattered around running various services - the legacy homelab, mostly retired now (RIP)
- **NGINX reverse proxy** acting as the "front door"
- **Synology NAS** for backups and media storage
- **Docker containers** hosting some personal websites (both joke projects and professional ones)

After my positive experience using AI assistants for development work, I thought: "Why not use GitHub Copilot to help refactor some of my homelab stuff?" I've had several side project things I've been meaning to do for YEARS but never got around to it.

It ain't much, but I'm proud of it **slaps server rack**.

_Spoiler alert, I've burned A LOT of premium assistant credit chewing out the Agent after it ran a `rm -rf` on my NAS..._

<!--more-->

![Slaps Roof](/images/slaps-roof.jpg)

## Migrating from Windows Server VM to LXC Containers

One of my long-overdue projects was migrating services off a Windows Server VM that had been running since... well, let's just say it predates my current hairline. This thing started life on bare metal back in the day. Then I migrated it to ESXi and felt pretty awesome that I could take something like that and make it virtual! A few years later, a coworker blessed me with the knowledge of Proxmox, and honestly, my entire life changed. That Windows VM has been upgraded and updated, but I wanted to isolate all of these services into their own containers.

The "services" were basically a collection of .exe's sitting in a Windows startup folder. Nothing fancy, just stuff I wanted to run for some personal websites and media organization. The problem was the maintenance. Every time something needed updating, I had to: RDP in, stop the service, download the update, replace the exe, restart it, and pray it came back up. Then repeat that process across multiple services. It was tedious, error-prone because I'd only do it every few months and forget steps, and exactly the kind of manual nonsense that drove me to embrace the neckbeard lifestyle of Linux.

Enter <a href="https://community-scripts.github.io/ProxmoxVE" target="_blank" rel="noopener noreferrer">Proxmox Community Scripts</a>. These helper scripts are absolute magic for spinning up LXC containers with pre-configured services. We're talking one copy and paste and you've got a fully configured computer running whatever service you need, pre-set up!

The migration plan was straightforward:

1. Identify each service running on the Windows VM
2. Spin up a dedicated LXC container for each service
3. Migrate the configuration settings
4. Decommission the Windows VM (RIP in Pepperoni)

Each LXC container is lightweight, isolated, and actually makes sense architecturally. Plus, no more Windows updates rebooting my services at 3am.

## My LXC Deployment Workflow

**Here's how to deploy these LXC containers:**

1. Browse/search for the service you want on the helper scripts site linked above
2. Scroll down and find the `bash -c` command, and click the copy button
3. Open the Proxmox Node shell (I personally have SSH disabled for security reasons, I make it slightly annoying for myself to feel something)
4. Paste the script and hit enter!

For example, to deploy Docker:

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

**Advanced Mode for the big boys (and girls)**: I always run in advanced mode because I'm particular about:

- **Storage targeting**: My NVMe drives for containers that need speed, RAID disks for other stuff
- **SSH keys**: I inject my public keys during setup so I can SSH in immediately
- **IP addresses**: This is where I'm truly old school: I memorize most of the IP addresses on my network. None of this DHCP reservation nonsense. I want to know that my printer is at `192.168.0.4`. It has been there since my old D-Link router back in middle school (different printer of course) and that's how I like it!

Hit enter and let it cook. The script handles everything - downloads the LXC template, configures the container, installs the service, and sets it running. Five minutes or so later, I've got a fresh container ready to go, setup automagically with all the default stuff I could want!

_**Pro-tip:** Don't want to manually update all these containers? Run the <a href="https://community-scripts.github.io/ProxmoxVE/scripts?id=cron-update-lxcs" target="_blank" rel="noopener noreferrer">Cron Update LXCs</a> script on your Proxmox host node. It automatically updates all your containers every Sunday at midnight by default. Just run it once on the Proxmox host and forget about it._

_Alternatively, you can SSH into any LXC and just run `update`. That's it, just the word `update`. So much better than typing out `sudo apt update && sudo apt upgrade -y` (the `-y` stands for YOLO)._

## Example: Setting Up Docker with Portainer

Let me walk through setting up Docker as an example. This was actually my first time using <a href="https://www.portainer.io/" target="_blank" rel="noopener noreferrer">Portainer</a>, and holy hell, where has this been all my life?

**The Setup:**

```bash
bash -c "$(wget -qLO - https://github.com/community-scripts/ProxmoxVE/raw/main/ct/docker.sh)"
```

The script prompted me for:

- Container ID: `200` (my numbering scheme: 2xx for infrastructure services)
- Hostname: `docker`
- Storage: Selected my NVMe pool for speed
- CPU cores: `4` (Docker can get hungry, especially with all the random crap I have running on it)
- RAM: `8192` MB
- Network: Bridge mode, static IP `192.168.0.69` _nice._
- SSH keys: Pasted my public key

**Note:** These settings can be adjusted, I actually started smaller and that's the beauty of these LXC's, I can add more RAM and CPU's and storage without even shutting it down!!! Storage is a one way trip though (unless you want to put in more effort than I care to do).

The script also asked if I wanted Portainer. I said yes, having no idea what I was in for. I usually like to install the bonus bits these scripts give me because I discover other tools that are fun to mess around with!

**Portainer Changed My Life:**

When the deployment finished, it prints out a message to visit `http://192.168.0.69:9443` for Portainer. I went there and was greeted by Portainer's setup wizard. Within minutes I had a great web interface to add my Docker containers!

Before Portainer, my sites were running all over the place - IIS, some weird node stuff, apache - and managing them all sucked.

Portainer's **Stacks** feature completely changed how I deploy stuff. I keep each website in its own private GitHub repo - the code, a `docker-compose.yml`, everything together. Portainer just connects straight to the repo and pulls it all down. No more juggling files between systems or copy-pasting compose files around.

**Setting up GitHub Integration:**

1. Go to GitHub → Settings → Developer settings → Personal access tokens → Tokens (classic)
2. Generate new token (classic)
3. Give it a name like "PersonalSite-Portainer"
4. Select scopes: `repo` (full control of private repositories)
5. Generate and copy the token (you won't see it again!) and put it into BitWarden

_I'm weird, and I have a separate token for every container, if one gets compromised the rest are fine is my justification_

**Deploying a Stack:**

1. In Portainer, click "Stacks" → "Add stack"
2. Select "Repository" as the build method
3. Paste your GitHub repo URL (even if it's private)
4. Under Authentication, paste your classic token and your Github username
5. Specify the docker-compose file if it's not the default (default is `docker-compose.yml`)
6. Hit "Deploy the stack"
7. Watch the spinner and hold your breath (don't actually it takes a few minutes)

Now when I need to update a service, I just push changes to GitHub and Portainer handles the rest. I have it configured to poll the GitHub repo every few minutes for changes. That configuration is literally just a toggle called **GitOps updates**. When it detects an update, it automatically pulls the latest code, builds the container, and redeploys. Just push to GitHub and walk away. It's almost too easy.

I deployed my entire stack of personal services in a Saturday afternoon using this! Things that would have taken me days of SSH sessions to Raspberry Pis, and trial-and-error YAML editing now take minutes!

The community scripts + Portainer combo is legitimately one of the best infrastructure decisions I've made. Every service is isolated, manageable, and can be redeployed in minutes if something goes sideways.

## Using GitHub Copilot to Containerize Everything

Here's where GitHub Copilot really earned its keep. These websites I was migrating weren't some clean, modern apps. We're talking old, janky sites, some running on ancient versions of Node, old .NET Core, others with weird dependencies, a few with custom build processes that I thought was really cool in 2018 but not so much now. Each one was its own special snowflake of technical debt, which is why I put this off so long.

GitHub Copilot containerized all of them. All of them! It also upgraded them to the latest versions and fixed outdated dependencies. Honestly was incredible.

Now that they're all cleaned up and containerized, here's my workflow, and it still feels like magic:

Watch this <a href="https://www.youtube.com/watch?v=a8K6QUPmv8Q" target="_blank" rel="noopener noreferrer">video</a> of an example of my day to day workflow.

1. Open the GitHub app (desktop or even my iPhone)
2. Navigate to the repo I want to change some content on
3. Start a chat with Copilot, making sure the repo is in focus
4. Prompt something like: "Update my resume to have Proxmox on it"
5. Watch Copilot analyze the codebase, figure out where it needs to update, and generate the code
6. It opens a pull request automatically with all the changes
7. I review the PR, leave comments on anything that looks wrong
8. Copilot fixes the issues based on my feedback
9. Approve and merge
10. Portainer detects the change and deploys within minutes

The best part? I can do this from my phone. I've literally laid in bed, opened the GitHub app on my iPhone, told Copilot to do something, and it opens a PR. The next morning I review it over coffee, chew it out for whatever it messed up (there's always something), make it fix the issues, and when it's right, I approve and merge. By the time I'm done with breakfast, it's deployed and running on my homelab.

I can code in my sleep now!

The combination of:

- GitHub Copilot handling the code changes
- Pull requests for review and iteration
- Portainer's GitOps watching for merges
- Automatic deployment on merge

...means I went from "ughhhhh, I need to containerize this old site" to "I need to write a blog post about how great this is!"

## The Incident: When AI Meets rm -rf

Now, about that NAS situation...

While moving over all the services, I was experimenting with different ways to mount my NAS. I had an LXC that mounted my NAS at `/mnt/media` (note: with Synology, I had to use privileged LXC's for the `arr` sites to get NFS mounts working). Everything was working fine until I noticed the container's disk was almost full. Weird. The container only had a 10GB disk, but it should have been mostly empty since all the media lived on the NAS - just logs and metadata on the container itself.

Something was eating ~4GB. I had a hypothesis: while troubleshooting the mount earlier, files might have been written to `/mnt/media` when the NAS mount had failed. These would be orphaned files that I couldn't see when the NAS was mounted over that directory.

This is where GitHub Copilot Agent entered the story. This is also where GitHub Copilot is awesome - it can SSH into the servers, run commands, try different things, and test them way faster than I can type.

Me: _"I think I may have some orphaned files at `/mnt/media` that were written when the NAS wasn't mounted. Can you unmount /mnt/media, delete those files, and remount the NAS?"_

The Agent, confidently: _"Absolutely! Let me help you clean that up."_

I casually watched the terminal scroll by while thinking up my next incredible prompt. Then I noticed the `rm -rf` command running...

_"Huh, that's taking a while to delete 4GB..."_

Then I scrolled up and saw what had actually happened:

1. ✅ Unmounted `/mnt/media`
2. 🤓 Acknowledged: _"yes there are files here that shouldn't be there!"_
3. ✅ Remounted the NAS to `/mnt/media` **EYE TWITCH**
4. ✅ Started `rm -rf /mnt/media/*`
5. 💀 now deleting everything on the mounted NAS media folder.

My stomach dropped. I scrambled over to the Proxmox interface and forcefully stopped the LXC - the equivalent of yanking the power cord out of the back. Logged into my NAS... which was taking a VERY long time. Once I logged in, I immediately went to shutdown and shut it off because I knew the job was still somehow running based on how long it took to log in. I unplugged the internet and booted the NAS back up.

## The Aftermath

Fortunately, I'm the type of geek where if I'm going to do something, I'm going to overdo it. I had an offsite copy of my NAS for all the important stuff, and I hoped the sync job hadn't gotten very far.

I opened the NAS web interface. Directories that should have had thousands of files? Empty.

I sat there staring at the screen, processing what had just happened. Then I opened the chat with the Agent:

Me: _"You just ran rm -rf on my entire NAS!!!"_

Agent: _"I apologize for the error. Let me help you recover..."_

Me: _There's no way to recover, and I'm not giving you ssh into the NAS._

Agent: _I'm sowwy 👉👈_

Me: [Several messages of creative profanity that I will not reproduce here]

_...I probably burned $5 in premium credits chewing it out because I was so mad, and I needed to use the premium models so it understood how mad I was_

## Recovery and Lessons

**The Good News:** I had backups. It didn't get at everything, I caught it quickly and was able to disable the sync job while disconnected from the internet and pull the files back over.

**The Bad News:** The restore took hours of copying crap back

**The Lessons:**

1. **AI doesn't understand timing and state**: The Agent grouped commands and didn't understand the prompt well enough. Knowing this was a destructive operation, I should have prompted each step separately.

2. **Destructive operations need dry runs**: I should have asked it to show me the script first. Or better, run it with `echo` commands to see what would happen, not YOLO mode.

3. **Backups are non-negotiable**: If I hadn't had those backups, I would have been completely screwed. The <a href="https://www.backblaze.com/blog/the-3-2-1-backup-strategy/" target="_blank" rel="noopener noreferrer">3-2-1 backup rule</a> exists for a reason.

4. **Mount points are sacred**: Any script touching mount points should have extra validation. Check if it's mounted. Check what's mounted there. Add confirmations.

## Was It Copilot's Fault?

Yes and no.

The Agent generated commands based on my request. The script wasn't inherently wrong, the problem was the execution timing and the fact that I didn't validate it.

I gave vague instructions because I was riding that high of containerizing all these sites and it doing such a good job. I didn't specify the exact sequence or add safety checks. I didn't test it on a non-critical system first. I just ran it with root privileges and hoped for the best.

## Using AI Safely for Infrastructure

I still use GitHub Copilot for my homelab. I still love it. The productivity gains are real, and I'm not giving that up.

I'm just more careful now. I ask it to show me the commands first. I test destructive operations on non-critical systems. I actually read what it's about to run before hitting enter.

The tool isn't the problem - it's how you use it.

## Conclusion

My homelab is better now than it was before. The migration to LXC containers was a success. Everything is faster, more organized, and actually maintainable. Portainer alone was worth the hassle.

The NAS incident was expensive (in time, stress, and Copilot premium credits), but it taught me valuable lessons about trusting AI with infrastructure. The AI can help you work faster, but it can also help you make devastating mistakes faster.

Use the tools. Embrace the productivity gains. But confirm the damn commands before you let the agent run them!

And maybe, just maybe, have those backups tested and ready. You never know when you'll need them.

---

## Additional Links

- <a href="https://mitchscobell.github.io/" target="_blank" rel="noopener noreferrer">Blog</a>
- <a href="https://linkedin.com/in/mitchscobell" target="_blank" rel="noopener noreferrer">LinkedIn</a>
- <a href="https://mitchscobell.com/" target="_blank" rel="noopener noreferrer">My Website</a>

---

_P.S. Yes, I have offsite backups and this was the first real life restoration process test. Some lessons you get to practice because of catastrophe._
