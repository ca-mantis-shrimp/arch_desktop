# Overview
Be sure to read the documentation before you do anything:
- [README](./README.md)
## Tooling
- Leverage `mkosi` wherever you can
- `man` is your friend, do your research
    - `systemd`
        - `systemd-repart`
        - `systemd-sysext`
        - `systemd-boot`
        - `systemd-networkd`
        - `systemd-resolved`
        - `systemd-homed`
        - `machinectl`

### Respect Guidance
Im going to say this again to be really clear DONT just run commands willy-nilly, DO go through the MAN pages FIRST, THEN if you cant find what you need there go online to do research, THEN if you still cant find the information you need, ASK for help.

do NOT just go spelunking through image filesystems or trying to review `mkosi` source code directly, its a waste of your time and energy unless we are in deep debugging mode.

We need to THINK carefully and respect precedence here, we dont want to be jumping around, adding complexity, or making things worse.

This also goes for our configuration. just because you CAN put scripts everywhere the manually update the vm, doesnt mean you SHOULD. be thoughtful about where you put things, and try to keep things as simple as possible, leverage our DECLARATIVE configuration tools FIRST before you start adding imperative scripts.
