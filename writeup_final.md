#**Advanced Lane Finding Project**

---

**Advanced Lane Finding Project**

The goals / steps of this project are the following:

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[image1]: ./examples/undist_chess.png "Undistorted"
[image2]: ./examples/color_grad_output.png "Color and Threshold"
[image3]: ./examples/camera_cal_corner.png "Camera Calibration"
[image4]: ./examples/warp_output.png "Warp Example"
[image5]: ./examples/color_fit_lines.png "Fit Visual"
[image6]: ./examples/example_output1.png "Output"
[image7]: ./examples/example_output2.png "Output"
[image8]: ./examples/example_output3.png "Output"
[video1]: ./test_videos_output/project_video.mp4 "Video"
[video2]: ./test_videos_output/challenge_video.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---


### Camera Calibration

#### 1. Computing the camera matrix and distortion coefficients.

The code for this step is contained in the Camera Calibration cell of the Jupyter notebook located in "./Advanced Lane Finding.ipynb". 

I start by preparing `object points`, which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0, such that the object points are the same for each calibration image.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  
Example of detected corner image shown below.

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  

![alt text][image3]

### Pipeline (single images)

Below are the pipeline steps taken towards finding lanes in a single image.

#### Step 1. Distortion Correction

Image distortion occurs when a camera looks at 3D objects in the real world and transforms them into a 2D image; this transformation isn’t perfect. Distortion actually changes what the shape and size of these 3D objects appear to be. 

To correct for distortion, we use the computed camera distortion matrix using the provided chessboard images to undistort images.

Below is an example an image after distortion correction has been applied.

![alt text][image1]

#### Step 2. Perspective transform

I then use the perspective transform to obtain a bird’s eye view of the road. 
I first identified 4 points in the original camera image, and then stretched the image such that the region between the 4 points forms a rectangular section.

The code for my perspective transform includes a function called `warp_image()`, which appears in the 5th code cell of the IPython notebook.   

I chose to hardcode the source and destination points in the following manner:

```python
ht = 480
src = np.float32([[548, ht], 
				  [740, ht], 
				  [image_size[0]-150, image_size[1]],
				  [190, image_size[1]]])
dest = np.float32([[200,0],
				   [1130,0],
				   [1130,720],
				   [200,720]])
```

This resulted in the following source and destination points:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 548, 480      | 200, 0        | 
| 740, 480      | 1130, 0       |
| 1130, 720     | 1130, 720     |
| 190, 720      | 200, 720      |

I verified that my perspective transform was working as expected by drawing the `src` and `dst` points onto a test image and its warped counterpart to verify that the lines appear parallel in the warped image.

![alt text][image4]

#### Step 3. Thresholded Binary Image using Color transforms and gradients

I used a combination of color and gradient thresholds to generate a binary thresholded image (thresholding steps are in the cell 4 in the iPython notebook).

I first detect yellow lines and white lines seperately using HSV colorspace as it provides good color discrimination.

Next I apply Sobel Operator along X to detect edges or lines. 
For this step I converted the image to HLS (based on experimentation) and apply the sobel operator on the L and S channel seperately. 

I then combined the binary masks from Sobel filters and color masks to get the final binary thresholded image.
The thresholds were tuned using the interact package.


Here's an example of my output for this step. 

![alt text][image2]


#### Step 4. Detect Lanes

I have two methods for detecting lanes. 
* First one calculates from scratch
* The second method relies on the line fits of the previous frame

For individual images I always detect from scratch. 
I first take a histogram of the bottom half of the image and find the peak of the left and right halves of the histogram.
These will be the starting point for the left and right lines.
Assumption is that the bottom half will have some lane pixels in the thresholded image whose peaks will be good starting points.

I then use the sliding window strategy to identify all the left and right lane pixels. 
Using these points I fit my lane lines with a 2nd order polynomial.

![alt text][image5]

#### Step 5. Radius of curvature and lane deviation with respect to center

The radius of curvature is calculated assuming the camera is attached to the center of the car.
We convert the x and y pixels of the lanes into real world space and then calculate the curvature in meters.
We assume the lane is about 30 meters long and 3.7 meters wide to convert to real world space. 
The method `calculat_radius_of_curvature` calculates the curvature.

Once the curvature is calculated, we then compute the car's offset with respect to the center. 
To do that we first calculate the x intercepts of the left and right at the bottom of the image.
Then measure how much away it is from the center of the image. The display text on the screen turns red if the deviation exceeds 32 cm.


#### Step 6. Plot the lane on the Input Image

The lanes are plotted on the image in the pipeline method `lane_detect_pipeline`.

Below are examples of lane detection plotted.

![alt text][image6]
![alt text][image7]
![alt text][image8]

---

### Pipeline (video)

For the video pipeline, I introduce an additional Lane Sanity to combat difficult frames. Also decaying mean is performed in this step.

#### Step 4a. Lane Sanity Check 

During videos, we encounter difficult frames where we may detect either one or both lanes inaccurately. 
In order to track them, I check the mean sqaured error between left lane current and previous coefficients.
Similar check is performed for the right lane. If the error is greater than 0.0005 then I use the previous frame's lane fits.

I also keep track of the number of consecutive frames that failed lane detection. If this number goes more than 12 then I perform lane detection from scratch for the following frames.
 
Also I introduce a decaying mean even if the lanes pass the sanity check. I always bias the current fit with 35% of previous frame's fit.
This is one way to introduce smoothing between frames.

The detected left lane or right lane turns red if the detected lane fails the sanity check. 



Here is the [link to my video result](./test_videos_output/project_video.mp4)

Here is the [link to the challenge video result](./test_videos_output/challenge_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Getting the right source and destination points for warping was little tricky. I was initially trying to draw a trapezoid covering the entire lane. 
Figured out later that we neednt cover the entire lane. After a few trials was able to get the warp working.
The lanes still look a bit blurry at the top. Some more tweaking might help.

I also choose to do warp first after distrotion correction before color thresholding.
This helped me in tuning the thresholds as I am looking at only the warped area.

For color thresholding I started with the approach suggested in the course where we use S channel for color thresholding and calculating directional gradient on grey scale image. 
I noticed that it doesnt perform very well near the bridges.
I then experimented with HSV, HLS and RGB color spaces for seperating the lanes. It was hard to find a single threshold to have a good seperation for both yellow and white lanes.
So I detected them seperately and then combined them. HSV colorspace was used for this step.

I also used Sobel operator to eliminate all the horizontal lines detected. I converted the image to HLS and found that the L channel was very effective for this.
I also added S channel later and combined them.
 
Both the color mask and the gradient was combined to create the final binary thresholded image. 
The interact package prooved very useful in tuning the thresholds. 

I first experimented the pipeline on the test images and made sure they detected the lanes perfectly.
The test images also contains some examples of bridges where the intensity changes drastically.

Then I tried the pipeline on the first video by performing lane detection from scratch on all frames.
Then added the lane detection from previous frame info method.
There was lot of jitter from frame to frame in this video.
I then added a decaying mean to smoothen and also to make sure there is no drastic variation from frame to frame.

I then tried tried on the challenge video and found that I was detecting lanes outside the actual lane.
I also added the diagnostic view of the pipeline in the video which many people had cited in the forums. 
This really helped me understand the problem I was facing. 
There were multiple frames where the color gradient thresholding flagged additional pixels and this caused the lane to deviate.

I then added a sanity check for the lanes individually and use the previous frame values if the lane failed the sanity check.
I compute the mean squared error between the second degree coefficients of the current and previous frame.

With these changes, the challenge video worked.

I then tried with the harder challenge. It failed badly.
With diag mode, I noticed the color thresholding failed miserably when the overall intensity of the frame was very high.
Also in harder challenge video, there are many instances where there are no right markings.

I would try the following to make my pipeline more rigid:
* Histogram equalization
* LAB color scpace
* For lane detection, currently we compute the histogram for the bottom half of the image. This may not work on very curvy roads. Explore some other methods.






