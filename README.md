# pytorch-sample : data augmentation for pytorch

This package provides a set of transforms and data structures for sampling from in-memory or out-of-memory data. I'm openly taking requests for new transforms or new features to the samplers. 

## Transforms

### Torch Transforms
These transforms work directly on torch tensors

- `Compose()` : string together multiple transforms
- `AddChannel()` : add a dummy channel dimension to a tensor (useful if your images are of size `28x28` for example, and you need them to be size `1x28x28`)
- `SwapDims()` : swap the axes of the image to the given tuple order (numpy syntax). For example, to switch from CHW to HWC ordering, you can use `Transpose((1,2,0))`.
- `RangeNormalize()` : normalize a tensor between a `min` and `max` value (e.g. between (-1,1), (0,1), etc.)
- `StdNormalize()` : normalize a tensor to have zero mean and unit variance
- `Slice2D()` : take a random 2D slice from a 3D image (or video) along a given axis
- `RandomCrop()` : take a random crop of a given size from a 2D images
- `SpecialCrop()` : take a crop of a given size from one of the four corners of an image or the center of an image
- `Pad()` : pad an image by a given size
- `RandomFlip()` : randomly flip a given image horizontally and/or vertical with a given probability
- `ToTensor()` : convert numpy array or pil image to torch tensor

### Affine Transforms
The following transforms perform affine (or affine-like) transforms on torch tensors. 

- `Rotation()` : randomly rotate an image between given degree bounds
- `Translation()` : randomly translate an image horizontally and/or vertically between given bounds
- `Shear()` : randomly shear an image between given radian bounds
- `Zoom()` : randomly zoom in or out on an image between given percentage bounds

We also provide a class for stringing multiple affine transformations together so that only one interpolation takes place:

- `Affine()` : perform an affine transform with all of the above options available as arguments to this function, with the benefit of using only one interpolation

- `AffineCompose()` : perform a string of explicitly-provided affine transforms, with the benefit of using only one interpolation.

## Sampling
We provide the following datasets which provide general structure and iterators for sampling from and using transforms on in-memory or out-of-memory data:

- `TensorDataset()` : sample from and/or iterate through an input and target tensor, while providing transforms and a sampling procedure.

- `FolderDataset()` : sample from and/or iterate images or arbitrary data types existing in directories, which will only be loaded into memory as needed.

### Sampling Features
- sample/augmentation without any target tensor
- use a regular expression to find or filter out certain images
- Load input and target images from the same folder and identify them using regular expressions
- Apply the same augmentation/affine transforms to input and target images
- save transformed/augmented images to file

## Examples & Tutorial

### TensorDataset Examples
The `TensorDatset` provides a class structure for sampling from data that is already loaded into memory as torch tensors.

#### `TensorDataset` Explained

Here is the class signature for the `TensorDataset` class:

```python
class TensorDataset(Dataset):

    def __init__(self, 
                 input_tensor,
                 target_tensor=None,
                 transform=None, 
                 target_transform=None,
                 co_transform=None, 
                 batch_size=1,
                 shuffle=False,
                 sampler=None,
                 num_workers=0,
                 collate_fn=default_collate, 
                 pin_memory=False)
```

You can see the most obvious argument - `input_tensor`, which would be your input images/data. This can be of arbitrary size, but the first dimension should always be the number of samples and if the input represents an image then the channel dimension should be before the other dimensions (e.g. `(Channels, Height, Width)`).

There is also an <b>optional</b> target tensor, which might be a vector of integer classifications, continous values, or even another set of images or arbitrary data.

Next, there is the `transform`, which takes in the input tensor, performs some operation on it, and returns the modified version. The `target_transform` does the same thing with the `target_tensor`. 

There is also a `co_transform` argument to perform transforms on the `input_tensor` and `target_tensor` together. This is particularly useful for segmentation tasks, where we may want to perform the same affine transform on both the input image and the segmented image. Note that the `co_transform` will occur <b>after</b> the individual transforms.

There is a `batch_size` argument, which determines how many samples to take for each batch. There is also the boolean `shuffle` argument, which determines whether we will take samples sequentially as given or in a random order. 

Finally, there are a few arguments which relate to the multi-processing nature of the samping, which I wont get into but will refer interested readers to the official pytorch docs.

#### `TensorDataset` Image -> Class Label Sampling

Having an input tensor be a collection of images and the target tensor be a vector of class labels is perhaps the most common scenario. We'll start by loading the mnist dataset from `torchvision`:

```
from torchvision.datasets import MNIST
train = MNIST(root='/users/ncullen/desktop/data/', train=True, download=True)
x_train = train.train_data
y_train = train.train_labels
test = MNIST(root='/users/ncullen/desktop/data/', train=False, download=True)
x_test = test.test_data
y_test = test.test_labels
print(x_train.size(), ' - ' , x_test.size())
```
Here we have a training set of 60k images of `28x28` size, and a testing st of 10k images of equal size, along with the digit labels for the images. 

We can look plot a few of these images:
```python
import matplotlib.pyplot as plt
%matplotlib inline
for i in range(5):
    print('DIGIT: ', y_train[i])
    plt.imshow(x_train.numpy()[i])
    plt.show()
```

Now, we can create a `TensorDataset` for the training images. We wont have any transforms, but we will have a batch size of 3.
```python
from ptsample import TensorDataset
train_data = TensorDataset(input_tensor=x_train, target_tensor=y_train,
                batch_size=3)
```

To get the first batch of 3 images and 3 labels, simply call `train_data.next()` or `next(train_data)`:
```python
x_batch, y_batch = train_data.next()
print(x_batch.size(), ' - ' , y_batch.size())
print(x_batch.min() , ' - ' , x_batch.max())
print(x_batch.type())
```

We see that the images are still of size `28x28` and the tensors are still from range `0 - 255`, and the tensor is of type `ByteTensor`. Let's use three standard torch transforms -- `AddChannel()`, `RangeNormalize()`, and `TypeConvert()` --  wrapped in a `Compose()` transform to change this:

```python
from ptsample.transforms import AddChannel, RangeNormalize, Compose

tform = Compose([TypeConvert('float'), AddChannel(), RangeNormalize(0,1)])
train_data = TensorDataset(x_train, y_train, transform=tform, batch_size=3)
x_batch, y_batch = train_data.next()
print(x_batch.size())
print(x_batch.min() , ' - ' , x_batch.max())
print(x_batch.type())
```

Very nice! Now let's add some simple augmentation transforms like `RandomCrop()` and `RandomFlip()`. 

`RandomCrop()` takes a crop size and `RandomFlip()` takes two booleans (whether to flip horizontal, and whether to flip vertical) and a probability with which to apply those flips.

```python
from ptsample.transforms import RandomCrop, RandomFlip

process = Compose([TypeConvert('float'), AddChannel(), RangeNormalize(0,1)])
augment = Compose([RandomFlip(h=True, v=False, p=0.5), RandomCrop((20,20))])
tform = Compose([process, augment])

train_data = TensorDataset(x_train, y_train, transform=tform, batch_size=3)
x_batch, y_batch = train_data.next()

for i in range(3):
    plt.imshow(x_batch.numpy()[i][0])
    plt.show()
```

Awesome! Now, let's take it one step further and apply some Affine transforms. We provide a few common affine transforms which can be individually used as transforms. For instance, let's randomly rotate the image using the `Rotate()` transform.

```python
from ptsample.transforms import Rotation
process = Compose([TypeConvert('float'), AddChannel(), RangeNormalize(0,1)])
augment = Rotation(30) # randomly rotate between (-30, 30) degrees
tform = Compose([process, augment])
train_data = TensorDataset(x_train, y_train, transform=tform, batch_size=3)
x_batch, y_batch = train_data.next()

for i in range(3):
    plt.imshow(x_train.numpy()[i])
    plt.title('ORIGINAL')
    plt.show()
    plt.imshow(x_batch.numpy()[i][0])
    plt.title('ROTATED')
    plt.show()
```

So cool! 

What if we wanted to string together multiple affine transforms? Well, affine transforms require interpolation, and having an interpolation for each affine transform might quickly cause the quality of the image details to distintegerate. For that reason, we provide an `AffineCompose()` transform which can compose multiple affine transforms together such that only one interpolation takes place. It does this by combining the transformation matrices of all the given transforms into a single matrix (without losing any info) before applying it.

Here, we will use both rotation and translation. The `Translate()` transform takes in a tuple `(x,y)` which provide lower and upper bounds, respectively, on the zoom in the image. `1.0` means no zoom, anything greater than `1.0` means zoom out, and less means zoom in.

```python
from ptsample.transforms import Rotation, Zoom, AffineCompose
process = Compose([TypeConvert('float'), AddChannel(), RangeNormalize(0,1)])
r_tform = Rotation(30) # randomly rotate between (-30, 30) degrees
z_tform = Zoom((1.0, 1.4)) # randomly zoom out btwn 100% and 140% of image size
affine_tform = AffineCompose([r_tform, z_tform]) # string affine tforms together
tform = Compose([process, affine_tform])
train_data = TensorDataset(x_train, y_train, transform=tform, batch_size=3)
x_batch, y_batch = train_data.next()

for i in range(3):
    plt.imshow(x_train.numpy()[i])
    plt.title('ORIGINAL')
    plt.show()
    plt.imshow(x_batch.numpy()[i][0])
    plt.title('ROTATED AND ZOOMED')
    plt.show()
```

Amazing! 

That was a good overview of the functionality of the `transforms` on torch tensors and the `TensorDataset` class for sampling from input and target tensors. 

#### `TensorDataset` Image Sampling

I want to demonstrate another useful feature, which is sampling without any target tensor. This functionality can be nice if you just want to randomly transform some images to create an augmented dataset, but you don't want to do include a target tensor. Even more, we provide a `ToFile()` transform that will save all of the transformed images to file. This is good for inspecting what the sampling and augmentation is doing before actually using it. 

We provide the option to save to the following formats:
- `.npy' (numpy format - no intensity rescaling will take place)
- `.png` or `.jpg` (will automatically rescale intensity to 0 - 255)
    - not currently supported

```python
#from ptsample.transforms import ToFile
process = Compose([TypeConvert('float'), AddChannel(), RangeNormalize(0,1)])
r_tform = Rotation(30) # randomly rotate between (-30, 30) degrees
z_tform = Zoom((1.0, 1.4)) # randomly zoom out btwn 100% and 140% of image size
affine_tform = AffineCompose([r_tform, z_tform]) # string affine tforms together
#save = ToFile(path='/users/ncullen/desktop/transformed_images/', save_format='npy')
tform = Compose([process, affine_tform])
train_data = TensorDataset(x_train, transform=tform, batch_size=3)
x_batch = train_data.next()
```
