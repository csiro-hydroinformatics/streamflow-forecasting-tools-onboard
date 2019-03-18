# Roadmap for ensemble forecasting tools

## Software architecture

As of Jan 2019 the python package pyswift uses CFFI for interop. `uchronia` started down the same path but while trying to get things at parity with the R packages, we looked at `pybind11` which is a serious candidate for our python/c++/c interop needs. Even more so since we are looking at `xtensor` for multidimensional data handling and [xtensor-python](https://github.com/QuantStack/xtensor-python) also uses pybind11. pybind11 could play a role very similar to Rcpp and indeed is a close conceptual equivalent.

* q1 2019 Set up a Testbed with a uchronia_p11 package
