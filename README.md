---
Title: Pass language ecosystem overlays as pkgs.config options
Author: jonringer
Discussions-To: https://github.com/NixOS/nixpkgs/issues/44426
Status: Draft
Type: Standards Track
Topic: Packaging
Created: 2024-9-17
---

# Summary

Extending language package sets (e.g. `python3Packages`) is notoriously difficult
as each ecosystem is created differently. This proposal attempts to provide a
series of `config.overlays.<scope>` options in which overlays can be passed to
the respective package set(s).

## Detailed Implementation

This is using python as an example, but applicable to almost all package sets.

Config support:
```
  # pkgs/top-level/config.nix

  overlayType = lib.mkOptionType {
    name = "overlay";
    description = "overlay";
    check = lib.isFunction;
    merge = lib.mergeOneOption;
  };

  ...

  options.overlays.python = mkOption {
    type = types.listOf overlayType;
    default = [];
    description = ''
      Overlays which will be applied to every python interpreter package set.
    '';
  };
```

Integration with python package sets, in this case it would replace the need
for the `pythonPackagesExtensions` you can define at the top-level:
```diff
--- a/pkgs/development/interpreters/python/passthrufun.nix
+++ b/pkgs/development/interpreters/python/passthrufun.nix
@@ -56,7 +56,7 @@
         pythonExtension
       ] ++ (optionalExtensions (!self.isPy3k) [
         python2Extension
-      ]) ++ pythonPackagesExtensions ++ [
+      ]) ++ config.overlays.python ++ [
         overrides
       ]);
```

## Example python use case

In a downstream overlay:
```nix
# flake.nix or default.nix
import poly-repo {
  ...
  config.overlays.python = [ (final: _: {
    myPythonPackage = final.callPackage ./package.nix { };
  )}];
}
```

Then building for any python interpreter would just be:
```bash
$ nix-build -A python311.pkgs.myPythonPackage
/nix/store/<hash>-python3.11-mypythonpackage-1.0
$ nix-build -A python312.pkgs.myPythonPackage
/nix/store/<hash>-python3.12-mypythonpackage-1.0
```

## Migration plan

- If a package set does expose a way to inject overlays (e.g. python + `pythonPackagesExtensions`), then deprecate old usage.
- `overlays` from the import statement should map to `overlays.pkgs`

## Future work

- Apply overlay options for every language ecosystem and package set.

## Relevant topics

- Recursive Overlays: https://github.com/NixOS/nixpkgs/pull/54266
- Common Override Interface: https://github.com/NixOS/rfcs/pull/67
- "Better extensible" attrsets: https://github.com/NixOS/nixpkgs/pull/51213

