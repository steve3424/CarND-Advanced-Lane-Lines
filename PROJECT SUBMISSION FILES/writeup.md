## Writeup Template

### You can use this file as a template for your writeup if you want to submit it as a markdown file, but feel free to use some other method and submit a pdf if you prefer.

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


### Camera Calibration

#### 1. Object points and image points
The first step in calibrating a camera is to create the object points and image points. I did this in the third cell of the project notebook. Using a list of 20 calibration images and the function cv2.findChessboardCorners() I was able to create a list of all the found corners paired with the object points to be used in the next calibration step. Here is one example of a calibration image with the corners drawn on it:
![Alt text](images/drawn_corners.jpg?raw=True)

#### 2. Camera matrix and distortion coefficients
The next cell contains the function used for finding the camera matrix and distortion coefficients:
![Alt text](images/camera_matrix.png?raw=True)

#### 3. Undistorting images
The final step is to actually use the calibration on each image in the pipeline. This was handled using a simple function that takes in the camera matrix and the distortion coefficients from the previous step and applies them to any input image:
![Alt text](images/UNDISTORT.png?raw=True)


### Pipeline (single images)

#### 1. Distortion correction
The first step in the image processing pipeline is to correct each image for distortion. Using the previously calculated camera matrix and distortion coefficients I could simply apply these numbers to every image. I simply passed each image from the video through the undistort function along with the mtx and dist values to correct each image:
![Alt text](images/undistort_pipeline.jpg?raw=True)

Here is an example of an image before and after the distortion. It is easy to see the effect on the appearance of the white car on the right of the image:
BEFORE
![Alt text](images/distorted_image.png?raw=True)

AFTER
![Alt text](images/undistorted_image.png?raw=True)

Here is another example of the distortion correction on a calibration image:
BEFORE
![Alt text](images/distorted_cal.png?raw=True)

AFTER
![Alt text](images/undistorted_cal.png?raw=True)

#### 2. Gradient and color thresholding
I experimented with different thresholds and decided that using a combination of a gradient threshold in the X direction as well as a threshold of the S channel is HLS colorspace gave me the best results. The function I used can be found in the "Thresholding functions" section of the notebook just below the camera calibration. The values I used are shown in the pipeline code here:
![Alt text](images/thresholding.png?raw=True)

Here is an example of the results I got with those values:
![Alt text](images/non_binary.png?raw=True)
![Alt text](images/binary_thresh.png?raw=True)

#### 3. Perspective transform
The next step after I created the binary thresholded image was to perform a perspective transform. This would give me a bird's eye view of the lane and make it easier to identify lane lines. I defined a function called warp_perspective() which appears in the section labeled 'Perspective Transform' underneath the thresholding section. This took in a binary image as well as predefined source points and destination points. This outputted the M matrix needed to warp the image. I also outputted the M_inverse matrix for later when I drew the found lines onto the image. I hardcoded the points as follows:
#define source points
tls = [605, 440]
trs = [675, 440]
brs = [1045, 670]
bls = [270, 670]

#define destination points
tld = [300,0]
trd = [1000,0]
brd = [1000,img.shape[0]]
bld = [300,img.shape[0]]

After some experimentation, I settled on the above values. The transform on a normal image looks like this:
BEFORE
![Alt text](images/normal_unwarped.png?raw=True)

AFTER
![Alt text](images/normal_warped.png?raw=True)

Here is the perspective transform with the binary image:
BEFORE
![Alt text](images/binary_unwarped.png?raw=True)

AFTER
![Alt text](images/binary_warped.png?raw=True)

#### 4. Finding lane lines and fitting a polynomial
At this point the image is sufficiently transformed and it is time to identify the lane lines. This is found in the 'Detecting Lane Lines' section of my notebook. The main function is the fit_polynomial function which takes in a binary warped image and outputs the polynomial fit. 

The first step in this process was to find out which pixel indices corrosponded to a lane line. I used a function called find_lane_pixels() which also took in the binary warped image. This created a historgram out of the values of the image in the vertical direction using the hist_peak() function. Then it split the image in half vertically and took the X values with the highest total on both the left and right sides. Since the perspective of the binary image was transformed to place the lines vertically, the max values of this histogram corrosponded to the lane lines.

Then I used the sliding window method in the find_lane_pixels() function to find the appropriate pixels. This method is needed because some of the pixels in the image may not be lane pixels. If they all were then it would be easy enough to just grab all activated pixels. Using 9 windows with a height of 80 pixels and a width of 75, I was able to identify lane pixels appropriately.

Here is an example of my results:
![Alt text](images/sliding_windows.png?raw=True)

I also utilized a function called search_around_poly(). This function was called only if both the left and right lines were previously detected. Instead of using sliding windows with a histogram, this loaded the previous polynomial fit and searched in an area around that. Since lane lines basically stay the same through multiple frames this worked out very well as a heuristic for where to search for lane lines.

Here is a visualization of the search_around_poly() function:
![Alt text](images/search_around_poly.png?raw=True)

#### 5. Curvature and offset
Once I had confident measurements of the lane lines I could measure the curvature at the base. This required first calculating the polynomial fit in meters. I did this in the fit_polynomial() function discussed in the previous section.
![Alt text](images/polynomial_meters.png?raw=True)

By using the conversion from pixels to meters I could define and save a polynomial adjusted to real world measurements. I then used these polynomial coefficients in the equation curve_radius = ((1 + (2Ay + B)^2)^(3/2)) / absolute(2A) along with the pixel to meters conversion shown in the above image to calculate the curve radius of each line. I then averaged these 2 values in my pipeline function to get my final value.
![Alt text](images/curvature.png?raw=True)

Finally I calculated the offset. I calculated the base X position by evaluating each polynomial fit at the bottom of the image. This gave me a reasonable estimate of where the lane started. I then subtracted the right base value from the left to get the width of the lane and divided in half to get the midpoint of the lane. Assuming the camera was in the center of the vehicle I could assume the center of the image was the location of the middle of the car. This was calculated by image.shape[1] / 2. I could then find the distance between this value and the lane center value and convert from pixels to meters to find the offset of the vehicle in meters.
![Alt text](images/offset.png?raw=True)


#### 6. Drawing the lines
Finally I could draw the lines I found onto the images. I used the function called draw_lines() which took in the binary_warped image used for finding the lines as well as the polynomial fit and the inverse perspective matrix used to reverse the perspective transform. This was able to draw lines onto the warped image, reverse the warp to the original perspective, then add the drawn polynomial lines to the original image. 
![Alt text](images/final_output.png?raw=True)

---

### Pipeline (video)

#### 1. FINAL OUTPUT VIDEO
My output video can be found in the PROJECT SUBMISSION FOLDER in the git hub account link

---

### Discussion

#### 1. Briefly discuss any problems / issues you faced in your implementation of this project.  Where will your pipeline likely fail?  What could you do to make it more robust?

My pipeline did reasonably well as it was on the project video, although it had one very noticable issue on the area where the pavement changed. The right lane line started to stray toward the center of the lane considerably. I added a patch of code that fixed this issue, but could potentially cause other issues.
![Alt text](images/lane_fix.png?raw=True)

I eliminated any fit that shifted the right line too far over and reverted back to the previous fit. This is a temporary fix and not an ideal solution that allows the pipeline to do well in the project video, but it could easily fail especially when switching lanes. 

Also my perspective transform could certainly be better chosen as it wasn't exactly centered over the lane on every part of the video. 

My gradient choices could most likely be more robust and allow for detection in many more conditions.

I would also like to add some features such as averaging over a certain number of frames as well as create a robust sanity check to throw out any fit that is obviously bad.
