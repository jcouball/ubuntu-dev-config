# ubuntu-dev-config

The configuration of `rama`, an x86_64 machine running Ubuntu 26.10 Desktop. It is managed by [chezmoi](https://www.chezmoi.io/).

rama has two jobs:

- A backup development machine that can do most of what the Mac
  (`neuromancer`) does.
- A Linux test machine that other machines drive over ssh, the way `sophon` is
  used for Windows.

This repo is a sibling of
[mac-dev-config](https://github.com/jcouball/mac-dev-config) (macOS)
and [windows-dev-config](https://github.com/jcouball/windows-dev-config)
(Windows). Each repo configures one operating system, and this one currently
configures one machine, rama. Shared dotfiles are copied between the repos,
not templated, so a change to one operating system never needs testing on
another.

The three are worked on together in
[dev-config](https://github.com/jcouball/dev-config), the workspace they are
cloned into. Issues that span more than one of them are filed there.

**Status: tested in a container, not yet on the machine.** The rebuild
procedure below runs to completion in an `ubuntu:26.04` container:
[Certify the rebuild](#certify-the-rebuild) repeats that test. See
[Not done yet](#not-done-yet) for what it leaves untested.

## Principle: conventional Ubuntu

rama is set up the way an Ubuntu administrator would expect. Software comes
from the first of these sources that has it:

1. **The Ubuntu archive, with `apt`.** This is the default for everything.
2. **The vendor's own apt repository**, when the archive lacks the software or
   carries a version too old to use.
3. **mise**, only for what neither of the first two provides: language
   runtimes at pinned versions, and command-line tools with no apt package.
4. **snap**, only for GUI applications that none of the first three provides.

Being conventional takes priority over matching the Mac. Where the two
conflict, rama follows Ubuntu practice and accepts that the commands differ
from `neuromancer`. The Mac uses Homebrew and asdf; rama uses neither.

Two consequences of the principle:

- **No Homebrew.** It works on Linux, but it is not how Ubuntu machines are
  managed, and it installs its own copies of libraries that builds can pick up
  by accident.
- **No snap for command-line tools.** Snap puts commands in `/snap/bin`, which
  only login shells have on `PATH`, so `ssh rama 'some-tool'` would not find
  them. Snaps also update on their own schedule. Snap is acceptable for GUI
  applications.

## Install Ubuntu

rama is a Dell Inspiron 3880 desktop with one 238 GB NVMe disk. Install from
the Ubuntu 26.10 Desktop ISO, with the network cable plugged in. The installer
shows these screens in this order; its wording changes between releases, so
match each by what it asks.

| Screen | Choice |
| --- | --- |
| Language | English |
| Accessibility | Nothing |
| Keyboard | English (US) |
| Network | The wired connection. Wi-Fi is not used |
| Try or install | Install Ubuntu |
| Interactive or automated | Interactive installation |
| Which apps to install | Default selection, the minimal desktop. Not the extended selection |
| Extra software | Leave both boxes unticked: third-party drivers, and support for more media formats |
| Disk setup | Erase the disk and install Ubuntu. Under the advanced features, ZFS, without encryption |
| Account | Name `James Couball`, computer name `rama`, username `james`, and a password required to log in |
| Time zone | `America/Los_Angeles` |

Why these:

- **26.10, not 26.04 LTS.** 26.10 is what the scripts have run on. It is an
  interim release: see [Updates](#updates) for when it has to be upgraded.
- **The minimal desktop and no extras.** The lists in this repo are the whole
  inventory of what is installed on top of Ubuntu. The Inspiron's Intel
  graphics needs no proprietary driver.
- **ZFS.** The installer labels it experimental. It puts the root pool,
  `rpool`, on the fourth partition, after the EFI, `/boot`, and swap
  partitions it creates. Script 08 exists because of this choice.
- **No encryption.** rama stays at home and has to come back from a reboot
  with nobody at the keyboard, so that other machines can drive it over ssh.
  Anyone who takes the disk can read it, including the `gh` and Claude Code
  sign-ins. The installer is the only place to change this.
- **The username `james`.** The ssh client entry under
  [Set up a client](#set-up-a-client) logs in as `james`.
- **The computer name `rama`.** chezmoi asks for the host name and offers the
  current one as the default, so naming the machine here makes that answer
  Enter.

## Rebuild from nothing

On the fresh install, in a terminal on the desktop, not over ssh: the scripts
that set the terminal font, the dock, and the mouse need the desktop session.

```bash
# 1. What the next command needs.
sudo apt-get update && sudo apt-get install -y curl git

# 2. Everything else.
sudo -v && sh -c "$(curl -fsLS get.chezmoi.io)" -- -b /tmp/chezmoi-bootstrap \
  init --apply --branch stable jcouball/ubuntu-dev-config
```

`stable` is the branch that holds the last certified commit;
[Main and stable](#main-and-stable) says how it moves.

Step 2 downloads a temporary copy of chezmoi and runs it. It asks what the
machine is called, then installs the apt packages, changes the login shell to
zsh, registers the vendor apt repositories, writes the config files, installs
the mise runtimes and tools, installs the VS Code extensions, installs any
snaps listed, restricts sshd to key logins, turns on the firewall, and sets up
the dock.

The temporary chezmoi in `/tmp` is only for this first run. mise installs the
permanent one.

Run `sudo -v` first and never `sudo chezmoi apply`: chezmoi would write
root-owned files into the home directory. The scripts call `sudo` themselves
where they need it.

If the apply stops because a download failed, run it again:
`sudo -v && chezmoi apply`. Every script is written to be run again, and a
download fails for reasons that have nothing to do with this repo: a 503 from
GitHub stopped a certification run on 2026-11-23, and the next run passed
unchanged.

Afterwards, by hand:

1. Give the machines that will drive rama the key: see
   [Set up a client](#set-up-a-client).
2. Sign in to Tailscale, so rama can be reached from outside the LAN: see
   [Tailscale](#tailscale).
3. Log out and back in so the login shell change and the `docker` group
   membership take effect.
4. Log into 1Password.
5. Sign in to mail in Thunderbird, and set View > Layout to Classic View.
6. Run `claude` and `gh auth login` to sign in to each.
7. Sign in to VS Code with GitHub and turn on Settings Sync.
8. Sign in to Telegram: scan its QR code from the phone app.
9. In Warp, set Terminal font to `MesloLGS NF` under Settings > Appearance >
   Text. Warp does not sync or restore the setting, and script
   `after_07_terminal_font` sets the font only for Ptyxis.

The rebuilt machine's chezmoi source is on `stable`, because that is what the
command above cloned. Leave it there until the configuration is next changed
on rama, or a certification is to be run. Then put the source on `main`, which
is where rama's source normally is:

```bash
chezmoi git -- fetch origin && chezmoi git -- switch main && sudo -v && chezmoi update
```

[Main and stable](#main-and-stable) says why.

## What installs what

| What | Source | Declared in |
| --- | --- | --- |
| Compilers, libraries, services, zsh, sshd, and every tool Ubuntu carries | apt, Ubuntu archive | [apt-packages.txt](apt-packages.txt) |
| VS Code, `gh`, the 1Password app and CLI, Claude Code, mise, Chrome, Slack, Warp, Typora, Docker Engine, Tailscale | apt, vendor repositories | script `before_03_vendor_apt_repositories` |
| Ruby, Node, Python, Go, Rust | mise | [dot_config/mise/config.toml](dot_config/mise/config.toml) |
| Command-line tools with no apt package, including chezmoi and topgrade | mise | the same file |
| VS Code extensions | `code --install-extension` | [vscode-extensions.txt](vscode-extensions.txt) |
| GUI applications with no apt source | snap | [snap-packages.txt](snap-packages.txt) |
| The powerlevel10k zsh theme, and the `jcouball/scripts` utilities | git checkouts kept by chezmoi | [.chezmoiexternal.toml](.chezmoiexternal.toml) |
| The MesloLGS NF font the prompt needs | files downloaded once by chezmoi | [.chezmoiexternal.toml](.chezmoiexternal.toml) |
| Dock launchers for web pages, and their icons | chezmoi | [dot_local/share/private_applications](dot_local/share/private_applications), [dot_local/share/icons](dot_local/share/icons) |
| The dock's pinned launchers and settings | gsettings | [dock-favorites.txt](dock-favorites.txt), [dock-settings.txt](dock-settings.txt) |
| Config files | chezmoi | the `dot_*` files |

### Why each vendor repository is there

| Package | Why not the Ubuntu archive |
| --- | --- |
| `code` | Not in the archive |
| `gh` | The archive has 2.46.0, more than two years old |
| `1password` | Not in the archive |
| `1password-cli` | Not in the archive |
| `claude-code` | Not in the archive |
| `mise` | Not in the archive |
| `warp-terminal` | Not in the archive |
| `typora` | Not in the archive |
| `docker-ce` and its plugins | The archive has `docker.io`, Ubuntu's own build; Docker's repository has Docker's releases and the Compose and Buildx plugins |
| `tailscale` | Not in the archive |
| `google-chrome-stable` | Not in the archive |
| `slack-desktop` | Not in the archive |

Claude Code follows the repository's `latest` channel, the same channel Claude
Code checks for updates, so its "Update available" notice means `topgrade` has
an upgrade to install. The `stable` channel runs about a week behind and skips
releases with major regressions. To switch, change the URI and suite in script
03 from `latest` to `stable`.

The script checks the Claude Code and Docker signing keys against the
fingerprints Anthropic and Docker document. The other nine keys are fetched
over HTTPS from the vendor, as each vendor's own instructions do.

The 1Password app, Chrome, and Slack are published for amd64 only. The script
installs them when the machine is amd64 and skips them otherwise.

**Docker is Docker Engine, not Docker Desktop.** Desktop runs the engine inside
a virtual machine, which Linux does not need. The script adds the user to the
`docker` group, so `docker` runs without `sudo`, including in a command started
over ssh. Membership of that group is equivalent to root access.

**Four packages register their vendor's repository themselves.** apt refuses to
update when two files define one repository with different keys, and warns on
every update when they differ in any other way. The script avoids a second
definition in each case:

| Package | What it does | What the script does |
| --- | --- | --- |
| `1password` | Rewrites `1password.sources` on every install and upgrade | Uses the same file name |
| `google-chrome-stable` | Rewrites `google-chrome.sources` on every install and upgrade | Uses the same file name |
| `warp-terminal` | Writes `warpdotdev.sources` only when it is missing | Uses the same file name, so the package leaves it alone |
| `slack-desktop` | A daily cron job writes `slack.list` | Turns the cron job's switch off in `/etc/default/slack` |

For 1Password and Chrome, the file on disk is the package's version after an
install or upgrade and the script's version after the script runs. Both work.

### What mise installs, and how

mise's own registry covers most tools and downloads them from the project's
GitHub releases. Four entries in the config use a different route:

- `npm:@github/copilot` and `npm:markdownlint-cli` are npm packages, installed
  with the Node that mise provides.
- `pipx:checkov` is a Python package. The registry's `checkov` entry downloads
  a release binary that is not published for every architecture.
- `github:` entries (`dblab`, `git-trim`, `mq`) are not in the registry and are
  installed straight from the project's GitHub releases.

Runtimes are pinned to exact versions. Tools are set to `latest`.

**Ruby and Python are precompiled.** mise downloads a prebuilt Ruby and Python
where one exists, which takes seconds. Ruby still loads the system's OpenSSL,
and gems with native extensions compile against apt's headers. To build Ruby
from source against apt's libraries, run `mise settings set ruby.compile true`
before installing it.

### PATH is set in .zshenv

`ssh rama 'ruby -v'` starts a zsh that is neither a login shell nor
interactive. That shell reads `.zshenv` and nothing else, so `.zshenv` is the
only file that can put mise's Ruby on `PATH` for a command run over ssh. It
does so with mise's shims directory.

Interactive shells also run `mise activate` from `.zshrc`, which manages `PATH`
directly and applies a project's own `mise.toml` or `.tool-versions` on `cd`.

## Remote access over SSH

Other machines on the LAN or the tailnet run commands and tests on rama over
ssh. A session
logs in as `james`, who has sudo and is in the `docker` group, so a login is
close to root access. Three things limit who gets one, the same three
[windows-dev-config](https://github.com/jcouball/windows-dev-config) uses:

- **Keys only.** [sshd-rama.conf](sshd-rama.conf) is installed as
  `/etc/ssh/sshd_config.d/00-rama.conf`. It allows public keys alone, so the
  account password does not open a session, and it refuses root.
- **Keys from this repo.** chezmoi writes `~/.ssh/authorized_keys` from
  [private_dot_ssh/private_authorized_keys](private_dot_ssh/private_authorized_keys).
  Each line must start with `restrict` or `restrict,pty`; script 09 fails the
  apply on any other.
- **LAN and tailnet only.** ufw blocks every inbound connection except the
  rules in [ufw-rules.txt](ufw-rules.txt), which allow port 22 from
  `192.168.0.0/16` and from IPv6 link-local addresses. A machine on the tailnet
  reaches port 22 with no rule, because ufw does not filter the tailnet: see
  [Tailscale](#tailscale).

Forwarding is off: `DisableForwarding yes` covers ports, the agent, X11, and
tunnels, and no key option can turn it back on. Use `scp` or `sftp` to move
files; both still work.

There is one key for rama, `id_ed25519_rama`, used for rama alone. Its private
half is kept in 1Password as an SSH Key item of the same name, and every
client holds a copy. Each copy should have a passphrase, so a copy of the key
file is useless on its own. Nothing here can enforce that.

**The firewall does not cover Docker.** Docker publishes ports with its own
packet-filter rules, which run ahead of ufw's, so `docker run -p 8080:80` is
reachable from the network whatever `ufw-rules.txt` says. To keep a container
private, publish it on the loopback address: `-p 127.0.0.1:8080:80`.

**The firewall blocks everything else inbound from the LAN.** A dev server or
Redis on rama is unreachable from another machine on the LAN until it has a
rule: see [Open a port](#open-a-port). Connections from rama to itself,
outbound connections, ping, and `rama.local` name lookups need no rule.

### Tailscale

rama is on a tailnet, the Tailscale network the other machines are signed in
to, and has a name there: `rama`. That is what makes `ssh rama` work from
outside the LAN. Script 03 registers Tailscale's apt repository and installs
the package. The package enables `tailscaled`, which starts at boot, so rama is
back on the tailnet after a restart with nobody logged in.

**Signing in is done by hand, once.** It needs a browser:

```bash
sudo tailscale up
```

Open the address it prints on any machine and approve the login. Then, in the
[admin console](https://login.tailscale.com/admin/machines), turn off key
expiry for rama. With it on, rama leaves the tailnet after 180 days and has to
be signed in again at its console. Script 03 says when rama is signed out, and
`verify.sh` fails on it.

A rebuilt rama comes back as a new machine. Delete the old entry in the admin
console first, or the new one is named `rama-1` and `ssh rama` reaches nothing.

**ufw does not filter the tailnet.** Tailscale adds its own packet-filter
rules, ahead of ufw's, that accept what arrives on its interface,
`tailscale0`. So a machine on the tailnet reaches every port rama listens on,
whatever `ufw-rules.txt` says, the same way a port Docker publishes is
reachable. Tested on 2026-11-26 with a listener on port 8099 and no rule for
it: reachable from Neuromancer and from sophon over the tailnet, blocked from
sophon over the LAN.

What limits the tailnet is who is on it. Only machines signed in to this
tailnet can send anything, and the tailnet's access rules, in the admin
console, say which of them may reach which ports. Those rules are not managed
here, and by default they allow everything. To keep a service off the tailnet,
bind it to the loopback address or to rama's LAN address.

**Check a machine that has restarted with `ssh <name> hostname`, not
`tailscale ping`.** From the Mac app, `tailscale ping` to a machine that has
just restarted fails until some other connection is made to it:
[dev-config#7](https://github.com/jcouball/dev-config/issues/7).

### Set up a client

The key is already listed in `private_dot_ssh/private_authorized_keys`, so a
new client needs only its own copy of the private half. Nothing changes on
rama.

On the client, save the private key from the `id_ed25519_rama` item in
1Password to `~/.ssh/id_ed25519_rama`. 1Password hands the key over without a
passphrase, so give the copy one:

```bash
chmod 600 ~/.ssh/id_ed25519_rama
ssh-keygen -p -f ~/.ssh/id_ed25519_rama
```

Add an entry to the client's `~/.ssh/config`. `UseKeychain` is for a Mac, and
stores the passphrase in the Keychain so commands run without a prompt; leave
it out elsewhere.

```
Host rama
  HostName rama
  User james
  IdentityFile ~/.ssh/id_ed25519_rama
  IdentitiesOnly yes
  ForwardAgent no
  AddKeysToAgent yes
  UseKeychain yes
```

`rama` is the name rama has on the tailnet, so the block works from anywhere
the client has a network. It needs Tailscale running on the client and on
rama: see [Tailscale](#tailscale). With it off at either end, name the LAN
address from a client on the LAN: `ssh -o HostName=rama.local rama`.

Check from the client: `ssh rama 'ruby -v'` prints mise's Ruby.

### Connect from rama to another machine

rama can open `neuromancer`, `cortex` and `sophon` by name once it holds the
key for that machine. The private keys are kept on the primary development
machine, which is Neuromancer. rama normally holds none, so `ssh cortex` from
rama is refused until it is given the key.

To give rama the key for cortex:

1. On Neuromancer, copy the key to rama:

   ```bash
   scp ~/.ssh/id_ed25519_cortex rama:.ssh/
   ssh rama 'chmod 600 ~/.ssh/id_ed25519_cortex'
   ```

2. On rama, connect: `ssh cortex`.

For another machine, use its name in place of `cortex`. The key can stay on
rama for as long as rama needs the connection.

**The passphrase is meant to be asked once.** ssh asks for it on the first
connection and hands the unlocked key to rama's ssh-agent, which answers for
it from then on (`AddKeysToAgent yes` in `~/.ssh/config`). This does not work
yet in an ssh session into rama: the shell there does not know where the agent
is, so ssh asks on every connection.
[#34](https://github.com/jcouball/ubuntu-dev-config/issues/34) is the fix.

**To take a key away from rama,** remove it from the agent and then delete the
file, in the session the key was used from:

```bash
ssh-add -d ~/.ssh/id_ed25519_cortex
rm ~/.ssh/id_ed25519_cortex
```

Deleting the file alone is not enough: the agent keeps its copy of the key
for as long as the agent runs. If `ssh-add` answers "Could not open a connection to your
authentication agent", that session has no agent and so gave it no key; delete
the file and carry on.

The config reaches each machine by its tailnet name, so Tailscale has to be
running on rama and on the machine being opened.

### Replace the key

Every client shares the one key, so replacing it locks all of them out until
each has the new private half.

Make the new key on any machine. `ssh-keygen` asks for a passphrase; give it
one.

```bash
ssh-keygen -t ed25519 -f ~/.ssh/id_ed25519_rama -C id_ed25519_rama
```

Replace the private key in the `id_ed25519_rama` item in 1Password. In a
checkout of this repo, delete the old key line from
`private_dot_ssh/private_authorized_keys`, then add the new one and push:

```bash
printf 'restrict,pty %s\n' "$(cat ~/.ssh/id_ed25519_rama.pub)" \
  >> private_dot_ssh/private_authorized_keys
git commit private_dot_ssh/private_authorized_keys -m "Replace rama's SSH key"
git push
```

On rama, run `chezmoi update`, at the machine or over a session that is
already open, because the old key stops working once its line is gone. Then
repeat [Set up a client](#set-up-a-client) on each client.

### Revoke the key

Delete its line from `private_dot_ssh/private_authorized_keys` and run
`chezmoi apply`. sshd reads the file on every login, so the key stops working
at once, on every client. A session the key already opened stays open:
`loginctl list-sessions` lists them and `sudo loginctl terminate-session ID`
ends one.

### Open a port

Add a line to `ufw-rules.txt`, then `chezmoi apply`. A line is the arguments to
`ufw`:

```
allow from 192.168.0.0/16 to any port 3000 proto tcp
```

`sudo ufw status` shows the rules in force. For a port needed once, run `sudo
ufw allow ...` by hand and delete the rule afterwards.

A machine on the tailnet needs no rule: see [Tailscale](#tailscale).

### Close a port

Delete the line from `ufw-rules.txt`, then remove the rule by hand: `sudo ufw
status numbered` lists the rules, and `sudo ufw delete N` removes one. Script
10 adds rules and never removes them.

### Change sshd's settings

Edit `sshd-rama.conf`, then `chezmoi apply`. Script 09 installs the file,
checks it with `sshd -t`, and checks with `sshd -T` that passwords are still
off. If either check fails it puts the previous file back and fails the apply.
An edit made directly in `/etc/ssh/sshd_config.d/00-rama.conf` lasts until the
next apply that runs the script.

## Deploy key

`chezmoi update` fetches this repo over https with no credential, because the
repo is public. A push needs one. `gh`'s login is kept in the desktop keyring,
which an ssh session can read only while james is logged in at rama's desktop.
After a restart with nobody logged in, a push over ssh fails with
`could not read Username`.

So rama has a deploy key: an ssh key that GitHub accepts for this repo alone,
with write access. Only a push from the chezmoi source uses it. It is there so
a certification can be recorded as a tag from an ssh session
([issue 16](https://github.com/jcouball/ubuntu-dev-config/issues/16)).

| What | Where |
| --- | --- |
| Private key, no passphrase | `~/.ssh/id_ed25519_ubuntu_dev_config_deploy` on rama |
| Public key | this repo's deploy keys on GitHub, titled `rama: push certified tags`, with write access |
| GitHub's host keys | `~/.ssh/known_hosts` on rama |
| Push URL and ssh command | the git config of the chezmoi source, `~/.local/share/chezmoi/.git/config` |

The key has no passphrase because the push runs with nobody there to type one.

**None of this is managed by chezmoi.** The key is a secret made by hand on the
machine, and the two settings are in the chezmoi source's own git config, which
a new `chezmoi init` does not have. After a rebuild, set it up again with a new
key and delete the old one from GitHub. The key is not in 1Password: it is
useful on rama alone, and making a new one takes less time than restoring it.

**What the key can do.** Read this repo, and push to it. What it is for is
the two things `record.sh` pushes: a new `certified/` tag, and `stable` moved
forward. It can also create, move and delete any other tag, and any branch
other than `main` and `stable`. It cannot change `main`, which takes changes
only through a pull request. It cannot move or delete a `certified/` tag, or
delete `stable` or move it backwards: rulesets stop both, for everyone. It
opens nothing in any other repo.

So a stolen key could move `stable` forward to a commit of its own making, and
a rebuild would then clone that commit.

### Set up the deploy key

On rama, make the key:

```bash
ssh-keygen -t ed25519 -N "" -C "rama deploy key for ubuntu-dev-config" \
  -f ~/.ssh/id_ed25519_ubuntu_dev_config_deploy
```

Register its public half with write access. This needs `gh` logged in, so run
it in a terminal at rama's desktop, or copy the `.pub` file to another machine
and run it there:

```bash
gh repo deploy-key add ~/.ssh/id_ed25519_ubuntu_dev_config_deploy.pub \
  -R jcouball/ubuntu-dev-config -w -t "rama: push certified tags"
```

Add GitHub's host keys. They come from GitHub's API over https, so they are not
taken on trust from the first ssh connection:

```bash
curl -fsSL https://api.github.com/meta \
  | jq -r '.ssh_keys[] | "github.com \(.)"' >> ~/.ssh/known_hosts
chmod 600 ~/.ssh/known_hosts
```

Point pushes from the chezmoi source at the key. The fetch URL stays https, so
`chezmoi update` is unchanged:

```bash
cd "$(chezmoi source-path)"
git config --local remote.origin.pushurl git@github.com:jcouball/ubuntu-dev-config.git
git config --local core.sshCommand \
  "ssh -i ~/.ssh/id_ed25519_ubuntu_dev_config_deploy -o IdentitiesOnly=yes -o BatchMode=yes"
```

Check it from an ssh session, with nobody logged in at the desktop. This pushes
nothing:

```bash
cd "$(chezmoi source-path)"
GIT_TERMINAL_PROMPT=0 git push --dry-run origin HEAD:refs/tags/certified/push-check
```

It prints ` * [new tag]         HEAD -> certified/push-check`.

`~/.ssh/config` is managed by chezmoi and is not involved: the key is named in
`core.sshCommand`, not in a `Host` block.

### Replace the deploy key

Delete the old key from GitHub, then from rama:

```bash
gh repo deploy-key list -R jcouball/ubuntu-dev-config
gh repo deploy-key delete <id> -R jcouball/ubuntu-dev-config
rm ~/.ssh/id_ed25519_ubuntu_dev_config_deploy ~/.ssh/id_ed25519_ubuntu_dev_config_deploy.pub
```

Then make and register a new one as in
[Set up the deploy key](#set-up-the-deploy-key). The host keys and the two git
settings are still in place, so finish with the dry-run push.

### If the deploy key is compromised

Treat the key as compromised if rama's disk, a backup of james's home
directory, or the key file itself has been somewhere it should not be.

1. **Delete the key from GitHub first.** GitHub refuses it from that moment.
   This can be done from any machine where `gh` is logged in, or in a browser
   under the repo's Settings, Deploy keys:

   ```bash
   gh repo deploy-key list -R jcouball/ubuntu-dev-config
   gh repo deploy-key delete <id> -R jcouball/ubuntu-dev-config
   ```

2. **Look for pushes you did not make.** The repo's
   [Activity](https://github.com/jcouball/ubuntu-dev-config/activity) page
   lists every push, including branches and tags that were created or deleted.
   `main` cannot have been changed, and no `certified/` tag can have been
   moved or deleted. `stable` can have been moved forward, and a new tag
   made. Check that `stable` is a commit of `main` with a `certified/` tag you
   made:

   ```bash
   git fetch origin
   git merge-base --is-ancestor origin/stable origin/main && echo "stable is on main"
   git tag --points-at origin/stable
   git ls-remote --tags origin
   ```

   If `stable` is somewhere it should not be, do not rebuild from it. Putting
   it back means turning its ruleset off first.

3. **Make a new key**, as in [Replace the deploy key](#replace-the-deploy-key).
4. **Find out how the key got out.** It was readable by anything that could
   read james's files on rama, and so was everything else in that home
   directory.

## Scripts

chezmoi runs these in order during `chezmoi apply`. All but the last are
`run_onchange_`: each runs again only when its rendered content changes.
Script 13 is `run_`, and runs on every apply.

| Script | What it does | Re-runs when |
| --- | --- | --- |
| `before_00_set_host_name` | `hostnamectl set-hostname`, and fixes `/etc/hosts` | the host name answer changes |
| `before_01_apt_install` | Installs the missing packages from `apt-packages.txt`, upgrading none; checks sshd is running | `apt-packages.txt` changes |
| `before_02_set_login_shell` | Makes zsh the login shell | the script changes |
| `before_03_vendor_apt_repositories` | Registers eleven vendor apt repositories and installs their packages; adds the user to the `docker` group; checks tailscaled is running, and says when Tailscale is not signed in | the script changes |
| `after_04_mise_install` | `mise install`, then verifies nothing declared is missing | the mise config changes |
| `after_05_vscode_extensions` | Installs the missing extensions from `vscode-extensions.txt`, trying a failed install up to three times | `vscode-extensions.txt` changes |
| `after_06_snap_install` | Installs the missing snaps from `snap-packages.txt`; skips itself where snapd is not running | `snap-packages.txt` changes |
| `after_07_terminal_font` | Rebuilds the font cache and sets the Ptyxis font to MesloLGS NF; fails outside a desktop session, so a later apply retries | the script changes |
| `after_08_hide_zfs_member` | Installs a udev rule that makes udisks ignore ZFS pool members, so the dock and Files stop listing the root pool as an unmountable `rpool` volume; skips the reload where udevd is not running | the script changes |
| `after_09_sshd_lockdown` | Installs `sshd-rama.conf` as an sshd drop-in, checks it with `sshd -t` and `sshd -T`, rolls it back if either fails, and reloads sshd; fails on a key line without `restrict` | `sshd-rama.conf` or the key file changes |
| `after_10_firewall` | Blocks inbound connections with ufw, adds the rules in `ufw-rules.txt`, and enables the firewall; skips itself where ufw cannot run | `ufw-rules.txt` changes |
| `after_11_dock` | Sets the dock's pinned launchers to `dock-favorites.txt` and its settings to `dock-settings.txt`, resetting every other dock setting to its default; fails outside a desktop session, so a later apply retries | either list changes |
| `after_12_mouse_scroll` | Turns on natural scrolling for the mouse wheel; skips itself where GNOME is not installed, and fails outside a desktop session, so a later apply retries | the script changes |
| `after_13_git_hooks_path` | Sets `core.hooksPath` to `.githooks` in the chezmoi source, so git runs [the hooks](#the-hooks) there | every apply |

`before_` scripts run before chezmoi writes any file; `after_` scripts run
after, because script 04 reads `~/.config/mise/config.toml` and script 09
needs `~/.ssh/authorized_keys` in place before it turns passwords off.

## Playbooks

Install a package or tool. Check the sources in order:

1. `apt-cache policy name`. If Ubuntu has a usable version, add the name to
   `apt-packages.txt`.
2. If the vendor publishes an apt repository, add an `add_repository` line and
   the package name to script 03.
3. Otherwise add it to `dot_config/mise/config.toml`. `mise registry | grep
   name` shows whether mise knows it.
4. For a GUI application that none of those has, add it to
   `snap-packages.txt`. `snap info name` shows who publishes the snap.

Then `chezmoi apply`.

Remove one: delete the line, then uninstall by hand (`sudo apt-get remove
name`, `mise uninstall name`, or `sudo snap remove name`). No script
uninstalls.

Upgrade a language version: edit the version in
`dot_config/mise/config.toml`, then `chezmoi apply`.

Test on a Ruby that is not installed: `mise install ruby@3.3`. It installs
beside the others and changes no default. See the note on `mise prune` under
[Updates](#updates).

Install a VS Code extension: add its id to `vscode-extensions.txt`, then
`chezmoi apply`.

Change the dock: pin, unpin, or reorder in the dock itself, and change its
settings in Settings > Ubuntu Desktop > Dock. Then run `dock-capture`, which
rewrites `dock-favorites.txt` and `dock-settings.txt` from the live dock, and
commit. `chezmoi status` does not report a dock changed by hand, and the next
apply that runs script 11 puts the dock back the way the lists have it.
Editing the lists and running `chezmoi apply` works too.

Add a dock icon for a web page: copy
`dot_local/share/private_applications/glinet.desktop` under a new name and
change `Name`, the URL in `Exec`, `Icon`, and `StartupWMClass`. Put a PNG icon
named after the `Icon` value in `dot_local/share/icons/hicolor/SIZE/apps/`,
where `SIZE` is the image's size, such as `256x256`. Add the launcher's file
name to `dock-favorites.txt`, then `chezmoi apply`.

Update everything: `topgrade`. See [Updates](#updates).

## Updates

Three things change the packages on rama, and each has one job.

| | What it does | When |
| --- | --- | --- |
| `chezmoi apply` | Installs what is in the lists and missing from the machine | When a list changed |
| `topgrade` | Upgrades everything already installed | When you run it |
| Ubuntu's `unattended-upgrades` | Installs security updates | Daily, with no action from you |

**chezmoi installs and never upgrades.** Both apt scripts pass `--no-upgrade`,
and the extension and snap scripts skip what is installed, so adding one item
to a list never upgrades the others as a side effect. If a list has not changed, its
script does not run.

**chezmoi never uninstalls.** Deleting a line from a list leaves the package on
the machine. Remove it by hand.

**topgrade is the upgrade path.** [dot_config/topgrade.toml](dot_config/topgrade.toml)
limits it to these steps:

- `system`: `apt update`, `apt dist-upgrade`, `apt clean`, and `apt
  autoremove`. This covers Ubuntu's archive and every vendor repository, so
  everything script 03 installs, mise itself included, upgrades here. It asks
  for the sudo password. The `autoremove` comes from `cleanup =
  true` and removes packages that were installed only as dependencies and are
  no longer needed.
- `mise`: `mise upgrade` and `mise prune`. The upgrade moves tools set to
  `latest` forward. Runtimes pinned to an exact version do not move.
- `vscode`: updates the extensions.
- `chezmoi`: pulls this repo and applies it.
- `shell` and `github_cli_extensions`.

**`mise prune` deletes versions no config refers to.** A Ruby installed by hand
for one test run, with `mise install ruby@3.3`, is removed the next time
topgrade runs. To keep one, add it to the mise config.

**`unattended-upgrades` covers security fixes between topgrade runs.** Ubuntu
enables it by default. It applies security updates from Ubuntu's own archive
only: not regular updates, and not vendor repositories. Those wait for
topgrade.

**Snaps are not topgrade's job.** snapd refreshes them itself, several times a
day.

**Nothing here moves rama to the next Ubuntu release.** 26.10 is an interim
release, supported for nine months, until about July 2027. Before then, move
to 27.04 with `sudo do-release-upgrade`, run by hand.

## Certify the rebuild

Certification has two scripts, as in mac-dev-config:

- `certification/certify.sh` runs [Rebuild from nothing](#rebuild-from-nothing)
  in a fresh `ubuntu:26.04` container, then runs `verify.sh` in it.
- `certification/verify.sh` checks a machine by its outcomes: what is
  installed, what is set, what resolves. It also runs on rama itself.
- `certification/record.sh` records a certification as a tag on the commit,
  and moves `stable` to it.

### Rebuild in a container

```bash
./certification/certify.sh                  # certify main
./certification/certify.sh --branch <name>  # try a branch before it merges
./certification/certify.sh --keep           # leave the container for inspection
./certification/certify.sh --verify-only    # re-run the checks in a kept container
```

It needs docker and git on the machine it runs on, and changes nothing there
apart from pulling the image and writing logs.

A pull request does not need a certification run to merge. It needs review, and
whatever shows the change itself works: a script run against stand-ins, a test
of the logic, an apply in a scratch directory. The pull request says what was
done.

Certify `main` when there is something on it that a rebuild should get. On
rama, in the chezmoi source, with `main` applied there first:

```bash
cd "$(chezmoi source-path)"
sudo -v && chezmoi update
./certification/certify.sh
```

That assumes the source follows `main`. On a rama that has just been rebuilt it
follows `stable`: see [Main and stable](#main-and-stable). A run takes about
eight minutes on rama.

**What needs a run before it reaches `stable`:** anything a rebuild runs or
reads. That is a script under `.chezmoiscripts`, one of the package lists,
`.chezmoi.toml.tmpl`, `.chezmoiignore`, a managed config file, and `verify.sh`.
A tightened check counts exactly as a changed script does, because the previous
run passed a test that no longer exists.

**What does not:** the README, `CLAUDE.md` and `record.sh`. A commit that
changes only those can sit on `main` until the next run takes it along.
`certify.sh` is tested by the next run it makes.

**One run can cover several changes.** It certifies the tip of `main`, and
everything merged since the last tag is in that commit. When a run of several
changes fails, the logs usually name the script. If they do not, certify the
changes one at a time with `--branch`.

**To try one change in a container before it merges**, name its branch. This
is worth the wait for a new script, where a clean machine finds what nothing
else does. Run it once, after review, on the commit that will merge: a new
commit on the branch means a new run. Because the checkout has to be at the
branch's commit, make the run from a scratch clone on rama, not from the
chezmoi source.

The container clones the repo from GitHub, as the README's command does, and
`verify.sh` comes from the local checkout, copied at the start of the run. So
for a branch the script stops before starting a container unless all three of
these hold:

- the branch is pushed,
- the local checkout is at the commit the branch has on GitHub, and
- `certification/verify.sh` has no uncommitted changes.

The transcripts go to `certification/logs/`, which git ignores:
`bootstrap-<stamp>[-<branch>].log`, `verify-<stamp>[-<branch>].log`, and the
copy of `verify.sh` that produced the second. The bootstrap log ends with the
commit and the tree the container built, what the container was, and the
verdict. For a branch run made before a merge, put the commit, the tree and the
verdict in the pull request.

**The verdict.** CERTIFIED needs both of:

- the last line `verify.sh` printed is its `VERIFY-COMPLETE` record, and the
  record counts no failed check, and
- the commit the container built is the one GitHub named before the run
  started.

The exit status of the bootstrap is reported and decides nothing. `verify.sh`
runs whatever it was, because a script that exits 0 has proved nothing.

`--verify-only` never certifies. It re-runs the checks in a container that a
run with `--keep` left, and says whether they passed. It did not rebuild the
container and does not know which commit it holds, so its result is for
looking into a failure. It cannot be recorded.

**What a pass does not cover.** `verify.sh` reports each of these as `SKIP`,
and counts it as excluded:

- **systemd.** A container has none, so the host name, the snaps, the
  firewall and a running sshd are not verified.
- **udev.** No udevd runs in a container, so script 08 writes its rule and
  skips the reload and the trigger. Whether the rule is in force is not
  verified.
- **A desktop session.** GNOME is not installed in the container, so the
  terminal font, the dock and the mouse are not verified.

A run in a container on rama ends with 6 excluded. `verify.sh` on rama itself
covers all six.

The container has the architecture of the machine it runs on. This repo
targets amd64: on an Apple silicon Mac the container is arm64, the amd64-only
packages are not installed, and `verify.sh` fails on them. Certify on rama.

The script's header names each place it departs from the README's procedure:
it makes a user first, because the image has only root; it answers the host
name question with `--promptString`; and it always names the branch it
clones, `main` or the one given with `--branch`, where the README's command
names `stable`.

### Main and stable

A change merges to `main`. A rebuild clones `stable`. Between the two is a
certification.

| Branch | Holds | Moved by |
| --- | --- | --- |
| `main` | every merged change, certified or not | a pull request |
| `stable` | the last commit that was certified | `certification/record.sh`, and nothing else |

`main` is often ahead of `stable`, and that is the normal state, not a fault:
several changes can merge and be certified together.

```bash
git fetch origin
git log --oneline origin/stable..origin/main    # merged, not yet certified
```

rama's own chezmoi source follows `main`, not `stable`. It is the one machine
this repo configures, so it is where the configuration is changed, and the
second half of a certification is `verify.sh` passing on it: it has to take a
commit before the commit can be certified. `stable` is for
[Rebuild from nothing](#rebuild-from-nothing).

A rama that has just been rebuilt is the exception: its source is on `stable`,
which is what the rebuild cloned. `chezmoi update` there keeps following
`stable`, and `record.sh` stops on a source that follows `stable` and says so.
Put the source on `main` before certifying or changing anything:

```bash
chezmoi git -- fetch origin && chezmoi git -- switch main && sudo -v && chezmoi update
```

A ruleset stops `stable` being deleted or moved backwards. Nothing on GitHub
can check that a commit pushed to it was certified. `record.sh` is what keeps
that true: it moves `stable` only to a commit it has just tagged, in the same
push as the tag.

mac-dev-config has two machines and so two roles, one following `stable`. Its
README describes them.

#### The hooks

That leaves one accident GitHub accepts: a commit made in a source that has
`stable` checked out, as a rama that has just been rebuilt does, and then
pushed. Two git hooks in `.githooks/` stop it in the clone. They are the same
two as in mac-dev-config:

| Hook | Stops | Lets through |
| --- | --- | --- |
| `pre-commit` | a commit while `main` or `stable` is checked out | any other branch, and a detached `HEAD`, which is what a rebase commits with |
| `pre-push` | a push to `stable`, from any local branch, of a commit with no `certified/` tag | a commit that has the tag in this clone or gets it in the same push, which is what `record.sh` pushes |

git runs a versioned hook only in a clone whose `core.hooksPath` names the
directory. Script 13 sets that in the chezmoi source on every apply, and
`verify.sh` checks it. `clone-repos.sh` in
[dev-config](https://github.com/jcouball/dev-config) sets it in a working
clone. In any other clone:

```bash
git config core.hooksPath .githooks
```

The hooks are a guard against accidents, not a control. Each lives in one
clone, and `--no-verify` skips it. The rulesets are the control.

### Record a run and move stable

A run gives the clean-machine result only. To record it, on rama, in the
chezmoi source:

```bash
cd "$(chezmoi source-path)"
sudo -v && chezmoi update       # after the merge, for a branch run
./certification/record.sh       # --dry-run shows the tag and makes nothing
```

`record.sh` reads the logs beside it, which is right for a run of `main` made
in the chezmoi source. A branch run is made in a scratch clone, and its logs
are there. Give that run's bootstrap log, as the end of the run prints it:

```bash
./certification/record.sh --log ~/<scratch clone>/certification/logs/bootstrap-<stamp>-<branch>.log
```

The verify log and the copy of `verify.sh` are found beside the log named.

`record.sh` reads the clean-machine result from the run's logs, then stops
unless all of these hold:

- this machine is rama,
- the run was CERTIFIED,
- the commit to certify is on `main` and has no `certified/` tag yet,
- `stable` is a commit of `main`,
- its tree is the tree the clean machine built,
- the `verify.sh` that checked the clean machine is the one at that commit,
- the chezmoi source on rama is at that commit, with nothing uncommitted, both
  before `verify.sh` runs and after it,
- the tag and `stable` can be pushed from this session, and
- `verify.sh` passes on rama with 0 warnings and 0 excluded, which is the
  real-hardware half. A warning or an excluded check there is a check that was
  not made.

It asks for the sudo password, because `verify.sh` reads sshd's settings and
the firewall's rules as root. So it needs a terminal: over ssh, use `ssh -t`.

It then shows the tag, creates it on the commit, and pushes the tag and
`stable` together: both arrive or neither does. The push uses the
[deploy key](#deploy-key). It certifies the tip of `main` unless `--commit`
names another commit, and reads the newest full run unless `--log` names
another bootstrap log. An older commit recorded with `--commit` gets its tag
and leaves `stable` where it is. If the trees differ, certify `main` and
record that run.

Nothing is committed, so no pull request is needed to record a certification.

### The certified run

Each certification is an annotated tag on the commit it certifies, named
`certified/<date>-<short commit>`. The tag's message holds the result:

```bash
git fetch --tags
git tag -l 'certified/*' --sort=-creatordate | head -1    # the latest
git tag -n99 -l 'certified/*'                             # all of them, with results
```

```text
Certified 1a2b3c4 on 2026-11-24

Clean-machine result: 52 passed, 0 failed, 0 warnings, 6 excluded
Real-hardware result: 63 passed, 0 failed, 0 warnings, 0 excluded

Machine:      container, ubuntu:26.04, x86_64
Host:         rama, Ubuntu 26.10
Built from:   main
Tree:         <the tree of the commit>
```

A tag that exists always means both halves: the rebuild in a clean machine,
and `verify.sh` on rama with the same commit applied. `Machine` says what the
clean machine was, and `Host` what rama runs. They are not the same release
today: the container is `ubuntu:26.04` and rama runs 26.10. While the clean
machine is a container, the six things under "What a pass does not cover" were
checked on rama alone, and the tag says so by naming the container.

What changed between two certifications is in the commits between their tags:
`git log <older tag>..<newer tag>`. A tag has one more line, `Triggered by:`,
when `record.sh` was given `--reason "<one line>"`. That is for a trigger no
commit records: an Ubuntu upgrade, a new image, or a second run of a commit
that has not changed.

The tags are not signed. A ruleset on `certified/*` stops one being deleted or
moved, so a record cannot change after it is made. To correct a wrong tag,
turn the ruleset off first, on purpose.

### Verify the machine

On rama, in a terminal:

```bash
sudo -v && ./certification/verify.sh
```

`sudo -v` first, because sshd's effective settings and the firewall's rules
can only be read as root. Without it those two checks print `WARN` and the
rest run. Nothing is skipped on rama: a clean run ends with 0 warnings and
0 excluded.

What `verify.sh` expects comes from the chezmoi source directory on the machine
it runs on: the package lists, the firewall rules, the dock, the sshd settings.
So a machine is checked against the commit it was built from, and those files
are not repeated in the script.

Each check prints one line: `PASS`, `FAIL`, `WARN` or `SKIP`. `WARN` means
something unexpected that is worth a look. `SKIP` means the machine cannot have
the thing at all, as a container cannot have a desktop. The last line is
`VERIFY-COMPLETE` with the four counts, and the exit status is 1 if any check
failed.

## Differences from mac-dev-config

| | neuromancer (macOS) | rama (Ubuntu) |
| --- | --- | --- |
| System packages and most tools | Homebrew | apt |
| Language runtimes | asdf, `.tool-versions` | mise, `config.toml` |
| VS Code extensions | Brewfile `vscode` lines | `vscode-extensions.txt` |
| powerlevel10k | Homebrew formula | git checkout kept by chezmoi |
| Claude Code | Homebrew cask | Anthropic's apt repository |
| 1Password, Chrome, Slack, Warp, Typora | Homebrew casks | each vendor's apt repository |
| Docker | Docker Desktop, a Homebrew cask | Docker Engine, from Docker's apt repository |
| Tailscale | the app on a MacBook, the daemon from a Homebrew formula on a Mac with no battery | the package from Tailscale's apt repository |
| What limits ssh to the LAN and the tailnet | sshd's `AllowUsers` | ufw for the LAN; the tailnet is not filtered |
| `PATH` setup | `.zprofile` | `.zshenv`, because of ssh |

Not carried over, because they are macOS-only:

- App Store apps, and every cask not named in the table above.
- The OpenJDK link script (Java comes from apt's `default-jdk`), the Amphetamine
  Power Protect script, and the Warp quarantine script.
- `scutil`, `plutil`, `xcrun`, `path_helper`, `java_home`, and the Amazon Q
  blocks in the shell files.
- The `sophon` and Proxmox entries in the ssh config. Their keys live on the
  Mac.

No PostgreSQL server: the Mac has `postgresql@17`, and rama has only `libpq-dev`,
the headers the `pg` gem builds against.

## Not done yet

- **A rebuild of rama.** The procedure has run to completion in a container.
  A container has no systemd, no udevd and no desktop session, so the host
  name, snap, firewall, terminal font, dock and mouse steps skipped
  themselves, sshd was not reloaded, and the udev rule was written but not
  loaded. `verify.sh` checks the outcome of every one of those on rama as it
  stands, but rama has not been rebuilt from nothing by these scripts.
- **A real topgrade run.** `topgrade --dry-run` in the container printed the
  commands listed under [Updates](#updates). It has not been run for real.
- **GitHub rate limits.** mise asks the GitHub API for release information.
  The container run did not hit the unauthenticated limit. If a rebuild does,
  set `GITHUB_TOKEN` and run `chezmoi apply` again.
- **A snap install.** The container has no snapd, so script 06 has only run
  there with a stand-in `snap` command.
- **More GUI applications.** VS Code, 1Password, Chrome, Slack, Warp, and
  Typora are installed. Zoom, Discord, Synology Drive, and GitButler publish a
  deb with no apt repository; each needs a source chosen (snap, or a pinned
  download) and recorded here.
