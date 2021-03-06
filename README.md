# My Video Streaming
 
# required package
pip install imutils
pip install flask
pip install opencv

OpenCV – Stream video to web browser/HTML page
by Adrian Rosebrock on September 2, 2019

Click here to download the source code to this post
In this tutorial you will learn how to use OpenCV to stream video from a webcam to a web browser/HTML page using Flask and Python.

Ever have your car stolen?

Mine was stolen over the weekend. And let me tell you, I’m pissed.

I can’t share too many details as it’s an active criminal investigation, but here’s what I can tell you:

My wife and I moved to Philadelphia, PA from Norwalk, CT about six months ago. I have a car, which I don’t drive often, but still keep just in case of emergencies.

Parking is hard to find in our neighborhood, so I was in need of a parking garage.

I heard about a garage, signed up, and started parking my car there.

Fast forward to this past Sunday.

My wife and I arrive at the parking garage to grab my car. We were about to head down to Maryland to visit my parents and have some blue crab (Maryland is famous for its crabs).

I walked to my car and took off the cover.

I was immediately confused — this isn’t my car.

Where the #$&@ is my car?

After a few short minutes I realized the reality —  my car was stolen.

Over the past week, my work on my upcoming Raspberry Pi for Computer Vision book was interrupted — I’ve been working with the owner of the the parking garage, the Philadelphia Police Department, and the GPS tracking service on my car to figure out what happened.

I can’t publicly go into any details until it’s resolved, but let me tell you, there’s a whole mess of paperwork, police reports, attorney letters, and insurance claims that I’m wading neck-deep through.

I’m hoping that this issue gets resolved in the next month — I hate distractions, especially distractions that take me away from what I love doing the most — teaching computer vision and deep learning.

I’ve managed to use my frustrations to inspire a new security-related computer vision blog post.

In this post, we’ll learn how to stream video to a web browser using Flask and OpenCV.

You will be able to deploy the system on a Raspberry Pi in less than 5 minutes:

Simply install the required packages/software and start the script.
Then open your computer/smartphone browser to navigate to the URL/IP address to watch the video feed (and ensure nothing of yours is stolen).
There’s nothing like a little video evidence to catch thieves.

While I continue to do paperwork with the police, insurance, etc, you can begin to arm yourself with Raspberry Pi cameras to catch bad guys wherever you live and work.

To learn how to use OpenCV and Flask to stream video to a web browser HTML page, just keep reading!


Looking for the source code to this post?
JUMP RIGHT TO THE DOWNLOADS SECTION 
OpenCV – Stream video to web browser/HTML page
In this tutorial we will begin by discussing Flask, a micro web framework for the Python programming language.

We’ll learn the fundamentals of motion detection so that we can apply it to our project. We’ll proceed to implement motion detection by means of a background subtractor.

From there, we will combine Flask with OpenCV, enabling us to:

Access frames from RPi camera module or USB webcam.
Process the frames and apply an arbitrary algorithm (here we’ll be using background subtraction/motion detection, but you could apply image classification, object detection, etc.).
Stream the results to a web page/web browser.
Additionally, the code we’ll be covering will be able to support multiple clients (i.e., more than one person/web browser/tab accessing the stream at once), something the vast majority of examples you will find online cannot handle.

Putting all these pieces together results in a home surveillance system capable of performing motion detection and then streaming the video result to your web browser.

Let’s get started!

The Flask web framework

Figure 1: Flask is a micro web framework for Python (image source).
In this section we’ll briefly discuss the Flask web framework and how to install it on your system.

Flask is a popular micro web framework written in the Python programming language.

Along with Django, Flask is one of the most common web frameworks you’ll see when building web applications using Python.

However, unlike Django, Flask is very lightweight, making it super easy to build basic web applications.

As we’ll see in this section, we’ll only need a small amount of code to facilitate live video streaming with Flask — the rest of the code either involves (1) OpenCV and accessing our video stream or (2) ensuring our code is thread safe and can handle multiple clients.

If you ever need to install Flask on a machine, it’s as simple as the following command:

OpenCV - Stream video to web browser/HTML page
$ pip install flask
While you’re at  it, go ahead and install NumPy, OpenCV, and imutils:

OpenCV - Stream video to web browser/HTML page
$ pip install numpy
$ pip install opencv-contrib-python
$ pip install imutils
Note: If you’d like the full-install of OpenCV including “non-free” (patented) algorithms, be sure to compile OpenCV from source.

Project structure
Before we move on, let’s take a look at our directory structure for the project:

```
OpenCV - Stream video to web browser/HTML page
$ tree --dirsfirst
.
├── pyimagesearch
│   ├── motion_detection
│   │   ├── __init__.py
│   │   └── singlemotiondetector.py
│   └── __init__.py
├── templates
│   └── index.html
└── webstreaming.py

3 directories, 5 files
```

To perform background subtraction and motion detection we’ll be implementing a class named SingleMotionDetector — this class will live inside the singlemotiondetector.py file found in the motion_detection submodule of pyimagesearch.

The webstreaming.py file will use OpenCV to access our web camera, perform motion detection via SingleMotionDetector, and then serve the output frames to our web browser via the Flask web framework.

In order for our web browser to have something to display, we need to populate the contents of index.html with HTML used to serve the video feed. We’ll only need to insert some basic HTML markup — Flask will handle actually sending the video stream to our browser for us.

Implementing a basic motion detector

Figure 2: Video surveillance with Raspberry Pi, OpenCV, Flask and web streaming. By use of background subtraction for motion detection, we have detected motion where I am moving in my chair.
Our motion detector algorithm will detect motion by form of background subtraction.

Most background subtraction algorithms work by:

Accumulating the weighted average of the previous N frames
Taking the current frame and subtracting it from the weighted average of frames
Thresholding the output of the subtraction to highlight the regions with substantial differences in pixel values (“white” for foreground and “black” for background)
Applying basic image processing techniques such as erosions and dilations to remove noise
Utilizing contour detection to extract the regions containing motion
Our motion detection implementation will live inside the SingleMotionDetector class which can be found in singlemotiondetector.py.

We call this a “single motion detector” as the algorithm itself is only interested in finding the single, largest region of motion.

We can easily extend this method to handle multiple regions of motion as well.

Let’s go ahead and implement the motion detector.

Open up the singlemotiondetector.py file and insert the following code:

```
OpenCV - Stream video to web browser/HTML page
# import the necessary packages
import numpy as np
import imutils
import cv2
class SingleMotionDetector:
	def __init__(self, accumWeight=0.5):
		# store the accumulated weight factor
		self.accumWeight = accumWeight
		# initialize the background model
		self.bg = None
Lines 2-4 handle our required imports.
```


All of these are fairly standard, including NumPy for numerical processing, imutils for our convenience functions, and cv2 for our OpenCV bindings.

We then define our SingleMotionDetector class on Line 6. The class accepts an optional argument, accumWeight, which is the factor used to our accumulated weighted average.

The larger accumWeight is, the less the background (bg) will be factored in when accumulating the weighted average.

Conversely, the smaller accumWeight is, the more the background bg will be considered when computing the average.

Setting accumWeight=0.5 weights both the background and foreground evenly — I often recommend this as a starting point value (you can then adjust it based on your own experiments).

Next, let’s define the update method which will accept an input frame and compute the weighted average:

```
OpenCV - Stream video to web browser/HTML page
	def update(self, image):
		# if the background model is None, initialize it
		if self.bg is None:
			self.bg = image.copy().astype("float")
			return
		# update the background model by accumulating the weighted
		# average
		cv2.accumulateWeighted(image, self.bg, self.accumWeight)
		
```

In the case that our bg frame is None (implying that update has never been called), we simply store the bg frame (Lines 15-18).

Otherwise, we compute the weighted average between the input frame, the existing background bg, and our corresponding accumWeight factor.

Given our background bg we can now apply motion detection via the detect method:
```
OpenCV - Stream video to web browser/HTML page
	def detect(self, image, tVal=25):
		# compute the absolute difference between the background model
		# and the image passed in, then threshold the delta image
		delta = cv2.absdiff(self.bg.astype("uint8"), image)
		thresh = cv2.threshold(delta, tVal, 255, cv2.THRESH_BINARY)[1]
		# perform a series of erosions and dilations to remove small
		# blobs
		thresh = cv2.erode(thresh, None, iterations=2)
		thresh = cv2.dilate(thresh, None, iterations=2)
		
```
The detect method requires a single parameter along with an optional one:

image: The input frame/image that motion detection will be applied to.
tVal: The threshold value used to mark a particular pixel as “motion” or not.
Given our input image we compute the absolute difference between the image and the bg (Line 27).

Any pixel locations that have a difference > tVal are set to 255 (white; foreground), otherwise they are set to 0 (black; background) (Line 28).

A series of erosions and dilations are performed to remove noise and small, localized areas of motion that would otherwise be considered false-positives (likely due to reflections or rapid changes in light).

The next step is to apply contour detection to extract any motion regions:
```
OpenCV - Stream video to web browser/HTML page
		# find contours in the thresholded image and initialize the
		# minimum and maximum bounding box regions for motion
		cnts = cv2.findContours(thresh.copy(), cv2.RETR_EXTERNAL,
			cv2.CHAIN_APPROX_SIMPLE)
		cnts = imutils.grab_contours(cnts)
		(minX, minY) = (np.inf, np.inf)
		(maxX, maxY) = (-np.inf, -np.inf)
Lines 37-39 perform contour detection on our thresh image.
```
We then initialize two sets of bookkeeping variables to keep track of the location where any motion is contained (Lines 40 and 41). These variables will form the “bounding box” which will tell us the location of where the motion is taking place.

The final step is to populate these variables (provided motion exists in the frame, of course):
```
OpenCV - Stream video to web browser/HTML page
		# if no contours were found, return None
		if len(cnts) == 0:
			return None
		# otherwise, loop over the contours
		for c in cnts:
			# compute the bounding box of the contour and use it to
			# update the minimum and maximum bounding box regions
			(x, y, w, h) = cv2.boundingRect(c)
			(minX, minY) = (min(minX, x), min(minY, y))
			(maxX, maxY) = (max(maxX, x + w), max(maxY, y + h))
		# otherwise, return a tuple of the thresholded image along
		# with bounding box
		return (thresh, (minX, minY, maxX, maxY))
On Lines 43-45 we make a check to see if our contours list is empty.
```
If that’s the case, then there was no motion found in the frame and we can safely ignore it.

Otherwise, motion does exist in the frame so we need to start looping over the contours (Line 48).

For each contour we compute the bounding box and then update our bookkeeping variables (Lines 47-53), finding the minimum and maximum (x, y)-coordinates that all motion has taken place it.

Finally, we return the bounding box location to the calling function.

Combining OpenCV with Flask

Figure 3: OpenCV and Flask (a Python micro web framework) make the perfect pair for web streaming and video surveillance projects involving the Raspberry Pi and similar hardware.
Let’s go ahead and combine OpenCV with Flask to serve up frames from a video stream (running on a Raspberry Pi) to a web browser.

Open up the webstreaming.py file in your project structure and insert the following code:
```
OpenCV - Stream video to web browser/HTML page
# import the necessary packages
from pyimagesearch.motion_detection import SingleMotionDetector
from imutils.video import VideoStream
from flask import Response
from flask import Flask
from flask import render_template
import threading
import argparse
import datetime
import imutils
import time
import cv2
Lines 2-12 handle our required imports:

Line 2 imports our SingleMotionDetector class which we implemented above.
The VideoStream class (Line 3) will enable us to access our Raspberry Pi camera module or USB webcam.
Lines 4-6 handle importing our required Flask packages — we’ll be using these packages to render our index.html template and serve it up to clients.
Line 7 imports the threading library to ensure we can support concurrency (i.e., multiple clients, web browsers, and tabs at the same time).
Let’s move on to performing a few initializations:

OpenCV - Stream video to web browser/HTML page
# initialize the output frame and a lock used to ensure thread-safe
# exchanges of the output frames (useful when multiple browsers/tabs
# are viewing the stream)
outputFrame = None
lock = threading.Lock()
# initialize a flask object
app = Flask(__name__)
# initialize the video stream and allow the camera sensor to
# warmup
#vs = VideoStream(usePiCamera=1).start()
vs = VideoStream(src=0).start()
time.sleep(2.0)
```
First, we initialize our outputFrame on Line 17 — this will be the frame (post-motion detection) that will be served to the clients.

We then create a lock on Line 18 which will be used to ensure thread-safe behavior when updating the ouputFrame (i.e., ensuring that one thread isn’t trying to read the frame as it is being updated).

Line 21 initialize our Flask app itself while Lines 25-27 access our video stream:

If you are using a USB webcam, you can leave the code as is.
However, if you are using a RPi camera module you should uncomment Line 25 and comment out Line 26.
The next function, index, will render our index.html template and serve up the output video stream:
```
OpenCV - Stream video to web browser/HTML page
@app.route("/")
def index():
	# return the rendered template
	return render_template("index.html")

```
This function is quite simplistic — all it’s doing is calling the Flask render_template on our HTML file.

We’ll be reviewing the index.html file in the next section so we’ll hold off on a further discussion on the file contents until then.

Our next function is responsible for:

Looping over frames from our video stream
Applying motion detection
Drawing any results on the outputFrame
And furthermore, this function must perform all of these operations in a thread safe manner to ensure concurrency is supported.

Let’s take a look at this function now:
```
OpenCV - Stream video to web browser/HTML page
def detect_motion(frameCount):
	# grab global references to the video stream, output frame, and
	# lock variables
	global vs, outputFrame, lock
	# initialize the motion detector and the total number of frames
	# read thus far
	md = SingleMotionDetector(accumWeight=0.1)
	total = 0
```
Our detection_motion function accepts a single argument, frameCount, which is the minimum number of required frames to build our background bg in the SingleMotionDetector class:

If we don’t have at least frameCount frames, we’ll continue to compute the accumulated weighted average.
Once frameCount is reached, we’ll start performing background subtraction.
Line 37 grabs global references to three variables:

vs: Our instantiated VideoStream object
outputFrame: The output frame that will be served to clients
lock: The thread lock that we must obtain before updating outputFrame
Line 41 initializes our SingleMotionDetector class with a value of accumWeight=0.1, implying that the bg value will be weighted higher when computing the weighted average.

Line 42 then initializes the total number of frames read thus far — we’ll need to ensure a sufficient number of frames have been read to build our background model.

From there, we’ll be able to perform background subtraction.

With these initializations complete, we can now start looping over frames from the camera:
```
OpenCV - Stream video to web browser/HTML page
	# loop over frames from the video stream
	while True:
		# read the next frame from the video stream, resize it,
		# convert the frame to grayscale, and blur it
		frame = vs.read()
		frame = imutils.resize(frame, width=400)
		gray = cv2.cvtColor(frame, cv2.COLOR_BGR2GRAY)
		gray = cv2.GaussianBlur(gray, (7, 7), 0)
		# grab the current timestamp and draw it on the frame
		timestamp = datetime.datetime.now()
		cv2.putText(frame, timestamp.strftime(
			"%A %d %B %Y %I:%M:%S%p"), (10, frame.shape[0] - 10),
			cv2.FONT_HERSHEY_SIMPLEX, 0.35, (0, 0, 255), 1)
```
Line 48 reads the next frame from our camera while Lines 49-51 perform preprocessing, including:

Resizing to have a width of 400px (the smaller our input frame is, the less data there is, and thus the faster our algorithms will run).
Converting to grayscale.
Gaussian blurring (to reduce noise).
We then grab the current timestamp and draw it on the frame (Lines 54-57).

With one final check, we can perform motion detection:
```
OpenCV - Stream video to web browser/HTML page
		# if the total number of frames has reached a sufficient
		# number to construct a reasonable background model, then
		# continue to process the frame
		if total > frameCount:
			# detect motion in the image
			motion = md.detect(gray)
			# check to see if motion was found in the frame
			if motion is not None:
				# unpack the tuple and draw the box surrounding the
				# "motion area" on the output frame
				(thresh, (minX, minY, maxX, maxY)) = motion
				cv2.rectangle(frame, (minX, minY), (maxX, maxY),
					(0, 0, 255), 2)
		
		# update the background model and increment the total number
		# of frames read thus far
		md.update(gray)
		total += 1
		# acquire the lock, set the output frame, and release the
		# lock
		with lock:
			outputFrame = frame.copy()
```
On Line 62 we ensure that we have read at least frameCount frames to build our background subtraction model.

If so, we apply the .detect motion of our motion detector, which returns a single variable, motion.

If motion is None, then we know no motion has taken place in the current frame. Otherwise, if motion is not None (Line 67), then we need to draw the bounding box coordinates of the motion region on the frame.

Line 76 updates our motion detection background model while Line 77 increments the total number of frames read from the camera thus far.

Finally, Line 81 acquires the lock required to support thread concurrency while Line 82 sets the outputFrame.

We need to acquire the lock to ensure the outputFrame variable is not accidentally being read by a client while we are trying to update it.

Our next function, generate , is a Python generator used to encode our outputFrame as JPEG data — let’s take a look at it now:
```
OpenCV - Stream video to web browser/HTML page
def generate():
	# grab global references to the output frame and lock variables
	global outputFrame, lock
	# loop over frames from the output stream
	while True:
		# wait until the lock is acquired
		with lock:
			# check if the output frame is available, otherwise skip
			# the iteration of the loop
			if outputFrame is None:
				continue
			# encode the frame in JPEG format
			(flag, encodedImage) = cv2.imencode(".jpg", outputFrame)
			# ensure the frame was successfully encoded
			if not flag:
				continue
		# yield the output frame in the byte format
		yield(b'--frame\r\n' b'Content-Type: image/jpeg\r\n\r\n' + 
			bytearray(encodedImage) + b'\r\n')
```
Line 86 grabs global references to our outputFrame and lock, similar to the detect_motion function.

Then generate starts an infinite loop on Line 89 that will continue until we kill the script.

Inside the loop, we:

First acquire the lock (Line 91).
Ensure the outputFrame is not empty (Line 94), which may happen if a frame is dropped from the camera sensor.
Encode the frame as a JPEG image on Line 98 — JPEG compression is performed here to reduce load on the network and ensure faster transmission of frames.
Check to see if the success flag has failed (Lines 101 and 102), implying that the JPEG compression failed and we should ignore the frame.
Finally, serve the encoded JPEG frame as a byte array that can be consumed by a web browser.
That was quite a lot of work in a short amount of code, so definitely make sure you review this function a few times to ensure you understand how it works.

The next function, video_feed calls our generate function:
```
OpenCV - Stream video to web browser/HTML page
@app.route("/video_feed")
def video_feed():
	# return the response generated along with the specific media
	# type (mime type)
	return Response(generate(),
		mimetype = "multipart/x-mixed-replace; boundary=frame")
```
Notice how this function as the app.route signature, just like the index function above.

The app.route signature tells Flask that this function is a URL endpoint and that data is being served from http://your_ip_address/video_feed.

The output of video_feed is the live motion detection output, encoded as a byte array via the generate function. Your web browser is smart enough to take this byte array and display it in your browser as a live feed.

Our final code block handles parsing command line arguments and launching the Flask app:
```
OpenCV - Stream video to web browser/HTML page
# check to see if this is the main thread of execution
if __name__ == '__main__':
	# construct the argument parser and parse command line arguments
	ap = argparse.ArgumentParser()
	ap.add_argument("-i", "--ip", type=str, required=True,
		help="ip address of the device")
	ap.add_argument("-o", "--port", type=int, required=True,
		help="ephemeral port number of the server (1024 to 65535)")
	ap.add_argument("-f", "--frame-count", type=int, default=32,
		help="# of frames used to construct the background model")
	args = vars(ap.parse_args())
	# start a thread that will perform motion detection
	t = threading.Thread(target=detect_motion, args=(
		args["frame_count"],))
	t.daemon = True
	t.start()
	# start the flask app
	app.run(host=args["ip"], port=args["port"], debug=True,
		threaded=True, use_reloader=False)
# release the video stream pointer
vs.stop()
```
Lines 118-125 handle parsing our command line arguments.

We need three arguments here, including:
```
--ip: The IP address of the system you are launching the webstream.py  file from.

--port: The port number that the Flask app will run on (you’ll typically supply a value of 8000 for this parameter).

--frame-count: The number of frames used to accumulate and build the background model before motion detection is performed. By default, we use 32  frames to build the background model.
```
Lines 128-131 launch a thread that will be used to perform motion detection.

Using a thread ensures the detect_motion function can safely run in the background — it will be constantly running and updating our outputFrame so we can serve any motion detection results to our clients.

Finally, Lines 134 and 135 launches the Flask app itself.

The HTML page structure
As we saw in webstreaming.py, we are rendering an HTML template named index.html.

The template itself is populated by the Flask web framework and then served to the web browser.

Your web browser then takes the generated HTML and renders it to your screen.

Let’s inspect the contents of our index.html file:
```
OpenCV - Stream video to web browser/HTML page
<html>
  <head>
    <title>Pi Video Surveillance</title>
  </head>
  <body>
    <h1>Pi Video Surveillance</h1>
    <img src="{{ url_for('video_feed') }}">
  </body>
</html>
```
As we can see, this is super basic web page; however, pay close attention to Line 7 — notice how we are instructing Flask to dynamically render the URL of our video_feed route.

Since the video_feed function is responsible for serving up frames from our webcam, the src of the image will be automatically populated with our output frames.

Our web browser is then smart enough to properly render the webpage and serve up the live video stream.

Putting the pieces together
Now that we’ve coded up our project, let’s put it to the test.

Open up a terminal and execute the following command:
```
OpenCV - Stream video to web browser/HTML page
$ python webstreaming.py --ip 0.0.0.0 --port 8000
 * Serving Flask app "webstreaming" (lazy loading)
 * Environment: production
   WARNING: This is a development server. Do not use it in a production deployment.
   Use a production WSGI server instead.
 * Debug mode: on
 * Running on http://0.0.0.0:8000/ (Press CTRL+C to quit)
127.0.0.1 - - [26/Aug/2019 14:43:23] "GET / HTTP/1.1" 200 -
127.0.0.1 - - [26/Aug/2019 14:43:23] "GET /video_feed HTTP/1.1" 200 -
127.0.0.1 - - [26/Aug/2019 14:43:24] "GET /favicon.ico HTTP/1.1" 404 -
```
As you can see in the video, I opened connections to the Flask/OpenCV server from multiple browsers, each with multiple tabs. I even pulled out my iPhone and opened a few connections from there. The server didn’t skip a beat and continued to serve up frames reliably with Flask and OpenCV.

Join the embedded computer vision and deep learning revolution!


I first started playing guitar twenty years ago when I was in middle school. I wasn’t very good at it and I gave it up only a couple years after. Looking back, I strongly believe the reason I didn’t stick with it was because I wasn’t learning in a practical, hands-on manner.

Instead, my music teacher kept trying to drill theory into my head — but as an eleven year old kid, I was just trying to figure out whether I even liked playing guitar, let alone if I wanted to study the theory behind music in general.

About a year and a half ago I decided to start taking guitar lessons again. This time, I took care to find a teacher who could blend theory and practice together, showing me how to play songs or riffs while at the same time learning a theoretical technique.

The result? My finger speed is now faster than ever, my rhythm is on point, and I can annoy my wife to no end rocking Sweet Child of Mine on my Les Paul.

My point is this — whenever you are learning a new skill, whether it’s computer vision, hacking with the Raspberry Pi, or even playing guitar, one of the fastest, fool-proof methods to pick up the technique is to design (small) real-world projects around the skill and try to solve it.

For guitar, that meant learning short riffs that not only taught me parts of actual songs but also gave me a valuable technique (such as mastering a particular pentatonic scale, for instance).

In computer vision and image processing, your goal should be to brainstorm mini-projects and then try to solve them. Don’t get too complicated too quickly, that’s a recipe for failure.

Instead, grab a copy of my Raspberry Pi for Computer Vision book, read it, and use it as a launchpad for your personal projects.

When you’re done reading, go back to the chapters that inspired you the most and see how you can extend them in some manner (even if it’s just applying the same technique to a different scenario).

Solving the mini-projects you brainstorm will not only keep you interested in the subject (since you personally thought of them), but they’ll teach you hands-on skills at the same time.

Today’s tutorial — motion detection and streaming to a web browser — is a great starting point for such a mini-project. I hope that now that you’ve gone through this tutorial, you have brainstormed ideas on how you may extend this project to your own applications.

But, if you’re interested in learning more…

My new book, Raspberry Pi for Computer Vision, has over 40 projects related to embedded computer vision + Internet of Things (IoT). You can build upon the projects in the book to solve problems around your home, business, and even for your clients. Each of these projects have an emphasis on:

Learning by doing.
Rolling up your sleeves.
Getting your hands dirty in code and implementation.
Building actual, real-world projects using the Raspberry Pi.
A handful of the highlighted projects include:

Daytime and nightime wildlife monitoring
Traffic counting and vehicle speed detection
Deep Learning classification, object detection, and instance segmentation on resource constrained devices
Hand gesture recognition
Basic robot navigation
Security applications
Classroom attendance
…and many more!
The book also covers deep learning using the Google Coral and Intel Movidius NCS coprocessors (Hacker + Complete Bundles). We’ll also bring in the NVIDIA Jetson Nano to the rescue when more deep learning horsepower is needed (Complete Bundle).

In case you missed the Kickstarter, you may wish to watch my announcement video:


Are you ready to join me to learn about computer vision and how to apply embedded devices such as the Raspberry Pi, Google Coral, and NVIDIA Jetson Nano?

If so, take a look at the book using the link below!

Pre-order my Raspberry Pi for Computer Vision book!
Summary
In this tutorial you learned how to stream frames from a server machine to a client web browser. Using this web streaming we were able to build a basic security application to monitor a room of our house for motion.

Background subtraction is an extremely common method utilized in computer vision. Typically, these algorithms are computationally efficient, making them suitable for resource-constrained devices, such as the Raspberry Pi.

After implementing our background subtractor, we combined it with the Flask web framework, enabling us to:

Access frames from RPi camera module/USB webcam.
Apply background subtraction/motion detection to each frame.
Stream the results to a web page/web browser.
Furthermore, our implementation supports multiple clients, browsers, or tabs — something that you will not find in most other implementations.

Whenever you need to stream frames from a device to a web browser, definitely use this code as a template/starting point.

To download the source code to this post, and be notified when future posts are published here on PyImageSearch, just enter your email address in the form below!
