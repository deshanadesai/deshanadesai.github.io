---
layout: post
title: Fine-Tuning ImageNet model for Classification
category: posts
---



### What am I doing today?
> I have installed caffe and the required libraries using [this](https://github.com/tiangolo/caffe/blob/ubuntu-tutorial-b/docs/install_apt2.md) really good guide. **The aim of my experiment is to fine-tune tha VGG-16 network for classification.** The VGG-16 network I use is pre-trained on ImageNet for classification. I will exploit it's caffemodel to fine-tune the weights for my own purpose.

<div class="divider"></div>



### Data-Preparation


Getting the data prepared correctly finishes most of your work.

**Step 1-** Go to caffe/data folder and create your own data folder there. I shall name it 'deshana-data'. This is the folder where I will
put my data files for training and testing. 

```
Caffe/
    - data/
        - deshana-data/
            - train.txt
            - val.txt
            - Train_Images
            - Val_Images
        
```

**Step 2-** Put all the images used for training in Train_Images and all the Images used for validation in Val_Images.

**Step 3-** In train.txt, I put the name of all the images with the labels separated by a space.

```
Train_Images/0908989.jpg 0
Train_Images/0908988.jpg 2
Train_Images/0908987.jpg 27
Train_Images/0908986.jpg 15
Train_Images/0908985.jpg 5

```
Notice that the labels are integer values. 
Make sure you have a mapping of your string categories to integer values handy. 

In the same way, val.txt will contain names of images inside the Val_Images folder followed by the corresponding label.

**Step 4-** In the data folder, go to ilsrvc12/ and run in the terminal: sh get_ilsrvc_aux.sh In case this throws an error, you shall need to make the file executable with sudo chmod a+x <filename>. This downloads the train.txt, val.txt, test.txt and information on synsets used for ImageNet as useful reference.

**Step 5-** Now go to caffe (the root directory) and run ``` ./scripts/download_model_binary.py models/bvlc_reference_caffenet ``` This will download the caffemodel. This may take some time. 
So meanwhile, I am going to make the lmdb files required for data loading.

**Step 6-** First of all, make a folder named 'deshana' (anything, really!) on caffe/. And copy all the files from examples/imagenet inside it. <br>Change the following paths in create_imagenet.sh:

```
EXAMPLE=examples/imagenet # changed it to deshana
DATA=data/ilsvrc12 # changed it to data/deshana-data
TOOLS=build/tools 

TRAIN_DATA_ROOT=/path/to/imagenet/train/ # changed it to data/deshana-data/Train_Images/
VAL_DATA_ROOT=/path/to/imagenet/val/ # changed it to data/deshana-data/Val_Images/
```
You may need to provide absolute paths. 
_A common mistake: Make sure there is no space after the assignment operator ("=") in the path assignment lines_

and also set

```
RESIZE=true
```
if your images are not resized to 256 x 256. This script will do it for you.

Run the script with sh create_imagenet.sh. This will create a folder inside deshana na,ed ilsrvc_train_lmdb and ilsrvc_val_lmdb.

**Step 7-** After creating the lmdb datasets, we must compute the mean values. In the same way as step 6. Change the paths in the make_imagenet_mean.sh. Then run it. It generates deshana_binaryproto file for me.

### Fine-Tuning


**Step 8-** We need to make changes in the train_val.prototxt file which defines the architecture of the model. The file is located in caffe/models/bvlc_reference_caffenet/

In the data layers, change the paths of the binaryproto file and the source lmdb file.

```
name: "CaffeNet"
layer {
  name: "data"
  type: "Data"
  top: "data"
  top: "label"
  include {
    phase: TRAIN
  }
  transform_param {
    mirror: true
    crop_size: 227
    mean_file: "data/ilsvrc12/imagenet_mean.binaryproto"
  }
  data_param {
    source: "examples/imagenet/ilsvrc12_train_lmdb"
    batch_size: 256
    backend: LMDB
  }
}
```
Do the same for the validation data layer.

**Step 9-** Change the name of the last layer from "fc8" to "fc8_deshana". This makes sure that it does not carry the weights of the pre-trained caffemodel.
It will be assigned random weights in the beginning with gaussian mean 0 and standard deviation 0.01.

**Change the number of outputs in this fc8_deshana layer to the number of categories you have in the dataset you are using to fine-tune.**
Finally, change the learning rates so that it learns faster.

```
layer {
  name: "fc8_deshana"
  type: "InnerProduct"
  bottom: "fc7"
  top: "fc8"
  param {
    lr_mult: 10
    decay_mult: 1
  }
  param {
    lr_mult: 20
    decay_mult: 0
  }
  inner_product_param {
    num_output: 57
    ....
```

**Step 10-** Modify the solver.prototxt in the same folder.

My modifications:
```
net: "models/bvlc_reference_caffenet/train_val.prototxt"
test_iter: 1000
test_interval: 1000
base_lr: 0.001
lr_policy: "step"
gamma: 0.1
stepsize: 10000
display: 20
max_iter: 50000
momentum: 0.9
weight_decay: 0.0005
snapshot: 5000
snapshot_prefix: "models/bvlc_reference_caffenet/caffenet_train"
solver_mode: GPU
```

**Step 11-** Train the model!
```
./build/tools/caffe train \
-solver models/bvlc_reference_caffenet/solver.prototxt \
-weights models/bvlc_reference_caffenet/bvlc_reference_caffenet.caffemodel
```

**Step 12-** Test the model!
```
./build/tools/caffe test -model=models/bvlc_reference_caffenet/train_val.prototxt -weights=models/myself_caffenet/caffenet_iter_1000.caffemodel
```

### Yay! It's done!

![Minion](https://octodex.github.com/images/foundingfather_v2.png)
