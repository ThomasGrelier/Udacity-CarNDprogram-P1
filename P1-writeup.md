
# **Finding Lane Lines on the Road** 

---

The goal of this project is to make a pipeline that finds lane lines on the road.

This document provides a written report which includes description of the pipeline, validation results, reflections on the limitations of the developed pipeline, and proposals for improvement.

The validation of the pipeline is done in two steps:

1) first on a series of individual images

2) then on a video stream

---


## 1. Description of the pipeline

My pipeline consisted of 6 steps:

1) Conversion to grayscale

2) Gaussian smoothing

*Size of the Gaussian kernel: 5

3) Canny edge detector

*Threshold values: 50, 150

4) Selection of the region of interest

*Vertices of the polygon: [(100,imshape[0]),(420, 330), (530, 330), (890,imshape[0])]

5) Hough transform

*rho = 2; theta = pi/180; threshold = 15; min_line_length = 10; max_line_gap = 10

6) Averaging and extrapolation of the lines:

This consists in the following steps:

- Computation of the segment slopes

$$
slope = (y_2-y_1)/(x_2-x_1)
$$

```python
lines_f = lines.astype(float)       # float conversion
slope = (lines_f[0,:,3]-lines_f[0,:,1])/(lines_f[0,:,2]-lines_f[0,:,0])
```

- Removal of incoherent slopes

I defined thresholds for the absolute value of the slopes and removed the segments out of range

```python
max_slope = 0.8  # higher threshold for absolute slope
min_slope = 0.5  # lower threshold for absolute slope  
slope[np.logical_or(abs(slope)>max_slope,(abs(slope)<min_slope))]=0
```

- Selection of positive and negative slopes

I separated positive slopes (which are supposed to correspond to the right line) from negative slopes (left line)

```python
ind_neg = slope<0
ind_pos = slope>0
```

- Definition of a weight associated to each segment

I tried weighted and unweighted averaging, where the weight is equal to the length of the segment.
```python
if bool_weight: # weighted average
    weight = np.sqrt((lines_f[:,0,2]-lines_f[:,0,0])**2+(lines_f[:,0,3]-lines_f[:,0,1])**2)
else: # unweighted average
    weight = np.ones_like(lines_f[0,:,0])
```

**I wanted to assess if bringing more weight to longer segments improves the final result.**

- Weighted averaging

Definition :
$$
slope = \frac{\sum slope*weight}{\sum weight}
$$

```python
mean_neg_slope = np.sum(slope[ind_neg]*weight[ind_neg])/np.sum(weight[ind_neg])
mean_pos_slope = np.sum(slope[ind_pos]*weight[ind_pos])/np.sum(weight[ind_pos])
```    
- Computation of weighted/unweighted average of the coordinates of the segments for each of the lines

```python
# x-coord of left line
mean_neg_coor_0 = np.sum(lines_f[0,ind_neg,2]*weight[ind_neg])/np.sum(weight[ind_neg])
# y-coord of left line
mean_neg_coor_1 = np.sum(lines_f[0,ind_neg,3]*weight[ind_neg])/np.sum(weight[ind_neg])
# x-coord of right line
mean_pos_coor_0 = np.sum(lines_f[0,ind_pos,2]*weight[ind_pos])/np.sum(weight[ind_pos])
# y-coord of right line
mean_pos_coor_1 = np.sum(lines_f[0,ind_pos,3]*weight[ind_pos])/np.sum(weight[ind_pos])
```
    
- Computation of the y-intercept of each line
```python
b_neg = -mean_neg_coor_0*mean_neg_slope+mean_neg_coor_1; 
b_pos = -mean_pos_coor_0*mean_pos_slope+mean_pos_coor_1;  
```

- Computation of the boundaries of the segments
```python
Xmax_neg = np.int((Ymin-b_neg)/mean_neg_slope)  # higher left boundary
Xmin_neg = np.int((Ymax-b_neg)/mean_neg_slope)  # lower left boundary
Xmin_pos = np.int((Ymin-b_pos)/mean_pos_slope)  # higher right boundary
Xmax_pos = np.int((Ymax-b_pos)/mean_pos_slope)  # lower right boundary
```

- And we finally construct the two lines
```python
solid_lines = np.array([[[Xmin_neg,Ymax,Xmax_neg,Ymin],[Xmin_pos,Ymin,Xmax_pos,Ymax]]])
``` 

## 2. Results

### 2.1 Images
Below I display the lanes I detected for the 6 input images, using the weighted averaging.

**Using weighted averaging brought little improvement, except for one of the images.**

![image1](./test_images_output/results/solidWhiteCurve.jpg "solidWhiteCurve")
![image2](./test_images_output/results/solidWhiteRight.jpg "solidWhiteRight")
![image3](./test_images_output/results/solidYellowCurve.jpg "solidYellowCurve")
![image4](./test_images_output/results/solidYellowCurve2.jpg "solidYellowCurve2")
![image5](./test_images_output/results/solidYellowLeft.jpg "solidYellowLeft")
![image6](./test_images_output/results/whiteCarLaneSwitch.jpg "whiteCarLaneSwitch")


### 2.2 Videos
My pipeline behaved well on the first two videos. However it went into failure for the last one.

I had to add in my code a management of the cases where no negative slope segments or no positive slope segments are found.
The code did not break anymore then but result is really poor!

## 3. Potential shortcomings of my current pipeline

1)  Region of interest is fixed, then it may not work with a different size of the image and orientation of the camera
--> this is put in evidence on the challenging video, where the region of interest is smaller in the vertical dimension. 

2) Canny edge threshold parameters are fixed, whereas it should self-adapt to the light conditions. Indeed gradient values will be smaller when image is darker. I guess it will be even worse within rainy or foggy conditions!

3) Algorithm is not robust to curved lanes as we are making the assumption that we are looking for two straight lines

4) Algorithm is not robust to the cases where there are more than two lines in the region of interest. If there are three lines (which may happen when the car changes lane or when there are road works with different line color markings), then the algorithm will potentially average two lines (which present the same slope sign) and we will get one line in the middle of the two. There may also be "false" lines appearing, as for example in the challenging video where the wall edges are detected and corrupt the true left line.



## 4. Propositions of improvements to your pipeline

I see the following ways to improve my pipeline:

1) Adapt in real time the region of interest: may be done by selection a region of the image which has the colour of the road

2) Determine average lighting of the image and set canny edge thresholds in accordance

3) To cope with curved lanes (and furthermore to improve straight line detection), I think it would be better to perform a polynomial fitting (second order?) on the detected segments, instead of performing a simple averaging

4) Not only should we consider the slope of the segments but also the location. Indeed my pipeline gathers segments by slope (negative, positive). It should also classify the segments using the y-intercept. It would then allow to distinguish between parallel lines. 
We could also add a detection of the color of the lines, to assess the presence of road works or to remove false lines.






```python

```
