---
layout: post
title: Fancy PCA (Data Augmentation) with Scikit-Image
category: posts
---


Let's start with the basics! We know that an integer variable is stored in 4 bytes. An integer array would be a consecutive stream of many such 4 bytes. A string of text would store number of bytes proportional to the characters perhaps with a little padding. Storage of numbers and text is understood, but how on earth, would we store an image? How do we turn an image into something that can be processed and stored in memory?

Well, images are stored as numpy matrices. You make a m x n x 3 matrix- where the three different layers denote the Red, Green and Blue dimensions. The 2D space m x n helps to locate a particular pixel value in the RGB matrix. Every pixel is essentially made up of varied proportions of Red, Green and Blue intensities. This RGB color model uses a space of 8 bits x 3 (color channels) for every pixel. Every cell in the matrix therefore occupies 8 bits. The total possible combinations are 2^8 = 256 and the intensities thereby vary from 0-255. This can be scaled from 0-1 by dividing the intensity by 255. These color images may also form a matrix of shape m x n x 3 x 4. Why? Well, the 4th layer can be added to encode transparency of the image.

If every image is a matrix of values, you can apply all sorts of mathematical transformations to the matrix and voila! you have done the same to your image.

Today's task consists of altering the intensities of the RGB channels in training images as suggested by Alex Krizhevsky in his seminal paper [ImageNet Classification with Deep Convolutional Neural Networks](https://www.nvidia.cn/content/tesla/pdf/machine-learning/imagenet-classification-with-deep-convolutional-nn.pdf).

### Task Goal: (Quoting Alex Krizhevsky et al.)

> The second form of data augmentation consists of altering the intensities of the RGB channels in training images. Specifically, we perform PCA on the set of RGB pixel values throughout the ImageNet training set. To each training image, we add multiples of the found principal components, with magnitudes proportional to the corresponding eigenvalues times a random variable drawn from a Gaussian with mean zero and standard deviation 0.1.

Let's first import a few libraries useful down the road and read an image-


```python
import numpy as np
import matplotlib.pyplot as plt
from skimage import io,transform

# Using six sample images.
imnames = ['n00.jpg','n01.jpg','n02.jpg','n03.jpg','n04.jpg','n05.jpg']

# Read collection of images with imread_collection
imlist = (io.imread_collection(imnames))
```


Now before proceeding, we must transform all the images to a standard 256 x 256 size so there is no inconsistency in size.


```python
for i in range(len(imlist)):
	# Using the skimage.transform function-- resize image (m x n x dim).
	m=transform.resize(imlist[i],(256,256,3))
```


Now we come to the real deal - performing PCA. What is PCA? It is a way of finding patterns in the data and once you have found these patterns, it reduces the the number of dimensions without much loss of information. Let's start doing this and learn on the way.


<strong>Step 1: Get your data.</strong>
Well, we already have a set of images. But in this case, we are going to treat every pixel as a data point. This means you have a ton of data points which are vectors with 3 values: R, G and B. We are then going to compute PCA on these data points. Let's make a consolidated dataset with all these different data points. We need to make an array of all these vectors containing the RGB values from all the images.

Essentially, we want to turn the image matrix of m x n x 3 to lists of rgb values i.e. (m*n) x 3.

For visualization purposes, you can try playing with img[:,:,3] and img[0,0,3] to see a sample of the output-format we want.


```python
# initializing with zeros.
res = np.zeros(shape=(1,3))

for i in range(len(imlist)):
	m=transform.resize(imlist[i],(256,256,3))
	# Reshape the matrix to a list of rgb values.
	arr=m.reshape((256*256),3)
	# concatenate the vectors for every image with the existing list.
	res = np.concatenate((res,arr),axis=0)

# delete initial zeros' row
res = np.delete(res, (0), axis=0)
# print list of vectors - 3 columns (rgb)
print res

Output:

[[ 0.60784314 0.59215686 0.47843137]
[ 0.6 0.58823529 0.4627451 ]
[ 0.59607843 0.58627451 0.44313725]
...,
[ 0.2635723 0.09494485 0.03219975]
[ 0.28074449 0.11550245 0.05275735]
[ 0.32495404 0.19500613 0.11413909]]

```



<strong>Step 2: Subtract the mean. For PCA to work properly, you must subtract the mean from each of the dimensions.</strong>


```python
m = res.mean(axis = 0)

Output (m):
[ 0.46348368 0.43890141 0.41217445]

res = res - m

Output(res):
[[ 0.14435945 0.15325546 0.06625693]
[ 0.13651632 0.14933389 0.05057065]
[ 0.13259475 0.1473731 0.03096281]
...,
[-0.19991138 -0.34395655 -0.37997469]
[-0.1827392 -0.32339895 -0.35941709]
[-0.13852964 -0.24389528 -0.29803535]]
```


<strong>Step 3: Calculate the covariance matrix. Since the data is 3 dimensional, the cov matrix will be 3x3. No surprises there.</strong>


```python
R = np.cov(res, rowvar=False)

Output (R):
[[ 0.06814677 0.06196288 0.05043152]
[ 0.06196288 0.06608583 0.06171048]
[ 0.05043152 0.06171048 0.07004448]]
```

There seems to be very weak covariance of the pixel intensities in the non-diagonal elements. Interesting concept point- Because the covariance of the ith random variable with itself is simply that random variable's variance, each element on the principal diagonal of the covariance matrix is just the variance of each of the elements in the vector. however, the current example is only for sample purposes and the results would be better with more colorful images.


<strong>Step 4: Calculate the Eigenvectors and Eigenvalues of the covariance matrix. </strong> 
Since the cov matrix is a square matrix, this can be done. Brief recap: Geometrically, an eigenvector corresponding to a real, nonzero eigenvalue points in a direction that is stretched by the transformation and the eigenvalue is the factor by which it is stretched. If the eigenvalue is negative, the direction is reversed.

Now, the eigenvector with the highest value is also the principal component of the dataset. For example if your data was most widely spread across the x-axis, the eigenvector representing the x-axis would be your principal component. Once the eigenvectors are found, we shall sort them from the highest to lowest. Now we can ignore the lower eigenvalues, since the loss of information will be small (they are hardly important components and so the variations along those components in the data will be very small). In this way, by leaving out some components your final data will have lesser components than the original dataset. Voila! this is how dimensionality reduction is performed.

So now we basically form something called a feature vector, which is just a fancy name for the matrix of the chosen eigenvectors. So let's do this!

```python
from numpy import linalg as LA
evals, evecs = LA.eigh(R)
idx = np.argsort(evals)[::-1]
evecs = evecs[:,idx]

# sort eigenvectors according to same index
evals = evals[idx]

# select the best 3 eigenvectors (3 is desired dimension
# of rescaled data array)
evecs = evecs[:, :3]

# make a matrix with the three eigenvectors as its columns.
evecs_mat = np.column_stack((evecs))
```


<strong>Step 5: The Final Step- Performing PCA. </strong> 
Once we have chosen the eigenvectors or the components that we wish to keep in our data and formed a feature vector- we simply take the transpose of the vector and multiply it to the left of the original dataset, transposed. you can work out this matrix math by yourself.

what will this give us? It will give us the original data solely in terms of the components we chose. Further, the eigenvectors are always perpendicular to each other- to express the data in the most efficient manner.


```python
# carry out the transformation on the data using eigenvectors
# and return the re-scaled data, eigenvalues, and eigenvectors
m = np.dot(evecs.T, res.T).T
```


We have now transformed our data so that it is expressed in terms of the patterns between them where patterns are lines that most closely describe the relationships between the data. In the case of transformations by eigenvalues, we have simply altered the magnitude or 'level of stretch' (visually speaking) of these eigenvectors without changing its direction.

Now, having performed PCA on the pixel values, <strong>we need to add multiples of the found principal components with magnitudes proportional to the corresponding eigenvalues times a random variable drawn from a Gaussian distribution with mean zero and standard deviation 0.1.</strong> Ok! Let's keep rowing!


```python
def data_aug(img = img):
	mu = 0
	sigma = 0.1
	feature_vec=np.matrix(evecs_mat)

	# 3 x 1 scaled eigenvalue matrix
	se = np.zeros((3,1))
	se[0][0] = np.random.normal(mu, sigma)*evals[0]
	se[1][0] = np.random.normal(mu, sigma)*evals[1]
	se[2][0] = np.random.normal(mu, sigma)*evals[2]
	se = np.matrix(se)
	val = feature_vec*se

	# Parse through every pixel value.
	for i in xrange(img.shape[0]):
		for j in xrange(img.shape[1]):
			# Parse through every dimension.
			for k in xrange(img.shape[2]):
				img[i,j,k] = float(img[i,j,k]) + float(val[k])

# Calling function for first image.
# Re-scaling from 0-255 to 0-1.
img = imlist[0]/255.0
data_aug(img)
plt.imshow(img)
```


And that's it! We are done!
We have now perturbed the image colours along these PCA axes.

If you can figure out faster ways to do this, please let me know :)

Putting it all together:


```python
import numpy as np
import matplotlib.pyplot as plt
from skimage import io,transform
%matplotlib inline

def data_aug(img = img):
	mu = 0
	sigma = 0.1
	feature_vec=np.matrix(evecs_mat)

	# 3 x 1 scaled eigenvalue matrix
	se = np.zeros((3,1))
	se[0][0] = np.random.normal(mu, sigma)*evals[0]
	se[1][0] = np.random.normal(mu, sigma)*evals[1]
	se[2][0] = np.random.normal(mu, sigma)*evals[2]
	se = np.matrix(se)
	val = feature_vec*se

	# Parse through every pixel value.
	for i in xrange(img.shape[0]):
		for j in xrange(img.shape[1]):
			# Parse through every dimension.
			for k in xrange(img.shape[2]):
				img[i,j,k] = float(img[i,j,k]) + float(val[k])

imnames = ['n00.jpg','n01.jpg','n02.jpg','n03.jpg','n04.jpg','n05.jpg']
#load list of images
imlist = (io.imread_collection(imnames))

res = np.zeros(shape=(1,3))
for i in range(len(imlist)):
	# re-size all images to 256 x 256 x 3
	m=transform.resize(imlist[i],(256,256,3))
	# re-shape to make list of RGB vectors.
	arr=m.reshape((256*256),3)
	# consolidate RGB vectors of all images
	res = np.concatenate((res,arr),axis=0)
res = np.delete(res, (0), axis=0)

# subtracting the mean from each dimension
m = res.mean(axis = 0)
res = res - m

R = np.cov(res, rowvar=False)
print R

from numpy import linalg as LA
evals, evecs = LA.eigh(R)

idx = np.argsort(evals)[::-1]
evecs = evecs[:,idx]
# sort eigenvectors according to same index

evals = evals[idx]
# select the first 3 eigenvectors (3 is desired dimension
# of rescaled data array)

evecs = evecs[:, :3]
# carry out the transformation on the data using eigenvectors
# and return the re-scaled data, eigenvalues, and eigenvectors
m = np.dot(evecs.T, res.T).T

# perturbing color in image[0]
# re-scaling from 0-1
img = imlist[0]/255.0
data_aug(img)
plt.imshow(img)
```


Huzza! We have done it!
