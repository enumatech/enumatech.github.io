# Introduction

Static website of Enuma Technologies.
 
The website is managed by [Jekyll](https://jekyllrb.com).

# Building

A [`shell.nix`](https://nixos.wiki/wiki/Development_environment_with_nix-shell)
is provided to automatically setup the build environment. Refer to the
[Nix website](https://nixos.org/nix/) for installation instruction to
install Nix, a purely functional package manager.

Once in a nix-shell, run

```
jekyll build
``` 

to generate the site in the `_site` folder.
