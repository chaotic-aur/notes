# Internal documentation

## Setting up the main node

- UFSCar needs a `ufscar-hpc` user which needs its pubkey added in `authorized_keys`
- Permissions in `srv/http/repos/chaotic-aur/` need to be setup as follows so nodes can upload their packages:
    - `chmod g+s x86_64 logs`
    - `chown -R :chaotic_op x86_64 logs`
    - `chmod 775 x86_64 logs`
    - `chmod 664 x86_64/* logs/*`
- If builders can't execute `chaotic dbb` and adding to database fails due to this, symlink `/usr/local/bin/chaotic` to `/usr/bin/chaotic`

## Deploying new mirrorlist or keyring

```
cd /srv/http/repos/chaotic-aur
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst chaotic-mirrorlist.pkg.tar.zst
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst.sig chaotic-mirrorlist.pkg.tar.zst.sig
```

## The packages repo - containing lists of packages to be built by our builders

- The individual folders represent our builders
- The individual `*.txt` files are so called routines, package lists that are going to be built at individual times. Ressource consuming packages are likely not put into hourly routines while smaller packages are.
- Stable packages are only downloaded if an update is detected, `-git` ones are always downloaded but only built if its version changed (down/upgrade doesn't matter)
- AUR packages just need their pkgname added
- Non-AUR packages are added as follows: `pkgname:https://some.url.git` (this can also be used to force the download of a package that never changes version but is still updated, eg. a package building from git but incorrectly not having the `-git` suffix

## Our interfere repo

- Used to fix up AUR packages or PKGBUILDs which we don't control ourselves
- Every folder represents a package. To fix a package with `pkgname=somepackage` the folder would be somepackage
- Several possibilities to proceed exist:
    - `PKGBUILD.append`: everything in there is going to be the updated content of the original PKGBUILD. Fixing `build()` as is easy as adding the fixed `build()` into this file. This can be used for all kinds of fixes. If something needs to be added to an array, this is as easy as `makedepends+=somepackage`
    - `interfere.patch`: a patch file which can be used to fix either multiple files or PKGBUILD if a lot of changes are required. All changes need to be added in this file.
    - `prepare`: A script which is being executed after the building chroot has been setup. It can be used to source envvars or modify other things before compilation starts.

## Handling the toolbox:

- The toolbox repo contains good instructions concerning CLI usage
- Usual workflow could look like this:
    - Rebuilding an AUR `-git` package:
      - `chaotic rm somepackage-git`
      - `chaotic get somepackage-git`
      - `chaotic mkd somepackage-git`
    - Updating a non-AUR package:
      - `git clone someurl.git`
      - `chaotic mkd justclonedfolder`
    - Cleaning sources of a package when a package expects a different one
      - `chaotic cls somepackage`

