.. _tutorial_aspace:

Tutorial: Working with common sense
===================================

Divisi is particularly designed for working with knowledge in semantic
networks. Here, we'll reason over ConceptNet, which contains general
common-sense knowledge in natural language.

Building a common sense matrix
------------------------------

The first thing we'll need to do is to import Divisi, and then load its graph
representation of ConceptNet.

.. note::

    **This is going to be slow.** We're building the matrix from nothing, and
    in most cases, you'd want to load a pickle file where you've already done
    this. In fact, this particular pickle file is already included in Divisi.

    If you want to get to the point faster, you can type::

        >>> A = divisi2.network.conceptnet_matrix('en')

    and jump to :ref:`aspace_svd`. But if you keep reading this section,
    we'll show you how to *make* that matrix.

    The exact results you get, of course, may depend on which release of the
    ConceptNet matrix you use, or which release of the ConceptNet graph you
    built it from.

The graph file is packaged with Divisi and stored in the csc/divisi2/data
directory, so we can use a special path beginning with `data:` to get at it.
You could give a normal path as well to load a different file.

>>> import divisi2
>>> conceptnet = divisi2.load('data:graphs/conceptnet_en.graph')

Now we build a Divisi sparse matrix out of this data. The rows should be the
nodes of the graph (which are the concepts in ConceptNet), and the columns
should represent their *features*, or connections to other nodes.

The :func:`divisi2.network.sparse_matrix`
function can automate this for us -- we just need to ask for the `'nodes'` and
`'features'`. `cutoff=5` means to keep only nodes and features with a degree of
5 or more.

>>> A = divisi2.network.sparse_matrix(conceptnet, 'nodes', 'features', cutoff=5)
>>> print A
SparseMatrix (12564 by 19719)
         IsA/spor   IsA/game   UsedFor/   UsedFor/   person\\C ...
baseball 3.609584   2.043731   0.792481   0.500000   0.500000  
sport       ---     1.292481      ---     1.000000      ---    
yo-yo       ---        ---        ---        ---        ---    
toy         ---     0.500000      ---     1.160964      ---    
dog         ---        ---        ---     0.792481      ---    
...

We can see here a preview of :math:`A`, a :class:`SparseMatrix` of 12564 concepts and
19719 features.

The rows of this matrix are labeled with common-sense *concepts*, and the
columns are labeled with *features* that describe them.

.. note::

   Divisi2 has a special way of printing node-feature matrices. The features
   are actually represented as tuples of a direction (`'left'` | `'right'`), the
   name of a relation, and the node that is related. The direction
   distinguishes "baseball is a sport" from "sport is a baseball". When
   printed as labels, however, these tuples are abbreviated:
   `('right', 'IsA', 'sport')` looks like `IsA/sport`, and
   `('left', 'IsA', 'sport')` looks like `sport\\IsA`.

Now that you've spent all this time constructing a matrix, you might want to
save a copy of it in the current directory::

    >>> divisi2.save(A, 'conceptnet_en.pickle')

Then you could load it again with::

    >>> A = divisi2.load('conceptnet_en.pickle')

.. _aspace_svd:

Singular value decomposition
----------------------------

By now, you've either built a ConceptNet matrix, or you've loaded a
pre-built matrix.

Given this matrix, we can factor it using SVD and ask for 100
principal components, and then reconstruct its approximation from the factors.
(See :ref:`svd_intro` for more explanation of what happens here.)

>>> concept_axes, axis_weights, feature_axes = A.svd(k=100)

We could also refer to these objects as :math:`U`, :math:`\Sigma`, and
:math:`V`, to match the traditional notation of SVD.

`concept_axes`, or :math:`U`, relates the concepts of the input matrix to the
principal axes. Its rows are labeled with concepts, like the rows of the input
matrix. Its columns are unlabeled, because they simply refer to the 100 largest
eigenvectors, in order. We can look at an entry of this matrix as follows:

>>> concept_axes.entry_named('baseball', 0)  # Where's "baseball" on axis 0?

Or we can get the concepts that have the most positive and most negative values
on the largest eigenvector, by looking up the first column of the matrix and
asking for its top items.

>>> concept_axes[:,0].top_items(5)
>>> (-concept_axes[:,0]).top_items(5)

`feature_axes`, or :math:`V`, relates features to the principal axes in the
same way.  Its rows, then, are labeled with feature tuples:

>>> feature_axes.row_labels[:5]
[('right', u'IsA', u'sport'),
 ('left', u'IsA', u'baseball'),
 ('right', u'IsA', u'toy'),
 ('left', u'IsA', u'yo-yo'),
 ('right', u'IsA', u'write')]

`axis_weights` is a list of eigenvalues. Mathematically, it is the diagonal of
:math:`\Sigma` from the SVD. Conceptually, it represents the strength of each
eigenvector.

Making predictions
------------------

To make predictions about previously unknown statements, we want to look up
entries in the *reconstructed* matrix :math:`A^\prime = U \Sigma V^T`.

To do this, we use the :meth:`divisi2.reconstruct` function. This does not
actually multiply the matrices; instead, it provides a
:class:`ReconstructedMatrix` object that *acts* like the product of those
matrices when you look at its entries.

>>> predictions = divisi2.reconstruct(concept_axes, axis_weights, feature_axes)

As one example, we look up the concept "pig" and ask for the predicted values
of two features it can take on the right side:

Does a pig have legs?

>>> predictions.entry_named('pig', ('right', 'HasA', 'leg'))
0.15071150848740383

Can a pig fly?

>>> predictions.entry_named('pig', ('right', 'CapableOf', 'fly'))
-0.26456066802309008

Calculating similarity
----------------------

Because the `concept_axes` matrix relates concepts to the 100 axes, we can
consider each concept to have a position in a space defined by a
100-dimensional vector. The intuition we have about an SVD of a semantic
network is that similar concepts (and similar features) have vectors that point
in similar directions.

To look up the position of "cow":

>>> cow = concept_axes.row_named('cow')
>>> print cow

The amount of similarity of concepts to each other, in this space, could be
represented by the dot products of all concepts with all others. This
similarity matrix can be computed as :math:`U \Sigma^2 U^T`. Another way to say
this is that we want to multiply the matrix :math:`U \Sigma` by its own
transpose. In our terms, :math:`U \Sigma` is `concept_axes` weighted by
`axis_weights`.

Like before, we have a method that simulates this product,
:meth:`divisi2.reconstruct_similarity`.

But if we do this alone, the results we get are on no meaningful numerical
scale. Consider this example where we look up the similarity between "horse"
and "cow":

>>> sim = divisi2.reconstruct_similarity(concept_axes, axis_weights, post_normalize=False)
>>> sim.entry_named('horse', 'cow')
36.693964805281276

So "horse" and "cow" are 36.69 similar to each other. Is that a lot? Who
can tell?

If we're looking for similarities between particular concepts, we can deal with
the scale problem by neutralizing the magnitudes of the concepts altogether.
We simply *normalize* every row of :math:`U \Sigma` to be a unit vector.
Then the dot products in the similarity matrix are simply the cosines of the
angles between the corresponding vectors, creating a well-defined similarity
scale that ranges from 1.0 (exactly similar) to -1.0 (exactly dissimilar).

It would be somewhat difficult and verbose to ask Divisi to normalize the rows
at this particular step, so Divisi has a shorthand for this:

>>> sim_n = divisi2.reconstruct_similarity(concept_axes, axis_weights, post_normalize=True)
>>> sim_n.entry_named('horse', 'cow')
0.82669084520494984
>>> sim_n.entry_named('horse', 'stapler')
-0.031207494261339251

Varations on normalization
..........................

In many applications, we want to rank similarities or predictions and choose
the best ones. If we don't normalize anything, the concepts and features that
have the most information about them will show up at the top of the results:

>>> sim.row_named('table').top_items()
[('table', 134.82), ('desk', 60.77), ('chair', 47.08), ('kitchen', 41.74),
('house', 40.16), ('bed', 38.14), ('restaurant', 37.04), ('plate', 30.25),
('paper', 29.86), ('person', 29.80)]

A table isn't that similar to a person; ConceptNet just happens to know a lot
about people. So what if we normalize the rows as above?

>>> sim_n.row_named('table').top_items()
[('table', 1.000), ('newspaper article', 0.694), ('dine table', 0.681),
('dine room table', 0.676), ('table chair', 0.669), ('dine room', 0.663),
('bookshelve', 0.636), ('table set', 0.629), ('home depot', 0.591),
('wipe mouth', 0.587)]

Newspaper article? Home Depot? How did those get there? The problem is that
normalization is too generous to some concepts. If a concept is not well
described by the components in the SVD, it will end up with a smaller magnitude
than it started with, as most of the information about that concept is dropped.
Normalizing all the rows magnifies those concepts enormously, in whatever
direction they happen to weakly point.

What we need to do is normalize the input matrix *before* the SVD. This
way, all concepts are created equal, but after the SVD, the ones that are
poorly represented are reduced in magnitude, and will not rank highly in queries
such as this one.

Because we already did the normalization we wanted before the SVD, we set
`post_normalize` back to False. 

>>> A_pre = A.normalize_all()
>>> U_pre, S_pre, V_pre = A_pre.svd(k=100)
>>> sim_pre = divisi2.reconstruct_similarity(U_pre, S_pre, post_normalize=False)
>>> sim_pre.row_named('table').top_items()
[('table', 1.718), ('desk', 1.195), ('kitchen', 0.988), ('chair', 0.873),
('restaurant', 0.850), ('plate', 0.822), ('bed', 0.772), ('cabinet', 0.678), 
('refrigerator', 0.652), ('cupboard', 0.617)]

.. note::

    Normalizing the rows and columns of a matrix presents a bit of a
    mathematical conundrum: the input matrix has both concepts and features we
    would need to normalize, and if we normalize just one direction, we let the
    other direction distort the results. But it's impossible to normalize an
    arbitrary matrix so that all its rows and columns are unit vectors.

    The compromise that Divisi2 provides is to divide each entry by the
    *geometric mean* of its row norm and its column norm. The rows and columns
    don't actually become unit vectors, but they all become closer to unit
    vectors, at least.

Spreading activation
--------------------

With the similarity measure, we have been able to tell whether two concepts
have similar common-sense properties to each other. In applications such as
sentiment analysis or topic detection, however, we are looking for something
more general: whether two concepts are *related* in any way by common sense.

For example, the concepts "sad" and "cry" are only a bit similar, but they are
very related.

Spreading activation works by assigning values, or *activations*, to nodes in a
network. Then, those nodes spread some of their activation to their neighboring
nodes. This process can be iterated many times. The result is that, given a
starting node or set of nodes, it associates every node with a quantity of
activation.

Divisi2 provides the *reconstruct_activation* function, which takes in SVD
results and constructs an operator that simulates an infinite number of steps
of spreading activation (with diminishing effects).

.. note::

    We do not include the `conceptnet_assoc_en` matrix with Divisi2, so the
    first time you run this, it will have to build it. This takes some time.

>>> assoc = divisi2.network.conceptnet_assoc('en')
>>> U, S, _ = assoc.svd(k=100)
>>> spread = divisi2.reconstruct_activation(U, S)

Each entry of the matrix says how much activation would spread from one concept
to another, with a maximum of 1.0.

>>> spread.entry_named('cat', 'cat')
1.0000000000000007
>>> spread.entry_named('cat', 'dog')
0.80290202113709208
>>> spread.entry_named('cat', 'tree')
0.2546597941841342
>>> spread.entry_named('cat', 'buddhism')
0.092909665436672548

We can examine the rows of this matrix like other Divisi2 results. For example,
what are the concepts most related to thinking?

>>> spread.row_named('think').top_items()
[(u'think', 1.0), (u'create idea', 0.920), (u'brain activity', 0.919), (u'contemplate', 0.911), (u'cogitate', 0.906), (u'solve problem', 0.892), (u'insight', 0.886), (u'cognition', 0.883), (u'mind', 0.882), (u'answer question', 0.880)]

What are the concepts most related to sadness? (This could be useful in
determining the emotional affect of things!)

>>> spread.row_named('sad').top_items()
[(u'sad', 1.0), (u'sob', 0.910), (u'wipe eye', 0.900), (u'watery eye', 0.896), (u'weep tear', 0.895), (u'shed tear', 0.894), (u'weep', 0.894), (u'sad emotion', 0.894), (u'produce tear', 0.893), (u'sob tear', 0.893)]

This operation becomes even more powerful when used together with :ref:`categories`.

