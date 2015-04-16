## Steps to reproduce:

1. Globally enable SSH ControlMaster by placing this at the top of your `~/.ssh/config`:

  ```
  ControlMaster auto
  ControlPath ~/.ssh/master-%r@%h:%p
  ControlPersist 10m
  ```

2. `npm cache clean`
2. `npm install`

