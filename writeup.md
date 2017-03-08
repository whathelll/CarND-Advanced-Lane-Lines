##Writeup Template
###You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/undistort_output.png "Undistorted"
[image2]: ./examples/undistort_road.png "Road Transformed"
[lane-filtered-yes]: ./examples/lane_filter_yes.png "Lane Filtered"
[lane-filtered-no]: ./examples/lane_filter_no.png "Original Image"
[warp_original]: ./examples/warp_original_reverted.png "Original Image"
[warp_warped]: ./examples/warp_warped.png "Warped Image"
[lane-fitting]: ./examples/lane_fitting.png "Lane Fitting"
[lane-fitting2]: ./examples/lane_fitting2.png "Colour Fit"
[final]: ./examples/final.png "Final Output"
[video1]: ./output_images/project_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points
###Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---
###Writeup / README

####1. Provide a Writeup / README that includes all the rubric points and how you addressed each one.  You can submit your writeup as markdown or pdf.  [Here](https://github.com/udacity/CarND-Advanced-Lane-Lines/blob/master/writeup_template.md) is a template writeup for this project you can use as a guide and a starting point.  

You're reading it!
###Camera Calibration

####1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the 2 to 4 cell of the IPython notebook located in "./project-notebook.ipynb"

I start by preparing "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result:

![alt text][image1]

I've since changed these cells to markdown format as I don't need to run them again.

###Pipeline (single images)

####1. Provide an example of a distortion-corrected image.
To demonstrate this step, I will describe how I apply the distortion correction to one of the test images like this one:
![alt text][image2]
####2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

I played around with both applying the Sobel operator in both the X (horizontal) and Y (vertical) dimensions to detect gradients. I also looked at applying a threshold to the magnitude of the combined gradients as well as doing a threshold filter on the direction of these gradients. All of the Sobel operators were executed on Lightness channel of the HLS color channels.

I also played around with the thresholds on the L & S channel of the HLS color channels.   

I came to settle on a combination of x sobel gradient filter and a L & S channel filter.  

```
x_grad_binary = abs_sobel_thresh(image, orient='x', thresh=(20, 100))
hsl_channel_binary = hsl_channel_threshold(image, s_thresh=(100, 255), h_thresh=(0, 100))
```

This can be see in the notebook code cell 10 where I define the 'filter_lanes' function, with threshold functions above it.  Here's an example of my output for this step.  (note: this is not actually from one of the test images)

![alt text][lane-filtered-yes]
![alt text][lane-filtered-no]

####3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is in the notebook code cell 12 where I define the function 'warp_image'. The function can do a warp and it can reverse the warp by passing mode='inverse'.

```
warp_image(image, mode='normal')
warp_image(image, mode='inverse')
```

The src and dst points are hardcoded. Upon having issues with my lanes going off screen, I decided to do my perspective transform into a bigger image 1920 wide instead of the original 1280, after shifting everything to the right by a few hundred pixels I have a modified image with the lanes in view. This could have other impacts I'm not aware of but for now it seems to suit my needs.
```
warp_vertices_src = [(195, 720), (593, 450), (690, 450), (1120, 720)]
src = np.float32(warp_vertices_src)
dst = np.float32([[495, 720], [495, 0], [1420, 0], [1420, 720]])

if mode=='normal':
    M = cv2.getPerspectiveTransform(src, dst)
    warped = cv2.warpPerspective(image, M, (1920, 720), flags=cv2.INTER_LINEAR)
elif mode=='inverse':
    M = cv2.getPerspectiveTransform(dst, src)
    warped = cv2.warpPerspective(image, M, (1280, 720), flags=cv2.INTER_LINEAR)

```

This resulted in the following source and destination points:

| Source        | Destination   |
|:-------------:|:-------------:|
| 195, 720      | 495, 720      |
| 593, 450      | 495, 0        |
| 690, 450      | 1420, 0       |
| 1120, 720     | 1420, 720     |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.


![alt text][warp_original]
![alt text][warp_warped]


####4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

I applied the 2nd method in the lectures. Details can be found in the 'find_lane_pixels' function. I adjusted the margin width from 100 to 150 and changed the initial search from 25% to 40% of bottom of the image. I found the convolution approach interesting and sought to understand it.

![alt text][lane-fitting]

Once I've got the lanes detected I fitted my lines with a 2nd order polynomial

![alt text][lane-fitting2]


####5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

Radius of curvature was calculated in my 'pipeline' function in the notebook.

####6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

My 'pipeline' function in the notebook combines all the other functions together to transform an original image to one such as below:

![alt text][final]

---

###Pipeline (video)

####1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_video.mp4)

---

###Discussion

####1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

It was not easy trying to find the right filter combination between all the different Sobel filters and the HLS filters. It took a while for me to settle on something that worked well on the 7 test images. In the end I don't think it worked that well on the challenge videos.

The next challenge was trying to find the right locations to warp the image. After trial and error I settled on a set of points, however later on I discovered that during sharp turns my lanes disappear into the side of my warped image, after some thought I ended up doing a warp onto a wider image of 1920 pixels wide instead.

The following challenge was trying to understand the sample codes provided in the lessons on finding and colouring in the lane lines. This was easy to slot in, however I had to come back and tweak this a few times, increased the initial lane centre searches to be bottom 40% of the screen and also increased my window margin to 150 to cater for the bigger warped picture.

Taking the lane lines and fitting a 2nd order polynomial line to it was fairly straight forward. As was overlaying the lane line drawings back onto the original image.

The time consuming part was tuning the pipeline after I had lined up all the pieces. I used a generator to help save every frame to a file so I can troubleshoot problematic images.   

**Possible Improvements**
- This pipeline was all done in Jupyter Notebook, it can be improved into a class encapsulating all the logic to build a better pipeline
- This pipeline also doesn't keep history, if we do the first improvement then we can also start keeping a memory of past lane lines and make decisions to whether to throw away certain frames. I feel like this should the last resort tuning.
- Given more time I can tune my filters a bit more to provide more robust detection as the pipeline fails pretty bad on the challenge video.
