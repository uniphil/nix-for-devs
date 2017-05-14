# Nix for devs

This is collection of recipes focused on [`nix-shell`](ttp://nixos.org/nixpkgs/manual/#how-to-create-ad-hoc-environments-for-nix-shell) to make setting up project environments easy.
Pragmatism and low project impact are prioritized:
it's not a goal to change a nodejs project to use nix instead of npm,
but it is a goal to run npm projects without having to install any OS dependencies (including nodejs itself!).
`nix-ops` and other super-cool nix tech are completely out of scope.

Every solution should Work Today (TM).
If something is broken, [open an issue!](https://github.com/uniphil/nix-for-devs/issues/new).
If there is a better solution that will be supported by nix soon, we wait until it's released before mentioning it here (but don't hesitate to open a tracking issue!).

Maybe in the best case, this document can be a kind of gateway to nix and nixos.
I keep telling my developer friends about the cool things I do with nix, but I have to add huge warnings about how much time it's taken me to get to my current level of productivity.
I'm annoyingly aware of the amount of trivia I'm carying in my head (though I still feel
like a beginner!), and I constantly go back to old projects to look up what I did last time to fix problems.
I frequently find myself running into hard-to-google issues.
So, I'm finally writing it all down, for myself as much as for my friends :)


## Work-in-progress

Current state: rough sketches and todos. Pull requests welcome!


## `nix-shell`

TODO


### Run stuff without installing anything

Stuff I used to do with docker. TODO :)


## Node.js

`shell.nix`

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
    name = "node";
    buildInputs = [
        nodejs
    ];
    shellHook = ''
        export PATH="$PWD/node_modules/.bin/:$PATH"
        npm install
    '';
}
```

The `nodejs` build input sets up `node` and `npm` in the shell.
The shell hook adds the `.bin/` folder to your `PATH`, so you can install node command-line tools like `grunt` with `npm` without the `-g` flag.
Finally, it automatically installs any dependencies found in `package.json`, if one exists.
If you prefer to run `npm install` manually inside the shell, just delete that line from `shellHook`.


### React native

TODO


## Python

`shell.nix`

```nix
with import <nixpkgs> {};
with pkgs.python27Packages;

stdenv.mkDerivation {
  name = "python";

  buildInputs = [
    pip
    python27Full
    virtualenv
  ];

  shellHook = ''
    SOURCE_DATE_EPOCH=$(date +%s)  # so that we can use python wheels
    YELLOW='\033[1;33m'
    NC="$(printf '\033[0m')"

    echo -e "''${YELLOW}Creating python environment...''${NC}"
    virtualenv --no-setuptools venv > /dev/null
    export PATH=$PWD/venv/bin:$PATH > /dev/null
    pip install -r requirements.txt > /dev/null
  '';
}
```

This expression sets up a python 2 environment, and installs any dependencies from `requirements.txt` with `pip` in a `virtualenv`.

### OS library dependencies

There is a `LD_LIBRARY_PATH` attribute for nix packages that can help python dependencies find the libraries they may need to complete installation.
You generally need to format two pieces of information together: the path to the dependency in the nix store, and `/lib`.

Usually this means you just need `LD_LIBRARY_PATH=${<NAME>}/lib`.
However, sometimes nix packages divide up their outputs, with the `/lib` folder and its contents at their own path in the nix store. _Usually_, the output with `lib//` is called `out`, and can be used like `LD_LIBRARY_PATH=${<NAME>.out}/lib`.

If you need to put more than on dependency into `LD_LIBRARY_PATH`, separate them with a colon `:`, like for `$PATH`.

#### `geos` and `gdal`

`shell.nix`

```nix
with import <nixpkgs> {};
with pkgs.python27Packages;

stdenv.mkDerivation {
  name = "python";

  buildInputs = [
    gdal
    geos
    pip
    python27Full
    virtualenv
  ];
  
  LD_LIBRARY_PATH="${geos}/lib:${gdal}/lib";

  shellHook = ''
    SOURCE_DATE_EPOCH=$(date +%s)  # so that we can use python wheels
    YELLOW='\033[1;33m'
    NC="$(printf '\033[0m')"

    echo -e "''${YELLOW}Creating python environment...''${NC}"
    virtualenv --no-setuptools venv > /dev/null
    export PATH=$PWD/venv/bin:$PATH > /dev/null
    pip install -r requirements.txt > /dev/null
  '';
}
```


#### zlib

TODO


#### sqlite3

TODO

---

also, http://nixos.org/nixpkgs/manual/#using-python


## Rust

TODO: mozilla nix overlay for nightly

`shell.nix`

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
    name = "rust";
    buildInputs = [
        rustChannels.nightly.cargo
        rustChannels.nightly.rust
    ];
}
```

### OpenSSL

`shell.nix`

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
    name = "rust";
    buildInputs = [
        openssl
        rustChannels.nightly.cargo
        rustChannels.nightly.rust
    ];
    shellHook = ''
        export OPENSSL_DIR="${openssl.dev}"
        export OPENSSL_LIB_DIR="${openssl.out}/lib"
    '';
}
```


### diesel

`shell.nix`

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
    name = "rust";
    buildInputs = [
        postgresql
        rustChannels.nightly.cargo
        rustChannels.nightly.rust
    ];
    shellHook = ''
        export OPENSSL_DIR="${openssl.dev}"
        export OPENSSL_LIB_DIR="${openssl.out}/lib"
        export PATH="$PWD/bin:$PATH"
        export DATABASE_URL="postgres://postgres@localhost/db"
        if ! type diesel > /dev/null 2> /dev/null; then
          cargo install diesel_cli --no-default-features --features postgres --root $PWD
        fi
        diesel setup
    '';
}
```

TODO: talk about running postgres in a nix-shell, and don't hard-code `DATABASE_URL` in such an ugly way

`cargo.toml`

```toml
[dependencies.diesel]
version = "0.11"
features = ["postgres"]

[dependencies.diesel_codegen]
version = "0.11"
features = ["postgres"]
```

`.gitignore`

```
bin/
```
