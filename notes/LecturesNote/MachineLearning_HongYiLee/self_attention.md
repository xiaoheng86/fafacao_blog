# Self-attention

## Background

When the input is a set of vectors instead of just one single vector, how to let the model consider the context?

![outputofset](media/outputofset.png)



Some thoughts about this:

![sequencelabel](media/sequencelabel.png)

* If we can make the window big enough to cover the whole context? No, cause the length of sequence is not fixed as for each input



Self-attention is a layer of network, after which the vector will be filled with context

![selfatten](media/selfatten.png)

![selfatten_whitebox](media/selfatten_whitebox.png)