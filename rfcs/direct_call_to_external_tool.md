# Direct call to external toolchain

The compiler shouldn't require a shell in order to work.

# Motivation for the change

I try to fix this https://github.com/esy/esy/issues/1344
I can reduce process in esy but half of this has to be fixed in ocaml/flexdll

# Technical details of the change

Replace Sys.command by Unix.create_process or Unix.open_process_args

# Drawbacks of the change and alternatives to the change

pros:
- You earn some time without launching useless shell
- You avoid the [quoting nightmare](https://github.com/ocaml/ocaml/pull/10727) on OS with exec*
- You handle better all path (I have not tested path with $,; and other niceties)
- You workaround cmd limitation of 8187

cons:
- The flags -pp/-ppx 'binary -args' will not work
  - User can still create a wrapper script
  - We can add --ppopt, -ppxopt like -ccopt
  - We can add -direct_pp,-direct_ppx so we don't break any projects. All good citizens that use dune will get a boost benefit freely.
- You loose usage of %COMSPEC% on windows (but is it still needed ?)
  - We can still read that variable and act in consequence

cons without workaround:
- Unix will not be optional anymore (or maybe at least the creation part)

# Unresolved questions
