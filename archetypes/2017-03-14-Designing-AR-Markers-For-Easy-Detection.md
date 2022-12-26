---
layout: post
title: Designing Markers for Easy Detection in Real Life Images.
category: posts
---

If you had an image and you wanted to identify what is shown in the image, or if you had products in an image and you wanted to 
identify the products, you could tag them with a sticker. What would this sticker look like? And how would you detect the sticker?
Your sticker must have a design embedded in it which is detectable and decode-able at any distance from the camera, illumination 
levels, different angles etc. 

![Example Marker Detection]({{site.baseurl}}/images/s990x600_stap7.0.png)

### Making stickers that can be Detected with ease.

#### Thick Black Framing of Marker

Pointed sharp edges are easier to detect than rounded edges. Squares and rectangles are therefore ideal to use for framing the pattern.
We can detect the corners easily and check if the shape approximates a quadrilateral.

I use [Python's Cairo library](https://cairographics.org/pycairo/) to generate this black border in an image.

This is the code:
```
#!/usr/bin/env python

import math
import cairo
import random
import time

def printsmth(name):
	WIDTH, HEIGHT = 100,100

	surface = cairo.ImageSurface (cairo.FORMAT_ARGB32, WIDTH, HEIGHT)
	ctx = cairo.Context (surface)

	ctx.scale (WIDTH, HEIGHT) # Normalizing the canvas
	ctx.set_source_rgb(255,255,255)

	# BORDER FRAME GENERATION
	ctx.rectangle(0,0,1,1)
	ctx.set_source_rgb(255,255,255)
	ctx.fill()

	ctx.rectangle(0.05,0.05,0.90,0.90)
	ctx.set_source_rgb(0,0,0)
	ctx.fill()

	ctx.rectangle(0.2,0.2,0.6,0.6)
	ctx.set_source_rgb(255,255,255)
	ctx.fill()
  surface.write_to_png (str(name)+".png") # Output to PNG

printsmth('sample')
```  

The Result:

![Example Marker Frame Design]({{site.baseurl}}/images/sample.png)


#### Hierchical Black Squares at the Corners with Single Black Square at the Center.

Now, there are different things you can do to detect the orientation of the sticker and then correct it. This helps in re-orienting the design inside the frame so that it is upright. In my case, I am embedding words inside the frame of the sticker. My word identification system can only decode horizontal text, so it is extremely important for me to have orientation correction methods.

The idea here is to find the three hierarchical square patterns at the three corners and then rotate them in such a manner that they match the inverted L position. The black square in the center is useful for sticker detection since sometimes the hierarchical patterns are missed by the algorithm (due to small size).

![Example Hierarchical Square Frame Design]({{site.baseurl}}/images/milk.png)

#### Thick Black Border Frame with Orientation Correction.

Some of the disadvantage of using hierarchical squares for orientation are: <br>
1) They are small and therefore hard to detect. <br>
2) If any of the corners are undetected, due to noisy image or occlusion; the orientation correction shall fail. <br>
3) It is not completely reliable to always be able to differentiate between the fourth corner and the other corners. <br>

Here's an alternative design, where a slab of black square is used in the left hand side of the image. If we use the black square for orientation correction, we will be able to produce two images:

![Example Orientation Correction Design]({{site.baseurl}}/images/fig_a.png)
![Example Orientation Correction Design]({{site.baseurl}}/images/fig_b.jpg)

My text identification pipeline will not be able to decode fig. (b) but it should ideally be able to decode fig. (a) with high confidence score. I can use this confidence score to filter out the False Positive (Fig. b).

The code to generate this:

```
#!/usr/bin/env python

import math
import cairo
import random
import time

def printsmth(num):
	WIDTH, HEIGHT = 220,70

	surface = cairo.ImageSurface (cairo.FORMAT_ARGB32, WIDTH, HEIGHT)
	ctx = cairo.Context (surface)

	#ctx.scale (WIDTH, HEIGHT) # Normalizing the canvas
	ctx.set_source_rgb(255,255,255)

	# BORDER FRAME GENERATION

	ctx.rectangle(0,0,220,70)
	ctx.set_source_rgb(255,255,255)
	ctx.fill()

	ctx.rectangle(5,5,210,60)
	ctx.set_source_rgb(0,0,0)
	ctx.fill()
	
	ctx.rectangle(15,15,190,40)
	ctx.set_source_rgb(255,255,255)
	ctx.fill()	

	ctx.rectangle(20,20,25,25)
	ctx.set_source_rgb(0,0,0)
	ctx.fill()	

	ctx.select_font_face("None", cairo.FONT_SLANT_NORMAL, cairo.FONT_WEIGHT_BOLD)
	ctx.set_font_size(35)
	ctx.move_to(80,50)
	ctx.show_text("food")
	surface.write_to_png (str(num)+".png") # Output to PNG


printsmth('sample')
```

Now the next steps involved in here are: <br>
1) Detect the sticker. <br>
2) Orient the sticker. <br>
3) Pass the system through Text Identification/OCR pipelines. <br>
4) Filter False Positives based on the results of Text Identification pipeline confidence scores. <br>
5) Output the final identified word! Tada! We are done. <br>
