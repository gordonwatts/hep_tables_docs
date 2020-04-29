# Discussion

Below is some discussion of things I encountered along the way.

## Architectural Decisions

### How the two packages work together

There isn't really a python library that helps one work with array programming. The code to work with array slicing, manipulation, capturing `numpy` calls, etc., is common in several different implementations, but it is tightly integrated with the respective backend.

The first package, [`dataframe_expressions`](https://github.com/gordonwatts/dataframe_expressions) attempts to split that code out. It makes few assumptions about the data being manipulated - instead pushing as many decisions as possible to the backend. The [`hep_tables`](https://github.com/gordonwatts/hep_tables) code _renders_ a dataframe expression into code, actions, etc.

Internally, most of the manipulations are stored using the python `ast` package. This was chosen because it is something I understand and am familiar with, and is a rather complete way to represent computational expressions. And since this DSL is the python language - it is a natural fit. That said, it isn't obvious that a AST is the same thing as a computational compute graph - and something else would better serve.

### Iterators

Anytime one has `numpy` semantics one has to decide where the _implied_ iterators are. For example, `df.jets.pt` means:

```
for event in df:
  for j in event.jets:
    pt = j.pt
    ....
```

What does `df.jets.pt + df.jets.pt` mean? How about `df.electrons.pt + df.jets.pt` mean? In the case here, `df.jets.pt + df.jets.pt` translates to `2*df.jets.pt` and the second case is undefined: I decided it was ambiguous as to what the user meant by that statement (a 2D matrix?).

The implementation is very strict about this - so much so that as this project was finishing off it became clear it would be rather difficult to look at the invariant mass of $Z \rightarrow ee$ if one wanted to pair all electrons found in the event. The natural thing to write is, given a function `invar(p1, p2)` that calculates the invariant mass:

```
invar_masses = df.electrons.map(lambda e1: df.electrons.map(lambda e2: invar (e1, e2)))
```

However, the code will spot `df.electrons` in two places, and make them the same iterator, and thus the invariant mass will always be the mass of the electrons!

The fix is to say the second time the `df.electrons` appears, start it again. However, I am nervous about the long-range implications of that - as I do think if the user writes `df.jets.pt + df.jets.pt` they mean to double the jet `pt`. A possible solution is to do have this behavior only inside a `lambda` function.

Regardless, a clear set of rules must be define to prevent confusion.

### Functions over Sequences

Most folks from HEP would agree that `abs(df.jets.eta)` meant a column of the absolute value of jet $\eta$ values. In short, it translates to:

```
for event in df:
  for j in df.jets:
    a = abs(j.eta)
    ...
```

But what happens if one writes something like `deltaR(df.jets, df.electrons)`? This gets back to the same issue of iterators mentioned in the previous section. Is this a 2D matrix? Or something else?

In the prototype I've punted on this decision: the above code will generate an error, just as in the case of the `df.jets.pt + df.electrons.pt` expression. Even `deltaR(df.jets, df.jets)` will currently generate an error, though it is likely we know what the user meant.

In the prototype it is possible to write helper functions to specify explicitly ones intent - using the `map` function.

## Painful Places

There are some places where I struggled - not just with writing the code behind. I outline some of the user issues I encountered in this section.

### Nested Expressions

I would note that the function `associate_particles` didn't just write itself. As a reminder:

```
def associate_particles(source, pick_from):
    '''
    Associate each particle from source with a close by one from the particle list pick_from. Extend source's data model.
    
    Args:
        source            The particles we want to start from
        pick_from         For each particle from source, we'll find a close by one from pick_form.
        name              Naming we can use when we extend the data model.
        
    Returns:
        with_assoc        The source particles that had a close by match
    '''
    def dr(p1, p2):
        'short hand for calculating DR between two particles.'
        return DeltaR(p1.eta(), p1.phi(), p2.eta(), p2.phi())

    def very_near(picks, p):
        'Return all particles in picks that are DR less than 0.1 from p'
        return picks[lambda ps: dr(ps, p) < 0.1]

    source['all'] = lambda source_p: very_near(pick_from, source_p)
    
    source['has_match'] = lambda e: e.all.Count() > 0
    with_assoc = source[source.has_match]
    with_assoc['mc'] = lambda e: e.all.First()
    
    return with_assoc
```

I had to think a bit carefully. I had first attempted to write it the definition of the column `all` in a single line. However, it gets very hard to track how many levels deep I was in an array. And you could certainly do things by using arrays only - but the data model extension makes it significantly easier to reason about (at least for me). Further, the ability to split things into smaller functions - bite sized bits - seems absolutely crucial to both writing it,and figuring out what it is doing when I return to code like this 6 months after having last looked at it.

I believe the reason this is hard to understand is that things are pulled in from different contexts. For example, the line `return picks[lambda ps: dr(ps, p) < 0.1]` is pulling things from the list of MC electrons (`picks`), which is translated from a column to a single particle (`ps`) - there is one context switch. And then the `dr` is calculated vs this `p`, which is a reconstructed electron that was passed in - a second context switch.

Without violating the requirement that all this be valid python, I'm not sure how to make this better at the moment. I'm happy with this: complex intent is expressed unambiguously, and, for the most part, clearly.

Testing and designing the backend implementation for this section probably took the longest - and there are some architecture decisions I would have made differently had I started with this rather than making simple plots (part of the reason to call this a prototype!).

OTOH, I think everything after the `all` line was very straight forward and easy to understand.

### Type System

Every effort was made to keep the type system out of `dataframe_expressions`. While this means it is fairly easy to write, it also means there is almost not hope for an editor to make suggestions on what the user might type next. This is one of the most high-productivity/help/discovery mechanisms.

Second, the type system in `hep_tables` is mostly heuristics. For example, every leaf is assumed to be a `double` return. This works fine for `df.jets.pt` - the transverse momentum is a `double`. However, it can be a mistake for `df.truth.pdgId`.

Collections are declared (e.g. jets, electrons, etc.). These are hardwired into a dictionary in the code.

The proper way to do this is using python typeshed files. Some of these can be generated automatically (ROOT, for example, has a run time type system built in). But this is a big chunk of work! As a reward, however, it would mean the code generated would be more "correct" and efficient.

Lastly, the type system would allow one to generate errors quickly. Especially in this prototype it is fairly easy to generate an impossible data query. A robust type system would help catch these errors. As would more careful checking in the code!

That said, types are tracked through the code. The system knows if you are counting something that it is looking at integers, and if you are summing over all jet `pt`'s, it knows to use a `double` as an accumulator.


### Integration with common python libraries

There are a lot of data science libraries out there that it would be nice to be able to re-use. As a canonical example, I've used [seaborn](http://seaborn.pydata.org/) as a way to reason about this issue. The library takes, most naturally, a `pandas DataFrame` object as input, and will make categorical plots. It is a quite powerful way to quickly visualize a large amount of data.

In the Run 4 time frame bringing the data to the users laptop won't always be an option. This means `seaborn` will have to run in the _Analysis_ box, perhaps in an analysis facility. This feels like a great deal of work.

One option is to implement functions explicitly (as was done with the `histogram` function in the demo in an earlier section). A second option is to try to automate things, with something like a function decorator. I suspect the final answer will be something in between these two. But this is something that needs to be addressed: there is too much work in the python data science ecosystem to leave behind.

### Moving `dataframe_expressions` over the wire

Raw `python` functions are included in the data model. Some of them are evaluated as soon as their are written, but many can't be implemented as they are written and instead are called as part of the rendering process by `hep_tables`. This means that somehow these functions have to be sent over the wire from the user to the Analysis facility. It isn't obvious how to implement this.

## Fundamental Limitations

There are some "this is not designed to do" things to keep in mind.

### Loop Algorithms

The data model has hierarchical data and sequence semantics built in. There are a class of algorithms that are not well suited to these semantics: loops. For example one would be hard pressed to express a tracking algorithm using this syntax. Perhaps more fundamentally, lets say you wanted to know if an electron's parents were $Z$ boson. That implies walking the parent chain until you reach the end or a $Z$ - a loop. This is a limitation baked into the design of the `dataframe_expressions`. The only way around this is to implement a special function that implements this. It is possible one might be able to implement a generic function to cover this class of algorithms. Or piggy back on the `Aggregate` operation.

