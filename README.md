# Efficient Relative Attention

## How to use
```py
from relative_attention import RelativeAttention2d
# you can also use RelativeAttention1d and RelativeAttention3d
net = RelativeAttention2d(num_heads, model_depth, max_relative_positions_past=[width, height],
                          max_relative_positions_future=None, # same as past
                          heads_share_relative_embeddings=[True, False], # share in width but not height
                          # extend embedding by using sin and cosine for the width dim, and zero padding for h
                          embedding_padding_modes=[EmbeddingPaddingMode.Extend, EmbeddingPaddingMode.Zero],
                          position_embedding_types=PositionEmbeddingType.Fixed,
                          key_start_positions=KeyStartPosition.BeforeQuery, 
                          add_bias_to_query_for_relative_logits=True, # the D term in transformer-xl
                          add_bias_to_query_for_key_logit=True, # the C term in transformer-xl
                          # use my custom kernel or the vanilla pytorch implementation
                          use_custom_cuda_kernel=True).cuda() 
q = torch.randn(batch_size, num_heads, q_height, q_width, model_depth // num_heads).cuda()
k = torch.randn(batch_size, num_heads, k_height, k_width, model_depth // num_heads).cuda()
if use_mask:
  mask = (torch.randn(batch_size * num_heads, q_height * q_width, k_height * k_width) > 0).cuda()
  # or just (q_height * q_width, k_height * k_width)
else:
  mask = None
logits = net(q, k, mask)
print(logits.size()) # batch_size * num_heads, q_height * q_width, k_height * k_width
```

## Algorithm
![Efficient Relative Position Encoding](algorithm.png?raw=true "Efficient Relative Position Encoding")

## Reasoning
I was trying to use a relative position encoding in my 2d attention network
and there wasn't a good implementation for pytorch, so I decided to adopted the
tensor2tensor implementation into pytorch.
Furthermore our architecture, uses this operation at each layer, so I decided
to make it a bit more efficient by writing a custom **cuda** kernel. It's not
a general purpose kernel and it might be slower than vanilla pytorch code, it
depends on your GPU, your batch_size, query_size and query_dim, so profile it
on your settings before using it.

## How to profile
You can see how to profile it by checking the `speed_check()` and
`run_profiler()` function in `check.py`. for example in my settings(large batch size with 32x32 patch images) I get **1.7x** speedup in my forward and backward calls.

## Further Improvements
I also tried to fuse the logit calculation in my kernel, but it was way too
slow, compared to cublas. You can check my experiment in `experiment.py`, even
tough it's really slow for training, it has good performance in small inference
mode. For example with batch_size=1, path_size=8x8, num_heads=1, and model_depth=16 
you can get 6.5x performance gain!

## Embedding Class
`DistanceEmbedding` class is an awesome wrapper for most of the common usages,
check it out! :))

## References
* [Attention Augmented Convolutional Networks](https://arxiv.org/abs/1904.09925)
* [Transformer-XL: Attentive Language Models Beyond a Fixed-Length Context](https://arxiv.org/abs/1901.02860)
* [Tensor2Tensor](https://github.com/tensorflow/tensor2tensor/blob/master/tensor2tensor/layers/common_attention.py)
* [fairseq](https://github.com/pytorch/fairseq/blob/master/fairseq/modules/sinusoidal_positional_embedding.py)

