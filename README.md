# Internal documentation

## Setting up the main node

- UFSCar needs a `ufscar-hpc` user which needs its pubkey added in `authorized_keys`
- Permissions in `srv/http/repos/chaotic-aur/` need to be setup as follows so nodes can upload their packages:
    - `chmod g+s x86_64 logs`
    - `chown -R :chaotic_op x86_64 logs`
    - `chmod 775 x86_64 logs`
    - `chmod 664 x86_64/* logs/*`
- If builders can't execute `chaotic dbb` and adding to database fails due to this, symlink `/usr/local/bin/chaotic` to `/usr/bin/chaotic`
