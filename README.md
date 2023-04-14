# Py.test plugin for validating Jupyter notebooks

[![Tests](https://github.com/computationalmodelling/nbval/actions/workflows/tests.yml/badge.svg)](https://github.com/computationalmodelling/nbval/actions/workflows/tests.yml)
[![PyPI Version](https://badge.fury.io/py/nbval.svg)](https://pypi.python.org/pypi/nbval)
[![Documentation Status](https://readthedocs.org/projects/nbval/badge/)](https://nbval.readthedocs.io/)

The plugin adds functionality to py.test to recognise and collect Jupyter
notebooks. The intended purpose of the tests is to determine whether execution
of the stored inputs match the stored outputs of the `.ipynb` file. Whilst also
ensuring that the notebooks are running without errors.

The tests were designed to ensure that Jupyter notebooks (especially those for
reference and documentation), are executing consistently.

Each cell is taken as a test, a cell that doesn't reproduce the expected
output will fail.

See [`docs/source/index.ipynb`](http://nbviewer.jupyter.org/github/computationalmodelling/nbval/blob/HEAD/docs/source/index.ipynb) for the full documentation.

## This Fork ([`ouseful-PR/nbval@table-test`](https://github.com/ouseful-PR/nbval/tree/table-test))

This fork currently recognises the following additional tags to the tags provided by `nbval` (`nbval-skip`, `nbval-ignore-output`, `nbval-raises-exception`)):

- `nbval-variable-output`: some cells return randomised or changeable output that cannot be easily sanitised using a regular eexpression. The output of cells tagged with `nbval-variable-output` are ignored as per `nbval-ignore-output`;
- `folium-map`: specify that the cell is a folium map output. The cell output type is then checked to see whether it is a folium map object type;
- `nbval-test-linecount`: where cells contain printed output that changes in content but not structure (eg the same number of lines are printed on each run), the `nbval-test-linecount` will check that the same number of lines are printed by a cell in the test notebook as in the reference notebook;
- `nbval-test-df` tag: attempt to cast a cell outut to a *pandas* dataframe, then check that the test dataframe has a similar structure to the reference dataframe, even if the content differs. Structural tests currently include: shape test (same number of rows and columns; common column names test);
- `nbval-test-listlen` tag: attempt to cast a cell output to a list, then compare the length of the lists;
- `nbval-list-membership` tag: attempt to cast cell output to a list and then see whether the list elements are the same, irrespective of order *(this currently fails to handle nested lists?)*;
- `nbval-set-membership` tag: attempt to cast cell output to a set, then compare membership;
- `nbval-test-dictkeys` tag: attempt to cast cell output to a dict, then perform strutural equivalence tests *(currently limited to check that top-level dictionary keys are equivalent)();
- `nbval-figure` tag: `matplotlib` returns text labels as last line output so we need to defend against that. A figure tag is useful... [See `nb_workflow_tools` for a tool to autotag pre-run notebook cells with an `nbval-figure` tag, if appropriate.]
- `nbval-test-series` tag: test for series, compare series length

These tags provide a way of weakening test equivalence in a way that still allows useful tests to be performed. This is particularly useful where instructional notebooks are being tested that do not return strictly reproducible cell outputs, but where the shape or type of the returned elements from a particular cell should be consistent.

These tests are being developed as part of an ongoing process to support the use and deployment of notebook based teaching materials over several years of course presentation in a distance education context. The notebooks themselves are intended to remain largely unchanged over time, but the Docker container based environment is updated on a per course presentation basis. Previously run notebooks are automatically tested in updated Docker containers to ensure that outputs are consistent with previous runs, if not stricly identical to them. Warnings and errors arising from updates will also be captured.

A regular expression sanitiser can also be used to reduce the number of cell run failures by rewriting conventionally variable content.

### Useful Sanitiser Regular Expressions

*The first line is IPython magic that would write the remaining contents to the specified file. Omit that first line if you are copying and pasting the regexes for your own sanitiser file.*

```
%%writefile ouseful-sanitiser.cfg
[regex1]
regex: Figure size \d.*x\d.*
replace: FIGURE-SIZE
[regex2]
regex: .* per loop (mean ± std. dev. of \d+ runs, \d+ loops? each)
replace: TIMING-REPORT
[regex3]
regex: peak memory: .* MiB, increment: .* MiB
replace: MEMORY-REPORT
[regex4]
regex: File size \(.*\): .*B
replace: FILE_SIZE
[regex5]
regex: <pymongo.results.InsertOneResult at.*>
replace: <pymongo.results.InsertOneResult>
[regex6]
regex: ObjectId\('.*'\)
replace: ObjectId(...)
[regex7]
regex: <pymongo.results.UpdateResult at .*>
replace: <pymongo.results.UpdateResult>
[regex8]
regex: <pymongo.results.InsertManyResult at .*>
replace: <pymongo.results.InsertManyResult>
[regex9]
regex: 0x[0-9a-f]*
replace: 0xHASH
[regex10]
regex: <Graph identifier=.* \(<class 'rdflib.graph.Graph'>\)>
replace: <Graph identifier=IDENTIFER (<class 'rdflib.graph.Graph'>)>
```

We can use the sanitiser file with a command of the form `py.test --nbval */*.ipynb --sanitize-with ouseful-sanitiser.cfg`

## Installation

__Install this branch:__

`pip install --upgrade git+https://github.com/ouseful-PR/nbval.git@table-test`

Original / official package available on PyPi:

    pip install nbval

or install the latest version from cloning the repository and running:

    pip install .

from the main directory. To uninstall:

    pip uninstall nbval

For HTML reports, also install:

    pip install pytest-html

## How it works

The extension looks through every cell that contains code in an IPython notebook
and then the `py.test` system compares the outputs stored in the notebook
with the outputs of the cells when they are executed. Thus, the notebook itself is
used as a testing function.
The output lines when executing the notebook can be sanitized passing an
extra option and file, when calling the `py.test` command. This file
is a usual configuration file for the `ConfigParser` library.

Regarding the execution, roughly, the script initiates an
IPython Kernel with a `shell` and
an `iopub` sockets. The `shell` is needed to execute the cells in
the notebook (it sends requests to the Kernel) and the `iopub` provides
an interface to get the messages from the outputs. The contents
of the messages obtained from the Kernel are organised in dictionaries
with different information, such as time stamps of executions,
cell data types, cell types, the status of the Kernel, username, etc.

In general, the functionality of the IPython notebook system is
quite complex, but a detailed explanation of the messages
and how the system works, can be found here

https://jupyter-client.readthedocs.io/en/latest/messaging.html#messaging

## Execution

To execute this plugin, you need to execute `py.test` with the `nbval` flag
to differentiate the testing from the usual python files:

    py.test --nbval

To generate an HTML report (requires `pytest-html`), add the following to the command:

 --html=report.html --self-contained-html

You can also specify `--nbval-lax`, which runs notebooks and checks for
errors, but only compares the outputs of cells with a `#NBVAL_CHECK_OUTPUT`
marker comment.

    py.test --nbval-lax

The commands above will execute all the `.ipynb` files and 'pytest' tests in the current folder.
Specify `-p no:python` if you would like to execute notebooks only. Alternatively, you can execute a specific notebook:

    py.test --nbval my_notebook.ipynb
    
To check numbered notebooks:

    py.test --nbval Part\ 01*/[[:digit:]]*.ipynb

By default, each `.ipynb` file will be executed using the kernel
specified in its metadata. You can override this behavior by passing
either `--nbval-kernel-name mykernel` to run all the notebooks using
`mykernel`, or `--current-env` to use a kernel in the same environment
in which pytest itself was launched.

If the output lines are going to be sanitized, an extra flag, `--nbval-sanitize-with`
together with the path to a confguration file with regex expressions, must be passed,
i.e.

    py.test --nbval my_notebook.ipynb --nbval-sanitize-with path/to/my_sanitize_file

where `my_sanitize_file` has the following structure.

```
[Section1]
regex: [a-z]*
replace: abcd

regex: [1-9]*
replace: 0000

[Section2]
regex: foo
replace: bar
```

The `regex` option contains the expression that is going to be matched in the outputs, and
`replace` is the string that will replace the `regex` match. Currently, the section
names do not have any meaning or influence in the testing system, it will take
all the sections and replace the corresponding options.

### Coverage

To use notebooks to generate coverage for imported code, use the pytest-cov plugin.
nbval should automatically detect the relevant options and configure itself with it.

### Parallel execution

nbval is compatible with the pytest-xdist plugin for parallel running of tests. It does
however require the use of the `--dist loadscope` flag to ensure that all cells of one
notebook are run on the same kernel.

## Documentation

The narrative documentation for nbval can be found at https://nbval.readthedocs.io.
## Help

The `py.test` system help can be obtained with `py.test -h`, which will
show all the flags that can be passed to the command, such as the
verbose `-v` option. Nbval's options can be found under the
`Jupyter Notebook validation` section.

## Acknowledgements
This plugin was inspired by Andrea Zonca's py.test plugin for collecting unit
tests in the IPython notebooks (https://github.com/zonca/pytest-ipynb).

The original prototype was based on the template in
https://gist.github.com/timo/2621679 and the code of a testing system
for notebooks https://gist.github.com/minrk/2620735 which we
integrated and mixed with the `py.test` system.

We acknowledge financial support from

- OpenDreamKit Horizon 2020 European Research Infrastructures project (#676541), http://opendreamkit.org

- EPSRC's Centre for Doctoral Training in Next Generation
  Computational Modelling, http://ngcm.soton.ac.uk (#EP/L015382/1) and
  EPSRC's Doctoral Training Centre in Complex System Simulation
  ((EP/G03690X/1),

- The Gordon and Betty Moore Foundation through Grant GBMF #4856, by the
  Alfred P. Sloan Foundation and by the Helmsley Trust.

## Authors

2014 - 2017 David Cortes-Ortuno, Oliver Laslett, T. Kluyver, Vidar
Fauske, Maximilian Albert, MinRK, Ondrej Hovorka, Hans Fangohr
