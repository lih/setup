Version 1.0
===========

- introduced the `prepare`, `setup` and `teardown` functions
- introduced the other core functions : `SETUP_JOBS`, `Setup.use`, `Setup.load`, `Setup.hook` and `Setup.state-file`
- introduced the basic Setup script utilities : `Setup.params` and `prepare-match`

Version 1.1
===========

- adjusted `prepare-match` to automatically anchor the regex. A
  matcher that used to look like `prepare-match '^xyz$' = ...` will
  have to be changed to `prepare-match 'xyz' = ...`
