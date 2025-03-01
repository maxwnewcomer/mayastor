# Building Mayastor

> You will not need to always build Mayastor. We will provide official x86_64
> & aarch64 binaries and images in future releases.

Mayastor is a multi-component [Rust][rust-lang] project that makes heavy use of
[Nix][nix-explore] for our development and build process.

If you're coming from a non-Rust (or non-Nix) background, **building Mayastor may be a bit
different than you're used to.** There is no `Makefile`, you won't need a build toolchain,
you won't need to worry about cross compiler toolchains, and all builds are reproducible.

## Table of Contents

- [Prerequisites](#Prerequisites)
- [Iterative Builds](#Iterative-Builds)
- [Artifacts](#Artifacts)

## Prerequisites

Mayastor **only** builds on modern Linuxes. We'd adore contributions to add support for
Windows, FreeBSD, OpenWRT, or other server platforms.

If you do not have a Linux system:

- **Windows:** We recommend using [WSL2][windows-wsl2] if you only need to
  build Mayastor. You'll need a [Hyper-V VM][windows-hyperv] if you want to use it.
- **Mac:** We recommend you use [Docker for Mac][docker-install]
  and follow the Docker process described. Please let us know if you find a way to
  run it!
- **FreeBSD:** We _think_ this might actually work, SPDK is compatible! But, we haven't
  tried it yet.
- **Others:** This is kind of a "Do-it-yourself" situation. Sorry, we can't be more help!

The only thing your system needs to build Mayastor is [**Nix**][nix-install].

Usually [Nix][nix-install] can be installed via (Do **not** use `sudo`!):

```bash
curl -L https://nixos.org/nix/install | sh
```

> **Can't install Nix?**
>
> That's totally fine. You can use [`docker`][docker-install] just fine for one-off or occasional PRs!
>
> This flow will get you a pre-fetched `nix` store:
>
> ```bash
> docker run --name mayastor-nix-prefetch -it -v $(pwd):/scratch:rw --privileged --workdir /scratch nixos/nix nix-shell --run "exit 0"
> docker commit mayastor-nix-prefetch mayastor/dev-env:latest
> docker rm mayastor-nix-prefetch
> docker run --rm -it -v $(pwd):/scratch:rw --workdir /scratch mayastor/dev-env:latest nix-shell
> ```
>
> To re-enter, just run the last command again.

- Some of our team uses [NixOS][nixos] which has `nix` baked in, but you don't need to.
- Some of our team uses [`direnv][direnv], but you don't need to.

**Want to run or hack on Mayastor?** _You need more configuration!_ See
[running][doc-run], then [testing][doc-test].

You can use a tool like [`direnv`][direnv] to automate `nix shell` entry.
If you are unable to use the Nix provided Rust for some reason, there are `rust` and
`spdk-path` arguments to Nix shell. `nix-shell --arg rust none`

After cloning the repository don't forget to run a:

```bash
git submodule update --init --recursive
```

to initialize the submodules.

## Iterative Builds

Contributors often build Mayastor repeatedly during the development process.

```bash
nix-shell
```

Once entered, you can start any tooling (eg `code .`) to ensure the correct resources are available.
The project can then be interacted with like any other Rust project.

Building:

```bash
cargo build
cargo build --release
```

**Want to run or hack on Mayastor?** _You need more configuration!_ See
[running][doc-run], then [testing][doc-test].

## Artifacts

There are a few ways to build Mayastor! If you're hacking on Mayastor, it's best to use
[`nix-shell`][nix-shell] (above) then turn to traditional Rust tools. If you're looking for releases,
use [`nix build`][nix-build] or [`nix bundle`][nix-bundle] depending on your needs.

> **Why is the build process this way?**
>
> Mayastor creates [_reproducible builds_][reproducible-builds], it won't use any of your
> local system dependencies (other than `nix`). This is a component of the best practices of the
> [Core Infrastructure Initiative][cii-best-practices]. More on how Nix works can be found in the
> [Nix paper][nix-paper].

### Building non-portable Nix derivations

You can build release binaries of Mayastor with [`nix build`][nix-build]:

```bash
nix build -f . -o artifacts/pkgs io-engine
ls artifacts/pkgs/bin
casperf  io-engine  io-engine-client
```

Try them as if they were installed:

```bash
nix shell -f . io-engine
```

### Building portable Nix bundles

In order to make an artifact which can be distributed, we use [`nix bundle`][nix-bundle].

> **TODO:** We currently don't generate bundles some executables, such as
> `io-engine-client`. This is coming.

```bash
for BUNDLE in io-engine io-engine-cli casperf; do
  echo "Bundling ${BUNDLE} to artifacts/bundle/${BUNDLE}"
  nix bundle -f . -o artifacts/bundles/${BUNDLE} units.release.${BUNDLE} --extra-experimental-features flakes
done
```

Test them:

```bash
for FILE in artifacts/bundles/*; do
 echo "Testing bundle ${FILE}..."
 ${FILE} --version
done
```

### Building Docker images

Build the Docker images with the CI build script:

```bash
  ❯ ./scripts/release.sh --help
  Usage: release.sh [OPTIONS]

    -d, --dry-run              Output actions that would be taken, but don't run them.
    -h, --help                 Display this text.
    --registry <host[:port]>   Push the built images to the provided registry.
                               To also replace the image org provide the full repository path, example: docker.io/org
    --debug                    Build debug version of images where possible.
    --skip-build               Don't perform nix-build.
    --skip-publish             Don't publish built images.
    --image           <image>  Specify what image to build and/or upload.
    --tar                      Decompress and load images as tar rather than tar.gz.
    --skip-images              Don't build nor upload any images.
    --alias-tag       <tag>    Explicit alias for short commit hash tag.
    --tag             <tag>    Explicit tag (overrides the git tag).
    --incremental              Builds components in two stages allowing for faster rebuilds during development.
    --skopeo-copy              Don't load containers into host, simply copy them to registry with skopeo.
    --skip-cargo-deps          Don't prefetch the cargo build dependencies.

  Environment Variables:
    RUSTFLAGS                  Set Rust compiler options when building binaries.

  Examples:
    release.sh --registry 127.0.0.1:5000

  ❯ ./scripts/release.sh --registry localhost:5000 --image "io-engine"
```

The container images are packaged and pushed using either docker or podman - whichever is run successfully with
--version cli argument.
If you want to specifically test one of these first, please set DOCKER env variable.

Build the Docker images with [`nix build`][nix-build]:

```bash
  nix-build --out-link artifacts/docker/mayastor-io-engine-image -A images.io-engine
```

**Optionally,** the generated Docker images will **not** tag to the `latest`. You may wish to do that if
you want to run them locally:

```bash
  ./scripts/release.sh --registry $registry --image "io-engine" --alias-tag latest
```

### Building KVM images

> **TODO:** We're still writing this! Sorry!

### Building Artifacts the Hard Way

> This isn't really the 'hard way', you'll still use `cargo`.

> **TODO:** We're still writing this! Sorry! For now, please refer to
> `spdk-rs` README on this matter.

[doc-run]: ./run.md

[doc-test]: ./test.md

[direnv]: https://direnv.net/

[nix-explore]: https://nixos.org/explore.html

[nix-install]: https://nixos.org/download.html

[nix-develop]: https://nixos.org/manual/nix/unstable/command-ref/new-cli/nix3-develop.html

[nix-paper]: https://edolstra.github.io/pubs/nixos-jfp-final.pdf

[nix-build]: https://nixos.org/manual/nix/unstable/command-ref/new-cli/nix3-build.html

[nix-bundle]: https://nixos.org/manual/nix/unstable/command-ref/new-cli/nix3-bundle.html

[nix-shell]: https://nixos.org/manual/nix/unstable/command-ref/new-cli/nix3-shell.html

[nix-channel]: https://nixos.wiki/wiki/Nix_channels

[nixos]: https://nixos.org/

[rust-lang]: https://www.rust-lang.org/

[windows-wsl2]: https://wiki.ubuntu.com/WSL#Ubuntu_on_WSL

[windows-hyperv]: https://wiki.ubuntu.com/Hyper-V

[docker-install]: https://docs.docker.com/get-docker/

[reproducible-builds]: https://reproducible-builds.org/

[cii-best-practices]: https://www.coreinfrastructure.org/programs/best-practices-program/

[direnv]: https://direnv.net/
