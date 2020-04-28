# Discussion

pulled from the functions stuff:

There is one architectural decision I made here - not at all sure it is the right one. But lets say we have $seq = (x_0, x_1, x_2, x_3,...)$ as before. Then `abs(seq)` is translated into the mathmatical equivalent of `seq.map(lambda s: abs(s))`. That does not happen for `user_func`'s. The reason is I'm not at all sure what is the right thing to do when two different sequences are passed to `DeltaR`. Instead, in the prototype this has been made explicit. One can always write helper functions and compose them, of course.

As a side note, it might be interesting to combine this with some of the `numpy.histogram` like methods to implement them on the backend.

From the text in the multiple objects:

I'm not at all sure what the 2's and 3's are due to - I'm assuming brems or similar objects. This is to investigate at a later time!

I would note that the function `associate_particles` didn't just write itself. I had to think a bit carefully. I had first attempted to write it the definition of the column `all` in a single line. However, it gets very hard to track how many levels deep you are in an array.
