The Contentful API delivers fast. Its CDN ensures that the media assets reach the audience with the lowest latency possible. However, mobile users – I’m talking about iOS throughout this post – may still experience slow images downloading due to the nature of cell networks, and they are forced to wait until the image is downloaded entirely, looking at the unpleasant *Loading...* spinner.

I thought that in 2015 we can definitely do better.

## First take: progressive download

One solution to this problem is *progressive downloading* – a technique which enables displaying parts of an image while it is being fetched. This can be quickly implemented with Apple's ImageIO framework [and a little effort](http://cocoaintheshell.com/2011/05/progressive-images-download-imageio/). 

While progressive downloading certainly improves the experience a bit, it still isn't miracles.

## A better way: progressive JPEG

Progressive JPEG is a format which stores multiple *passes* of an image of progressively higher detail. When such image is being loaded, the reader would first see a low-quality pixelated version which would gradually improve in detail, until the image is fully downloaded. The idea is to display the visuals as early as possible to entertain the visitor and make the layout look as designed.

See how progressively downloading a regular JPEG (left) is different from displaying progressive JPEG (right):
![THE GIF](https://camo.githubusercontent.com/24249b6f6909cb50379ffc62ffb8a3021e226f35/68747470733a2f2f7261772e67697468756275736572636f6e74656e742e636f6d2f636f6e74656e7466756c2d6c6162732f436f6e636f7264652f6d61737465722f7573652e676966)

Most likely you've already seen this approach – it's been very common on the web for quite some time. However, there is no out-of-the-box library for displaying progressive JPEGs in iOS apps. Photos in the Facebook iOS app [have been loading in this manner](https://code.facebook.com/posts/857662304298232/faster-photos-in-facebook-for-ios/) for some several months already, but their solution is not publicly available.

## Concorde helps

When there is no wheel at hand, it's a perfect opportunity to invent one. I have built [Concorde](https://github.com/contentful-labs/Concorde) – an open-source library for decoding progressive JPEGs on iOS and OS X. Download, plug in your projects, and make the user experience better.

## Install and use

Install Concorde via [CocoaPods](http://cocoapods.org) (version 0.36 or higher, as the code is partially written in Swift):

```
pod 'Concorde'
```

Then use `CCBufferedImageView` which progressively downloads and displays an image:

```objective-c
let imageView = CCBufferedImageView(frame: ...)
if let url = NSURL(string: "http://example.com/yolo.jpg") {
	imageView.load(url)
}
```

If you use [Contentful](https://www.contentful.com/), install the subspec:

```
pod 'Concorde/Contentful'
```

Then replace your calls to `UIImageView` with `CCBufferedImageView` to automatically use progressive JPEGs, in case you have been using the `UIImageView` category before. This will work regardless whether the original images were converted to progressive JPEGs or not – Contentful delivery API takes care of converting the assets.

Of course, you can also manually integrate the framework as a subproject or download it as a binary build.

## Implementation

Facebook based their solution on [libjpeg-turbo](http://www.libjpeg-turbo.org), a variant of the commonly used [libjpeg](http://libjpeg.sourceforge.net/), accelerated by using SIMD instructions. This seemed like the right approach, so Concorde is also built on that.

Since Apple's [WebKit](http://webkit.org) already supports progressive JPEGs, the solution is based on the existing code. The basic flow of decoding *buffered images*, which is their jargon for progressive JPEGs is as follows:

```
jpeg_create_decompress()
set data source
jpeg_read_header()
set overall decompression parameters
cinfo.buffered_image = TRUE;    /* select buffered-image mode */
jpeg_start_decompress()
for (each output pass) {
	adjust output decompression parameters if required
   jpeg_start_output()         /* start a new output pass */
   for (all scanlines in image) {
   	jpeg_read_scanlines()
     display scanlines
   }
   jpeg_finish_output()        /* terminate output pass */
}
jpeg_finish_decompress()
jpeg_destroy_decompress()
```

The part which deals directly with libjpeg-turbo is written in Objective-C, so that the framework does not have to expose all of `libjpeg.h` to the outside world. It also deals with a little oddity of error handling.

Fatal errors are handled by doing a `longjmp` which is a direct jump to a memory address, to some error-handling code. This was a bit painful to deal with, because a part of the libjpeg calls are made in the initializer of `CCBufferedImageDecoder`. But course we cannot jump back there once the instance has been initialized! The solution was to have a first jump target for errors during the initialization, and a second one for errors happening later in the process. 

Hopefully you would be happy to know that since I’ve dealt with all this already, you can simply enjoy displaying progressive JPEGs in the apps you build.

## Do I have to convert all my images?

Media content producers might wonder if they have to convert all their standard JPEGs into progressive ones, how to do it, and how do they check if a JPEG is progressive.

If you use Contentful for content delivery, please stop reading this paragraph. The Contentful API takes care of it all: regardless of the original format, the Contentful delivery API processes all the media assets to make them mobile-friendly.

Generally, to make an image loadable in this modern fashion – yes, you have to prepare the files so they are in the appropriate progressive format. You can do that with different tools: for example, [Photoshop](http://peteschuster.com/2013/01/saving-jpegs-for-the-web-setting-photoshop-up-for-progressive-jpegs/), [ImageMagick](http://www.imagemagick.org/), or command-line-friendly [jpegtran](http://jpegclub.org/jpegtran/).

To check whether a JPEG is progressive or not, use [this online tool](http://highloadtools.com/progressivejpeg) or [use command line interface of ImageMagick](http://www.hiddentao.com/archives/2013/06/11/how-to-check-if-a-jpeg-is-progressive/).

## Final notes

If you want to provide your users with pleasant user experience for image loading, regardless of connection speed, Concorde will help you achieve that. By delivering the content via Contentful you won't have to deal with converting the images specifically for this purpose - even better, you can also enjoy more kinds of [image manipulations](http://cocoadocs.org/docsets/ContentfulDeliveryAPI/1.6.0/Classes/CDAAsset.html#//api/name/imageURLWithSize:quality:format:fit:focus:radius:background:progressive:) which the Contentful API enables.
