# [npm issue 7979](https://github.com/npm/npm/issues/7979)

## Steps to reproduce:

0. **Use npm version 2.7.1 or below**.

 - `npm install -g npm@2.7.1` Somewhere between `2.7.1` and `2.7.6`, there's been a change that rewrites 'git+ssh://' uris to `git://` IFF host is github.com.

1. Globally enable SSH ControlMaster by placing this at the top of your `~/.ssh/config`:

  ```
  ControlMaster auto
  ControlPath ~/.ssh/master-%r@%h:%p
  ControlPersist 10m
  ```

2. `npm cache clean` to clean the cache
3. `rm -rf ~/.ssh/master-*` to remove open socket descriptors just in case
4. `npm install --loglevel=info`

# Actual resuls:

```
npm info it worked if it ends with ok
npm info using npm@2.8.3
npm info using node@v0.10.35
npm WARN package.json npm-ssh-controlmaster@3.0.0 No repository field.
npm info preinstall npm-ssh-controlmaster@3.0.0
npm WARN package.json test-repo-one@1.0.0 No description
npm WARN package.json test-repo-one@1.0.0 No repository field.
npm WARN package.json test-repo-two@1.0.0 No description
npm WARN package.json test-repo-two@1.0.0 No repository field.
npm info git [ 'clone',
npm info git   '--template=/Users/aleksey/.npm/_git-remotes/_templates',
npm info git   '--mirror',
npm info git   'ssh://git@github.com/lxe/test-repo-one.git',
npm info git   '/Users/aleksey/.npm/_git-remotes/ssh-git-github-com-lxe-test-repo-one-git-501932d7' ]
npm info git [ 'clone',
npm info git   '--template=/Users/aleksey/.npm/_git-remotes/_templates',
npm info git   '--mirror',
npm info git   'ssh://git@github.com/lxe/test-repo-two.git',
npm info git   '/Users/aleksey/.npm/_git-remotes/ssh-git-github-com-lxe-test-repo-two-git-58c17dcf' ]
npm info git [ 'rev-list', '-n1', 'master' ]
npm info git [ 'clone',
npm info git   '/Users/aleksey/.npm/_git-remotes/ssh-git-github-com-lxe-test-repo-two-git-58c17dcf',
npm info git   '/var/folders/zl/41hpvlz54xg0d603wxfqhlf80000gn/T/npm-18567-211df082/git-cache-8b59285f1293/fc06ac925bc46dd17f72f71441c7e9f0fc18e61e' ]
npm info git [ 'checkout', 'fc06ac925bc46dd17f72f71441c7e9f0fc18e61e' ]
npm info already installed test-repo-two@1.0.0

...
```

^ Hangs indefinitely

# Expected results:

```
npm info it worked if it ends with ok
npm info using npm@2.7.1
npm info using node@v0.10.35
npm WARN package.json npm-ssh-controlmaster@3.0.0 No repository field.
npm info preinstall npm-ssh-controlmaster@3.0.0
npm info git [ 'config', '--get', 'remote.origin.url' ]
npm info git [ 'config', '--get', 'remote.origin.url' ]
npm info git [ 'fetch', '-a', 'origin' ]
npm info git [ 'fetch', '-a', 'origin' ]
npm info git [ 'rev-list', '-n1', 'master' ]
npm info git [ 'rev-list', '-n1', 'master' ]
npm info git [ 'clone',
npm info git   '/Users/aleksey/.npm/_git-remotes/ssh-git-github-com-lxe-test-repo-two-git-58c17dcf',
npm info git   '/var/folders/zl/41hpvlz54xg0d603wxfqhlf80000gn/T/npm-92738-9d12c32d/git-cache-e8ed9eb06d69/fc06ac925bc46dd17f72f71441c7e9f0fc18e61e' ]
npm info git [ 'clone',
npm info git   '/Users/aleksey/.npm/_git-remotes/ssh-git-github-com-lxe-test-repo-one-git-501932d7',
npm info git   '/var/folders/zl/41hpvlz54xg0d603wxfqhlf80000gn/T/npm-92738-9d12c32d/git-cache-1c9bbe432686/990718389ff778ceb76d1a705e7953f596e64853' ]
npm info git [ 'checkout', 'fc06ac925bc46dd17f72f71441c7e9f0fc18e61e' ]
npm info git [ 'checkout', '990718389ff778ceb76d1a705e7953f596e64853' ]
npm info install test-repo-two@1.0.0 into /Users/aleksey/devel/npm-ssh-controlmaster
npm info install test-repo-one@1.0.0 into /Users/aleksey/devel/npm-ssh-controlmaster
npm info installOne test-repo-two@1.0.0
npm info installOne test-repo-one@1.0.0
npm info preinstall test-repo-two@1.0.0
npm info preinstall test-repo-one@1.0.0
npm info build /Users/aleksey/devel/npm-ssh-controlmaster/node_modules/test-repo-two
npm info linkStuff test-repo-two@1.0.0
npm info install test-repo-two@1.0.0
npm info build /Users/aleksey/devel/npm-ssh-controlmaster/node_modules/test-repo-one
npm info linkStuff test-repo-one@1.0.0
npm info install test-repo-one@1.0.0
npm info postinstall test-repo-two@1.0.0
npm info postinstall test-repo-one@1.0.0
npm info build /Users/aleksey/devel/npm-ssh-controlmaster
npm info linkStuff npm-ssh-controlmaster@3.0.0
npm info install npm-ssh-controlmaster@3.0.0
npm info postinstall npm-ssh-controlmaster@3.0.0
npm info prepublish npm-ssh-controlmaster@3.0.0
test-repo-two@1.0.0 node_modules/test-repo-two

test-repo-one@1.0.0 node_modules/test-repo-one
npm info ok
```

# Possible explanation:

When npm clones 2 repos from the same SSH host when [SSH ControlMaster](http://en.wikibooks.org/wiki/OpenSSH/Cookbook/Multiplexing) is enabled, it does so in parallel. Git invokes the ssh client for both repos at the same time. When ControlMaster is enabled, the ssh client attempts to create a `.ssh/master-git@github.com:22` file that serves as an open descriptor for multiple connections.

There's a reace condition here however, when cloning `repo-1`, the socket file is created, but the connection is not yet established, by the time `repo-2` is being cloned. The `ssh` session sees the existing socket file, attempts to use the connection associated with it, fails (since `repo-1` hasn't finished creating it yet), and hangs indefinitely.

This is an OpenSSH bug more than anything, I think.


