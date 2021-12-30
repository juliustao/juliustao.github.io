---
title: "VSCode SSH on Windows 11 🪟 with WSL2 Ubuntu 🐧"
published: true
---

# TL;DR

Below are steps for VSCode remote development on Windows 11 through SSH of WSL2 Ubuntu.

Why not use
[Windows SSH](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse)?
I need some packages for SSH that are only available with Ubuntu.

-   On Windows 11

    -   [Install WSL2 Ubuntu](https://docs.microsoft.com/en-us/windows/wsl/install)
    -   [Install VSCode](https://code.visualstudio.com/docs/setup/windows)
    -   [Install VSCode Remote Extension Pack](https://marketplace.visualstudio.com/items?itemName=ms-vscode-remote.vscode-remote-extensionpack)
    -   [VSCode SSH from WSL](https://stackoverflow.com/a/66048792/10039100)
        -   [SSH from interactive shell to source `~/.bashrc`](https://stackoverflow.com/a/70321075/10039100)

-   In WSL2 Ubuntu

    -   [Generate SSH key and add to `ssh-agent`](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
    -   [Add SSH key to GitHub](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/adding-a-new-ssh-key-to-your-github-account)
    -   [Auto start `ssh-agent`](https://stackoverflow.com/a/24902046/10039100)
    -   [Install Kerberos](https://tig.csail.mit.edu/operating-systems/csail-ubuntu/kerberos-gnu-linux/)
    -   [Customize `~/.ssh/config`](https://tig.csail.mit.edu/operating-systems/csail-ubuntu/kerberos-gnu-linux/)

-   Verify that your SSH command from a WSL2 Ubuntu terminal works.
-   [Typing that command into VSCode](https://code.visualstudio.com/docs/remote/ssh#_connect-to-a-remote-host) should give you the full remote editing experience!

# How it started

I got a great deal ($500 off!) on a
[Dell Inspiron 16 Plus](https://www.dell.com/en-us/shop/dell-laptops/inspiron-16-plus-laptop/spd/inspiron-16-7610-laptop/nn7610eyvoh)
laptop this Black Friday 🤑

The laptop runs Windows 11, but I usually use Ubuntu for development.
However, I'm tired of maintaining the compatibility hacks I need with
[my previous laptop](https://www.lenovo.com/us/en/outletus/laptops/ideapad/ideapad-300-series/Ideapad-320-Touch-15-Intel/p/88IP3000844)
that I dual booted Ubuntu.
For example, the microphone audio sounds deep fried, and
[the solution](https://gist.github.com/Therises/d2e91c81af1574f9069635d520fdc7ec)
is hacking around in PulseAudio config files.

In my current research project, I work over a
[SSH](https://www.ssh.com/academy/ssh) connection to a
[CSAIL](https://www.csail.mit.edu/) server.
Rather than edit locally and copy changes to remote, I prefer using
[VSCode Remote SSH](https://code.visualstudio.com/docs/remote/ssh)
to directly edit remote files.

Maybe I could set up my preferred workflow with Windows?

# Environment

On Windows,
[installing VSCode](https://code.visualstudio.com/docs/setup/windows),
[installing SSH](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_install_firstuse#install-openssh-using-windows-settings), and
[creating new SSH keys](https://docs.microsoft.com/en-us/windows-server/administration/openssh/openssh_keymanagement)
is straightforward.

Configuring the `ssh-agent` with
[automatic startup and the key](https://dmtavt.com/post/2020-08-03-ssh-agent-powershell/)
is also simple.

The last step is to check if I can SSH with Powershell to the server.
For security purposes, I must connect to the target server `deep-gpu-1.csail.mit.edu` through the jump server `jump.csail.mit.edu`.
For both connections, I must type my password.

Good news! I can SSH successfully.
What about VSCode?

Once I install
[VSCode Remote Development](https://code.visualstudio.com/docs/remote/remote-overview),
I type the same Powershell SSH command into the VSCode prompt.
After connecting, I can edit the remote files directly.

Success!

. . . except typing my password twice is annoying 😩

On my previous laptop, I set up
[passwordless SSH with Kerberos tickets](https://tig.csail.mit.edu/operating-systems/csail-ubuntu/kerberos-gnu-linux/).
Then I only type my password once every 12 hours instead of twice every time I login.
However, these steps are for Ubuntu.

The CSAIL website also has a
[Windows version](https://tig.csail.mit.edu/operating-systems/windows/kerberos-for-windows/).
Maybe that would work?

# Kerberos on Windows

## 3.2.x

The CSAIL TIG website shows instructions for
[installing Kerberos on Windows (KfW) 3.2.x](https://tig.csail.mit.edu/operating-systems/windows/kerberos-for-windows/).

Following the instructions steps, I install the 64-bit version of KfW before the 32-bit version.

I open the KfW GUI, create a new Kerberos ticket, and try connecting to the server.

Like on my previous laptop, I expect to see the server login text after a short delay.

Unluckily, I see the password prompt.

The website mentions a newer KfW, which I could try instead?

## 4.1.x

I remove the previous KfW before
[installing KfW 4.1.x](http://web.mit.edu/kerberos/kfw-4.1/kfw-4.1.html).

The GUI looks nicer, which seems more promising.

However, after I create a new Kerberos ticket and connect, I still see password prompts.

At this point, I file a help request with TIG, and they quickly confirm that using KfW with CSAIL servers is no longer supported.

Do I need to dual boot Ubuntu now?
Not just yet.

# Ubuntu inside Windows!

Microsoft recently released
[Windows Subsystem for Linux 2 (WSL2)](https://docs.microsoft.com/en-us/windows/wsl/about),
a Linux kernel inside Windows.
This feature is promising because I could have the convenience of Windows but develop inside a lightweight Ubuntu environment.

## SSH

[Installing WSL2 Ubuntu](https://docs.microsoft.com/en-us/windows/wsl/install)
is simple, and I can set up my
[SSH keys](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
and
[config](https://tig.csail.mit.edu/operating-systems/csail-ubuntu/kerberos-gnu-linux/)
with the same `~/.ssh` folder as my previous laptop.

[Adding my key](https://docs.github.com/en/authentication/connecting-to-github-with-ssh/generating-a-new-ssh-key-and-adding-it-to-the-ssh-agent)
to the `ssh-agent` is also identical.
The only difference is that WSL2 Ubuntu does not support `systemd`, so the `ssh-agent` does not already start automatically.
However, there exist
[multiple solutions](https://unix.stackexchange.com/a/90869),
and I choose to install `keychain` and write `eval $(keychain --eval -q ~/.ssh/id_ed25519)` to my `~/.bashrc`.

I can successfully open an SSH connection by typing my password twice.
So far, so good.

## Kerberos

Since I already see differences between WSL2 and desktop Ubuntu, I began to worry slightly that Kerberos login would not work with WSL2.

I again follow
[the Ubuntu Kerberos steps](https://tig.csail.mit.edu/operating-systems/csail-ubuntu/kerberos-gnu-linux/),

Luckily, I can connect to the server from WSL2 with no password prompt.

## VSCode hack

However, I still need to connect my VSCode installed on Windows.
How can I force VSCode to SSH from Ubuntu?

Fortunately, I find [this thread on Stack Overflow](https://stackoverflow.com/questions/60150466/can-i-ssh-from-wsl-in-visual-studio-code).
VSCode offers amazing customization for developers, so it's no surprise that we can change what SSH executable VSCode runs.

The default executable is the `ssh` on the `PATH`, which would be the Windows SSH executable.
The trick is replacing the SSH executable with a `ssh.bat` file that pipes VSCode arguments `%*` into the command `C:\Windows\system32\wsl.exe ssh %*`.
When VSCode runs the `.bat` file, it will run in a WSL shell the `ssh` command with arguments `%*`.

## Forward `ssh-agent`

I soon encounter an issue where I can `git push` properly in Windows terminal but encounter a public key error in a VSCode terminal.

Running `ssh -T git@github.com` also fails with a public key error, so my `ssh-agent` is not forwarded properly.

After some wrong approaches like modifying [local VSCode settings](https://github.com/microsoft/vscode-remote-release/issues/2926) or the
[remote SSH config](https://github.com/Microsoft/vscode-remote-release/issues/16), I finally realize the issue is that `wsl.exe` does not source `~/.bashrc`.

After searching through WSL2 Github Issues, I found a [helpful diagram](https://github.com/microsoft/WSL/issues/6339) of how a non-interactive shell like `wsl.exe` works.
The [new workaround](https://github.com/microsoft/WSL/issues/5801)
is to run `C:\Windows\system32\wsl.exe bash -ic 'ssh %*'` to SSH with an interactive shell.

Once I modify the `ssh.bat` with the new command, I can now `git push` with no errors.

# How it's going

I finally have my preferred setup of VSCode remote development on Windows 😄

The workarounds I needed are somewhat messy but so much cleaner than dual booting Ubuntu and fixing hardware compatibility issues!
