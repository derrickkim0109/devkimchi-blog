---
title: "Oh My Azure Cloud Shell"
slug: oh-my-azure-cloud-shell
description: "In this post, I'm going to introduce the way to install both oh-my-zsh and oh-my-posh onto Azure Cloud Shell."
date: "2021-12-22"
author: Justin-Yoo
tags:
- azure
- azure-cloud-shell
- oh-my-zsh
- oh-my-posh
cover: https://sa0blogs.blob.core.windows.net/devkimchi/2021/12/oh-my-azure-cloud-shell-00.png
fullscreen: true
---

You likely heard of or currently use [oh-my-zsh][om zsh] for your terminal on Linux or Mac, or [WSL][wsl] on Windows. It might also be possible to have heard of or currently use [oh-my-posh][om posh] for your PowerShell. [Azure][az] offers [Azure Cloud Shell][az csh] service, which uses both Bash Shell and PowerShell by default. Therefore, if you want either oh-my-zsh or oh-my-posh, or both, you should configure it by yourself.

Throughout this post, I'm going to show how to configure your shell environment for both.

> This [GitHub repository][gh sample] provides the working shell script source for your reference.


## For oh-my-zsh ##

Let's configure [oh-my-zsh][om zsh] on your [Azure Cloud Shell][az csh]. Make sure that you see the Bash Shell prompt. If not, enter the `bash` command to switch your prompt to Bash.

1. Install oh-my-zsh.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=01-install-oh-my-zsh.sh

2. Install plug-ins for oh-my-zsh. Although there are many good plug-ins, this post will install the three popular ones &ndash; [zsh-completions][om zsh plugins zsh-completions], [zsh-syntax-highlighting][om zsh plugins zsh-syntax-highlighting] and [zsh-autosuggestions][om zsh plugins zsh-autosuggestions]. If you want more plug-ins, follow the steps below.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=02-install-oh-my-zsh-plugins.sh

3. Install themes for oh-my-zsh. It's totally up to you which theme you're going to pick, but this post chooses either [Spaceship][om zsh themes spaceship] or [Powerlevel10k][om zsh themes p10k].

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=03-install-oh-my-zsh-themes.sh

4. If you choose the [Powerlevel10k][om zsh themes p10k] theme, run the following command for further configuration.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=04-configure-p10k.sh

5. As mentioned above, [Azure Cloud Shell][az csh] uses Bash Shell by default. Unfortunately, you can't run the `chsh -s $(which zsh)` command because `sudo` is not allowed. Therefore, update your `.bashrc` like below, as a workaround.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=05-update-bashrc.sh

6. Once everything so far is done, restart [Azure Cloud Shell][az csh]. Alternatively, run the command, `source .bashrc`. Then, you will have the oh-my-zsh applied shell prompt.

7. If you want to run all steps above in just one command, run the following:

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=06-install-oh-my-azure-cloud-shell.sh

8. If you are with the [Powerlevel10k][om zsh themes p10k] theme and want to turn on or off the current time, run the following script:

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=07-switch-p10k-clock.sh

Now, your [Azure Cloud Shell][az csh] starts using [oh-my-zsh][om zsh].


## For oh-my-posh ##

Let's configure [oh-my-posh][om posh] on your [Azure Cloud Shell][az csh] this time. Make sure that you see the PowerShell prompt. If not, enter the `pwsh` command to switch your prompt to PowerShell.

1. Install oh-my-posh.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=08-install-oh-my-posh.ps1

2. Install themes for oh-my-posh. Again, it's totally up to you which theme you're going to pick, but this post chooses either [Spaceship][om posh themes spaceship] or [Powerlevel10k - Rainbow][om posh themes p10k].

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=09-install-oh-my-posh-themes.ps1

3. Install plug-ins for oh-my-posh, like [Terminal Icons][om posh plugins terminal-icons].

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=10-install-oh-my-posh-plugins.ps1

4. Create the `$PROFILE` file to run when initiating a PowerShell session.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=11-update-profile.ps1

5. Run the following command to reload `$PROFILE`.

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=12-reload-profile.ps1

6. If you want to run all steps above in just one command, run the following:

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=13-install-oh-my-azure-cloud-shell.ps1

7. If you are with the [Powerlevel10k - Rainbow][om posh themes p10k] theme and want to turn on or off the current time, run the following script:

    https://gist.github.com/justinyoo/f5fa2a0e894a185828b76d64170de24e?file=14-switch-p10k-clock.ps1

Now, your [Azure Cloud Shell][az csh] starts using [oh-my-zsh][om posh].

---

So far, I've walked through how to configure either [oh-my-zsh][om zsh] or [oh-my-posh][om posh], or both. With both configurations, you will be able to extend your local shell scripting experiences to [Azure Cloud Shell][az csh].


[gh sample]: https://github.com/justinyoo/oh-my-azure-cloud-shell

[wsl]: https://docs.microsoft.com/windows/wsl/about?WT.mc_id=dotnet-52663-juyoo

[om zsh]: https://github.com/ohmyzsh/ohmyzsh
[om zsh plugins zsh-completions]: https://github.com/zsh-users/zsh-completions
[om zsh plugins zsh-syntax-highlighting]: https://github.com/zsh-users/zsh-syntax-highlighting
[om zsh plugins zsh-autosuggestions]: https://github.com/zsh-users/zsh-autosuggestions
[om zsh themes spaceship]: https://github.com/spaceship-prompt/spaceship-prompt
[om zsh themes p10k]: https://github.com/romkatv/powerlevel10k

[om posh]: https://ohmyposh.dev/
[om posh plugins terminal-icons]: https://github.com/devblackops/Terminal-Icons
[om posh themes spaceship]: https://ohmyposh.dev/docs/themes#spaceship
[om posh themes p10k]: https://ohmyposh.dev/docs/themes#powerlevel10k_rainbow

[az]: https://azure.microsoft.com/?WT.mc_id=dotnet-52663-juyoo
[az csh]: https://docs.microsoft.com/azure/cloud-shell/overview?WT.mc_id=dotnet-52663-juyoo
