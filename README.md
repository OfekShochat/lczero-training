# What's new here?

I've added a few features and modules to the training pipeline. Please direct any questions to the Leela Discord server.

# Auxiliary losses (WIP)
It would be expedient to follow the KataGo paper (https://arxiv.org/abs/1902.10565) to add some auxiliary losses. That paper found that training the model to predict the opponent's response and the final ownership of each piece of territory on the board roughly doubled training speed. In designing auxiliary losses for lc0 we should note several key points:

(1) The self-attention architecture will inevitably and quickly supersede ResNets. This means that we should design our auxiliary losses for the self-attention architecture.

(2) Auxiliary losses must capture information which is both rich and complicated enough that it is difficult to glean through backpropagation and useful to the task.

(3) There are a few types of auxiliary losses to choose from: scalars, per-square information, and, given (1), between-square information.

Because we are training large models, we shouoldn't have to worry about the time it takes to mask squares with no pieces. All of the below would require changes to the training data generation and parsing code.

## Scalars
What we have now is not very useful for (2).

(a) Value

(b) Moves left

We could also work with one-hot encodings of the number of moves left or something similar, but I don't see a way to improve this. KataGo predicts a pdf over the final score, and this is the closest we have to that.

## Per-square information
Things get interesting here. With a per-square auxiliary loss we can pass a signal to each square and hopefully speed up training. I am considering the following losses for each square:

(a) The next piece after the current one to occupy the square

(b) The amount of time the piece on this square will stay on the board (it can only be removed on the opponent's turn)

(c) The amount of time the piece on this square will stay on this square

## Between-square information
The benefit of this type of auxiliary loss is that it works well with the self-attention architecture. We already have

(a) Policy

calculated through self-attention. We could also consider the following: for each square,

(b) The square to which the piece on this square will move to.

(c) The square from which the piece will come that occupies this square next.

These would allow the model to generate connections between squares which are useful in the self-attention step but which suffer a weak signal from the policy loss.





# Quality of life
There are three quality of life improvements: a progress bar, metrics for value and policy accuracy, and pure attention code

Progress bar: A simple progress bar implemented in the Python `rich` module displays the current steps (including part-steps if the batches are split) and the expected time to completion.

More Metrics: I've added train value accuracy and train policy accuracy for the sake of completeness and to help detect overfitting. The speed difference is negligible.

Pure attention: The pipeline no longer contains any code from the original ResNet architecture, except in the protobuf stuff, which has yet to be updated. This makes for clearer yamls and code.

# Modules
There are a few modules I've introduced. I only list the apparently useful ones here. For reference, doubling the computation size (i.e., 40% larger embeddings or 100% more layers) seems to add 1.2% policy accuracy at 16h/256/1024dff.

Note that the large expansion ratios of 4x in the models I report here are not as useful in larger models. A 1536dff outperforms a 4096dff at 10x 8h/1024.

The main improvement is fullgen, which adds 1.2% policy accuracy to a 10x 16h/256/1024dff model. The other two are talking heads, which is necessary with fullgen but does not as of yet have a control condition to compare to, and DCD/dydense, which are dynamic kernel methods that both seem to add 0.5% ish policy acc at 16h/256/1024dff. However, DCD is faster and much more memory efficient with large embedding sizes.

## Fullgen

Fullgen is the best improvement by far. It adds around 1.2% policy accuracy to a 10x 16h/256/1024dff model. The motivation is simple: how can we encode global information into self-attention? The encoder architecture has two stages: self-attention and a dense feedforward layer. Self-attention only picks out information shaired between pairs of squares, while the feedforward layer looks at only one square. The only way global information enters the picture is through the softmax, but this cannot be expected to squeeze any significant information out. 

Of course repeated application of self-attention is sufficient with large enough embedding sizes and layers, but chess is fundamentally different from image recognition and NLP. The encoder architecture effectively partitions inputs into nodes and allows them at each layer to spread information and then do some postprocessing with the results. This works in image recognition since it makes sense to compare image patches and the image can be represented well by patch embeddings at each patch. In NLP, the lexical tokens are very suited for this spread of information since the simple structures of grammar allows self-attention (with distance embeddings of course so that tokens can interact locally at first).


Compared to these problems, chess is a nightmare. What it means that there are two rooks on the same file depends greatly on whether there are pieces between them. Even while the transformer architecture provides large gains against ResNets, which are stuck processing local information, it is still not suited for a problem which requires processing not at the between-square level bet on the global level. 

The first solution was logit gating. Arcturai observed that adding embeddings which represent squares which can be reached through knight, bishop, or rook moves vastly improved the architecture. My first attempt at improving upon this was logit gating. Because the board structure is fixed, it makes sense to add an additive offset to the attention logits so that heads can better focus on what is important. A head focusing on diagonals could have its gating emphasize square-pairs which lie on a same diagonal. I achieved further improvements applying multiplicative factors to the attention weights.

This solution works well, but still has its shortcomings. In particular, it is static. We'd like our offsets to change with the input. If pawns are blocking out a diagonal, we would like to greatly reduce the information transfer between pieces on that diagonal. This leads to fullgen, which dynamically generates additional attention logits from the board state. Because it should focus on spatial information and a 13-hot vector is sufficient to completely describe the board state, the reprsentation is generated by applying a dense layer to compress each square into a representation of size 32 (of course, the embeddings already contain processed information which will be useful in the computation).

This is then flattened and put through two dense layers with hidden size 256 and swish nonlinearities (outperforms relu by 0.1% pol acc). Finally, a dense layer is applied to hx64x64, where h is the number of fullgen heads. This is then reshaped into (h, 64, 64) and concatenated with the logits calculated through self-attention. A good performance tradeoff is h=4, but h=1 can extract most of the performance gains. This last dense layer is extremely parameter intensive, so it is shared across all layers. This works well in practice.

It seems that logit gating no longer improves over fullgen, thought this may just be because the fullgen dense layer has a bias. More experiments are required.

## Talking heads


The second improvement is talking heads. It simply applies learned linear transformations to the attention logits and final attention weights. It is needed to convert the concatenated logits and fullgen weights so that the output has the right number of heads. This works well across self-attention layers in several applications, but it can be slow because of the small dimension sizes and memory inefficient because multiple copies of attention logits/weights are computed.

## Dynamic kernel methods


Finally, there are the dynamic kernel methods. DyDense applies softmax attention to several kernels. It has a channelwise option which does not improve performance. Two kernels on the value and feedforward out layers adds 0.5% pol acc on a 10x 16h/256/1024dff network. However, it is fairly slow, adding 20% to training time, and also heavily increases memory requirements on networks with large embedding sizes.

DCD in the original paper divides the kernel into a static one which is modulated by attention weights on channels (as is done in squeeze excitation) and also a dynamic component which is represented by the product MPN, where the dimensions are (C, L), (L, L), and (L, C), respectively. M and N are static and P is conditioned on the input. To keep parameters manageable, L is set to roughly the square root of C. I instead treat the static kernel as if it were the identity and apply three dense layers instead of computing the MPN factorization, which can get unwieldy with large embeddings. The results are added together. I only do this before the qkv vectors are computed, which is much faster (10% increase in training time) and lighter on memory while retaining the 0.5% pol acc improvement. Applying the full dcd everywhere adds 0.9% pol acc over vanilla but this is not justified by the added complexity.

I have experimented with dynamic activations but because most of the processing is done at the self-attention step, these provide very little improvement.


# Training

The training pipeline resides in `tf`, this requires tensorflow running on linux (Ubuntu 16.04 in this case). (It can be made to work on windows too, but it takes more effort.)

## Installation

Install the requirements under `tf/requirements.txt`. And call `./init.sh` to compile the protobuf files.

## Data preparation

In order to start a training session you first need to download training data from https://storage.lczero.org/files/training_data/. Several chunks/games are packed into a tar file, and each tar file contains an hour worth of chunks. Preparing data requires the following steps:

```
wget https://storage.lczero.org/files/training_data/training-run1--20200711-2017.tar
tar -xzf training-run1--20200711-2017.tar
```

## Training pipeline

Now that the data is in the right format one can configure a training pipeline. This configuration is achieved through a yaml file, see `training/tf/configs/example.yaml`:

The configuration is pretty self explanatory, if you're new to training I suggest looking at the [machine learning glossary](https://developers.google.com/machine-learning/glossary/) by google. Now you can invoke training with the following command:

```bash
./train.py --cfg configs/example.yaml --output /tmp/mymodel.txt
```

This will initialize the pipeline and start training a new neural network. You can view progress by invoking tensorboard:

```bash
tensorboard --logdir leelalogs
```

If you now point your browser at localhost:6006 you'll see the trainingprogress as the trainingsteps pass by. Have fun!

## Restoring models

The training pipeline will automatically restore from a previous model if it exists in your `training:path` as configured by your yaml config. For initializing from a raw `weights.txt` file you can use `training/tf/net_to_model.py`, this will create a checkpoint for you.

## Supervised training

Generating trainingdata from pgn files is currently broken and has low priority, feel free to create a PR.
