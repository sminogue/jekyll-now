---
layout: post
title: Simple Document Scanner in Java
---

So recently in my travels I came across [Adrian Rosebrock's blog](http://www.pyimagesearch.com/), hes doing a lot of really interesting Computer Vision applications which I have enjoyed reading about. One of his posts was about creating a [simple document scanner in 5 minutes](http://www.pyimagesearch.com/2014/09/01/build-kick-ass-mobile-document-scanner-just-5-minutes/) caught my eye and captured my imagination. I had to tinker a bit as the post is written using openCV2 and the current is openCV3 which has some differences.

Being a Java developer I thought it would be fun to do a similar application in Java rather than python. I also didnt want to use openCV for my application. So here is my version of the quick (not mobile) document scanner in pure Java.

To make this work and not use openCV I first needed to find a good Java computer vision library. After searching around a bit and discarding some Java wrappers for openCV I came across [boofCV](https://boofcv.org/) Ive included the XML from my POM to help you find it. So with that in hand lets get started!

```XML
<dependency>
    <groupId>org.boofcv</groupId>
    <artifactId>ip</artifactId>
    <version>0.26</version>
</dependency>
  <dependency>
    <groupId>org.boofcv</groupId>
    <artifactId>visualize</artifactId>
    <version>0.26</version>
</dependency>
  <dependency>
    <groupId>org.boofcv</groupId>
    <artifactId>io</artifactId>
    <version>0.26</version>
</dependency>
```
So first off I created a simple maven project (its just easier for me to do dependency management with it but feel free to use whatever you like). I then created a simple class with a main method that will do the work and proove the concept. I also chose to use some visualization classes provided by boofcv.

So to get started I added the following to my main method:
```Java
ListDisplayPanel panel = new ListDisplayPanel();

ShowImages.showWindow(panel,"Document Scanning", true);
```
When running this you should be presented with a screen that displays nothing. So now we have a running application which is ready to display images. So lets load a test image and do some prep work on it to prepare page detection.

```Java
//Load test image from disk
BufferedImage image = UtilImageIO.loadImage("test.jpg");

//Convert to gray scale image
GrayU8 gray = ConvertBufferedImage.convertFromSingle(image, null, GrayU8.class);
       
//Brighten the image to wash out light colors in the background
GrayU8 brighter = GrayImageOps.brighten(gray, 100, 255, null);

//Reverse the black and white to make the paper page solid black for detection
GrayU8 invert = GrayImageOps.invert(brighter, 0, null);

panel.addImage(VisualizeBinaryData.renderBinary(invert, false, null), "Enhanced");

```
Now that the image has been loaded and we have done a little prep work we can detect the page.

```Java
//Detect 4 sided black polygon
ConfigPolygonDetector config = new ConfigPolygonDetector(4, 4);
BinaryPolygonDetector<GrayU8> detector = FactoryShapeDetector.polygon(config, GrayU8.class);

int threshold = GThresholdImageOps.computeOtsu(invert, 0, 255);
GrayU8 binary = new GrayU8(invert.width, invert.height);
ThresholdImageOps.threshold(invert, binary, threshold, true);
detector.process(invert, binary);

FastQueue<Polygon2D_F64> found = detector.getFoundPolygons();

//Go through the found polygons and take the polygon with the largest area. 
//This SHOULD be the page since this should be a picture of a page you are scanning
Polygon2D_F64 polygon = null;
double polygonSize = 0;

for (int i = 0; i < found.size; i++) {

    Polygon2D_F64 poly = found.get(i);

    double size = poly.areaSimple();

    if (size > polygonSize) {
        polygonSize = size;
        polygon = poly;
    }

}

//If unable to identify page... Bail.
if (polygon == null) {
    throw new Exception("Unable to identify page in image");
}
```

Ok great so at this point with have enhanced the image and used a detector to find a 4 sided polygon (the page). Now we just need to identify the 4 corners of the polygon an put them in TL,TR,BL,BR order. This is necessary as the boofcv library we will use to transform the image needs the points in that specific order. The transform is going to change the perspective to be looking straight at the page and the page be straightened out. This way if the picture is not PERFECTLY aligned we still get a good scan. I wrote a little method which will sort a list of points into the required order. Its in the first code block below. the second contains how we use it.

```Java
public static List<Point2D_F64> sortPoints( List<Point2D_F64> pts ) {

    //Copy points into a working list
    List<Point2D_F64> points = new ArrayList<Point2D_F64>();
    points.addAll(pts);

    //Create list of points to be returned.
    List<Point2D_F64> returns = new ArrayList<Point2D_F64>();

    //Sort the points by Y value
    Collections.sort(points, new Comparator<Point2D_F64>() {

        @Override
        public int compare( Point2D_F64 a, Point2D_F64 b ) {

            if (a.y < b.y) {
                return -1;
            } else if (a.y == b.y) {
                return 0;
            } else {
                return 1;
            }

        }

    });

    // First 2 elements in the list are the top of the page. The one with
    // the lower X is TL the other is TR
    Point2D_F64 a, b;
    a = points.get(0);
    b = points.get(1);

    if (a.x < b.x) {
        returns.add(a);
        returns.add(b);
    } else {
        returns.add(b);
        returns.add(a);
    }

    // Second 2 elements are the bottom points
    a = points.get(2);
    b = points.get(3);

    if (a.x < b.x) {
        returns.add(b);
        returns.add(a);
    } else {
        returns.add(a);
        returns.add(b);
    }

    return returns;

}
```

```Java
//Get the four corners and add them to a list fot sorting.
List<Point2D_F64> points = new ArrayList<>();
points.add(polygon.get(2));
points.add(polygon.get(1));
points.add(polygon.get(0));
points.add(polygon.get(3));

//Sort the points so that the array is in TL, TR, BR, BL order
points = sortPoints(points);

//Pull the four points out into variables for ease of use
Point2D_F64 topLeft = points.get(0);
Point2D_F64 topRight = points.get(1);
Point2D_F64 bottomRight = points.get(2);
Point2D_F64 bottomLeft = points.get(3);
```
So at this point we now have the four corners identified and we can transform the image.

```Java
//Convert the ORIGINAL image
Planar<GrayF32> input = ConvertBufferedImage.convertFromMulti(image, null, true, GrayF32.class);

//Based on the four corners get the width and height of the page
Double width = topRight.x - topLeft.x;
Double height = bottomLeft.y - topLeft.y;

//Setup to make the page top down
RemovePerspectiveDistortion<Planar<GrayF32>> removePerspective = new RemovePerspectiveDistortion<>(width.intValue(), height.intValue(), ImageType.pl(3, GrayF32.class));

// Specify the corners in the input image of the region.
// Order matters! top-left, top-right, bottom-right, bottom-left
if (!removePerspective.apply(input, topLeft, topRight, bottomRight, bottomLeft)) {
    throw new RuntimeException("Failed!?!?");
}

//Get the re-oriented image
Planar<GrayF32> output = removePerspective.getOutput();

//Turn it into a BufferedImage. This should be the page straight on and nothind else.
BufferedImage flat = ConvertBufferedImage.convertTo_F32(output, null, true);
```
At this point we have the image cropped and the perspective shifted to be straight. The last thing to do is to convert the final image to black and white and apply an adaptive threshold to remove shadows or any kind of discoloration.

```Java
//Turn page into a grayscale image
GrayF32 f32 = ConvertBufferedImage.convertFromSingle(flat, null, GrayF32.class);

//Apply filter to give us that pure black and white paper look. This should also 
//Remove any artifacts like shadows or discolorations in the paper.
GrayU8 bw = new GrayU8(f32.width, f32.height);
GThresholdImageOps.localSauvola(f32, bw, 15, 0.2f, true);

//Turn the pure B&W image into a bufferedimage. Also invert the colors as the filter
//Turns the page black with white text.
BufferedImage finalInversed = VisualizeBinaryData.renderBinary(bw, true, null);

panel.addImage(finalInversed, "Final");
        
ShowImages.showWindow(panel,"Document Scanning", true);
```
And there it is! If everything has worked properly we should be left with just the document in the picture and it should be a nice clean black and white.

Complete source for this can be found as a [gist on github](https://gist.github.com/sminogue/7bd865e9e46a9a2ab079c22af08d1c0d "gist on github").
