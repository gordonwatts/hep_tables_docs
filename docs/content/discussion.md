# Discussion

pulled from the functions stuff:

There is one architectural decision I made here - not at all sure it is the right one. But lets say we have $seq = (x_0, x_1, x_2, x_3,...)$ as before. Then `abs(seq)` is translated into the mathmatical equivalent of `seq.map(lambda s: abs(s))`. That does not happen for `user_func`'s. The reason is I'm not at all sure what is the right thing to do when two different sequences are passed to `DeltaR`. Instead, in the prototype this has been made explicit. One can always write helper functions and compose them, of course.

As a side note, it might be interesting to combine this with some of the `numpy.histogram` like methods to implement them on the backend.