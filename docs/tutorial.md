
# PyWarm Basic Tutorial

## Import

To get started, first import PyWarm in your project:

```Python
import warm
import warm.functional as W
```

## Rewrite

Now you can replace child module definitions with function calls. 
For example, instead of:

```Python
# Torch
class MyModule(nn.Module):

    def __init__(self):
        super().__init__()
        self.conv1 = nn.Conv2d(in_channels, out_channels, kernel_size)
        # other child module definitions

    def forward(self, x):
        x = self.conv1(x)
        # more forward steps
```

You now use the warm functions:

```Python
# Warm
class MyWarmModule(nn.Module):

    def __init__(self):
        super().__init__()
        warm.up(self, input_shape_or_data)

    def forward(self, x):
        x = W.conv(x, out_channels, kernel_size) # no in_channels needed
        # more forward steps
```

Notice the `warm.up(self, input_shape_or_data)` at the end of the `__init__()` method.
It is required so that PyWarm can infer all shapes of itermediate steps and set up trainable parameters.
The only argument `input_shape_or_data` can either be a tensor, e.g. `torch.randn(2, 1, 28, 28)`,
or just the shape, e.g. `[2, 1, 28, 28]` for the model inputs. If the model has multiple inputs,
you may supple them in a list or a dictionary.

Although it is recommended that you attach `warm.up()` to the end of the `__init__()` of your model, you can actually
use it on the class instances outside of the definition, like a normal function call:

```Python
class MyWarmModule(nn.Module):

    def __init__(self):
        super().__init__() # no warm.up here

    def forward(self, x):
        x = W.conv(x, 10, 3)
        # forward step, powered by PyWarm


model = MyWarmModule() # call warm.up outside of the module definition

warm.up(model, [2, 1, 28, 28])
```

**Note**: If the model contains `batch_norm` layers, you need to specify the `Batch` dimension to at least 2.

# Advanced Topics

## Default shapes

PyWarm has a unified functional interface, that by default all functions accept and return tensors with shape
`(Batch, Channel, *)`, where `*` is any number of additional dimensions. For example, for 2d images,
the `*` usually stands for `(Height, Width)`, and for 1d time series, the `*` means `(Time,)`.

This convention is optimized for the performance of Convolutional networks. It may become less efficient if your
model relies heavily on dense (Linear) or recurrent (LSTM, GRU) layers. You can use different input and
output shapes by specifying `in_shape`, `out_shape` keyword arguments in the function calls. These keywords
accept only letters `'B'`, `'C'` and `'D'`, which stand for `Batch`, `Channel`, and `*` (extra Dimensions)
respectively. So for example if for a 1d time series you want to have `(Time, Batch, Channel)` as the output shape,
you can specify `out_shape='DBC'`.

## Dimensional awareness

PyWarm functions can automatically identify 1d, 2d and 3d input data, so the same function can be used on different
dimensional cases. For example, the single `W.conv` is enough to replace `nn.Conv1d, nn.Conv2d, nn.Conv3d`.
Similarly, you don't need `nn.BatchNorm1d, nn.BatchNorm2d, nn.BatchNorm3d` for differnt inputs, a single `W.batch_norm`
can replace them all.

## Shape inference

Many neural network layers will perform a transformation of shapes. For example, after a convolution operation,
the shape is changed from `(Batch, ChannelIn, *)` to `(Batch, ChannelOut, *)`. PyTorch nn Modules require the user to 
keep track of both `in_channels` and `out_channels`. PyWarm relieves this pain by inferring the `in_channels` for you,
so you can focus more on the nature of your tasks, rather than chores.

## Argument passdown

If the signature of a PyWarm function does not specify all possible argument of its torch nn Module couterpart, it will pass down
additional keyword arguments to the underlying nn Module. For example, if you want to specify strides of 2 for a conv layer,
just use `W.conv(..., stride=2)`. The only thing to remember is that you have to specify the full keyword, instead of 
relying on the position of arguments.

## Parameter initialization per layer

Unlike PyTorch's approach, paramter initialization can be specified directly in PyWarm's functional interface.
For example:

```Python
x = W.conv(x, 20, 1, init_weight='kaiming_uniform_')
```
This makes it easier to create layer specific initialization in PyWarm. You no long need to go through
`self.modules()` and `self.parameters()` to create custom initializations.

By default, PyWarm will look into `torch.nn.init` for initialization function names.
Alternatively, you may just specify a callable, or a tuple `(fn, kwargs)` if the callable accepts more than 1 input.

If the initialization is not specified or `None` is used, the corresponding layer will get default initializations as used
in torch nn modules. 

## Apply activation nonlinearity to the output

PyWarm's functional interface supports adding an optional keyword argument `activation=name`, where
name is a callable or just its name, which represents an activation (nonlinearity) function
in `torch.nn.functional` or just `torch`. By default no activation is used.

## Mix and Match

You are not limited to only use PyWarm's functional interface. It is completely ok to mix and match the old
PyTorch way of child module definitions with PyWarm's function API. For example:

```Python
class MyModel(nn.Module):

    def __init__(self):
        super().__init__()
        # other stuff
        self.conv1 = nn.Conv2d(2, 30, 7, padding=3)
        # other stuff

    def forward(self, x):
        y = F.relu(self.conv1(x))
        y = W.conv(y, 40, 3, activation='relu')
```

## Custom layer names

Normally you do not have to specify layer names when using the functional API.
PyWarm will track and count usage for the layer type and automatically assign names for you. For example,
subsequent convolutional layer calls via `W.conv` will create `conv_1`, `conv_2`, ... etc. in the parent module.

Nevertheless, if you want to ensure certain layer have particular names, you can specify `name='my_name'`
keyword arguments in the call.

Alternatively, if you still want PyWarm to count usage and increment ordinal for you, but only want to customize 
the base type name, you can use `base_name='my_prefix'` keyword instead. The PyWarm modules will then have
names like `my_prefix_1`, `my_prefix_2` in the parent module.

See the PyWarm [resnet example in the examples folder](https://github.com/blue-season/pywarm/blob/master/examples/resnet.py)
on how to use these features to load pre-trained model parameters into PyWarm models.
