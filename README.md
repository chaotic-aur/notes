# Chaotic-AUR documentation

## [The packages repo](https://github.com/chaotic-aur/packages) - containing lists of packages to be built by our builders

- The individual folders represent our builders.
- The individual `*.txt` files are so-called routines and package lists that are going to be built at individual times. Resource-consuming packages are likely not put into hourly routines while smaller packages are.
- Stable packages are only downloaded if an update is detected, `-git` ones are always downloaded but only built if their version changed (down/upgrade doesn't matter).
- AUR packages just need their pkgname added.
- Non-AUR packages are added as follows: `pkgname:https://some.url.git` (this can also be used to force the download of a package that never changes version but is still updated, eg. a package building from git but incorrectly not having the `-git` suffix.
- Packages that are part of a `pkgbase` need their `pkgbase` added instead of one of their `pkgnames`. Repoctl can't handle such packages yet, therefore one needs to treat it as `-git` package. Eg.: `pkgbase:https://some.url.git`
- Where to place newly added packages?
  - A general rule of thumb: `-bin` and all other, quick-to-build packages belong into `hourly` routines. Likewise, heavy packages belong in `daily`.
  - New, quick-to-build packages can always be added to `ufscar-hpc` - it is a big cluster with a lot of processing power. Balancing packages between `hourly.1` and `hourly.2` routines is a good idea.
  - The heaviest packages are built by `ufscar-hpc`, eg. `ungoogled-chromium`
  - The `garuda-cluster` handles packages of the Garuda team, therefore it is managed by Garuda staff only.
  - The `dragon-cluster` is handled by [@dr460nf1r3](https://github.com/dr460nf1r3), most of the packages here are used by him. Lately, also requested, heavier packages like kernels can be added to the `afternoon` routine.
  - The `dragon-cluster` also builds the complete `kde-git` stack. All things related to KDE can have their `-git` PKGBUILDs added to the `nightly` routine.
  - The `CatBuilder` is [@Edu4rdSHL](https://github.com/Edu4rdSHL) PC, therefore he is the only one managing the package list. TKG Kernels are also managed by him.
- Since its best to keep good track of where packages came from, the established way of formatting the package list:
  - For requested packages, we want to mention the issue followed by the requested package (and eventually its dependencies to make clear, why the particular package was added as well). Leave a space between issues and sort things alphabetically.
  - Issues should also be sorted in order.
  - Staff members may add their desired packages to the `ufscar-hpc` or `dragon-cluster` routines. Take a look at where other maintainers added their stuff and add it accordingly.
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

- If a package for some reason needs to be built before another one, a "barrier" can be put in place to force every makepkg before the barrier to finish before attempting to build the packages after it:
  - A good example of this can be seen in [this commit](https://github.com/chaotic-aur/packages/commit/ec2d70379dc9848af1942e504bbe47f178f5099f)

## Our [interfere repo](https://github.com/chaotic-aur/interfere)

- Used to fix up AUR packages or PKGBUILDs which we don't control ourselves.
- Every folder represents a package. To fix a package with `pkgname=somepackage` the folder would be `somepackage`. Likewise, several conditions can trigger actions.
- Several possibilities to proceed to exist:
  - `PKGBUILD.append`: everything in there is going to be the updated content of the original PKGBUILD. Fixing `build()` as is easy as adding the fixed `build()` into this file. This can be used for all kinds of fixes. If something needs to be added to an array, this is as easy as `makedepend+=(somepackage)`.
  - `interfere.patch`: a patch file that can be used to fix either multiple files or PKGBUILD if a lot of changes are required. All changes need to be added to this file.
  - `prepare`: A script which is being executed after the building chroot has been set up. It can be used to source environment variables or modify other things before compilation starts.
    - If somethign needs to be set up before the actual compilation process, commands can be pushed by inserting eg. `$CAUR_PUSH 'source /etc/profile'`. Likewise, package conflicts can be solved, eg. as follows: `$CAUR_PUSH 'yes | pacman -S nftables'` (single quotes are important because we want the variables/pipes to evaluate in the guest's runtime and not while interfering)
  - `on-failure.sh`: A script that is being executed if the build fails.
  - `on-success.sh`: A script that is being executed if the build succeeds.
- Incrementing `pkgrel` can be done by downloading the PKGBUILD (`chaotic get somepackage`), increasing its pkgrel temporarily (`chaotic bump somepackage`) and building it afterward (`chaotic mkd somepackage`).
- The `chaotic bump` command syncs the incremented pkgrel back to the interfere repo, which means it will be available for all other builders too. This can be useful for mass rebuilds as well, eg. in the case of Python version updates.

## Handling the [toolbox](https://github.com/chaotic-aur/toolbox)

- The toolbox repo contains [good instructions](https://github.com/chaotic-aur/toolbox#cli) concerning CLI usage
- The usual workflow could look like this:
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
  - Cleaning the build folder (removes successful `.log/.lock` files and build directories, keeping the failed logs):
    - `chaotic cleanpwd`
- The latest logfile of builds can always be found in the [log directory](https://builds.garudalinux.org/repos/chaotic-aur/logs/) of our main nodes URL. Here, every log file gets uploaded no matter if successful or failed.
- Builds via `chaotic mkd` or `chaotic routine` can be parallelized by adding `-j 10` before the command, eg. `chaotic -j 10 routine hourly` - this will save a lot of time, especially when building a lot of `-git` packages.

## Some examples on how to handle stuff

- A package didn't build:
  - Check the [logfiles](https://builds.garudalinux.org/repos/chaotic-aur/logs/) – a dependency is missing
  - Add the dependency via interfere using PKGBUILD.append: `makedepends+=(missingdep)`
  - Connect to a builder and update the interfere repo: `chaotic si`
  - Build the package: `chaotic get somepackage && chaotic mkd somepackage`
  - (optional, but _very_ recommended: report the missing dependency at the corresponding AUR page!)
- A package depends on an old shared library:
  - Either depends on reported issues or verify it's an issue by installing the package locally
  - Download the PKGBUILD of the package: `chaotic get somepackage`
  - Bump the `pkgrel` of the package: `chaotic bump somepackage`
  - Build the package using the updated shared library: `chaotic mkd somepackage`


## Administration
### Setting up the main node

- UFSCar needs a `ufscar-hpc` user which needs its public key added in `authorized_keys`
- Permissions in `srv/http/repos/chaotic-aur/` need to be set up as follows so nodes can upload their packages:
  - `chmod g+s x86_64 logs`
  - `chown -R :chaotic_op x86_64 logs`
  - `chmod 775 x86_64 logs`
  - `chmod 664 x86_64/* logs/*`
- If builders can't execute `chaotic dbb` and adding the packages to database fails due to this, symlink `/usr/local/bin/chaotic` to `/usr/bin/chaotic`

### Deploying new mirrorlist or keyring

```sh
cd /srv/http/repos/chaotic-aur
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst chaotic-mirrorlist.pkg.tar.zst
sudo ln -sfT x86_64/chaotic-mirrorlist-20211231-1-any.pkg.tar.zst.sig chaotic-mirrorlist.pkg.tar.zst.sig
```

### Resetting the repo

- Have repoctl's `config.toml` in `/root/.config/repoctl` 
- `su -` to ensure settings are present
- `repoctl reset` to create a new database with all files added
