Autograd
========

Autograd automatically differentiates native
[Torch](https://github.com/torch/torch7) code.

Scope
-----

Autograd has multiple goals:

* provide automatic differentiation of [Torch](https://github.com/torch/torch7)
  expressions
* support arbitrary Torch types (e.g. transparent and full support
  for CUDA-backed computations)
* full integration with [nn](https://github.com/torch/nn) modules: mix and match
  auto-differentiation with user-provided gradients
* represent complex evaluation graphs, which is very useful to describe models
  with multiple loss functions and/or inputs
* enable gradients of gradients for transparent computation of Hessians, ...

Performance
-----------

Long-term, `autograd`'s goal is to become the most flexible way to describe
arbitrary differentiable functions, with no compromise on performance. There
are currently two main limitations to performance:

* no support for sparse tensors: for excessively sparse tensor updates (e.g.
  selecting a row in a tensor, and updating it), the gradient computation
  will be naive and dense. The best way to address this would be to add
  support for sparse tensors in Torch.
* memory re-use: currently `autograd` doesn't do any memory caching, so each
  gradient estimation can potentially allocate lots of intermediate tensors.
  The best way to address this is to create a tensor pool, and have the tape
  re-use tensors from this pool as needed. This is one of the main TODOs below.

TODO
----

Autograd is work in progress. Current list of things to be developed includes:

- [ ] Gradients of gradients (Hessian)
- [ ] Helpers for building weights
- [ ] Debugging facilities (profile speed & gradient magnitude of each computation. Should just require nodeApply wrapper.)
- [ ] Add support for sparse gradients
- [ ] Implement auto-buffering so that native torch functions can re-use memory
- [ ] Implement missing gradients for `torch.max`, `torch.min`, ... more generally,
      allow operators on input data if no gradients will be computed on them (if x:max() is
      called to be used in a subsequent function, but we don't need the gradients wrt x,
      we should still be able to call it)
  (i.e. auto-generate code that's similar to what nn does for modules)
* Helpers for building nnfunc modules (tedious to write out convolution parameters)
  - [x] Basic helper logic for NN, CNN
  - [ ] Improve cascading, and parameter grouping when cascading functions
* Add more useful examples of models
  - [x] MNIST logistic regression
  - [x] MNIST MLP
  - [x] MNIST CNN
  - [ ] RNN (Penn?)
  - [ ] LSTM
- [x] Missing gradients for functions like `torch.sum(data,index)` if index is provided;
      these are critical to implement mini-batch support! (for now batches cannot be used)
- [x] make the process of returning different intermediate outputs easier: right
  now you have to define one function for each partial output, is it enough?
  => write examples
- [x] when calling `dparams = df(...)`, return the result of `f` as a second
  value: `dparams,loss = df(...)`, this way it doesn't have to be run twice.

Examples
--------

### Autograd example

A simple neural network with a multinomial logistic loss:

```lua
-- libraries:
t = require 'torch'
grad = require 'autograd'

-- define trainable parameters:
params = {
   W = {
      t.randn(100,50),
      t.randn(50,10),
   },
   b = {
      t.randn(50),
      t.randn(10),
   }
}

-- define model
neuralNet = function(params, x, y)
   local h1 = t.tanh(x * params.W[1] + params.b[1])
   local h2 = t.tanh(h1 * params.W[2] + params.b[2])
   local yHat = h2 - t.log(t.sum(t.exp(h2)))
   local loss = - t.sum(t.cmul(yHat, y))
   return loss
end

-- gradients:
dneuralNet = grad(neuralNet)

-- some data:
x = t.randn(1,100)
y = t.Tensor(1,10):zero() y[1][3] = 1

-- compute loss and gradients wrt all parameters in params:
dparams, loss = dneuralNet(params, x, y)

-- in this case:
--> loss: is a scalar (Lua number)
--> dparams: is a table that mimics the structure of params; for
--  each Tensor in params, dparams provides the derivatives of the
--  loss wrt to that Tensor.
```

Important note: only variables packed in the first argument of the
eval function will have their gradients computed. In the example above,
if the gradients wrt x are needed, then x simply has to be moved into
params. The params table can be arbitrarily nested.

See more complete examples in [examples](examples/).

Assuming the model defined above, and a training set of `{x,y}` pairs,
the model can easily be optimized using SGD:

```lua
for i,sample in datasetIterator() do
   -- estimate gradients wrt params:
   local grads, loss = dneuralNet(params, sample.x, sample.y)

   -- SGD step:
   for i = 1,#params.W do
      -- update params with an arbitrary learning rate:
      params.W[i]:add(-.01, grads.W[i])
      params.b[i]:add(-.01, grads.b[i])
   end
end
```

### Wrapping nn modules

The [nn](https://github.com/torch/nn) library provides with all sorts of very optimized
primitives, with gradient code written and optimized manually. Sometimes it's useful
to rely on these for maximum performance.

Here we rewrite the neural net example from above, but this time relying on a mix of
`nn` primitives and `autograd`-inferred gradients:

```lua
-- libraries:
t = require 'torch'
grad = require 'autograd'

-- define trainable parameters:
params = {
   W = {
      t.randn(50,100), -- note that parameters are transposed (nn convention for nn.Linear)
      t.randn(10,50),
   },
   b = {
      t.randn(50),
      t.randn(10),
   }
}

-- instantiate nn primitives:
-- Note: we do this outside of the eval function, so that memory
-- is only allocated once; moving these calls to within the body
-- of neuralNet would work too, but would be quite slower.
linear1 = grad.nn.Linear(100, 50)
acts1 = grad.nn.Tanh()
linear2 = grad.nn.Linear(50, 10)
acts2 = grad.nn.Tanh()

-- define model
neuralNet = function(params, x, y)
   local h1 = acts1(linear1(x, params.W[1], params.b[1]))
   local h2 = acts2(linear2(h1, params.W[2], params.b[2]))
   local yHat = h2 - t.log(t.sum(t.exp(h2)))
   local loss = - t.sum(t.cmul(yHat, y))
   return loss
end

-- gradients:
dneuralNet = grad(neuralNet)

-- some data:
x = t.randn(1,100)
y = t.Tensor(1,10):zero() y[1][3] = 1

-- compute loss and gradients wrt all parameters in params:
dparams, loss = dneuralNet(params, x, y)
```

This code is stricly equivalent to the code above, but will be more efficient
(this is espacally true for more complex primitives like convolutions, ...).

### Gradient checks

For ease of mind (and to write proper tests), a simple grad checker
is provided. See [test.lua](test.lua) for complete examples. In short, it can be
used like this:

```lua
-- Parameters:
local W = t.Tensor(32,100):normal()
local x = t.Tensor(100):normal()

-- Function:
local func = function(inputs)
   return t.sum(inputs.W * inputs.x)
end

-- Check grads:
tester:assert(gradcheck(func, {W=W, x=x}, 'x'), 'incorrect gradients on x')
tester:assert(gradcheck(func, {W=W, x=x}, 'W'), 'incorrect gradients on W')
```

### Model Primitives

To ease the construction of new models, we provide primitives to generate
standard models.

Each constructor returns 2 things:

* `f`: the function, can be passed to `grad(f)` to get gradients
* `params`: the list of trainable parameters

Once instantiated, `f` and `params` can be used like this:

```lua
input = torch.randn(10)
pred = f(params, input)
grads = autograd(f)(params, input)
```

Current list of model primitives includes:

#### autograd.model.NeuralNetwork

API:

```lua
f,params = autograd.model.NeuralNetwork({
   -- number of input features:
   inputFeatures = 10,

   -- number of hidden features, per layer, in this case
   -- 2 layers, each with 100 and 10 features respectively:
   hiddenFeatures = {100,10},

   -- activation functions:
   activations = 'ReLU',

   -- if true, then no activation is used on the last layer;
   -- this is useful to feed a loss function (logistic, ...)
   classifier = false,

   -- dropouts:
   dropoutProbs = {.5, .5},
})
```

#### autograd.model.SpatialNetwork

API:

```lua
f,params = autograd.model.SpatialNetwork({
   -- number of input features (maps):
   inputFeatures = 3,

   -- number of hidden features, per layer:
   hiddenFeatures = {16, 32},

   -- poolings, for each layer:
   poolings = {2, 2},

   -- activation functions:
   activations = 'Sigmoid',

   -- kernel size:
   kernelSize = 3,

   -- dropouts:
   dropoutProbs = {.1, .1},
})
```

### Loss Primitives

Similarly to model primitives, we provide common loss functions in
`autograd.loss`.
