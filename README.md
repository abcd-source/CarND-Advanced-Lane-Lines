## Advanced Lane Finding

---

**Goal of the Project**

The high level goal of the project is to write a pipeline which consumes an image or a set of images, applies some processing and determines the road curvature and vehicle position within the lane, and plots this information on top of the original image. This is accomplished via the following steps outlined below: 

* Compute the camera calibration matrix and distortion coefficients given a set of chessboard images.
* Apply a distortion correction to raw images.
* Use color transforms, gradients, etc., to create a thresholded binary image.
* Apply a perspective transform to rectify binary image ("birds-eye view").
* Detect lane pixels and fit to find the lane boundary.
* Determine the curvature of the lane and vehicle position with respect to center.
* Warp the detected lane boundaries back onto the original image.
* Output visual display of the lane boundaries and numerical estimation of lane curvature and vehicle position.

[//]: # (Image References)

[bev_lanes]: ./writeup_images/bev_lanes.jpg "Warped Lanes"
[calibrated_grid]: ./writeup_images/calibrated_grid.png "Calibrated Grid"
[raw_grid]: ./writeup_images/raw_grid.png "Raw Grid"
[undistorted_test1]: ./writeup_images/undistorted_test1.jpg "Undistorted Test Image"
[thresholded_binary]: ./writeup_images/thresholded_binary.jpg "Binary Example"
[histogram]: ./writeup_images/histogram.png "Activated Pixel Histogram"
[lane_polynomial]: ./writeup_images/lane_polynomial.jpg "Detected Lane Polynomial"
[processed_image]: ./writeup_images/processed_image.jpg "Processed Image"
[preprocessed_img]: ./writeup_images/preprocessed_img.jpg "Preprocessed Image"
[video]: ./writeup_images/project_video_output.mp4 "Video"

### Camera Calibration

#### 1. Briefly state how you computed the camera matrix and distortion coefficients. Provide an example of a distortion corrected calibration image.

I created a `class Camera():` to perform the calibration step on the input images extracted from the project videos. To use this class, the `calibrate()` method must be called by passing in a directory reference to a set of calibration images. 

In the `calibrate()` method, I read a set of calibration images, converting each image to greyscale, and then extracting the corners of the chess board using OpenCV's built in `findChessboardCorners` method. 

Note: I assume the chessboard is fixed on the (x,y) plane so the the object points are the same for each calibration image.

For each calibration image I append the corners and object points (assuming all nx,ny corners are actually detected). 

Once I have all of the imgpoints and objpoints, I pass them to OpenCV's built in `calibrateCamera` method to find the calibration coefficients and save them as local members of the class. 

In order to use the Camera() class, you simply pass a raw image or a file name to the `undistort` method which will return the undistored image (using the distortion correction coefficients calculated in `calibrate`).

Raw chessboard image:

![alt text][raw_grid] 

Corrected image after running the `undistort` function on the image. 

![alt text][calibrated_grid] 

### Pipeline (single images)

For processing single images I implemented a specific class `LaneDetector` which has a number of helper functions and a primary entry point `process` which takes as an argument a raw image, runs a series of operations to threshold, transform, find lane lines in a Bird's Eye View space, determine road curvature and vehicle position, plot those lane lines on the orignal image, and then return the final image.  

#### 1. Distortion Correction

The first step in the single image pipeline is to undistort a sample image using the `Camera` class. An example of an undistorted test image (example/test1.jpg) is shown below: 

![alt text][undistorted_test1]

#### 2. Thresholding

To threshold the raw images, I created the `SobelFilter` class which uses a combination of color and gradient thresholds to generate a binary image.

The specific sequence for thresholding is defined in the `process` method of the SobelFilter which takes as an input a raw image. The method performs the following: 
1) Finds the Sobel gradients in x, and y (using `absolute_threshold()`)
2) Finds the absolute magnitude of the Sobel gradient (using `magnitude_threshold()`)
3) Finds the direction Sobel gradient (using `direction_threshold()`)
4) Finds the saturation threshold filtered image (using `hls_select()`) 

After finding each threshold, a combined image is formed which is the logic OR of if a pixel is activated in the x AND y threshold, OR activated in the magnitude AND direction threshold OR activated in the saturation thresholding. The final result is a binary thresholded image, where the pixels will be either 0, or self.max (defined in the init() function of the Sobel class) 

Here's an example of my output for this step:

![alt text][thresholded_binary]

#### 3. Perspective Transform 
Once the images are thresholded, they are passed to the `transform_to_bev()` method in the `LaneDetector` class which takes as an argument a raw image and performs a perspective transform to warp the image from 2D space to a "Bird's Eye View" space. This method has hard coded `src` and `dst` points for the transform.  

Note: the perspective transform matrix is only calculated once, and used on all subsequent images, once the LaneDetector `self.M_BEV != None`

I used the following src and dst points. 

##### Source:
[[  585.           460.        ]
 [  203.3   720.        ]
 [ 1126.6   720.        ]
 [  695.           460.        ]]

##### Destination:
[[ 320.    0.]
 [ 320.  720.]
 [ 960.  720.]
 [ 960.    0.]]
 
 Which results in a transform matrix `self.M_BEV` defined as:
 
[[ -4.81574401e-01  -1.46015826e+00   9.26907230e+02]
 [ -2.01810962e-16  -1.92398227e+00   8.85031843e+02]
 [ -3.16767536e-19  -2.35384913e-03   1.00000000e+00]]
 
The following images shows the results of a thresholded and transformed image:

![alt text][bev_lanes]

#### 4. Identifying the lane-line pixels and finding a polynomial. 

After the image has been thresholded and transformed, I find the lane line pixels and fit a 2nd order polynomial to those pixels in the function `detect_lanes()`

The first step is take a histogram of activated pixels across the bottom half of the image. This is done using the NumPy method `sum()` across the bottom half of the input image along the x-axis. 

An example of the histogram plotted on top of the original image is shown below, where the green line represents the value of sum() pixels on top of the thresholded image.  

![alt text][histogram]

The histogram  is then broken into a left and right half, and the peak of each half is used to determine the starting point of the lane line, these are defined as internal variables `leftx_base` and `rightx_base`

From the base points along the bottom of the image, `nwindows` are created along the y-axis to track the peak pixel values in each window. The center of the window are updated to track the peaks when there are more than minpix activated pixels in a particular window. This moving window function identifies which pixels are to be used in creating the polynomial fit. 

After the relevant pixels are identified, I use the built in function `np.polyfit()` which takes the set of left and the set of right pixels and returns the 3 coefficients for a 2nd order polynomial.  

The coefficients for the left and right lane line are passed to instances of a low pass filter class `LaneLine` through the method `update_coeff()` which takes the new measurements as an argument and updates the LaneLine's internal filtered coefficients by applying an Alpha/Beta filter.   

Note that in the case of processing single images which are not temporarily linked, I must call `reset()` on the left and right lanes between images such that the filter is reset.

Once the polynomial coeffcients are identified, they can also be plotted on a sample image. 

An example of a processed image, showing the positions of the moving windows (green) the activated left lane line pixels (blue) the activated right lane line pixels (red), and the left and right polynomials (yellow) is below:

![alt text][lane_polynomial]

#### 5. Determining Curvature and Offset

After determining the polynomial coefficients in the previous step through `detect_lanes()` the LaneDetector class has updated values for `self.left_line`, `self.right_line` which can be used to calculate the curvature and offset. 

The function `calculate_curvature()` takes as inputs the y-pixel value to evaluate the curvature, and the midpoint of the image along the x-axis. 

The function calculate the left and right curvature values in meters by applying the defined equation from the class for curvature based on polynomial coefficients. These curvatures are averaged to find the "road curvature" which is stored as class member `self.curvature`

In addition to the curvature, this method determines of the lateral offset of the vehicle within the lane by calculating the pixel difference the center of the lanes and the center of the image. the center of the lanes is found by averaging the base point of the left and right lane as defined in the `self.left_line` and `self.right_line`. 

The values for curvature and offset (both in meters) are stored as class members, so this function does not need to return anything. 

#### 6. Plotting Lane Lines in Original image

The final step in the image processing function `process()` is to plot the found lane lines reprojected into image space on the original image. This step is done in the method `generate_lane_image()` which takes as an argument the original image only, and then uses the class members to generate an overlay of the lane edges and polygon. 

This function creates the final image in several steps: 
1) Create a blank image `out_img` in the same shape as the input image to draw on top. 
2) Determine the image indicies for the pixels which fall between the left and right polynomials.
3) Color in the pixels in `out_img` for the indices determined in (2)
4) Determine the image indicies for the pixels on either side of the left and right polynomial lines
5) Color in the pixels in `out_img` for the indices determined in (4)

An example of this image of the lane fill and the edges in "Bird's Eye View" can be seen here: 
![alt text][preprocessed_img]

This image is then warped back into the image space throug the function `transform_to_image()` which is the inverse of the function which was used to warp images from the camera space to the "Bird's Eye View" space. The inverse is achieved by simply swapping the dst and src points.  

This image is then combined with the input image using the OpenCV built-in function `addWeighted()` with the original image having an Alpha of 1.0 and the overlayed lane a Beta of 0.5 

Finally I use the OpenCV built-in function `putText` to write the lane curvature and offset positon to the top of the image. A final processed image can be seen here:

![alt text][processed_image]

---

### Pipeline (video)

#### 1. Processing Video

The pipeline for processing video is similar to processinga  single image. The function `process_image` is passed as a lamba to the video processing function `fl_image`. 

The `process_image` function copies the raw image frame, passes it to an instance of the `Camera` class `camera` to peform the undistortion, and then passes the undistored image to an instance of the `LaneDetector` class `lane_detector` to generate the final output frame with the lane lines overlayed on the original image. 

Here's a [link to my video result](./writeup_images/project_video_output.mp4)

---

### Discussion

This implementation does not seem very efficient, especially the section which fills in the the polygon defined between the two lane polynomials. I believe this is simply a result of my not using the right functions in OpenCV or numpy to identify the pixels correctly.

But this section of the code takes much too long to run, making the overal video generation slow.  
`
lane_inds = ((x > (left[0]*(y**2) + left[1]*y + left[2])) &
                       (x < (right[0]*(y**2) + right[1]*y + right[2])))
out_img[y[lane_inds], x[lane_inds]] = [0, 255, 255]
`

I did not spend enough time on the thresholding of the image to make the lane line detection as robust as it could be, and instead relied on heavier low pass filtering of the coefficients. This is in part due to the challenges with determining the right coefficients for the thresholding, which feels like a lot of trial and error. It would be better to write another function to perform a search over the parameter space and find the optimal hyper parameters, but it seemed like that would be out of the scope of the project. 

My pipeline is likely to fail where the lane lines are not well identified, or if there are multiple lane lines on an image since my code will not attempt to disambiguate multiple peaks. If there are groupings of bright pixels near the bottom of the image which are not the actual lane lines, my code will still identify them as such since it's only relying on a simple histogram. A more intelligent way of finding the base of the lanes by looking at previous values of the base of lanes would be more robust. 
