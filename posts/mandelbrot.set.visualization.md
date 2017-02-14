<a href="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot-set.jpg"><img src="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot-set.jpg" alt="mandelbrot-set" width="809" height="660" class="alignnone size-full wp-image-516" /></a>

# Drawing Fractals: Mandelbrot Set Visualization

*Using HTML canvas to visualize Mandelbrot set. Rendering large sets of points efficiently.*<br /><br /><br />

## Mandelbrot Set

Mandelbrot set is defined as a set of numbers *z* on the complex plane for  which the sequence of numbers computed using the following expression

<a href="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot_sequence.gif"><img src="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot_sequence.gif" alt="mandelbrot_sequence" width="117" height="45" class="alignnone size-full wp-image-517" /></a>

is such that their norm does not grow to the infinity as the index *n* increases indefinitely.

If at this point the notions of complex number, operations with complex numbers and computing the norm are not clear, please, refer for more details to the following Wikipedia article https://en.wikipedia.org/wiki/Complex_number for details.

Essentially a complex number can be considered to be a point on a two dimensional coordinate plane with operations *+*, *-*, */*, * defined for pairs of such points in a certain way.

An interesting thing about the Mandelbrot set is that despite the relative simplicity and mathematical formality of the definition, the set itself turns out to be quite curiously looking, as if it were some real thing occurring in nature and in addition to that it also has a property of repeating the same pattern infinitely at every scale. The structures with the property of repeating itself at every scale are called *fractals* https://en.wikipedia.org/wiki/Fractal.

On the other hand looking at the Mandelbrot set from a philosophical perspective we can expect that naturally occurring self repeating structures can probably be described by similar relatively simple mathematical laws, that is, sometimes what seems as utterly irregular at the first sight can in fact be described and studied formally.

But rather than go into the details about fractals and their importance, in this article we will focus on a task of computing and rendering the Mandelbrot set in a browser.

The rendition of the set itself we will obtain by the end of the article is shown in the beginning.

## Escape Algorithm

One algorithm that can allow us to determine whether a point belongs to the Mandelbrot set or not is the so called *Escape algorithm*.

The idea behind the algorithm is to compute the sequence of numbers from the definition of Mandelbrot set iteratively until a certain predefined maximum norm is reached or we have reached some predefined maximum number of iterations.

Then for every point on the complex plane we can find the number of iterations it takes for the numbers from the sequence to acquire a sufficiently large norm. If on the other hand the norm always stays too small, which is the case for points belonging to the Mandelbrot set, then we just observe that the maximum number of iterations has been reached and stop the iteration process for such a point.

Then we can color the points on the plane depending on the number of iterations corresponding to each point. The larger the number of iterations - the brighter the color, for example.

Now let's put this informally discussed algorithm into code

```javascript
  var MAX_VALUE = 4.0;
  var MAX_ITERATIONS = 30;

  function getEscapeIterationsNumber(pointX, pointY) {
    var currentIteration = 0;
    var x = 0;
    var y = 0;
    while ((currentIteration < MAX_ITERATIONS) 
      && (x * x + y * y < MAX_VALUE)) {
      const xOfSquare = x * x - y * y;
      const yOfSquare = 2 * x * y;
      x = xOfSquare + pointX;
      y = yOfSquare + pointY;
      currentIteration++;
    }
    return currentIteration;
  }
```

Here we compute the number of escape iterations for a complex number *pointX + i pointY* which is, like we noted before, can be represented by two coordinates on the complex plane. We use the rule for computing the next sequence number from the definition of the Mandelbrot set in the body of the while loop and increase the iteration number.

If there are any doubts it is possible to refer to the same Wikipedia article about complex numbers to verify that indeed this is how complex numbers are multiplied and added. We will however leave the detailed explanation out of the scope of the present article.

We continue to iterate until the norm has become sufficiently large or the maximum number of iterations has been reached, and after that point we return the total number of performed iterations.

Next we still need to compute the color of the point based on the computed number of iterations. This turns out to be quite easy to do, as we just assign the white color to the maximum number of iterations, black to 0 iterations and divide the color spectrum between black and white into *MAX_ITERATIONS* values.

```javascript
  var MAX_COLOR = 255;
  var COLOR_SCALE =
    Math.floor(MAX_COLOR / MAX_ITERATIONS);

  function getColorForIteration(iterationNumber) {
    return Math.min(
      iterationNumber * COLOR_SCALE,
      MAX_COLOR
    );
  }
```

The algorithm itself combines the two methods above

```javascript
  host.EscapeAlgorithm = {
    getColor: function(x, y) {
      return getColorForIteration(
        getEscapeIterationsNumber(x, y)
      );
    }
  };
```

So far everything seems quite simple, but how about rendering the set on a screen? For once we will have to scale the points so that the screen represents the interval *[-2, 2]* on both axis of the plane using a sufficiently large scale so that the Mandelbrot set is visible.

We would also like to show the progress of the computation and update the visualization dynamically as more points have been handled and we know whether they belong to the Mandelbrot set or not.

## Rendering on Canvas

Let's now start defining a few simple rendering primitives that will help us display the set.

We will first create a Display object that will provide convenient methods for interacting with the HTML canvas which we will ultimately use for drawing.

```javascript
  function Display(canvas, width, height) {
    this.canvas = canvas;
    this.width = width;
    this.height = height;
    this.context = null;
    this.imageData = null;
  }
```

We pass to the constructor a canvas DOM element, and the desired width and height of the canvas.

```javascript
  Display.prototype.initialize = function() {
    this.canvas.setAttribute('width', window.innerWidth);
    this.canvas.setAttribute('height', window.innerHeight);
    this.context = this.canvas.getContext('2d');
    this.context.font = FONT_SIZE_PX + 'px Arial';
    this.imageData = this.context.getImageData(0, 0,
      this.width, this.height);
  };
```

Next we resize the canvas to the desired width and height, get a 2D drawing context, set its font and initialize the image data.

We will use image data to draw points on a canvas in a batch fashion as there may be quite of them to draw. Using ```imageData``` here is an optimization opportunity provided by the canvas itself: rather than calling some drawing primitive of the canvas for every single pixel multiple times we will instead first set pixel data in ```imageData``` and only after that we will repaint the whole canvas once.

```javascript
  Display.prototype.drawPixel = function(x, y, color) {
    var index = (x + y * this.width) * 4;
    this.imageData.data[index + 0] = color;
    this.imageData.data[index + 1] = color;
    this.imageData.data[index + 2] = color;
    this.imageData.data[index + 3] = FULLY_OPAQUE_ALHPA;
  }
```

```drawPixel``` computes the index inside the ```imageData``` array given the coordinates on the canvas ```x``` and ```y``` and sets the color data of the pixel assuming that we will draw the image in black and white.

```javascript
  Display.prototype.repaint = function() {
    this.context.putImageData(this.imageData, 0, 0);
  }
```

The actual drawing happens inside the ```repaint``` method where all the pixels corresponding to the whole ```imageData``` array get repainted.

Here is one of the takeaways from this article: it is much faster to use canvas's image data to draw large sets of points rather than to try to dry them individually.

We would also like to show 'the computation border': horizontal green line moving from top to bottom of the screen such that all the pixels above it the set have been already computed and drawn.

```javascript
  Display.prototype.showComputationBorder = function(y) {
    this.context.fillRect(0, y, this.width, 1);
  }
```

Likewise we can show the current progress at the top left corner of the canvas: percentage of the total pixels computed and the average speed of computation in thousands of pixels.

```javascript
  Display.prototype.showProgress = function(progress, speed) {
    this.context.fillStyle = 'green';
    var progressInfo = progress.toFixed(2) + '%';
    var speedInfo = 'Speed ' + speed + 'K pixels/second';
    this.context.fillText('Drawing Mandelbrot set... '
      + progressInfo + ' ' + speedInfo, 20, 20);
  }
```

## Drawing Mandelbrot Set

Now that we have the implementation of the Escape algorithm and can draw pixels and overall progress on the canvas, we need to still put all this together.

```javascript
  MandelbrotSetVisualization.prototype.computeAndDraw = function() {
    var self = this;
    this.startTime = new Date().getTime();
    var width = this.width;
    var height = this.height;
    this.totalProgress = 0;
    var promises = [];
    for (var y = 0; y < height; y++) {
      promises.push(new Promise(function(resolve, reject) {
        setTimeout(function(y) {
          for (var x = 0; x < width; x++) {
            const color = EscapeAlgorithm.getColor(
              self.scaleX(x), self.scaleY(y)
            );
            self.display.drawPixel(x, y, color);
          }
          self.totalProgressPercent += self.progressPerLinePercent;
          self.drawCurrentState(y);
          resolve();
        }, 0, y);
      }));
    }
    Promise.all(promises).then(function() {
      self.display.repaint();
    });
  }
```

In the lines 8-22 we iterate through all the rows of the canvas and handle every row asynchronously by wrapping the computation into a promise at the line 9 and using ```setTimeout``` at the line 10.

The actual computation for every row happens in the lines 11-18. 11-16 we go through all the columns in the current row of the canvas, compute the color for the current pixel of the canvas with ```getColor``` and add it to the canvas data with ```drawPixel```. Note that we also scale the pixel coordinates using the functions ```scaleX``` and ```scaleY``` so that the canvas contains the interval *[-2, 2]* at an appropriate level of detail.

```javascript
  MandelbrotSetVisualization.prototype.scaleX = function(x) {
    return (SCALED_SIZE * x / this.size)
      - (SCALED_SIZE * this.width) / (2 * this.size);
  }
  MandelbrotSetVisualization.prototype.scaleY = function(y) {
    return (SCALED_SIZE * y / this.size)
     - (SCALED_SIZE * this.height) / (2 * this.size);
  }
```

Finally at the line 17 we update the total progress value and at the line 18 draw the current state of the canvas.

It also turns out to be convenient not to redraw the whole current state of the canvas for every row, but only for every 50 rows or so (value defined by the variable ```UPDATE_CANVAS_STEP```). Repainting the whole canvas is quite an expensive operation and skipping repainting for some of the rows significantly boosts the performance.

```javascript
  MandelbrotSetVisualization.prototype.drawCurrentState = function(y) {
    if (y % UPDATE_CANVAS_STEP === 0) {
      this.display.repaint();
      const elapsedTime = new Date().getTime() - this.startTime;
      const thousandPixelsPerSecond =
        Math.floor((((y + 1) * this.width) / elapsedTime));
      this.display.showComputationBorder(y);
      this.display.showProgress(
        this.totalProgressPercent, thousandPixelsPerSecond
      );
    }
  };
```

Otherwise we just repaint the canvas, get the elapsed time in milliseconds since the start of the visualization, compute the speed, show the computation border and the overall progress.

In the method ```computeAndDraw``` we finally wait for all the promises to resolve at the line 23 and do a final repaint of the whole computed Mandelbrot set at the line 24. This time we just display the whole set without the computation border or progress information.

Let's note that it is important to wait until all of the asynchronous tasks of computing each row of the Mandelbrot set fully complete before doing the final repaint of the canvas. Otherwise we may end up with a race condition and one of the asynchronous tasks being randomly chosen to be repainted last which will result in an incomplete set being shown. Hence the need to deal with promises and await the completion of asynchronous tasks.

Likewise it is important for parts of the computation to be asynchronous as otherwise the canvas will not be repainted by the browser with intermediate results as there will be no execution time to be allocated for that.

When we finally run the simulation in the ```main``` method the Mandelbrot set is gradually painted on the screen as the computation progresses.

```javascript
function main() {
  var width = window.innerWidth;
  var height = window.innerHeight;
  var canvas = document.querySelector('canvas');
  var display = new Display(canvas, width, height);
  var setOfMandelbrot = new MandelbrotSetVisualization(display, width, height);
  display.initialize();
  setOfMandelbrot.computeAndDraw();
}

window.addEventListener('load', main);
```

The full source code of the visualization is available as a gist https://gist.github.com/antivanov/59290f4048b03c7d01b88526fecf8afb and is hosted here https://output.jsbin.com/gowiruwiku/2

## Conclusion

* It is quite convenient to use HTML Canvas for visualizing data
* Canvas API provides optimization opportunities for drawing multiple pixels in batches
* Repainting the whole canvas is quite an expensive operation and some repaints can be skipped to increase the perceivable speed
* Large computation tasks are better split in smaller asynchronous parts so that the browser gets the chance to repaint the updated canvas