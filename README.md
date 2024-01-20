# nix-quickref

* formatter: https://github.com/nix-community/nixpkgs-fmt
* language server: https://github.com/oxalica/nil and https://github.com/nix-community/nixd/
  * TODO: pick one to recommend or explain why to chose one over the other 

# Flakes

### Structure

A "nix flake" is a directory that contains a file called `nix.flake`.

`nix.flake` must contain a single attribute set that must contain:
* `outputs` a function

and can optionally contain:
* `description` a string description of the flake
* `inputs` an attribute set specifying all of the flake and non-flake source code available to `outputs`

Thus, the minimum flake is:
```nix
{
  outputs = _: {};
}
```
If we run `nix flake show` we can see that the flake is successfully evaluated:
```
$> nix flake show
path:/Users/me/nix-test?lastModified=1705776049&narHash=sha256-dUFmwYUM0Jz2TYchyyH4ENHELaY/an9kdSUOnjVDzGY%3D
```

### The `outputs` function

The `outputs` function takes an **attribute set** of **inputs**.<br>
There is always at least one input called `self` which refers to the flake itself.<br>
Returns an **attribute set** of outputs which can be thought of as build targets.

Here is a flake.nix that takes only the default input and specifies a single output of `foo`:
```nix
{
  outputs = { self }: {
    foo = bar;
  };
}
```
Now if we run some flake commands we can see how nix interacts with our flake:
```
$> nix eval .#foo
"bar"
 
$> nix flake show  
path:/Users/me/nix-test?lastModified=1705777957&narHash=sha256-Z8HSIXz08K28JttmmRdBfC/ve6I9XhzVGiGEG4xlr%2B4%3D
└───foo: unknown
```


### The `inputs` attribute set

Detailed schema: https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-flake.html#flake-inputs

The `inputs` attribute set is a mapping of names to other flakes.<br>
It exactly defines all the context available to the output function.<br>
The output function will have nothing else available to it other than what is specified in `inputs`.<br>

Inputs can be sepcified in a few different ways: (adapted from https://nixos-and-flakes.thiscute.world/other-usage-of-flakes/inputs)
```nix
{
  inputs = {
    # GitHub, defaults to main branch
    nixpkgs.url = "github:Mic92/nixpkgs";
    # GitHub, specifying a branch
    nixpkgs.url = "github:Mic92/nixpkgs/main";
    # Git URL, applicable to any Git repository using the https/ssh protocol.
    git-example.url = "git+https://git.somehost.tld/user/path?ref=branch";
    # Similar to fetching a Git repository, but using the ssh protocol 
    # with key authentication. Also uses the shallow=1 parameter 
    # to avoid copying the .git directory.
    ssh-git-example.url = "git+ssh://git@github.com/ryan4yin/nix-secrets.git?shallow=1";
    # It's also possible to directly depend on a local Git repository.
    git-directory-example.url = "git+file:/path/to/repo?shallow=1";
    # Using the `dir` parameter to specify a subdirectory.
    nixpkgs.url = "github:foo/bar?dir=shu";
    # Local folder (if using an absolute path, the 'path:' prefix can be omitted).
    directory-example.url = "path:/path/to/repo";

    # If the data source is not a flake, set flake=false.
    # `flake=false` is usually used to include additional source code,
    #   configuration files, etc.
    # In Nix code, you can directly reference files within
    #   it using "${inputs.bar}/xxx/xxx" notation.
    # For example, import "${inputs.bar}/xxx/xxx.nix" to import a specific nix file,
    # or use "${inputs.bar}/xx/xx" as a path parameter for certain options.
    bar = {
      url = "github:foo/bar/branch";
      flake = false;
    };

    sops-nix = {
      url = "github:Mic92/sops-nix";
      # `follows` is the inheritance syntax within inputs.
      # Here, it ensures that sops-nix's `inputs.nixpkgs` aligns with 
      # the current flake's inputs.nixpkgs,
      # avoiding inconsistencies in the dependency's nixpkgs version.
      inputs.nixpkgs.follows = "nixpkgs";
    };

    # Lock the flake to a specific commit.
    nix-doom-emacs = {
      url = "github:vlaci/nix-doom-emacs?rev=238b18d7b2c8239f676358634bfb32693d3706f3";
      flake = false;
    };
  };

  outputs = { self, ... }@inputs: { ... };
}
```


### The `outputs` attribute set

Detailed schema: https://nixos.wiki/wiki/Flakes#Output_schema

`outputs` can produce an attribute set with arbitrary keys.<br>
There are some standard keys that are understood by various nix tools.

Here are many (all?) of the standard output attribute keys understood by nix tools:
* `<system>` is something like `x86_64-linux`, `aarch64-linux`, `i686-linux`, `x86_64-darwin`
* `<name>` is an attribute name like `hello`.
* `<flake>` is a flake name like `nixpkgs`.
* `<store-path>` is a /nix/store.. path.

```nix
{ self, ... }@inputs:
{
  # Executed by `nix flake check`
  checks."<system>"."<name>" = derivation;
  # Executed by `nix build .#<name>`
  packages."<system>"."<name>" = derivation;
  # Executed by `nix build .`
  packages."<system>".default = derivation;
  # Executed by `nix run .#<name>`
  apps."<system>"."<name>" = {
    type = "app";
    program = "<store-path>";
  };
  # Executed by `nix run . -- <args?>`
  apps."<system>".default = { type = "app"; program = "..."; };

  # Formatter (alejandra, nixfmt or nixpkgs-fmt)
  formatter."<system>" = derivation;
  # Used for nixpkgs packages, also accessible via `nix build .#<name>`
  legacyPackages."<system>"."<name>" = derivation;
  # Overlay, consumed by other flakes
  overlays."<name>" = final: prev: { };
  # Default overlay
  overlays.default = final: prev: { };
  # Nixos module, consumed by other flakes
  nixosModules."<name>" = { config }: { options = {}; config = {}; };
  # Default module
  nixosModules.default = { config }: { options = {}; config = {}; };
  # Used with `nixos-rebuild switch --flake .#<hostname>`
  # nixosConfigurations."<hostname>".config.system.build.toplevel must be a derivation
  nixosConfigurations."<hostname>" = {};
  # Used by `nix develop .#<name>`
  devShells."<system>"."<name>" = derivation;
  # Used by `nix develop`
  devShells."<system>".default = derivation;
  # Hydra build jobs
  hydraJobs."<attr>"."<system>" = derivation;
  # Used by `nix flake init -t <flake>#<name>`
  templates."<name>" = {
    path = "<store-path>";
    description = "template description goes here?";
  };
  # Used by `nix flake init -t <flake>`
  templates.default = { path = "<store-path>"; description = ""; };
}
```

### The `packages` key of the `outputs` attribute set

The `packages` key is where nix expects you to put instructions for building packages.<br>
Furthermore it expects you to specify the platform as an attribute set containing keys for specific platforms.
* ie: `packages.x86_64-linux = ...`

For example we can create a flake that simply re-builds `hello` for x64-linux and arm-mac like so:
```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs";
  };

  outputs = { self, nixpkgs }: {
    # For x64 linux
    packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;

    # For arm64 mac
    packages.aarch64-darwin.hello = nixpkgs.legacyPackages.aarch64-darwin.hello;
  };
}
```

And we can use nix to interact with it:
```
$> nix flake show --all-systems
path:/Users/me/nix-test?lastModified=1705780682&narHash=sha256-8JpfaktZHVlO6Hf5tyCjZKFY2NpeQiNnv9ae%2BzzoTTM%3D
└───packages
    ├───aarch64-darwin
    │   └───hello: package 'hello-2.12.1'
    └───x86_64-linux
        └───hello: package 'hello-2.12.1'

$> nix run .#hello  
Hello, world!
```

### The `devShell` key of the output attribute set

The `devShell` key is used with `nix develop` to produce shell environments for development of your package.<br>
Like the `packages` key you need to specify targets on the `devShell` attribute set.
* ie: `devShell.x86_64-linux = ...`
It is very common to use the [`mkShell`](http://ryantm.github.io/nixpkgs/builders/special/mkshell/) package to help you build a devShell package<br>.
`devShell` is a legacy package, and it is common to alias `nixpkgs.legacyPackages` to avoid some repetition.
* ie: `let pkgs = nixpkgs.legacyPackages`

Here is a `flake.nix` with a`devShell` for our above project that makes 2 tools available to us in a shell, `hello` and `cowsay`:
```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs";
  };

  outputs = { self, nixpkgs }:
    let
      # alias for nixpkgs.legacyPackages so we don't have to repeat it many times
      pkgs = nixpkgs.legacyPackages;
    in {
      # Package for x64 linux
      packages.x86_64-linux.hello = nixpkgs.legacyPackages.x86_64-linux.hello;

      # Package for arm64 mac
      packages.aarch64-darwin.hello = nixpkgs.legacyPackages.aarch64-darwin.hello;


      devShell = {
        # Dev shell for x64 linux
        x86_64-linux =
          pkgs.x86_64-linux.mkShell {
            # special key that lists packages that should be available in our dev shell
            buildInputs = [
              # the package specified by this `flake.nix`
              self.packages.x86_64-linux.hello
              # package from legacy nix packages
              pkgs.x86_64-linux.cowsay
            ]; 
          };

        # Dev shell for arm mac
        aarch64-darwin =
          pkgs.aarch64-darwin.mkShell {
            buildInputs = [
              self.packages.aarch64-darwin.hello
              pkgs.aarch64-darwin.cowsay
            ]; 
          };
          
      };
  };
}
```

We can use nix to interact with it like so:
```bash
$> nix flake show --all-systems
path:/Users/me/nix-test?lastModified=1705783469&narHash=sha256-eccJPL/qR27oUz5CAs3g5k%2B36K%2Bzgy72DrS2FBvJEpA%3D
├───devShell
│   ├───aarch64-darwin: development environment 'nix-shell'
│   └───x86_64-linux: development environment 'nix-shell'
└───packages
    ├───aarch64-darwin
    │   └───hello: package 'hello-2.12.1'
    └───x86_64-linux
        └───hello: package 'hello-2.12.1'

$> nix develop                 
FWM-20519:nix-test me$ hello | cowsay
 _______________ 
< Hello, world! >
 --------------- 
        \   ^__^
         \  (oo)\_______
            (__)\       )\/\
                ||----w |
                ||     ||
FWM-20519:nix-test me$ 
```

We can tell `nix develop` to use a specifc shell with the `-c` command line argument.<br>
For example to use your current shell pass in `$SHELL` like so: `nix shell -c $SHELL`<br>

### `flake-utils`

`flake-utils` is a commonly used library of pure Nix functions that are useful for writing flakes.

url: https://github.com/numtide/flake-utils

Our `flake.nix` above is rather verbose and has a lot of repetition where the only difference is the target architecture.<br>
We can abstract all of that away by including `flake-utils` as an input and using a function it provides:

```nix
{
  inputs = {
    nixpkgs.url = "github:nixos/nixpkgs";
    flake-utils.url = "github:numtide/flake-utils";
  };
  outputs = { self, nixpkgs, flake-utils }:
    # We call eachDefaultSystem to iterate over a list of common systems to
    # specify packages for.
    flake-utils.lib.eachDefaultSystem (system:
      let
        # We use interpolation of `${system}` to make `pkgs` represent the 
        # legacy nix packages for the specific target we're making packages for
        pkgs = nixpkgs.legacyPackages.${system};
      in {
        # Now in there's no more target specific code
        packages.hello = pkgs.hello;

        devShell = pkgs.mkShell {
          buildInputs = [
            self.packages.${system}.hello
            pkgs.cowsay
          ];
        };
    }
  );
}
```

And now we get even more supported platforms with less code:
```bash
$> nix flake show --all-systems
path:/Users/me/nix-test?lastModified=1705784732&narHash=sha256-jHGeSNBzH%2BmCLwrWwF3e4Y6eqMRmb3sYElbD7JqB5v0%3D
├───devShell
│   ├───aarch64-darwin: development environment 'nix-shell'
│   ├───aarch64-linux: development environment 'nix-shell'
│   ├───x86_64-darwin: development environment 'nix-shell'
│   └───x86_64-linux: development environment 'nix-shell'
└───packages
    ├───aarch64-darwin
    │   └───hello: package 'hello-2.12.1'
    ├───aarch64-linux
    │   └───hello: package 'hello-2.12.1'
    ├───x86_64-darwin
    │   └───hello: package 'hello-2.12.1'
    └───x86_64-linux
        └───hello: package 'hello-2.12.1'
```

# Formatting

TODO

`nix fmt --help`

https://nixos.org/manual/nix/stable/command-ref/new-cli/nix3-fmt.html




