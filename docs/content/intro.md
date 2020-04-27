# `hep_tables` - DataFrames over the wire to `servicex`

This site contains a few demos Jupyter notebooks showing some of the ideas behind the [hep_tables](https://github.com/gordonwatts/hep_tables) and [dataframe_expressions](https://github.com/gordonwatts/dataframe_expressions) packages. The two packages work together to allow easy columnar-like access to hierarchial data when the data processing backend is in an analysis facility and the user is using either Jupyter notebooks or python files.

While the code has fairly complete tests, it is certianly of prototype quality and coverage (for features you would need). The feature set has been driven by a a small number of tasks that were meant to see if this was possible.

- Basic plots should be easy to make
- Single object selection cuts using `numpy` like slicing
- Multiple object combinations

This site is organized into a chapter with the demo notebooks showing off this work, and then followed by some discussion.

## Acknowledgements

This was build in the context of the [IRIS-HEP](https://iris-hep.org) project.