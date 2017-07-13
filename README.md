# Nix for devs

This is collection of recipes focused on [`nix-shell`](ttp://nixos.org/nixpkgs/manual/#how-to-create-ad-hoc-environments-for-nix-shell) to make setting up project environments easy.
Pragmatism and low project impact are prioritized:
it's not a goal to change a nodejs project to use nix instead of npm,
but it is a goal to run npm projects without having to install any OS dependencies (including nodejs itself!).
`nix-ops` and other super-cool nix tech are completely out of scope.

Every solution should work today.
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
    '';
}
```

The `nodejs` build input sets up `node` and `npm` in the shell.
The shell hook adds the `.bin/` folder to your `PATH`, so you can install node command-line tools like `grunt` with `npm` without the `-g` flag.
Finally, it automatically installs any dependencies found in `package.json`, if one exists.
If you prefer to run `npm install` manually inside the shell, just delete that line from `shellHook`.

### `npm run <...>` shortcut

It's handy to have an alias `run` for `npm run`, which makes the following two commands equivalent:

```bash
$ npm run watch:test
$ run watch:test
```

You can add this to `shellHook`:

```bash
    alias run='npm run'
```

### Newer nodejs

`shell.nix`

```nix
with import <nixpkgs> {};

stdenv.mkDerivation {
    name = "node";
    buildInputs = [
        nodejs-8_x
    ];
    shellHook = ''
        export PATH="$PWD/node_modules/.bin/:$PATH"
    '';
}
```

Node versions are published as -<version>_x, so etg., `nodejs-7_x` and `nodejs-6_x` are also valid.

### React native

`shell.nix`

```nix
with import <nixpkgs> {};

let
  jdk = openjdk;
  node = nodejs-8_x;
  sdk = androidenv.androidsdk {
    platformVersions = [ "23" ];
    abiVersions = [ "x86" ];
    useGoogleAPIs = true;
    useExtraSupportLibs = false;
    useGooglePlayServices = false;
  };
  unpatched-sdk =
    let version = "3859397";
    in stdenv.mkDerivation {
      name = "unpatched-sdk";
      src = fetchzip {
        url = "https://dl.google.com/android/repository/sdk-tools-linux-${version}.zip";
        sha256 = "03vh2w1i7sjgsh91bpw40ryhwmz46rv8b9mp7xzg89zs18021plr";
      };
      installPhase = ''
        mkdir -p $out
        cp -r * $out/
      '';
      dontPatchELF = true;
    };
  run-android = pkgs.buildFHSUserEnv {
    name = "run-android";
    targetPkgs = (pkgs: [
      node
    ]);
    profile = ''
      export JAVA_HOME=${jdk.home}
      export ANDROID_HOME=$PWD/.android
      export PATH=$PWD/node_modules/.bin:$PATH
    '';
    runScript = "react-native run-android";
  };
in
  stdenv.mkDerivation {
    name = "react-native-android";
    nativeBuildInputs = [
      run-android
    ];
    buildInputs = [
      coreutils
      node
      sdk
      unpatched-sdk
    ];
    shellHook = ''
      export JAVA_HOME=${jdk}
      export ANDROID_HOME=$PWD/.android/sdk
      export PATH="$ANDROID_HOME/bin:$PWD/node_modules/.bin:$PATH"

      if ! test -d .android ; then
        echo doing hacky setup stuff:

        echo "=> pull the sdk out of the nix store and into a writeable directory"
        mkdir -p .android/sdk
        cp -r ${unpatched-sdk}/* .android/sdk/

        echo "=> don't track the sdk directory"
        echo .android/ >> .gitignore

        echo "=> get react-native-cli in here"
        npm install --no-save react-native-cli

        echo "=> set up react-native plugins... need an FHS env for... reasons."
        cd .android/sdk
          $PWD/bin/sdkmanager --update
          echo "=> installing platform stuff (you'll need to accept a license in a second)..."
          $PWD/bin/sdkmanager "platforms;android-23" "build-tools;23.0.1" "add-ons;addon-google_apis-google-23"
        cd ../../
      fi;
    '';
  }
```

This one is super-hacky and requires usage instructions :(
also probably only works on linux :/

#### 1. Enter the environment

```bash
$ nix-shell
[...lots of console spam the first time]
[nix-shell:]$
```

#### 2. If you don't have an avd set up, make one

```bash
[nix-shell:]$ android create avd -t android-23 -b x86 -d "Nexus 5" -n nexus
```

That command seeme to create slightly screwy avds. I ran `$ android avd` and then hit `edit` and `save` without any changes on mine, which seems to fix it :/

#### 3. Start the JS server and emulator

Both commands block: either background then (add `&` at the end) to run in the same terminal, or open a terminal for each

```bash
[nix-shell:]$ npm start
[nix-shell:]$ emulator -avd nexus
```


#### 4. Run it!

```bash
[nix-shell:]$ run-android
```

Note that this `run-android` is provided by [`shell.nix`](./shell.nix), and wraps the call to `react-native-cli`'s `react-native run-android` command in the FHS environment so that the build works.


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
