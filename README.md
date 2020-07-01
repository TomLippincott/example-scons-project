# Using SCons to organize experiments

Run the following to clone this repository and get a basic environment set up:

```bash
git clone https://github.com/TomLippincott/example-scons-project
cd example-scons-project
python3 -m venv venv
source venv/bin/activate
pip install scons steamroller
```

I use these layout conventions when starting a new project:

* `data/` for raw material drawn from outside the project: primary sources, dictionaries, pretrained embeddings, etc
* `src/` for code, mostly scripts corresponding to one step in the experimental pipeline
* `work/` is where everything is built by SCons
* `SConstruct` is pure Python, and describes the entire experimental structure
* `custom.py` is a site-specific file for setting/overriding configuration variables (I don't keep it under version control, but instead check in a `custom.py.template` file that's copied and modified for e.g. a particular grid/laptop/etc)

Set this structure up with the following commands:

```bash
mkdir data src work
touch SConstruct custom.py
```

In general, after making changes to any of the files outside of the `work/` directory, running:

```bash
scons -Q
```

builds all experiment artifacts for which dependencies have changed.  Since the `SConstruct` file is blank, there is nothing to build.  I usually start a new project with an `SConstruct` file like this (in fact, `emacs` prepopulates it for me):

```python
import sys
import os
import steamroller

# variable and environment objects
variables = Variables("custom.py")
variables.AddVariables(
    ("OUTPUT_WIDTH", "How wide a screen for debug output?", 80),
)

env = Environment(variables=variables, ENV=os.environ, TARFLAGS="-c -z", TARSUFFIX=".tgz",
                  tools=["default", steamroller.generate],
)

# function for width-aware printing of commands
def print_cmd_line(s, target, source, env):
    if len(s) > int(env["OUTPUT_WIDTH"]):
        print(s[:int(float(env["OUTPUT_WIDTH"]) / 2) - 2] + "..." + s[-int(float(env["OUTPUT_WIDTH"]) / 2) + 1:])
    else:
        print(s)

# and the command-printing function
env['PRINT_CMD_LINE_FUNC'] = print_cmd_line

# and how we decide if a dependency is out of date
env.Decider("timestamp-newer")

env.Append(BUILDERS={
})
```

This still has nothing to build, it just sets a few things up how I like them, and highlights two important places to fill in: variables, and build rules.  Variables are used in lots of ways (I'm trying not to duplicate too much of the [SCons manual](https://scons.org/documentation.html) here), but most notably for our purposes, they get substituted into command-line build rules.  Since command-line build rules are also pretty easy to execute e.g. on a grid, my policy is to limit myself to command-line build rules, each a script under `src/`, with behavior, hyper-parameters, etc governed by switches (`argparse` makes this very easy).  This has an additional advantage that each build rule can be executed outside of SCons.

So, now I start defining build rules as entries in the dictionary passed as the `BUILDERS` argument.  For example, I might have a script for turning some raw data into a clean list of JSON objects, maybe `src/raw_to_json.py`, that only reads up to a limited number of items.  The entry could be:

```python
    "RawToJSON" : env.Builder(**env.ActionMaker("python",
                                                "src/raw_to_json.py",
                                                "-i ${SOURCES[0]} -o ${TARGETS[0]} -l ${LIMIT}")),
```

You can probably ignore `ActionMaker` internals, it's a steamroller wrapper that makes sure SCons tracks the script file and can (if needed) run the command on a grid.  Just know the three arguments correspond to *interpreter*, *script*, and *arguments*.  Notice the substitution syntax has access, when the rule is invoked, to the sources and targets, and is also using a new variable, `LIMIT`.  Again, see the SCons documentation for more on all of this, but we'll want to add a `LIMIT` variable to the `AddVariables` invocation, an entry like:

```python
    ("LIMIT", "How much data?", 10000),
```

Ok, let's pretend you've now defined all the important stages of your experimental pipeline as build rules, along with variables to control how they behave: `PrepareData`, `TrainModel`, `ApplyModel`, `PlotOutputs`, and you have a bunch of data sets and hyper-parameters against which you want to plot performance.  You might do something like this:

```python
clean_data = {}
for name, dataset in env["DATASETS"].items():
    clean_data[name] = env.PrepareData("work/${NAME}.json", 
                                       dataset,
                                       NAME=name)

outputs = []
for hyperparam in range(env["MIN_HYPERPARAM"], env["MAX_HYPERPARAM"]):
    for name, datum in clean_data.keys():
        model = env.TrainModel("work/${HYPERPARAM}/${NAME}.model", 
                               datum, 
                               HYPERPARAM=hyperparam, NAME=name)
        for other_name, other_datum in clean_data.keys():
            outputs.append(env.ApplyModel("work/${HYPERPARAM}/${NAME}/${OTHER_NAME}.output",
                                          [model, other_datum],
                                          HYPERPARAM=hyperparam, NAME=name, OTHER_NAME=other_name))
env.PlotOutputs("work/plots.png",
                outputs)
```
As this shows, build rules are invoked by specifying the `targets`, the `sources`, and then any number of variables can be specified or overridden.  In reality this snippet is a lot more verbose than it needs to be: SCons has facilities for inferring output file names and much more.  But it illustrates the idea of quickly sweeping hyperparameters, dataset combinations, etc, how SCons uses string-substitutions in commands, and so forth.

With steamroller loaded, if the variable `USE_GRID` exists and is set to `True`, SCons will try to submit all build rules using `qsub` using job-dependencies to make sure the experimental structure is respected.  This has only been tested on a couple of grids, so it will almost surely need to be tweaked to work elsewhere, but if you let me know I can probably get it patched quickly.

## Example project

TBD
