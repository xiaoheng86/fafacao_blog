# Transformer

Transformer is a **seq2seq** model. Input a sequence, output a sequence, the output length is determined by model.

![seq2seq](media/seq2seq.png)

## Encoder

![encoder](media/encoder.png)

## Decoder

using softmax to amplify distribution vector

![autoregressive](media/autoregressive.png)

Decoder will take its own last ouput vector as input for next ouput.

![selfinput](media/selfinput.png)