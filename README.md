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

```sh
cd /srv/http/repos/chaotic-aur
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst chaotic-mirrorlist.pkg.tar.zst
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst.sig chaotic-mirrorlist.pkg.tar.zst.sig
```

## [The packages repo](https://github.com/chaotic-aur/packages) - containing lists of packages to be built by our builders

- The individual folders represent our builders.
- The individual `*.txt` files are so called routines, package lists that are going to be built at individual times. Resource consuming packages are likely not put into hourly routines while smaller packages are.
- Stable packages are only downloaded if an update is detected, `-git` ones are always downloaded but only built if its version changed (down/upgrade doesn't matter).
- AUR packages just need their pkgname added.
- Non-AUR packages are added as follows: `pkgname:https://some.url.git` (this can also be used to force the download of a package that never changes version but is still updated, eg. a package building from git but incorrectly not having the `-git` suffix.
- Where to place newly added packages?
  - General rule of thumb: `-bin` and all other, quick to build packages belong into `hourly` routines. Likewise, heavy packages belong into `daily`.
  - New, quick to build packages can always be added to `ufscar-hpc` - it is a big cluster with a lot of processing power. Balancing packages between `hourly.1` and `hourly.2` routines is a good idea.
  - The heaviest packages are built by `ufscar-hpc`, eg. `ungoogled-chromium`
  - The `garuda-cluster` handles packages of the Garuda team, therefore it is managed by Garuda staff only.
  - The `dragon-cluster` is handled by [@dr460nf1r3](https://github.com/dr460nf1r3), most of the packages here are used by him. Since lately, also requested, heavier packages like kernels can be added to the `afternoon` routine.
  - The `dragon-cluster` also builds the complete `kde-git` stack. All things related to KDE can have their `-git` PKGBUILDs added to the `nightly` routine.
  - The `CatBuilder` is [@Edu4rdSHL](https://github.com/Edu4rdSHL) PC, therefore he is the only one managing the package list. TKG Kernels are also managed by him.
- Since its best to keep good track of where packages came from, the established way of formatting the package list:
  - For requested packages we want to mention the issue followed by the requested package (and eventually its deps to make clear, why the particular package was added as well). Leave a space between issues and sort things alphabetically.
  - Issues should also be sorted in order.
  - Staff members may add their desired packages to the `ufscar-hpc` or `dragon-cluster` routines. Take a look where other maintainers added their stuff and add it accordingly.
  - If in doubt, the staff chat is always the best place to ask questions! 🐱

    ```md
    # Issue 1337
    absolutelyrequireddependency # (dep superfancypackage)
    superfancypackage

    # Nico
    everything-git
    firedragon
    octopi-git
    sudo-git
    ```

## Our [interfere repo](https://github.com/chaotic-aur/interfere)

- Used to fix up AUR packages or PKGBUILDs which we don't control ourselves.
- Every folder represents a package. To fix a package with `pkgname=somepackage` the folder would be `somepackage`.
- Several possibilities to proceed exist:
  - `PKGBUILD.append`: everything in there is going to be the updated content of the original PKGBUILD. Fixing `build()` as is easy as adding the fixed`build()` into this file. This can be used for all kinds of fixes. If something needs to be added to an array, this is as easy as `makedepend+=somepackage`.
  - `interfere.patch`: a patch file which can be used to fix either multiple files or PKGBUILD if a lot of changes are required. All changes need to be added in this file.
  - `prepare`: A script which is being executed after the building chroot has been setup. It can be used to source envvars or modify other things before compilation starts.
- Incrementing `pkgrel` can be done be by downloading the PKGBUILD (`chaotic get somepackage`), increasing its pkgrel temporarily (`chaotic bump somepackage`) and building it afterwards (`chaotic mkd somepackage`).
- The `chaotic bump` command syncs the incremented pkgrel back to the interfere repo, which means it will be available for all other builders too. This can be useful for mass rebuilds as well, eg. in case of Python version updates.

## Handling the [toolbox](https://github.com/chaotic-aur/toolbox)

- The toolbox repo contains [good instructions](https://github.com/chaotic-aur/toolbox#cli) concerning CLI usage
- Usual workflow could look like this:
  - Building or updating an AUR package:
    - `chaotic get somepackage`
    - `chaotic mkd somepackage`
  - Rebuilding an AUR package:
    - `chaotic get somepackage`
    - `chaotic bump somepackage`
    - `chaotic mkd somepackage`
  - Updating or building a non-AUR package that also isn't defined in any routine:
    - `git clone someurl.git`
    - `chaotic mkd justclonedfolder`
  - Cleaning sources of a package when a package expects a different one (useful when source changed):
    - `chaotic cls somepackage`
  - Updating the local interfere repo to build a package using the fix:
    - `chaotic si`
- The latest logfile of builds can always be found at the [log directory](https://builds.garudalinux.org/repos/chaotic-aur/logs/) of our main nodes URL. Here, every logfile gets uploaded no matter if successful or failed.

## Some examples on how to handle stuff

- A package didn't build:
  - Check the [logfiles](https://builds.garudalinux.org/repos/chaotic-aur/logs/) – a dependency is missing
  - Add the dependency via interfere using PKGBUILD.append: `makedepends+=(missingdep)`
  - Connect to a builder and update the interfere repo: `chaotic si`
  - Build the package: `chaotic get somepackage && chaotic mkd somepackage`
  - (optional, but _very_ recommended: report the missing dependency at the corresponding AUR page!)
- A package depends on an old shared library:
  - Either depend on reported issues or verify it's an issue by installing the package locally
  - Download the PKGBUILD of the package: `chaotic get somepackage`
  - Bump the `pkgrel` of the package: `chaotic bump somepackage`
  - Build the package using the updated shared library: `chaotic mkd somepackage`
