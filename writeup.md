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

[image1]: ./camera_cal/calibration01.jpg "Calibration Image"
[image2]: ./output_images/calibration/test_undist.jpg "Undistorted"
[image3]: ./test_images/test1.jpg "Test Image"
[image4]: ./output_images/pipeline/undist_test_1.jpg "Undistorted Test Image"
[image5]: ./output_images/pipeline/warp_area.png
[image6]: ./output_images/pipeline/pipeline.png "Pipeline"
[image7]: ./output_images/pipeline/sliding_windows.png "Sliding Windows"
[image8]: ./output_images/pipeline/polynomials.png "Fitted Polynomials"
[image9]: ./output_images/pipeline/detected_lane.png "Detected Lane"
[image10]: ./output_images/pipeline/warped_back.png "Lane Overlay"
[image11]: ./result.gif
[video1]: ./project_result.mp4 "Video"

## [Rubric](https://review.udacity.com/#!/rubrics/571/view) Points

### Here I will consider the rubric points individually and describe how I addressed each point in my implementation.  

---

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

The code for this step is contained in the first section "Step 1: Camera Calibration" of the IPython notebook.  

I start by preparing lists for "object points" and "image points". Then, I get all files from the "camera_cal" folder. Because the first try showed that not in all images the chessboard pattern was detected, I decided to loop over a variety of patterns. This means that patterns from 6x5 to 9x6 were covered. In the end this resulted in 22 images with chessboard detection and covered 19 out of 20 calibration images. The first step for every pattern was to prepate the "object points", which will be the (x, y, z) coordinates of the chessboard corners in the world. Here I am assuming the chessboard is fixed on the (x, y) plane at z=0.  Thus, `objp` is just a replicated array of coordinates, and `objpoints` will be appended with a copy of it every time I successfully detect all chessboard corners in a test image.  `imgpoints` will be appended with the (x, y) pixel position of each of the corners in the image plane with each successful chessboard detection.  

I then used the output `objpoints` and `imgpoints` to compute the camera calibration and distortion coefficients using the `cv2.calibrateCamera()` function.  I applied this distortion correction to the test image using the `cv2.undistort()` function and obtained this result: 

### Original Image
![alt text][image1]
### Undistorted Image
![alt text][image2]

### Pipeline (single images)

#### 1. Provide an example of a distortion-corrected image.

The distortion coefficients and the matrix extracted from the calibration images in the previous step are used to undistort the following image. 

### Original
![alt text][image3]
### Undistorted
![alt text][image4]

#### 2. Describe how (and identify where in your code) you used color transforms, gradients or other methods to create a thresholded binary image.  Provide an example of a binary image result.

First, I used a combination of color and gradient thresholds to generate a binary image. This was used to filter for yellow and white color. The color filtering was done in the HLS color space. The HLS color space is better suited for this task than the RGB color space because it gives the oportunity to filter the color type by hue and set a range for lightness and saturation. For the gradient a Sobel filter was used to get gradients in x-direction as lane lines are rather vertical from a cars perspective. This part is implemented in the second code cell of section "Step 2: Pipeline".
Especially the sobel part was leading to difficulties in some of the videos. This is why it was replaced with two single channel filters to support the color detection. Therefore, a lightness filter was added that filters values between 225 and 255 in the L channel of a HLS image and a filter for the b channel in a Lab images was added that filters values between 155 and 200.

#### 3. Describe how (and identify where in your code) you performed a perspective transform and provide an example of a transformed image.

The code for my perspective transform is also located in the second code cell of the section "Step 2: Pipeline". Besides the mentioned filter functions it includes a function called `transform_bird_perspective()`.  The `transform_bird_perspective()` function takes as inputs an image (`img`), as well as a boolean (`reverse`) that controls the direction of the transformation. After a little experimentation I chose to hardcode the source and destination points in the following manner:

```python
# Source points of the image for persepctive transform
src = np.float32([[708,460], [1055,680], [250,680], [577,460]])
# Destination points the source points should be mapped to
dst = np.float32([[830,0], [830,720], [450,720], [450,0]])
```
This covers the following are in test images with straight lane lines:
![alt text][image5]

I verified that my perspective transform was working as expected by warping the test images and checking that the lines appear almost vertical and parallel to each other.

![alt text][image6]

#### 4. Describe how (and identify where in your code) you identified lane-line pixels and fit their positions with a polynomial?

In the 14th cell of the notebook I defined the functions to get the start positions of the lane lines (`find_lane_start_positions()`), to perform slidign windows and collect all lane line pixels (`find_lane_from_start()`), to fit a polynomial to lane pixels (`fit_polynomial()`), and to find lane lines from previous detected polynomials (`find_lane_from_polynomial()`).
The first step is to find the starting positions, which is done by taking the binary image which was the output of color and gradient filtering and is already transformed to bird perspective. For the lower half of this image a histogram is calculated and peaks are assumed to be the starting points of the lane lines. The discoverd starting positions are the input for the sliding windows approach. Here the image is sliced into 9 pieces starting from the bottom. Then a mask is put at the start position of the lane and covers a range left and right of the start position with a margin of 100 pixels. If this rectangle covers more than 50 pixels the mask is recentered for the next slice. All the covered pixels are appended to a list which in the end contains all pixels of the lane line. This list of pixels is then used to fit a second order polynomial to the lane line. If a polynomial was already detected it is not necesary to perform sliding windows and instead the previous polynomial is used to search within a certain area around the previous polynomial to find lane line pixels. 

### Sliding Windows
![alt text][image7]
### Fitted Polynomial
![alt text][image8]
### Detected Lane
![alt text][image9]

#### 5. Describe how (and identify where in your code) you calculated the radius of curvature of the lane and the position of the vehicle with respect to center.

In the same code cell as above there is a function implemented to calculate the curvature of a polynomial (`compute_curvature()`). The function makes use of the mathematical formula to calculate the curvature radius and uses the first and second derivative of the second order polynomial. The same code snipped is implemented later as a method within the `Line` class. Here the pixel values are converted to meter scale before calculating the radius.

#### 6. Provide an example image of your result plotted back down onto the road such that the lane area is identified clearly.

After the lane line detection and the calculation of the polynomial for each line, the two polynomials are used to create a polygon and fill it. This covers the detected lane. To visualize it in the original image it has to be transformed back to the original perspective. The result can be seen in the image below:

![alt text][image10]

---

### Pipeline (video)

#### 1. Provide a link to your final video output.  Your pipeline should perform reasonably well on the entire project video (wobbly lines are ok but no catastrophic failures that would cause the car to drive off the road!).

Here's a [link to my video result](./project_result.mp4)

![alt text][image11]

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

The project went well for the standard video. The reason for this is that lighting conditions are good and distracting shadows are rare. The challenge video made problems with the high contrast of the different road materials in the center of the lane and the high contrast shadow at the left side of the road. This was mainly solved by increasind the number of past polynomials that are averaged and by adding a mechanism that denies lanes that are too close together. However, for the hard challenge video the pipeline was not good enough. The lighting conditions are tough with the sun directly shining into the front window which leads to reflections in the windshield. Because of the changes between shadow and light caused by the trees, the images are overexposed for a short time until the camera adapts. Additionally, the lane lines at the sides are covered by leaves and the curves are very tight. This causes problems with the region of interest the pipeline uses, which is focused more small changes in curvature and long distances. This might be resolved with changes to make the pipeline more resistant to changes in lighting and to adapt the region of interest according to previously detected lanes or by weighting close areas higher than ares farther away. 
