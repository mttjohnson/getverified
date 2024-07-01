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
