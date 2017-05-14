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

The `node` build input sets up `node` and `npm` in the shell. The shell hook adds the `.bin/` folder to your `PATH`, so you can install node tools like `grunt` with `npm` without the `-g` flag. Finally, it automatically installs any dependencies found in `package.json`, if one exists. If you prefer to run `npm install` manually inside the shell, just delete that line from `shellHook`.


### React native

TODO


## Python

TODO

also, http://nixos.org/nixpkgs/manual/#using-python


## Rust

TODO
