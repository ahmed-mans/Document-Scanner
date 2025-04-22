# 📄 Document Scanner using OpenCV in C++

This project demonstrates a lightweight document scanner built with OpenCV in C++. The figure below shows the original input image on the left and the resulting scanned output on the right.


![resultimages1](https://github.com/user-attachments/assets/0db648f0-c485-4450-a1e0-9620bf83dca4)

To transform the input into a clean, top-down view of the document, the algorithm follows these key steps (the complete code can be found in the ```doc_scanner.cpp ``` file):

- Loads and resizes a scanned image.
  ```
  // Read an image in grayscale
  string imagePath = "scanned-form.jpg";
  Mat src = imread(imagePath, IMREAD_GRAYSCALE);
  
  // Resize the image to a ratio = 1.33
  float imageRatio = 1.333;
  int imageWidth = 500;
  int imageHeight = imageWidth * imageRatio;
  
  Mat resizedImg;
  resize(src, resizedImg, cv::Size(imageWidth, imageHeight));
  ```
  
- Applies morphological operations and thresholding to isolate the document.
  ```
  // Get structuring element/kernel which will be used for opening operation
  int openingSize = 3;
  
  // Selecting a elliptical kernel 
  Mat element = getStructuringElement(MORPH_ELLIPSE, Size(2 * openingSize + 1, 2 * openingSize + 1),Point(openingSize, openingSize));
  
  // Apply closing operation
  Mat imageMorphClosed;
  morphologyEx(resizedImg, imageMorphClosed, MORPH_CLOSE, element, Point(-1, -1), 3);
  
  // Threshold the closed image
  Mat thresholdedImage;
  threshold(imageMorphClosed, thresholdedImage, 200, 256, THRESH_BINARY);
  ```
  
- Detects contours and finds the largest one (assumed to be the document)
  ```
  // Find all contours in the thresholded image
  vector<vector<Point> > contours;
  vector<Vec4i> hierarchy;
  findContours(thresholdedImage, contours, hierarchy, RETR_LIST, CHAIN_APPROX_SIMPLE);
  
  // Make a copy of the resized image
  Mat image = resizedImg.clone();
  
  // Draw all the contours on the copied image
  drawContours(image, contours, -1, Scalar(0, 255, 0), 3);
  
  // Find the largest contour based on area. The largest countour will be the one of the document.
  double maxArea = 0;
  int largestContourIdx = -1;
  for (size_t i = 0; i < contours.size(); i++) {
  	double area = contourArea(contours[i]);
  	
  	if (area > maxArea) {
  		maxArea = area;
  		largestContourIdx = i;
  	}
  }
  ```
  
- Approximates the contour to identify its corners

  ```
  // Create a blank mask
  Mat mask = Mat::zeros(thresholdedImage.size(), CV_8UC1);
  
  // Draw only the largest contour and fill it
  if (largestContourIdx != -1) {
  	drawContours(mask, contours, largestContourIdx, Scalar(255), FILLED);
  }
  
  // Use approxPolyDP to find the rectangle for the document
  vector<Point> approxCurve;
  double epsilon = 0.02;  // Approximation accuracy
  approxPolyDP(contours[largestContourIdx], approxCurve, epsilon, true);
  
  // Vizualize the polynome with polylines
  Mat result = Mat::zeros(thresholdedImage.size(), CV_8UC3);
  polylines(result, approxCurve, true, Scalar(0, 255, 0), 2);
  
  // Print approxCurve to see what it contains
  cout << "approxCurve : " << approxCurve << endl;
  cout << "approxCurve size : " << approxCurve.size() << endl;
  
  // Extract the four corners of the polynome
  int minX = 100000, maxX = 0, minY = 100000, maxY = 0;
  int minXIdx = -1, minYIdx = -1, maxXIdx = -1, maxYIdx = -1;
  // if the rectangle is not straight, there will be more than 4 points
  if (approxCurve.size() > 4) {
  
  	// Loop through all points in the approxCurve
  	for (size_t j = 0; j < approxCurve.size(); j++) {
  		Point p = approxCurve[j];
  
  		if (p.x < minX) { minX = p.x; minXIdx = j; }
  		if (p.x > maxX) { maxX = p.x; maxXIdx = j; }
  		if (p.y < minY) { minY = p.y; minYIdx = j; }
  		if (p.y > maxY) { maxY = p.y; maxYIdx = j; }
  	}
  }
  cout << "maxX: " << approxCurve[maxXIdx] << endl;
  cout << "maxY: " << approxCurve[maxYIdx] << endl;
  cout << "minX: " << approxCurve[minXIdx] << endl;
  cout << "minY: " << approxCurve[minYIdx] << endl;
  
  ```
  
- Computes a homography to warp the image to a top-down view of the document
  ```
  // Initializing Point objects for the 4 coordinate of the corners of the polynome
  Point topLeft;
  Point bottomLeft;
  Point bottomRight;
  Point topRight;
  
  // Get  the correct top left corner of the document.
  if (approxCurve[maxXIdx].y > approxCurve[minXIdx].y)
  
  {
  	topLeft = Point(approxCurve[minXIdx].x, approxCurve[minXIdx].y);
  	bottomLeft = Point(approxCurve[maxYIdx].x, approxCurve[maxYIdx].y);
  	bottomRight = Point(approxCurve[maxXIdx].x, approxCurve[maxXIdx].y);
  	topRight = Point(approxCurve[minYIdx].x, approxCurve[minYIdx].y);
  }
  
  else if (approxCurve[maxXIdx].y < approxCurve[minXIdx].y)
  
  {
  	topLeft = Point(approxCurve[minYIdx].x, approxCurve[minYIdx].y);
  	bottomLeft = Point(approxCurve[minXIdx].x, approxCurve[minXIdx].y);
  	bottomRight = Point(approxCurve[maxYIdx].x, approxCurve[maxYIdx].y);
  	topRight = Point(approxCurve[maxXIdx].x, approxCurve[maxXIdx].y);
  }
  
  // Compute homography matrix to fit the final image size of width 500
  vector<Point> srcPoints = vector<Point>{ topLeft,bottomLeft,bottomRight,topRight};
  vector<Point> dstPoints = vector<Point>{ Point2f(0,0),Point2f(0,imageHeight),Point2f(imageWidth, imageHeight),Point2f(imageWidth,0) };
  Mat h = findHomography(srcPoints, dstPoints);
  cout << h;
  
  // Apply the warp using the computed homography
  Mat warpedImage;
  Size outDim = resizedImg.size();
  warpPerspective(resizedImg, warpedImage, h, outDim);

  ```
- Displays the final, rectified "scanned" version
  ```
  imshow("original", resizedImg);
  imshow("Scanned image", warpedImage);
  
  waitKey();
  destroyAllWindows();
  ```
