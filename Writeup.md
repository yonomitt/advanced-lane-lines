
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

[original_cb]: ./images/checkerboard-00-orig.jpg "Original checkerboard image"
[corners_cb]: ./images/checkerboard-01-corners.jpg "Detected corners of the checkerboard are shown"
[undistort_cb]: ./images/checkerboard-02-undistorted.jpg "The same checkerboard image undistorted"
[orig_lane]: ./images/test2-00-orig.jpg "Original image"
[undist_lane]: ./images/test2-01-undist.jpg "Undistorted image"
[thresh_lane]: ./images/test2-02-thresh.jpg "Thresholded image"
[warp_lane]: ./images/test2-03-warp.jpg "Birds-eye view image"
[detect_lane]: ./images/test2-04-detect.jpg "Detected lane markings image"
[overlay_lane]: ./images/test2-05-overlay.jpg "Overlay image"

---

I have provided an [HTML](./Advanced-Lane-Finding.html) file with all of the cells run, for convenience.

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained under the **1. Camera Calibration** and **2. Undistort** sections of the [IPython notebook](./Advanced-Lane-Finding.ipynb).

My `camera_calibration` function takes in a series of checkerboard images taken at different angles and distances. Each checkerboard is known to have 9 internal corners in the horizontal direction and 6 internal corners in the vertical direction.

Using OpenCV's `findChessboardCorners` function, I then detect the 9x6 internal corners of each checkerboard image and append them to an array (`imgpoints`).

The location of the internal corners of the checkerboards are then fed to OpenCv's `calibrateCamera` function along with their respective relative coordinates (stored in the parallel `objpoints` array) to calculate the camera matrix and the distortion coefficients.

Here is an example:

![alt text][original_cb]
![alt text][corners_cb]
![alt text][undistort_cb]

### Pipeline (single images)

I wrote a function for each step in the pipeline:

1. **camera_calibration** - Calibrate the camera. Only needs to be performed once, so not technically in the pipeline.
2. **undistort** - Undistorts an image based on the camera matrix and distortion parameters
3. **threshold** - Thresholds the image to isolate the lane markings
4. **transform_birdseye** - Warps the image to a birds-eye view
5. **detect_lane_markings** - Detects the lane markings
6. **calc_lane_curvature** - Calculates the lane curvature
7. **calc_lane_center_offset** - Calculates the offset of the car from the center of the lane
8. **transform_lane_to_perspective** - Warps the calculated lane polygon back to the perspective view
9. **overlay_detected_lane** - overlays the lane polygon onto the undistorted version of the original image (from step 2.)
10. **overlay_lane_info** - overlays information about the lane curvature and vehicle offset onto the image

#### 1. Provide an example of a distortion-corrected image.

The first step is to correct the distortion to the image caused by the camera. Taking the camera matrix and distortion coefficients calculated during the camera calibration step, I could use OpenCV's `undistort` function to take care of undistortion.

![alt text][orig_lane]
![alt text][undist_lane]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

My thresholding functions can be found in section *3. Thresholding functions*. I used a combination of color and gradient thresholds to generate a binary image. 

I converted the image to HLS and used the saturation channel in my thresholding function. Additionally, I used the magnitude and direction gradients together.

I tried applied a mask to the thresholded image in an attempt to remove extra noise from the binary image, but found that the results were no better off. More on this in the next section.

Here is the result of my thresholding functions on the undistorted image.

![alt text][undist_lane]
![alt text][thresh_lane]

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform includes a function called `transform_birdseye`, which appears in section *4. Perspective Transformation*. The `transform_birdseye` function takes as inputs an image (`img`), and calls `gen_transform_matrices` to get the hard coded transformation matrices.

I started with the values for the source polygon from the examples in the lessons and tweaked them until I was happy.

This resulting source and destination points are:

| Source        | Destination   | 
|:-------------:|:-------------:| 
| 140, 720      | 140, 720        | 
| 581, 450      | 140, 0      |
| 699, 450     | 1140, 0      |
| 1140, 720      | 1140, 720        |

Here is an example of a thresholded image before and after being warped to birds-eye view

![alt text][thresh_lane]
![alt text][warp_lane]

I think my masking attempts did not help because of the way I performed the birds-eye view warp. In the process of warping the image, a lot of the noise in the thresholded image was automatically cut out of the image.

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

Lane line detection functions can be found under section *5. Detect Lane Markings*. I tried the two different sliding window approaches discussed in the lessons, but found the convolution approach always gave me inferior results on a noisy input.

The functions used to detect the lane pixels are:
 - **detect_lane_markings** - calculates the pixels that belong to the lanes based on the calculated windows
 - **find_windows_0** - calculates sliding windows centered over potential lane markings
 - **fit_curve** - calculates the polynomial based on the pixels determined to be part of the lane

My `detect_lane_markings` function begins by taking a histogram of the bottom half of the warped image. Finding the peaks to the left and the right of the middle gives a starting point for searching for pixels belonging to the lane markers.

The image is then divided horizontally into equal pieces for searching for lane pixels. A window is used to identify target pixels at each height and is centered around the mean of the detected lane pixels from the window below it.

I experimented with different window sizes and numbers, but settled on 9 windows per lane at a width of 200 pixels.

Having kept the coordinates for each detected pixel, I then used NumPy's poly fit to calculate a 2nd degree polynomial to represent the lane line.

In the following picture, I have overlaid the sliding windows in green and the calculated polynomial for the lane in yellow:

![alt text][detect_lane]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

I calculated the radius of the curvature of the lane and the position of the car in the lane in section *6. Calculate Curvature and Position*.

The function **calc_lane_curvature** calculates the radius of the lane curvature by calculating the radius of the polynomial describing the left and right lines. It then averages these together to arrive at the overall radius.

The function **calc_lane_center_offset** calculates the offset of the vehicle with respect to the center of the lane. This is accomplished by taking the width of the lane in pixels at the bottom of the image and seeing how well centered it is in the image. As the camera is centered on the car, the lane should be centered on the image when the car is centered in the lane.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

Having calculated the polynomials for the left and right lane markers, I then use the inverse of the warp matrix to create a polygon representing the lane in the perspective view. This is done in the function `transform_lane_to_perspective`, found in section *7. Warp Lane Markings to Perspective View*.

After the lane polygon is warped, I overlay it on the undistorted version of the original image.

I then overlay the information about the curvature of the lane and the offset of the car on top of this to produce the final image.

![alt text][overlay_lane]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

I then combined all functions together into one single `find_lane` function, which is found in section titled `Pipeline`.

Here's a [link to my video result](./project_video.mp4)

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

Like all of my projects this term, so far, I experimented a great deal with this one. I wish I had more time to experiment. I had a lot of fun discovering how changes could affect the quality of the output.

In fact, as I type this, I'm running one final experiment that may or may not make it into the final submission!

I have a thresholding function that works reasonably well for the project video. Unfortunately, it does not do so well for the two challenge videos. Given more time, I would play around to find out what other forms of thresholding could be incorporated to make it overall more robust.

Additionally, I feel I could improve the lane pixel detection algorithm. I attempted briefly (see `find_windows_1`) to play with the convolution algorithm. I even tried to modify it to see the effect on the output. While I was able to get it working better with my improvements, it never performed better than `find_windows_0` on the challenge videos. Part of the issue here is that the thresholding needs to first be improved.

Part of the issue my pipeline had with the challenge video is the shadow from the median barrier. I was just looking through the pictures, specifically these three:

![Thresholded challenge image][./images/challenge001-02-thresh.jpg]
![Warped challenge image][./images/challenge001-03-warp.jpg]
![Lane detection in challenge image][./images/challenge001-02-detect.jpg]

One can see that the shadow from the median barrier made it into the thresholded image. From the last picture, it is clear that it is causing havoc to my lane detection algorithm.

Three possible solutions:
1. Make a more robust lane pixel detecting algorithm
2. Threshold the image better
3. Find a better mask to leave only lane pixels

Currently, my experiments in masking had a hardcoded mask defined. One thought I just had to improve upon this would be the following algorithm:

1. Threshold as normal
2. Warp to birds-eye view
3. Histogram the lower 1/3 or 1/2 to determine the most likely x positions of the left and right lane at the bottom of the image
4. Convert these x positions back to the perspective view
5. Create a dynamic mask that uses these x positions, with a little buffer, as the edges for the trapezoidal mask
