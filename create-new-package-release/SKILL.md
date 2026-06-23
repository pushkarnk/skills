# Skill to update a Debian package for a new upstream release

This skill describes how to update a non-native Debian package for a new release of its upstream project.

## 1. Clone the package repository
If a package name is provided, run this command:
```
git ubuntu clone <package-name>
```

If a package URL is provided, run this command:
```
git clone <package-url>
```

If this warning is reported:
```
warning: remote HEAD refers to nonexistent ref, unable to checkout
```
then run this command:
```
cd <package-name>
git checkout <main or ubuntu/devel>
```

If the warning above is not reported, just run:
```
cd <package-name>
```
For the rest of the skill, maintain this directory as the working directory.

## 2. Check for new a upstream version

Run the uscan command:
```
uscan
```

If a new version is available, you will see a message like:
```
Newest version of <package-name> on remote site is A.B.C, local version is X.Y.Z
 => Newer package available from:
```

The command will also download <package-name>_<new-version>.tar.gz in the parent directory.

## 3. Importing the new version
### 3.1 Create a new branch named `upstream`

```
git branch upstream
```

### 3.2. Run gbp import-orig
```
gbp import-orig --debian-branch=<main or ubuntu/devel> --upstream-branch=upstream ../<package-name>_<new-version>.tar.gz.
```

The tool will ask:
```
What is the upstream version? [A.B.C]
```

Just press the ENTER key. If this command passes (exit code is 0), the source has been imported.

## 4. Add a debian/changelog entry
Use dch to create a new debian/changelog entry:

```
dch -i
```

The entry text should be of this format:
```
  * New upstream release <upstream-version>.
```
The version string in this entry must be: <upstream-version>-<package-version>.

If this package is Ubuntu-only, <package-version> is `0ubuntu1`.
If this package is a Debian package, <package-version> is `1`.
Ask the user if the above is not clear.

The Ubuntu release name should be provided by the user. Please prompt if not.

Save the file.

## 5. Update debian/patches
1. Run `quilt push -a` to apply all patches.
2. If no patches fail to apply, run `quilt pop -a` to unapply all patches. Move to step 6.
3. If any patch fails to apply, run `quilt pop -a` to and list all the patches that have conflicts. These patches need updates.
4. For each patch that needs an update, referring to the debian/patches/series file:
   1. Run `quilt push` for all patches up to the patch that needs an update.
   2. Run `quilt push -f` for the failing patch.
   3. Update the patch as per the new upstream code.
   4. Run `quilt refresh -p ab --no-index --no-timestamps` to update the patch.
5. Run `quilt pop -a` to unapply all patches.

## 6. Run a build
Test the package build using `sbuild`:
```
sbuild --no-clean-source -c <ubuntu-release-name>-amd64 -d <ubuntu-release-name>
```

Please report the final outcome of the sbuild command.
