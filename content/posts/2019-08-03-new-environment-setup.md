---
title: 'New Environment Setup'
date: Sat, 03 Aug 2019 04:14:48 +0000
draft: false
tags: ['automation', 'bash', 'zsh']
---

UPDATE: See the end of this post for a discussion on new revisions.

Every Professional Computer Toucher has a just-so way they want their environment. When I was but a wee lad, that environment consisted of Windows XP with various arcane registry tweaks applied (I vaguely recall changing my TCP congestion control algorithm to Westwood solely because of Red Alert 2), and Notepad. Things improve with age.

My current environment is a non-Retina Macbook Air, which suits me just fine. \*nix OS, with homebrew filling in the gaps. I use Atom (once you get past the abysmal load time, it's basically Sublime Text), and iTerm2 with zsh. You may note in a previous post that I extolled the virtues of fish. Well, I got tired of its lack of POSIX compatibility, and having to rewrite every script I downloaded. I toyed with the idea of writing a fish-to-zsh transpiler, but decided the better option was to [make zsh like fish.](https://github.com/abhigenie92/zsh_to_fish)

For CLI editing, when that is necessary, I have become infatuated with [micro.](https://micro-editor.github.io/) It's as if nano and emacs had a baby, but they left out the more troubling aspects of emacs, like its desire to become an OS. Seriously, it's amazing. All the normal stuff you'd expect, along with tab/space compatibility (tabs, but implemented as spaces), auto-spacing for functions and if blocks, and plugins for days. Did I mention [Solarized](https://ethanschoonover.com/solarized/) color support?

That's all well and good, but what about the cloud? I spin up a t2.micro at least every few days for something or other, and while sometimes it's a job that only takes a few minutes, it's annoying to not have my environment setup, and needing to manually run commands is the worst.

[Enter this repo (mine).](https://github.com/stephanGarland/useful-scripts) It's just two scripts (plus a caller) that do precisely the above - get micro, move it to /usr/local/bin, then install zsh, oh-my-zsh, and the necessary plugins to make it like fish. Its one Achilles' heel is the need for git. While sane and reasonable distros like Debian have this installed already, others like Fedora and SuSE (what are you DOING?) don't, at least not with AWS' AMIs. That will be fixed in an upcoming Terraform bootstrap which I almost have working, but for now, git clone and enjoy.

zsh\_setup.sh has a few checks, which I enjoyed implementing. First, it checks if you're on a Mac - if so, it directs you to install zsh using homebrew, and exits. I could/should add that in as a call, and also the git repos. Next, it checks your distro, and installs zsh. Currently supported are Debian and its derivatives, Red Hat and its derivatives, and SuSE. I'm not sure who is using SuSE, but it hovers between 9-10 on DistroWatch, so clearly some people are. After it has your distro, it updates your local repo, and installs zsh. Then it installs oh-my-zsh, clones the scripts, and edits ~/.zshrc using sed to install them. Finally, it runs chsh (oh-my-zsh's install script will do this for you, but it interrupts my script by dropping into zsh before it's completed, so I do it manually at the end) to set the current user's login shell to their zsh install location.

Code snippets are below with comment, for those wanting to learn more about shell scripts.

```
mac_check=`uname`
if [[ $mac_check == "Darwin" ]]; then
printf "\nPlease install zsh using Homebrew or a similar package manager"
exit 127 # command not found, since the next command would fail
fi
```

*   mac\_check = ... would fail, due to the whitespace. No whitespace for variable assignment.
*   exit codes can be applied to indicate the status of the exit. In this instance, it's saying that the command wasn't found, since a Mac isn't going to have apt, yum, or zypper.

    ```
    which zsh
    if [[ $? -eq 0 ]]; then
    ```

*   which asks for the location of the command that you're trying to execute. In this case, it's a fast way to see if zsh is installed (at least, with a valid PATH - it would fail if the user had simply dropped the binary into a directory not in PATH).
*   $? is a shortcut for the last command executed. If the exit code of that command was 0, i.e. successful, continue.

    ```
    else
    distro=`cat /etc/os-release | grep ID_LIKE`
    if [[ $distro =~ "debian" ]]; then
    ```

*   Here, we're just looking for a match in os-release, a file that contains information about the distro, such as its common name.
*   \=~ is for a substring match.

    ```
    which git
    if [[ $? -eq 0 ]]; then
    curl -fsSL https://raw.githubusercontent.com/robbyrussell/oh-my-zsh/master/tools/install.sh -o install.sh
    chmod +x install.sh
    `./install.sh --unattended`
    ```

*   I've found that curl is more universally installed by default than wget, so curl is used here. Note that if you piped this to bash, it would auto-install, which is the intent of the script, but I'm skipping that to tweak it. As an aside, I've read many people's opinions about piping curl outputs to bash, and they're usually negative. I agree that it can go south very quickly, but read the script first, then run it. If you're really paranoid, spin up an instance (hmm...) first and test it there.
*   The --unattended switch here is defined by the script to avoid doing anything that would interfere with a headless installation, so I'm using it.

    ```
    git clone...
    git clone...
    git clone...
    rm install.sh
    sed -i '/plugins=(git)/c\plugins=(git zsh-autosuggestions zsh-history-substring-search zsh-syntax-highlighting)' ~/.zshrc
    ```

*   Pull the scripts listed in the post using git clone
*   We don't need the install script after it's done, so remove it.
*   I heart sed like I heart regexes. They're both immensely powerful, and when used incorrectly, [break everything](https://blog.cloudflare.com/details-of-the-cloudflare-outage-on-july-2-2019/) with astonishing speed. In this case, I'm looking for the line in ~/.zshrc that adds plugins, which after oh-my-zsh is installed, will consist solely of git. I then replace that line using /c, which replaces the entire line, as opposed to /s, which substitutes the found text with yours. Note also that, per the zsh script's repos, the order of the scripts is important here.

    ```
    sudo chsh -s `which zsh` `whoami` &> /dev/null
    zsh
    ```

*   Finally, change the logged-in user's (whoami returns that) shell (which zsh finds its location), and directs the stdout to /dev/null so it doesn't clutter up the screen.
*   Then, drop into the shiny new shell, replete with glory.

    UPDATE: I've since re-written much of the script, mostly modularizing it. A script called check\_env is now sourced first, which also has support for a broader array of distros. Next, the user is given the option to update packages, as that may be annoying if you're deliberately using certain versions. Then, some tools that I find helpful (git \[I recognize the irony in installing git from within a git repo\], htop, mc, and tree) are installed with get\_tools. Finally, in zsh\_setup is now more POSIX-compliant (yay?), and also sets up oh-my-zsh as I like it.