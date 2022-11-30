# GIt
## How to change an ssh key for git
If you'd like to change it only for the current terminal session, then setting up `GIT_SSH_COMMAND` should be enough.
```sh
GIT_SSH_COMMAND="ssh -i ~/.ssh/alternative_id_rsa" git fetch
```
However if you'd like to set it for the particular repo:
```sh
git config core.sshCommand "ssh -i ~/.ssh/alternative_id_rsa"
```

