Version 1.0.0.1
==============

- implemented cycle detection in the dependency graph to warn
  users of incomplete builds
- adjusted `prepare-match` to automatically anchor the regex. A
  matcher that used to look like `prepare-match '^xyz$' = ...` will
  have to be changed to `prepare-match 'xyz' = ...`

Version 1.0
===========

- introduced the basic Setup script utilities : `Setup.params` and
  `prepare-match`
- introduced the other core functions : `SETUP_JOBS`, `Setup.use`,
  `Setup.load`, `Setup.hook` and `Setup.state-file`
- introduced the `prepare`, `setup` and `teardown` functions
