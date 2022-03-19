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
- Every folder represents a package. To fix a package with `pkgname=somepackage` the folder would be `somepackage`
- Several possibilities to proceed exist:
    - `PKGBUILD.append`: everything in there is going to be the updated content of the original PKGBUILD. Fixing `build()` as is easy as adding the fixed `build()` into this file. This can be used for all kinds of fixes. If something needs to be added to an array, this is as easy as `makedepends+=somepackage`
    - `interfere.patch`: a patch file which can be used to fix either multiple files or PKGBUILD if a lot of changes are required. All changes need to be added in this file.
    - `prepare`: A script which is being executed after the building chroot has been setup. It can be used to source envvars or modify other things before compilation starts.

## Handling the toolbox:

- The toolbox repo contains [good instructions](https://github.com/chaotic-aur/toolbox#cli) concerning CLI usage
- Usual workflow could look like this:
    - Building or updating an AUR package:
      - `chaotic get somepackage`
      - `chaotic mkd somepackage`
    - Rebuilding an AUR `-git` package:
      - `chaotic rm somepackage-git`
      - `chaotic get somepackage-git`
      - `chaotic mkd somepackage-git`
    - Updating or building a non-AUR package:
      - `git clone someurl.git`
      - `chaotic mkd justclonedfolder`
    - Cleaning sources of a package when a package expects a different one (useful when source changed)
      - `chaotic cls somepackage`
    - Updating the local interfere repo to build a package using the fix:
      - `chaotic si`
- The always latest logfile of builds can be found at the [log directory](https://builds.garudalinux.org/repos/chaotic-aur/logs/) of our main nodes URL. Here, every logfile gets uploaded no matter if successful or failed.

## Some examples on how to handle stuff:

- A package didn't build:
  - Check the [logfiles](https://builds.garudalinux.org/repos/chaotic-aur/logs/) - a dependency is missing
  - Add the dependency via interfere using PKGBUILD.append: `makedepends+=(missingdep)`
  - Connect to a builder and update the interfere repo: `chaotic si`
  - Build the package: `chaotic get somepackage && chaotic mkd somepackage`
  - (optional, but very recommended: report the missing dependency at the corresponding AUR page!)
- A package depends on an old shared library:
  - Either depend on reported issues or verify its an issue by installing the package locally
  - Remove it from the repo in order to start a rebuild: `chaotic rm somepackage`
  - Build the package using the updated shared library: `chaotic get somepackage && chaotic mkd somepackage`
