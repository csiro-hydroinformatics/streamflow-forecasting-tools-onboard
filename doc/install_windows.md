# Installing ensemble streamflow forecasting tools on Windows

You will have been given one or more links to the location of C++ compiled libraries (windows DLLs), and R and/or Python packages.

* libs.7z
* R_pkgs.7z.7z
* python.7z

## Installing shared (dynamic) libraries

This documentation is fairly prescriptive in terms of the location where you would install dynamic libraries. You can still choose the top level folder where you would install these libraries. This document uses "c:\local", and you are encourage to adhere to this convention if you can logistically can.

You may have `7z` in your command line prompt, otherwise use Windows Explorer to extract the zip file from the 7Zip graphical user interface :

```bat
mkdir c:\local
cd c:\local
7z x libs.7z
```

This should have created the folder `C:\local\libs\64`. A 32 bits folder may be present but is deprecated since 2017-12: operating systems are now largely 64 bits and we strongly recommend you use 64 bits toolsets.

### Adding environment variable LIBRARY_PATH

The packages in R, python, (etc.) need a way to search for these native libraries. To do so, you should set up a User- or System-level environmental variable `LIBRARY_PATH`. The approach borrows from what is customary (albeit variable) on most Linux system for dynamic library resolution.

Go to the Control Panel, System and Security\System, Search for the string "environment", and it should give you the link to edit environment variables for your account.

* for the field "Variable name:" use `LIBRARY_PATH`
* for the field "Variable value:" use c:\local\libs  (if you unzipped libs.7z into c:\local in the previous section)

## Installing packages

You can choose to install various higher level languages: R, Python, Matlab, C# and potentially others.

This section documents R and Python. You can install either, or both, depending on your needs.

### R packages

At the time of writing, instructions should work for R versions 3.3 and 3.4. It should be possible to install in ulterior versions but a bit of a workaround may be required.

#### Adding an R_LIBS environment variable

This step is recommended but not compulsory.

While not required, we advise you set up an additional R library location, specified via an environment variable `R_LIBS` at the machine or user level. This can facilitate access to R packages for all users and upgrades to newer version of R down the track. you can install the packages in the other library folders that R creates (user-specific if you do not have admin rights)

#### Installing an R package

You should have been given pre-compiled R packages. If you have for instance a USB drive mapped to E:, the following command should install swift, and all its depencency packages. This may take some time to set up once, but is worth it - most of these packages are publicly, popular R packages providing powerful statistical and visualisation tools to complement SWIFT2.

`swift` is the main modelling package, `uchronia` is a closely related package for multi-dimensional time series handling. 

```R
install.packages(c('uchronia','swift'), repos=c('file:///E:/software/R_pkgs', 'https://cran.csiro.au'), type='win.binary')
```

Other related packages but less mature or less often used are available:

```R
install.packages(c('rpp','calibragem'), repos=c('file:///E:/software/R_pkgs', 'https://cran.csiro.au'), type='win.binary')
```

### Python packages

#### __Python environments__

We **strongly** advise you to use python virtual environments to execute these tools via python.

There are various methods for these, but typically `venv` (virtual environments) or `conda`

This section documents the process using `conda`.

We recommend you use [miniconda3](https://docs.conda.io/en/latest/miniconda.html), but you may have already it installed on your Windows box by e.g. your IT department installation management software.

This document assumes you start from the base environment, for instance:

```text
(base) C:\Users\xxxyyy>conda env list
# conda environments:
#
base                  *  C:\Users\xxxyyy\Miniconda3
```

We recommend you use the `mamba` package, a newer and faster drop-in replacement for `conda`. This is optional.

```bat
conda install -c conda-forge mamba
```

#### __Installing hydroforecast packages__

Below remember to replace `mamba` by `conda` if you have not installed mamba.

```bat
set env_name="hydrofc"
mamba create -n %env_name% -c conda-forge python=3.9 xarray cffi pandas numpy matplotlib ipykernel jsonpickle netcdf4 seaborn
REM note to self seaborn used in swift2 sample notebooks, so somewhat optional.
```

Register the new conda environment as a "kernel" for jupyter notebooks

```bat
conda activate %env_name%
python -m ipykernel install --user --name %env_name% --display-name "HFC"
```

From here on all commands are done from within this new conda environment

You may already have jupyter-lab installed in another conda environment. You may use it to run 'hydrofc' notebooks. If not, install jupyter-lab in this new environment with:

```bat
mamba install -c conda-forge jupyterlab
```

You may need to install some additional conda packages depending on the notebooks you are using.

We can now install "our" packages. we can install from 'wheels', or from source code (for developers)

Two dependencies `refcount` and `cinterop` are on pypi, but should also be in the zip archive as wheels as well; prefer the latter.

To install "wheels", which you may have gotten from a zip file:

```bat
cd  C:\tmp\sf
7z x python.7z

:: Adapt the following to the versions you have.
pip install --force-reinstall --no-deps refcount-0.9.3-py2.py3-none-any.whl
pip install --force-reinstall --no-deps cinterop-0.9.0-py2.py3-none-any.whl
pip install --force-reinstall --no-deps uchronia-2.3.7-py2.py3-none-any.whl
pip install --force-reinstall --no-deps swift2-2.3.7-py2.py3-none-any.whl
pip install --force-reinstall --no-deps fogss-0.3-py2.py3-none-any.whl
```

#### __Installing hydroforecast packages in dev mode__

To install instead in development mode, for some or all of the 4 packages:

```bat
set GITHUB_REPOS=c:\src
set CSIRO_BITBUCKET=c:\src
```

```bat
cd %GITHUB_REPOS%\pyrefcount
python setup.py develop
cd %GITHUB_REPOS%\rcpp-interop-commons\bindings\python\cinterop
python setup.py develop

cd %CSIRO_BITBUCKET%\datatypes\bindings\python\uchronia\
python setup.py develop
cd %CSIRO_BITBUCKET%\swift\bindings\python\swift2\
python setup.py develop
:: fogss...
```

Now to exploresample notebooks

`mamba install -c conda-forge seaborn`

```bat
cd %userprofile%\Documents
mkdir notebooks
xcopy %CSIRO_BITBUCKET%\swift\bindings\python\swift2\notebooks\* .\

jupyter-lab .
```

start with getting_started.ipynb

TODO: recommend using `nbstripout` and `jupytext`

### Matlab functions

This section is a placeholder
