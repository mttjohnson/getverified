# Trying to get verified commits

Initial commit that is unverified.

![Initial commit showing unverified tag and where to find more info.](files/unverified_initial_commit.png?raw=true "Unverified commit info")

The link highlighted goes to "[Learn about vigilant mode](https://docs.github.com/en/authentication/managing-commit-signature-verification/displaying-verification-statuses-for-all-of-your-commits)".

The reason why this "Unverified" tag shows up is because my account was set in Vigilant mode.

![GitHub > Account > Settings > SSH and GPG keys > Vigilant mode. Flag unsigned commits as unverified is checked.](files/settings_vigilant_mode.png?raw=true "Vigilant mode setting")

Note that I have SSH keys listed as "Authentication keys" and the same keys in the Singing keys.

Need to add some information to the local system git config to enable signing when commits are created.

```bash
# View the existing configs
git config --global --get user.signingKey
git config --global --get gpg.format

# Set the configs for signing with an ssh key
git config --global user.signingKey ~/.ssh/id_ed25519.pub
git config --global gpg.format ssh
```

Using an ssh key for signing and specifying the public key in the git config requires the ssh-agent having the associated private key loaded.

```bash
ssh-add -l # list the key fingerprints of the keys the agent has loaded
ssh-add -L # list the full public keys the agent has loaded
```

This should add some additional settings to the `~/.gitconfig` git config file.
```
[user]
	signingKey = /Users/mattjohnson/.ssh/id_ed25519.pub
[gpg]
	format = ssh
```

Testing a commit

```bash
git add *
git commit -m "testing a commit after setting git configs for signing commits"
git push
```

...this still results in an Unverified commit. It turns out you have to add an additional flag `-S` to the git commit in order to sign it.

```bash
git add *
git commit -S -m "testing a commit after using -S with the git commit command"
git push
```

This results in a Verified commit, but it's possible to add a config so this additional `-S` is not required on every commit.

https://docs.gitlab.com/ee/user/project/repository/signed_commits/ssh.html

```bash
git config --global commit.gpgsign true
```

Having this config in place

```bash
git add *
git commit -m "testing another Verified signed commit"
git push
```

To verify commits locally there are a few additional things needed

Setup an allowed_signers file
```bash
touch ~/.ssh/allowed_signers
git config --global gpg.ssh.allowedSignersFile ~/.ssh/allowed_signers
echo "$(git config --get user.email) namespaces=\"git\" $(cat ~/.ssh/id_ed25519.pub)" >> ~/.ssh/allowed_signers
```

Then you'll be able to verify the git commits against the expected allowed signers.
```bash
git log --show-signature
```
...and see something like this:
```
commit c04908e6120002c5ade30066759b3ca9845d9f2c (HEAD -> main, origin/main)
Good "git" signature for m@ttjohnson.com with ED25519 key SHA256:JEx/Jj5vRc+Cfy8pvPzURGQZyEqrvhAGdOj5mtylHGw
Author: Matt Johnson <m@ttjohnson.com>
Date:   Sun Jun 30 20:46:20 2024 -0500

    testing another Verified signed commit
```

Adding a sign-off line to a commit using the cli
```bash
git add *
git commit -s -m "testing a sign-off message with -s"
git push
```

It's possible to add a git hook to a specific repo that will automatically signoff the commit

```bash
(cat <<'BASH_CODE'
#!/bin/sh

# Automatically signoff all commits.

# $RANDOM will be undefined if not using bash, so don't use set -u
random=$( (whoami ; hostname ; date; cat $1 ; echo $RANDOM) | git hash-object --stdin)
dest="$1.tmp.${random}"

trap 'rm -f "${dest}"' EXIT

GIT_AUTHOR="$(git var GIT_AUTHOR_IDENT | sed 's/^\(.*>\).*$/\1/')"
GIT_COMMITTER="$(git var GIT_COMMITTER_IDENT | sed 's/^\(.*>\).*$/\1/')"
git config --bool --get commit.signoff >/dev/null
if [ $? = 0 -a "${GIT_AUTHOR}" = "${GIT_COMMITTER}" ]; then
  SOB="Signed-off-by: ${GIT_AUTHOR} (.git/hooks/commit-msg)"
  git -c trailer.ifexists=doNothing interpret-trailers \
        --trailer "${SOB}" < "$1" > "${dest}"
  mv "${dest}" "$1"
fi

if ! grep -q "^Signed-off-by:" "$1"; then
  echo "Commit message is not signed off: $1"
  echo "Use 'git commit -s' to sign off the commit message."
  echo "Use 'git config commit.signoff true' to automatically sign off commits."
  exit 1
fi
BASH_CODE
) > .git/hooks/commit-msg
chmod +x .git/hooks/commit-msg
```
