# MLJTuning

Hyperparameter optimization for
[MLJ](https://github.com/alan-turing-institute/MLJ.jl) machine
learning models.

[![Build Status](https://travis-ci.com/alan-turing-institute/MLJTuning.jl.svg?branch=master)](https://travis-ci.com/alan-turing-institute/MLJTuning.jl)
[![Coverage Status](https://coveralls.io/repos/github/alan-turing-institute/MLJTuning.jl/badge.svg?branch=master)](https://coveralls.io/github/alan-turing-institute/MLJTuning.jl?branch=master)

### Contents

 - [Who is this repo for?](#who-is-this-repo-for)
 - [What's provided here?](#what-is-provided-here)
 - [How do I implement a new tuning strategy?](#how-do-i-implement-a-new-tuning-strategy)
 - [How do I implement a new selection heuristic?](#how-do-i-implement-a-new-selection-heuristic)

*Note:* This component of the [MLJ
  stack](https://github.com/alan-turing-institute/MLJ.jl#the-mlj-universe)
  applies to MLJ versions 0.8.0 and higher. Prior to 0.8.0, tuning
  algorithms resided in
  [MLJ](https://github.com/alan-turing-institute/MLJ.jl).


## Who is this repo for?

This repository is not intended to be directly imported by the general
MLJ user. Rather, MLJTuning is a dependency of the
[MLJ](https://github.com/alan-turing-institute/MLJ.jl) machine
learning platform, which allows MLJ users to perform a variety of
hyperparameter optimization tasks from there.

MLJTuning is the place for developers to integrate hyperparameter
optimization algorithms (here called *tuning strategies*) into MLJ,
either by adding code to [/src/strategies](/src/strategies), or by
importing MLJTuning into a third-pary package and implementing
MLJTuning's [tuning strategy interface](#how-do-i-implement-a-new-tuning-strategy).

MLJTuning is a component of the [MLJ
  stack](https://github.com/alan-turing-institute/MLJ.jl#the-mlj-universe)
  which does not have
  [MLJModels](https://github.com/alan-turing-institute/MLJModels.jl)
  as a dependency (no ability to search and load registered MLJ
  models). It does however depend on
  [MLJBase](https://github.com/alan-turing-institute/MLJBase.jl) and,
  in particular, on the resampling functionality currently residing
  there.


## What is provided here?

This repository contains:

- a **tuning wrapper** called `TunedModel` for transforming arbitrary
  MLJ models into "self-tuning" ones - that is, into models which,
  when fit, automatically optimize a specified subset of the original
  hyperparameters (using cross-validation or other resampling
  strategy) before training the optimal model on all supplied data

- an abstract **[tuning strategy
  interface](#how-do-i-implement-a-new-tuning-strategy)** to allow
  developers to conveniently implement common hyperparameter
  optimization strategies, such as:

  - [x] search models generated by an arbitrary iterator, eg `models = [model1,
	model2, ...]` (built-in `Explicit` strategy)

  - [x] grid search (built-in `Grid` strategy)

  - [ ] Latin hypercubes

  - [x] random search (built-in `RandomSearch` strategy)

  - [ ] bandit

  - [ ] simulated annealing

  - [ ] Bayesian optimization using Gaussian processes

  - [x] structured tree Parzen estimators (`MLJTreeParzenTuning` from
	[TreeParzen.jl](https://github.com/IQVIA-ML/TreeParzen.jl))

  - [ ] multi-objective (Pareto) optimization

  - [ ] genetic algorithms

  - [ ] AD-powered gradient descent methods

- a selection of **implementations** of the tuning strategy interface,
  currently all those accessible from
  [MLJ](https://github.com/alan-turing-institute/MLJ.jl) itself.

- the code defining the MLJ functions `learning_curves!` and `learning_curve` as
  these are essentially one-dimensional grid searches


## How do I implement a new tuning strategy?

This document assumes familiarity with the [Evaluating Model
Performance](https://alan-turing-institute.github.io/MLJ.jl/dev/evaluating_model_performance/)
and [Performance
Measures](https://alan-turing-institute.github.io/MLJ.jl/dev/performance_measures/)
sections of the MLJ manual. Tuning itself, from the user's
perspective, is described in [Tuning
Models](https://alan-turing-institute.github.io/MLJ.jl/dev/tuning_models/).


### Overview

What follows is an overview of tuning in MLJ. After the overview is an
elaboration on those terms given in *italics*.

All tuning in MLJ is conceptualized as an iterative procedure, each
iteration corresponding to a performance *evaluation* of a single
*model*. Each such model is a mutated clone of a fixed prototype. In the
general case, this prototype is a composite model, i.e., a model with
other models as hyperparameters, and while the type of the prototype
mutations is fixed, the types of the sub-models are allowed to vary.

When all iterations of the algorithm are complete, the optimal model
is selected by applying a *selection heuristic* to a *history*
generated according to the specified *tuning strategy*. Iterations are
generally performed in batches, which are evaluated in parallel
(sequential tuning strategies degenerating into semi-sequential
strategies, unless the batch size is one). At the beginning of each
batch, both the history and an internal *state* object are consulted,
and, on the basis of the tuning strategy, a new batch of models to be
evaluated is generated. On the basis of these evaluations, and the
strategy, the history and internal state are updated.

The tuning algorithm initializes the state object before iterations
begin, on the basis of the specific strategy and a user-specified
*range* object.

- Recall that in MLJ a *model* is an object storing the
  hyperparameters of some learning algorithm indicated by the name of
  the model type (e.g., `DecisionTreeRegressor`). Models do not
  store learned parameters.

- An *evaluation* is an object `E` returned by some call to the
  `evaluate!` method, when passed the resampling strategy (e.g.,
  `resampling=CV(nfolds=9)`) and a battery of user-specified
  performance measures (e.g., `measures=[cross_entropy,
  accuracy]`). An evaluation object `E` contains a list of measures
  `E.measure` and a list of corresponding measurements
  `E.measurement`, each of which is the aggregrate of measurements for
  each resampling of the data, which are stored in `E.per_fold` (a
  vector of vectors). In the case of a measure that reports a value
  for each individual observation (to obtain the per-fold measurement,
  by aggregation) the per-observation values can be retrieved from
  `E.per_observation`. This last object includes `missing` entries for
  measures that do not report per-observation values
  (`reports_per_observation(measure) = false`) such as `auc`. See
  [Evaluating Model
  Performance](https://alan-turing-institute.github.io/MLJ.jl/dev/evaluating_model_performance/)
  for details. There is a trait for measures called `orientation`
  which is `:loss` for measures you ordinarily want to minimize, and
  `:score` for those you want to maximize. See [Performance
  measures](https://alan-turing-institute.github.io/MLJ.jl/dev/performance_measures/)
  for further details.

- A *tuning strategy* is an instance of some subtype `S <:
  TuningStrategy`, the name `S` (e.g., `Grid`) indicating the tuning
  (optimization) algorithm to be applied. The fields of the tuning
  strategy - called *tuning hyperparameters* - are those tuning
  parameters specific to the strategy that **do not refer to specific
  models or specific model hyperparameters**. So, for example, a
  default resolution to be used in a grid search is a hyperparameter
  of `Grid`, but the resolution to be applied to a *specific*
  hyperparameter (such as the maximum depth of a decision tree) is
  *not*. This latter parameter would be part of the user-specified
  range object. A *multi-objective* tuning strategy is one that
  consults the measurements of all `measures` specified by the user;
  otherwise only the **first** measure is consulted, although
  measurements for all measures are nevertheless reported.

- A *selection heuristic* is a rule describing how the outcomes of the
  model evaluations will be used to select the *best (optimal) model*,
  after all iterations of the optimizer have concluded. For example,
  the default `NaiveSelection()` heuristic
  simply selects the model whose evaluation `E` has the smallest or
  largest `E.measurement[1]` value, according to whether the metric
  `E.measure[1]` is a `:loss` or `:score`. Most heuristics are
  *generic* in the sense they will apply no matter what tuning
  strategy is applied.  A selection heuristic supported by a
  multi-objective tuning strategy must select *some* "best" model
  (e.g., a random Pareto optimal solution).

- The *history* is a vector of identically-keyed named tuples, one
  tuple per iteration. The tuple keys include:

  - `model`: for the MLJ model instance that has been evaluated

  - `measure`, `measurement`, `per_fold`: for storing the values of
	`E.measure`, `E.measurement` and `E.per_fold`, where `E` is the corresponding
	evaluation object.

  - `metadata`: for any tuning strategy-specific information required
	 to be recorded in the history *but not intended to be reported to
	 the user* (for example an implementation-specific representation
	 of `model`).

  There may be additional keys for tuning-specific information that
  *is* to be reported to the user (such as temperature in
  simulated annhealing).

- A *range* is any object whose specification completes the
  specification of the tuning task, after the prototype, tuning
  strategy, resampling strategy, performance measure(s), selection
  heuristic, and total iteration count are given. This definition is
  intentionally broad and the interface places no restriction on the
  allowed types of this object, although **all strategies should
  support the one-dimensional range objects** defined in `MLJBase`
  (see [below](#range-types)). Generally, a range may be understood as
  the "space" of models being searched *plus* strategy-specific data
  explaining how models from that space are actually to be generated
  (e.g., grid resolutions or probability distributions specific to
  particular hyper-parameters). For more on range types see [Range
  types](#range-types) below.


### Interface points for user input

Recall, for context, that in MLJ tuning is implemented as a model
wrapper. A model is tuned by *fitting* the wrapped model to data
(which also trains the optimal model on all available data). This
process determines the optimal model, as defined by the selection
heuristic (see above). To use the optimal model one *predicts* using
the wrapped model. For more detail, see the [Tuning
Models](https://alan-turing-institute.github.io/MLJ.jl/dev/tuning_models/)
section of the MLJ manual.

In setting up a tuning task, the user constructs an instance of the
`TunedModel` wrapper type, which has these principal fields:

- `model`: the prototype model instance mutated during tuning (the
  model being wrapped)

- `tuning`: the tuning strategy, an instance of a concrete
  `TuningStrategy` subtype, such as `Grid`

- `resampling`: the resampling strategy used for performance
  evaluations, which must be an instance of a concrete
  `ResamplingStrategy` subtype, such as `Holdout` or `CV`

- `measure`: a measure (loss or score) or vector of measures available
  to the tuning algorithm, the first of which is optimized in the
  common case of single-objective tuning strategies

- `selection_heuristic`: some instance of `SelectionHeuristic`, such
  as `NaiveSelection()` (default)

- `range`: as defined above - roughly, the space of models to be searched

- `n`: the number of iterations (number of distinct models to be
  evaluated)

- `acceleration`: the computational resources to be applied (e.g.,
  `CPUProcesses()` for distributed computing and `CPUThreads()` for
  multi-threaded processing)

- `acceleration_resampling`: the computational resources to be applied
  at the level of resampling (e.g., in cross-validation)


### Implementation requirements for new tuning strategies

As sample implementations, see [/src/strategies/](/src/strategies)


#### Summary of functions

Several functions are part of the tuning strategy API:

- `setup`: for initialization of state (compulsory)

- `extras`: for declaring and formatting additional user-inspectable information
  going into the history

- `tuning_report`: for declaring any other strategy-specific information
  to report to the user (optional)

- `models`: for generating batches of new models and updating the
  state (compulsory)

- `update_metadata`: ;;;

- `default_n`: to specify the total number of models to be evaluated when
  `n` is not specified by the user

- `supports_heuristic`: a trait used to encode which selection
  heuristics are supported by the tuning strategy (only needed if you
  define a strategy-specific heuristic)

**Important note on the history.** The initialization and update of the
history is carried out internally, i.e., is not the responsibility of
the tuning strategy implementation. The history is always initialized to
`nothing`, rather than an empty vector.

The above functions are discussed further below, after discussing types.


#### The tuning strategy type

Each tuning algorithm must define a subtype of `TuningStrategy` whose
fields are the hyperparameters controlling the strategy that do not
directly refer to models or model hyperparameters. These would
include, for example, the default resolution of a grid search, or the
initial temperature in simulated annealing.

The algorithm implementation must include a keyword constructor with
defaults. Here's an example:

```julia
mutable struct Grid <: TuningStrategy
	goal::Union{Nothing,Int}
	resolution::Int
	shuffle::Bool
	rng::Random.AbstractRNG
end

# Constructor with keywords
Grid(; goal=nothing, resolution=10, shuffle=true,
	 rng=Random.GLOBAL_RNG) =
	Grid(goal, resolution, MLJBase.shuffle_and_rng(shuffle, rng)...)
```

#### Range types

Generally new types are defined for each class of range object a
tuning strategy should like to handle, and the tuning strategy
functions to be implemented are dispatched on these types. It is
recommended that every tuning strategy support at least these types:

- one-dimensional ranges `r`, where `r` is a `MLJBase.ParamRange` instance

- (optional) pairs of the form `(r, data)`, where `data` is extra
  hyper-parameter-specific information, such as a resolution in a grid
  search, or a distribution in a random search

- abstract vectors whose elements are of the above form

Recall that `ParamRange` has two concrete subtypes `NumericRange` and
`NominalRange`, whose instances are constructed with the `MLJBase`
extension to the `range` function.

Note in particular that a `NominalRange` has a `values` field, while
`NumericRange` has the fields `upper`, `lower`, `scale`, `unit` and
`origin`. The `unit` field specifies a preferred length scale, while
`origin` a preferred "central value". These default to `(upper -
lower)/2` and `(upper + lower)/2`, respectively, in the bounded case
(neither `upper = Inf` nor `lower = -Inf`). The fields `origin` and
`unit` are used in generating grids or fitting probability
distributions to unbounded ranges.

A `ParamRange` object is always associated with the name of a
hyperparameter (a field of the prototype in the context of tuning)
which is recorded in its `field` attribute, a `Symbol`, but for
composite models this might be a be an `Expr`, such as
`:(atom.max_depth)`.

Use the `iterator` and `sampler` methods to convert ranges into
one-dimensional grids or for random sampling, respectively. See the
[tuning
section](https://alan-turing-institute.github.io/MLJ.jl/dev/tuning_models/#API-1)
of the MLJ manual or doc-strings for more on these methods and the
`Grid` and `RandomSearch` implementations.


#### The `setup` method: To initialize state

```julia
state = setup(tuning::MyTuningStrategy, model, range, verbosity)
```

The `setup` function is for initializing the `state` of the tuning
algorithm (available to the `models` method). Be sure to make this
object mutable if it needs to be updated by the `models` method.

The `state` is a place to record the outcomes of any necessary
intialization of the tuning algorithm (performed by `setup`) and a
place for the `models` method to save and read transient information
that does not need to be recorded in the history.

The `setup` function is called once only, when a `TunedModel` machine
is `fit!` the first time, and not on subsequent calls (unless
`force=true`). (Specifically, `MLJBase.fit(::TunedModel, ...)` calls
`setup` but `MLJBase.update(::TunedModel, ...)` does not.)

The `verbosity` is an integer indicating the level of logging: `0`
means logging should be restricted to warnings, `-1`, means completely
silent.

The fallback for `setup` is:

```julia
setup(tuning::TuningStrategy, model, range, verbosity) = range
```

However, a tuning strategy will generally want to implement a `setup`
method for each range type it is going to support:

```julia
MLJTuning.setup(tuning::MyTuningStrategy, model, range::RangeType1, verbosity) = ...
MLJTuning.setup(tuning::MyTuningStrategy, model, range::RangeType2, verbosity) = ...
etc.
```


#### The `extras` method: For adding user-inspectable data to the history

```julia
MLJTuning.extras(tuning::MyTuningStrategy, history, state, E) -> named_tuple
```

This method should return any user-inspectable information to be
included in a new history entry, that is in addition to the `model`,
`measures`, `measurement` and `per_fold` data.
***This method must return a named tuple***,
human readable if possible. Each key of the
returned named tuple becomes a key of the new history entry.

Here `E` is the full evalutation object for `model` and `history` the
current history (before adding the new entry).

The fallback for `extras` returns an empty named tuple.


#### The `models` method: For generating model batches to evaluate

```julia
MLJTuning.models(tuning::MyTuningStrategy, model, history, state, n_remaining, verbosity)
	-> vector_of_models, new_state
```

This is the core method of a new implementation. Given the existing
`history` and `state`, it must return a vector ("batch") of *new*
model instances `vector_of_models` to be evaluated, and the updated
`state`. Any number of models can be returned (and this includes an
empty vector or `nothing`, if models have been exhausted) and the
evaluations will be performed in parallel (using the mode of
parallelization defined by the `acceleration` field of the
`TunedModel` instance). ***An update of the history, performed
automatically under the hood, only occurs after these evaluations.***

Most sequential tuning strategies will want include the batch size as
a hyperparameter, which we suggest they call `batch_size`, but this
field is not part of the tuning interface. In tuning, whatever models
are returned by `models` get evaluated in parallel.

In a `Grid` tuning strategy, for example, `vector_of_models` is a
random selection of `n_remaining = n - length(history)` models from
the grid, so that `models` is called only once (in each call to
`MLJBase.fit(::TunedModel, ...)` or `MLJBase.update(::TunedModel,
...)`). In a bona fide sequential method which is generating models
non-deterministically (such as simulated annealing),
`vector_of_models` might be a single model, or a small batch of models
to make use of parallelization (the method becoming "semi-sequential"
in that case).

##### Including model metadata

If a tuning strategy implementation needs to record additional
metadata in the history, for each model generated, then instead of
model instances, `vector_of_models` should be vector of *tuples* of the
form `(m, metadata)`, where `m` is a model instance, and `metadata`
the associated data. ***To access the metadata for the `j`th element of
the existing history, use `history[j].metadata`.***

If the tuning algorithm exhausts it's supply of new models (because,
for example, there is only a finite supply) then `vector_of_models`
should be an empty vector or `nothing`. Under the hood, there is no
fixed "batch-size" parameter, and the tuning algorithm is happy to
receive any number of models. If `vector_of_models` contains more
models than required to complete all tuning iterations, then it is
simply truncated.

Some simple tuning strategies, such as `RandomSearch`, will want to
return as many models as possible in one hit. The argument
`n_remaining` is the difference between the current length of the
history and the target number of iterations `tuned_model.n` set by the
user when constructing his `TunedModel` instance, `tuned_model` (or
`default_n(tuning, range)` if left unspecified).


####  The `tuning_report` method: To add to the user-inspectable report

As with any model, fitting a `TunedModel` instance generates a
user-accessible report. Note that the fallback report already includes
additions to the history created by the `extras` method mentioned
above. To add more strategy-specific information to the report, one
overloads `tuning_report`.

Specically, the report generated by fitting a `TunedModel` is
constructed with this code:

```julia
report1 = (best_model         = best_model,
		   best_history_entry = best_user_history_entry,
		   history            = user_history,
		   best_report        = best_report)

report = merge(report1, tuning_report(tuning, history, state))
```

where:

- `best_model` is the best model instance (as selected according to
  the user-specified `selection heuristic`).

- `best_user_history` is the corresponding entry in the history with `metadata` removed.

- `best_report` is the report generated when fitting the `best_model`
  on all available data.

- `user_history` is the full history with `metadata` entries removed.

- `tuning_report(::MyTuningStrategy, ...)` is a method the implementer
  may overload that ***must return a named tuple, preferably human readable

The fallback for `tuning_report` returns an empty named-tuple.


#### The `default_n` method: For declaring the default number of iterations

```julia
MLJTuning.default_n(tuning::MyTuningStrategy, range)
```

The `models` method, which is allowed to return multiple models in
it's first return value `vector_of_models`, is called until one of the
following occurs:

- The length of the history matches the number of iterations specified
by the user, namely `tuned_model.n` where `tuned_model` is the user's
`TunedModel` instance. If `tuned_model.n` is `nothing` (because the
user has not specified a value) then `default_n(tuning, range)` is
used instead.

- `vector_of_models` is empty or `nothing`.

The fallback is

```julia
default_n(tuning::TuningStrategy, range) = DEFAULT_N
```

where `DEFAULT_N` is a global constant. Do `using MLJTuning;
MLJTuning.DEFAULT_N` to see check the current value.


#### The `supports_heuristic` trait

If you define a selection heuristic `SpecialHeuristic` (see
[below](#how-do-i-implement-a-new-selection-heuristic)) and that
heuristic is specific to a tuning strategy `TuningStrategy` then you
must define

```julia
MLJTuning.supports_heuristic(::TuningStrategy, ::SpecialHeuristic) = true
```


### Implementation example: Search through an explicit list

The most rudimentary tuning strategy just evaluates every model
generated by some iterator, such iterators constituting the only kind
of supported range. The models generated must all have a common type
and, in th implementation below, the type information is conveyed by
the specified prototype `model` (which is otherwise ignored).  The
fallback implementations for `extras` and `report_history`
suffice.

```julia

mutable struct Explicit <: TuningStrategy end

mutable struct ExplicitState{R,N}
	range::R
	next::Union{Nothing,N} # to hold output of `iterate(range)`
end

ExplicitState(r::R, ::Nothing) where R = ExplicitState{R,Nothing}(r,nothing)
ExplictState(r::R, n::N) where {R,N} = ExplicitState{R,Union{Nothing,N}}(r,n)

function MLJTuning.setup(tuning::Explicit, model, range, verbosity)
	next = iterate(range)
	return ExplicitState(range, next)
end

# models returns all available models in the range at once:
function MLJTuning.models(tuning::Explicit,
                          model,
                          history,
                          state,
                          n_remaining,
						  verbosity)

	range, next  = state.range, state.next

	next === nothing && return nothing, state

	m, s = next
	vector_of_models = [m, ]

	next = iterate(range, s)

	i = 1 # current length of `vector_of_models`
	while i < n_remaining
		next === nothing && break
		m, s = next
		push!(vector_of_models, m)
		i += 1
		next = iterate(range, s)
	end

    new_state = ExplicitState(range, next)

    return vector_of_models, new_state

end

function default_n(tuning::Explicit, range)
	try
		length(range)
	catch MethodError
		DEFAULT_N
	end
end

```

For slightly less trivial example, see
[/src/strategies/grid.jl](/src/strategies/grid.jl)


## How do I implement a new selection heuristic?

Recall that a *selection heuristic* is a rule which decides on the
"best model" given the model evaluations in the tuning history. New
heuristics are introduced by defining a new struct `SomeHeuristic` subtyping
`SelectionHeuristic` and implementing a method

```julia
MLJTuning.best(heuristic::SomeHeuristic, history) -> history_entry
```
where `history_entry` is the entry in the history corresponding to the model deemed "best".

Below is a simplified version of [code](src/selection_heuristics.jl)
defining the default heuristic `NaiveSelection()`
which simply chooses the model with the lowest (or highest) aggregated
performance estimate, based on the first measure specified by the user
in his `TunedModel` construction (she may specify more than one).

```julia
struct NaiveSelection <: MLJTuning.SelectionHeuristic end

function best(heuristic::NaiveSelection, history)
	measurements = [h.measurement[1] for h in history]
	measure = first(history).measure[1]
	if orientation(measure) == :score
		measurements = -measurements
	end
	best_index = argmin(measurements)
	return history[best_index]
end
```

Because this selection heuristic is generic (applies to all tuning
strategies) we additionally define

```julia
MLJTuning.supports_heuristic(strategy, heuristic::NaiveSelection) = true
```

For strategy-specific selection heuristics, see
[above](#the-supportsheuristic-trait) on how to set this trait.
