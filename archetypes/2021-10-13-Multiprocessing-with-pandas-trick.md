---
layout: post
title: Multiprocessing with pandas trick
category: posts
---

In an effort to retain more of what I learn everyday, I decided to experiment with more frequent blogging. This is meant to be a fun short post of what sparked my curiosity or brought me joy today as a programmer. For me, that was a neat multiprocessing trick i used today.

I use the pool.map() function to speed up tasks that can be broken down into asynchronous tasks often. Today, I needed to apply it to a dataframe. More specifically, I had a target coordinate system that I wanted to project each row in my dataframe representing a point in the image coordinate system to. 

So there are two tasks at hand:
- project each point (ie process each row of the dataframe) in parallel
- create a new dataframe which stores the outputs of all the projections

I created a class called Projection to store values for the point of origin and the axes of the target coordinate frame.

```
class Projection:
    def __init__(self, ori, axX, axY, axZ):
        self.ori = ori
        self.axX = axX
        self.axY = axY
        self.axZ = axZ
        
        
    def project_point(self,rows):
        for idx, row in rows.iterrows():
            pt = np.array([row['x'], row['y'], row['z']])
            vec_a = pt-self.ori
            rows.loc[idx,'x'] = np.dot(vec_a, self.axX)
            rows.loc[idx,'y'] = np.dot(vec_a, self.axY)
            rows.loc[idx,'z'] = np.dot(vec_a, self.axZ)        
        return rows
```
Since we have a finite number of cores, we first calculate the number of cores to be used. Then the next task is to split the dataframe into multiple chunks (a quick google search showed the most efficient way to do this would be using the numpy array_split function - the function handles the case when the split may not be exact for each chunk). The number of chunks it is split into is equal to the number of cores to parallelize the task with.


Next, we call the pool object as usual but this time we return the modified dataframe chunk from the projection function and use the pandas concat function to concatenate all the modified chunks together. With this, we have the modified dataframe that can be used for further processing.
    
``` 
    proj = Projection(ori, axX, axY, axZ)

    num_cores = multiprocessing.cpu_count()-1  
    df_split = np.array_split(frame, num_cores)
    pool = multiprocessing.Pool(num_cores)
    df = pd.concat(pool.map(proj.project_point, df_split))
    pool.close()
    pool.join()
```    

For a quick comparison, this method with the parallelization took ~6 mins instead of ~14 mins without it (2.33x faster) for 900 frames.
