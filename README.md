# filter-experiment: long-running smudge filters in `git clone` vs. `gix clone`

This investigates how smudge (checkout) process filters interact with executable permissions in `git clone` and gitoxide's `gix clone`.

The experiment here relates to the mention of filters in [RUSTSEC-2025-0001](https://rustsec.org/advisories/RUSTSEC-2025-0001.html) (CVE-2025-22620, [GHSA-fqmf-w4xh-33rh](https://github.com/GitoxideLabs/gitoxide/security/advisories/GHSA-fqmf-w4xh-33rh)). But this is a separate experiment from the proof of concept there, and it is *not* an alternative proof of concept: the case this exercises does not appear to be vulnerable even in versions of `gix-worktree-state` affected by that vulnerability, at least with the `gix clone` command itself.

However, this does demonstrate a separate, non-security bug where, when a file is tracked as executable, and the configuration and filesystem for `gix clone` are such that it should set executable bits for files tracked as executable, it fails to do so if the file has an attribute applied to it and a long-running smudge filter is configured to be used when checking out files with that attribute. Other executable files (including in the same checkout) that do not have the attribute applied are still checked out as executable.

This happens both with `gix-worktree-state` 0.17.0 (which fixed [RUSTSEC-2025-0001](https://rustsec.org/advisories/RUSTSEC-2025-0001.html)) and earlier versions, with 0.16.0 having also been tested. The results of the experiment are unchanged between the tested versions.

*This repository supersedes [that older gist](https://gist.github.com/EliahKagan/13664496a3a33b79e3ad1640a05879cf) (which is not likely to be updated).*

## Setup

This is meant to be run on Unix-like systems and was tested on Arch Linux. It tests using the `arrow.rs` filter example from `gix-filter`. (The arrow filter works on Windows as well as Unix-like systems. It's just this experiment that may not.) The experiment would not be very useful to attempt on Windows, because it pertains to how Unix-style executable permissions are set in files on disk.

As written, the `run-experiment` script and `arrow` symlink assume the [`gitoxide`](https://github.com/GitoxideLabs/gitoxide) repository is cloned, with that name, as a sibling directory of this one--that is, that it can be accessed at `../gitoxide`. The script can be modified accordingly if that is not the case, and the symlink either modified or deleted.

It does not assume that the example (or any part of `gitoxide`) has been built. It will build the example if it has not already been built or if it is out of date compared to whatever is checked out in the `gitoxide` repository. A working Rust toolchain and `cargo` command is assumed. Because this attempts to build the example, **it will write, and may over write, to files in the `../gitoxide/target` directory**. (Ordinarily that would not be a problem, since one rarely puts anything that has to be preserved there.)

The reason this builds the example rather than installing it with `cargo run` is that, at least as of this writing, it is not listed in `examples` in any `Cargo.toml` file, so it cannot easily be installed from crates.io in that way. (Building the example from the `gitoxide` repository also allows an arbitrarily selected version of the `arrow.rs` filter to be used more easily.)

## The experiment

The `run-experiment` script takes an argument, `git` or `gix`, to tell it what clone command to test. This can actually be arbitrarily many arguments, in case you want to test it with options like `--trace` (for `gix`) or extra `-c var=value` pairs. Usually it would be run just as `./run-experiment git` or `./run-experiment gix`.

It runs the command specified by its arguments, with the additional arguments `-c filter.arrow-example.process=... clone`, where `...` is a full path to the `arrow` symlink in the current directory.

The test repository it creates and clones locally has three important files other than its `.gitattributes` file:

- `a`, a *non-executable* regular file (mode 100644) that *is* subjected to the filter.
- `b`, an *executable* regular file (mode 100755) that *is* subjected to the filter.
- `c`, an *executable* regular file (mode 100755) that is *not* subjected to the filter.

Between runs of the experiment, `./clean` can be used to remove the test repositories that the script creates and uses. (The script will not proceed if either exist.)

## Results

### `git clone`

With `git clone`, **both `b` and `c`** are checked out with executable permissions, which is the expected behavior:

```text
-rw-r--r-- 1 ek ek    5 Jan  8 21:56 a
-rwxr-xr-x 1 ek ek    5 Jan  8 21:56 b*
-rwxr-xr-x 1 ek ek    2 Jan  8 21:56 c*
```

For full output of the `git clone` experiment, see [`transcript-1-git.txt`](transcript-1-git.txt).

### `gix clone`

With `gix clone` (in current versions of `gitoxide`, last rechecked as of 19 January 2025), **only `c`, and not `b`** is checked out with executable permissions:

```text
-rw-r--r-- 1 ek ek    5 Jan  8 21:57 a
-rw-r--r-- 1 ek ek    5 Jan  8 21:57 b
-rwxr-xr-x 1 ek ek    2 Jan  8 21:57 c*
```

This is to say that, when a long running smudge filter (i.e. process smudge filter) is used in the checkout, the files it applies to do not have `+x` set. This makes no difference on `a`, which is not tracked as executable, nor `c`, which is tracked as executable but does not have the attribute that causes the filter to apply. But it prevents `b` from being set executable as intended.

For full output of the `gix clone` experiment, see [`transcript-2-gix.txt`](transcript-2-gix.txt).

## Caveat

If I understand correctly, the arrow filter, when run as a process filter, enables delays automatically unless told not to. However, it may be that the small number of files I was using were insufficient to actually produce any delays, or interesting ones. I am unsure how, if at all, that might affect this. Unfortunately, process filters, especially with delays, are not an aspect of Git behavior that I have much prior experience with.

## License

[0BSD](LICENSE)

## Other notes

- The part of the script that may be most confusing is actually the `sed` command. This quotes a path for a shell, in a way that works even if the path starts out with single quotes characters in it, so that directories under e.g. `/Users/O'Shaughnessy` don't break. It is conceptually unrelated to the actual goal, but I was unable to avoid it by using relative paths, for the commented reason that `git` and `gix` interpret relative paths in this particular situation (of a new clone) as being relative to different locations, which would prevent a generically written test from working automatically with both of them.

- If you're looking for the proof-of-concept code associated with [RUSTSEC-2025-0001](https://rustsec.org/advisories/RUSTSEC-2025-0001.html) then, as noted above, this is not what you're looking for. You'll most likely want to look it in the context of that advisory, which is likely to be sufficient. But if you also want it in a repository, with an associated `Cargo.toml` and `Cargo.lock` with an affected `gix-worktree-state` version, see the [`checkout-index`](https://github.com/EliahKagan/checkout-index) repository.
