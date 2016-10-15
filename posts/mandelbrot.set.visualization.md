<a href="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot-set.jpg"><img src="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot-set.jpg" alt="mandelbrot-set" width="809" height="660" class="alignnone size-full wp-image-516" /></a>

# Drawing Fractals: Mandelbrot Set Visualization

*Using HTML Canvas to visualize Mandelbrot set. Rendering large sets of points using HTML canvas efficiently.*<br /><br /><br />

## Mandelbrot Set

Mandelbrot set is defined as a set of numbers *z* on the complex plane for  which the sequence of numbers computed using the following expression

<a href="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot_sequence.gif"><img src="https://smthngsmwhr.files.wordpress.com/2016/10/mandelbrot_sequence.gif" alt="mandelbrot_sequence" width="117" height="45" class="alignnone size-full wp-image-517" /></a>

is such that their norm does not grow to the infinity as the index *n* increases indefinitely.

If at this point such notions as complex number, operations with complex numbers and computing the norm are not clear, please, refer for more details to the following Wikipedia article https://en.wikipedia.org/wiki/Complex_number for details.

Informally complex number is a point on a coordinate plane with operations *+*, *-*, */*, *** defined in a certain way.

An interesting thing about the Mandelbrot set is that despite the relative simplicity and mathematical formality of the definition the set itself turns out to look a bit irregular, as if it is a natural thing and has a property of repeating the same pattern infinitely at every scale. Such structures are called *fractals* https://en.wikipedia.org/wiki/Fractal.

Looking at the Mandelbrot set from a philosophical perspective we can suggest that natural self repeating structures can probably be described by similar relatively simple mathematical laws, that is, sometimes what seems as utterly irregular at the first sight can in fact be described and studied formally.

But rather than go into the details about fractals and their importance, in this article we will focus on a task of simply rendering Mandelbrot set in a browser.

The rendition of the set itself is shown at the beginning of this post.

## Escape Algorithm

One algorithm that will allow us to determine whether a point belongs to the set or not is the so called *Escape algorithm*.

The idea behind the algorithm is to compute the sequence of numbers from the definition of Mandelbrot set iteratively until a certain predefined maximum norm is reached or we have reached some predefined maximum number of iterations.

Then for every point on the complex plane we can find the number of iterations it takes for the numbers from the sequence to acquire sufficiently large norm. If the norm always stays too small which is the case for points from the Mandelbrot set we say that the maximum number of iterations corresponds to such a point.

Then we can color the points on the plane depending on the number of iterations it takes for the norm to grow sufficiently large. The larger the number of iterations the brighter the color, for example.

Now let's put this algorithm into code

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

First computing the number of escape iterations for a complex number *pointX + i pointY* which is like we noted is represented by two coordinates on the complex plane. We apply the rule for computing the next sequence number in the body of the while loop and increase the iteration number.

If necessary, please, refer to the same Wikipedia article to verify that indeed this is how complex numbers are multiplied and added.

We continue to iterate until the norm has become sufficiently large or the maximum number of iterations has been reached and return the number of iterations.

Next we need to compute the color of the point based on the number of iterations. This is quite easy to do, we just assign the white color to the maximum number of iterations, black to 0 iterations and divide the color spectrum between black and white into *MAX_ITERATIONS* values.

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

And the algorithm itself combines the two defined methods

```javascript
  host.EscapeAlgorithm = {
    getColor: function(x, y) {
      return getColorForIteration(
        getEscapeIterationsNumber(x, y)
      );
    }
  };
```

It seems quite simple, but how about rendering the set on a screen? For once we will have to scale the points on the screen so that the screen represents the interval *[-2, 2]* on both axis of the plane on a sufficiently large scale so that the Mandelbrot set is visible.

We would also like to show the progress of the computation and update the visualization dynamically are more points have been handled and we know whether they belong to the Mandelbrot set or not.

## Rendering on Canvas

Let's first start with a few simple rendering primitives that will help us display the set.

TODO

## Conclusion

TODO