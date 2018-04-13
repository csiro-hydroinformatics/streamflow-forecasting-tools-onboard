Installing ensemble streamflow forecasting tools on Windows
===========================================================

# Installing shared (dynamic) libraries

This documentation is fairly prescriptive in terms of the location where you would install dynamic libraries. You can still choose the top level folder where you would install these libraries. This document uses "c:\local".

You may have `7z` in your command line prompt, otherwise use Windows Explorer:

```bat
mkdir c:\local
cd c:\local
7z x libs.7z
```

This should have created the folder `C:\local\libs\64`. A 32 bits folder may be present but is deprecated as of 2017-12: operating systems are now largely 64 bits and we recommend you use 64 bits toolsets.

## Adding environment variable LIBRARY_PATH

The packages in R, python, (etc.) need a way to search for these native libraries. To do so, you should set up a User- or System-level environmental variable `LIBRARY_PATH`. The approach borrows from what is customary (albeit variable) on most Linux system for dynamic library resolution.

Go to the Control Panel, System and Security\System, Search for the string "environment", and it should give you the link to edit environment variables for your account.

* for the field "Variable name:" use `LIBRARY_PATH`
* for the field "Variable value:" use c:\local\libs  (if you unzipped libs.7z into c:\local in the previous section)

# Installing packages

## R packages

At the time of writing, instructions should work for R versions 3.3 and 3.4. It should be possible to install in ulterior versions but a bit of a workaround may be required.

### Adding an R_LIBS environment variable

This step is recommended but not compulsory.

While not required, we advise you set up an additional R library location, specified via an environment variable `R_LIBS` at the machine or user level. This can facilitate access to R packages for all users and upgrades to newer version of R down the track. you can install the packages in the other library folders that R creates (user-specific if you do not have admin rights)

### Installing an R package

You should have been given pre-compiled R packages. If you have for instance a USB drive mapped to E:, the following command should install swift, and all its depencency packages. This may take some time to set up once, but is worth it - most of these packages are publicly, popular R packages providing powerful statistical and visualisation tools to complement SWIFT2.

`swift` is the main modelling package, `uchronia` is a closely related package for multi-dimensional time series handling. 

```R
install.packages(c('uchronia','swift'), repos=c('file:///E:/software/R_pkgs', 'https://cran.csiro.au'), type='win.binary')
```

Other related packages but less mature or less often used are available:

```R
install.packages(c('rpp','calibragem'), repos=c('file:///E:/software/R_pkgs', 'https://cran.csiro.au'), type='win.binary')
```

## Python packages

This section is a placeholder

## Matlab functions

This section is a placeholder

