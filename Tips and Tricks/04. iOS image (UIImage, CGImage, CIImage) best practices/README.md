
## Image (UIImage, CGImage, CIImage) efficient handling on iOS

In this tutorial we will check the ways to work with images on iOS and find the most efficient one. For testing we are gonna use high resolution image (taken from Flickr.com [Launch of Mars Explorer Rover-B](https://www.flickr.com/photos/nasacommons/19138397548/in/album-72157650353459778/) with its size 6.4 mb on hard drive.

<img src="./Images/rocket.jpg" width="10%" height="10%"/>

First of all let's check how this image we will bundle to our application and what we can do to make it better.

### Adding image as application bundle resource

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.13.03.png" width="30%" height="30%"/>

When we do like this then the image will be copied inside the application bundle:

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.13.31.png" width="30%" height="30%"/>

and final application bundle size gonna be 6.6 mb :

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.16.32.png" width="30%" height="30%"/>

Run the application without any logic yet and check its memory consumption.

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.44.14.png" width="30%" height="30%"/>

### Find a better way to create UIImage from image file

We see that the application consumes 27.6 mb of ram memory in idle state. Now let's create UIImage instance for this bundled image:

```swift
final class ViewController: UIViewController {

    private var image: UIImage?    
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"

        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        let data = try! Data(contentsOf: url)
        let image = UIImage(data: data)!
        self.image = image
    }
}
```

We just store the image to a property, there is no showing of it on UI yet. Let's look as application memory goes:

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 21.58.09.png" width="30%" height="30%"/>

We can see the memory consumption increased by ~6.4 mb, that is the size of our image. And we don't show it yet on UI! Assume a server gave us an array of images and when we creating UIImage as above we will get a lot of ram memory usage even when images are not shown to a user yet. With code like this

```swift
final class ViewController: UIViewController {

    private var images: [UIImage] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        for _ in 0...5 {
            let data = try! Data(contentsOf: url)
            let image = UIImage(data: data)!
            self.images.append(image)
        }
    }
}
```

the app memory will look like this

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.56.39.png" width="30%" height="30%"/>

That is huge and there is no any business logic yet! Why is this happening ? Let's walk through the code and find a place from where the issue goes. 

```swift
final class ViewController: UIViewController {

    private var data: [Data] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        for _ in 0...5 {
            let data = try! Data(contentsOf: url)
            self.data.append(data)
        }
    }
}
```

Then check the memory

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.56.39.png" width="30%" height="30%"/>

Nothing has changed. Seems the problem is in Data object. Let's check is there a way to fix this. Go in Data's initializer documentation and see

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 16.07.50.png" width="30%" height="30%"/>

And these is nothing at all. Okay, will look at Data header at Foundation framework

```swift
    /// Initialize a `Data` with the contents of a `URL`.
    ///
    /// - parameter url: The `URL` to read.
    /// - parameter options: Options for the read operation. Default value is `[]`.
    /// - throws: An error in the Cocoa domain, if `url` cannot be read.
    public init(contentsOf url: URL, options: Data.ReadingOptions = []) throws
```

It is something. We see that initializer has second parameter `options` of type `Data.ReadingOptions` that is OptionSet. The default value for initializer is empty that is []. Let's see what these options are. Finally there is something.

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 16.12.27.png" width="30%" height="30%"/>

Now will look at the values of this option set.

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 17.02.45.png" width="30%" height="30%"/>

Let's try with `uncached` and find out what it means.

```swift
final class ViewController: UIViewController {

    private var data: [Data] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        for _ in 0...5 {
            let data = try! Data(contentsOf: url, options: .uncached)
            self.data.append(data)
        }
    }
}
```

And there are no changes, app still eats a lot of memory

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 15.56.39.png" width="30%" height="30%"/>

One more try with `alwaysMapped` option.

```swift
final class ViewController: UIViewController {

    private var data: [Data] = []
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        for _ in 0...5 {
            let data = try! Data(contentsOf: url, options: .alwaysMapped)
            self.data.append(data)
        }
    }
}
```

And look at the memory

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 17.09.22.png" width="30%" height="30%"/>

the value is like the app has in idle state. What happened there ? Take a closer look at description of options.

`uncached`

>A hint indicating the file should not be stored in the file-system caches.

That means the data loaded from URL will be read to the ram memory at creation of Data.

`alwaysMapped`

> Hint to map the file in if possible.

What means do not load data to the memory, but create some temporal file (not sure, might be just uses the origin data file) with this data and link the Data object to. This is why there is no memory consumption. Sure this behaviour will slow data loading when it is going to be used as OS first must read it from file system, that is much slower than reading from ram memory.

Okay, first takeaway is to use `alwaysMapped`/`mappedIfSafe` for Data object creation for files that cause big pressure on application memory usage.

Let's go back to UIImage and rewrite it with new knowledge.

```swift
final class ViewController: UIViewController {

    private var image: UIImage?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        let data = try! Data(contentsOf: url, options: .alwaysMapped)
        let image = UIImage(data: data)!
        self.image = image
    }
}
```

and finally we got what we wanted
 
<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 17.39.47.png" width="30%" height="30%"/>

Now let's get rid off `data` object as UIImage has initializer with URL path argument.

```swift
final class ViewController: UIViewController {

    private var image: UIImage?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        let image = UIImage(contentsOfFile: url.path)!
        self.image = image
    }
}
```

with memory consumption

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 17.39.47.png" width="30%" height="30%"/>

this means that UIImage under hood also uses `Data` with `alwaysMapped`/`mappedIfSafe` options or similar behaviour. Next takeaway is that creation of `UIImage` from `URL` costs nothing for memory.

### Optimize application bundle size

As you remember we put the image inside application bundle as resource file. Apple recommends store all images at an assets catalog. Let's see what we get when move the image from application bundle to assets catalog.

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 17.59.56.png" width="30%" height="30%"/>

Then build the project and check final application bundle

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 18.05.30.png" width="30%" height="30%"/>

We see that there is no `rocket.jpg` file anymore, but we got there `Assets.car` file, that bundles all asset's resources in itself. We can view its content via open source project [Asset Catalog Tinkerer](https://github.com/insidegui/AssetCatalogTinkerer). Check application bundle size

<img src="./Images/Screenshots/1/Screenshot 2020-03-08 at 18.02.13.png" width="30%" height="30%"/>

and unfortunately there no changes, it still is 6.6 mb. We can conclude that using raster images (jpg, png, bmp, gif, etc.) for assets catalog has no significant benefits, but it still is recommended way and best practise to store images in application bundle. Seems there only are 2 ways to reduce application bundle size:

1. Reduce the image quality.
2. Remove the image from application bundle and download it later from a server.

And last note, as we moved image to assets catalog, now we access to it directly via UIImage class

```swift
final class ViewController: UIViewController {

    private var image: UIImage?
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        self.image = UIImage(named: imageName)!
    }
}
```

### A better way to preview an image in UIImageView

Moving to most interesting part of this article - showing an image to a user. For day to day coding work this sounds as the simplest task to todo, it's just couple lines of code

```swift
final class ViewController: UIViewController {

    @IBOutlet private var imageView: UIImageView!
    
    override func viewDidLoad() {
        super.viewDidLoad()
        
        let imageName = "rocket.jpg"
        self.imageView.image = UIImage(named: imageName)
    }
}
```

I set `imageView` size to 320x480. 

<img src="./Images/Screenshots/2/Screenshot 2020-03-08 at 20.06.00.png" width="30%" height="30%"/>

The result on screen

<img src="./Images/Screenshots/2/Simulator Screen Shot - iPhone 11 - 2020-03-08 at 19.58.00.png" width="30%" height="30%"/>

Check memory consumption

<img src="./Images/Screenshots/2/Screenshot 2020-03-08 at 20.04.07.png" width="30%" height="30%"/>

Let's change image view size to 100x100
 
<img src="./Images/Screenshots/2/Simulator Screen Shot - iPhone 11 - 2020-03-08 at 20.10.19.png" width="30%" height="30%"/>
 
and check the memory one more time

<img src="./Images/Screenshots/2/Screenshot 2020-03-08 at 20.04.07.png" width="30%" height="30%"/>

and it is the same! Change the size to full screen

<img src="./Images/Screenshots/2/Simulator Screen Shot - iPhone 11 - 2020-03-08 at 20.12.04.png" width="30%" height="30%"/>

and memory consumption still remains the same!  

<img src="./Images/Screenshots/2/Screenshot 2020-03-08 at 20.04.07.png" width="30%" height="30%"/>

What is going on ? Let's do a math. Application idle state is ~27 mb, image size on the disk is 6.4 mb, so when we loaded the image to UIImageView the application memory consumption size should has been ~33.4 mb, but we got ~36.7 mb, that is the image now takes ~9.7 mb (~3.3 mb more (+34%)). To understand this we have to go back to our original image. As you remember it is `rocket.jpg`, seems the reason is a format of the file. Let's cite [Wikipedia](https://en.wikipedia.org/wiki/JPEG)

> JPEG is a commonly used method of lossy compression for digital images.

As this image is compressed then before showing it UIImageView must decompress it to restore image's original state. That is why we got more memory consumption than we expected (~36.7 mb instead of ~33.4 mb). I think you also has a question on your mind as it is in mine - can we know how much an image will take memory after decompression ? Let's find out. Before measurements make helper byte counter formatter

```swift
let byteCounter = ByteCountFormatter()
byteCounter.allowedUnits = .useMB
byteCounter.countStyle = .file
``` 

1 attempt

```swift
let imageName = "rocket.jpg"
let image = UIImage(named: imageName)!
let cgImage = image.cgImage!
byteCounter.string(fromByteCount: Int64(cgImage.bytesPerRow * cgImage.height))
```

gives us `24,3 MB` result, that seems isn't what we expect, we expect ~9.7 mb. We used `cgImage.height` that is image's height in pixels, as we know iPhone screen uses pixel per point value, may be if use this instead of pixels we will get right result.  

```swift
byteCounter.string(fromByteCount: Int64(cgImage.bytesPerRow * cgImage.height / Int(UIScreen.main.scale)))
```

we getting `12,1 mb` that is a bit closer to our math `9.7 mb`, but still not the same. Answers from [stackoverflow.com](https://stackoverflow.com/a/9076993/3614746) suggest to use `UIImage.jpegData(compressionQuality:)` method. Let's check it out

```swift
byteCounter.string(fromByteCount: Int64(image.jpegData(compressionQuality: 1)!.count))
```

and the result is `~9,3 mb`, seems what we wanted. What we can say after this is the size of UIImageView does nothing with image memory footprint, even if change size of UIImageVew to 10x10 pt the image still will consume its full uncompressed size. Also image size on disk and rendered on a screen is not the same because the image file on disk is in compressed state, UI have to decopress it to display it first.

Okay, we have only one image that consumes 9 mb of memory - not a big deal. But what will occur if we have UICollectionView with a lot of small cells with images in it ? It will consume a lot of memory be sure. Even Apple has spoken about this issue at its [WWDC 2018](https://developer.apple.com/videos/play/wwdc2018/219/) conference. What Apple recommends is to use image downsampling technic. They even provided a source code for

```swift
func downsample(imageAt imageURL: URL, to pointSize: CGSize, scale: CGFloat) -> CGImage {
    let imageSourceOptions = [kCGImageSourceShouldCache: false] as CFDictionary
    let imageSource = CGImageSourceCreateWithURL(imageURL as CFURL, imageSourceOptions)!
    let maxDimensionInPixels = max(pointSize.width, pointSize.height) * scale
    let downsampleOptions =
        [kCGImageSourceCreateThumbnailFromImageAlways: true,
         kCGImageSourceShouldCacheImmediately: true,
         kCGImageSourceCreateThumbnailWithTransform: true,
         kCGImageSourceThumbnailMaxPixelSize: maxDimensionInPixels] as CFDictionary
    return CGImageSourceCreateThumbnailAtIndex(imageSource, 0, downsampleOptions)!
}
```

Let's apply this to our application.

```swift
final class ViewController: UIViewController {

    @IBOutlet private var imageView: UIImageView!
    
    override func viewDidAppear(_ animated: Bool) {
        super.viewDidAppear(animated)
    
        // Put this logic to `viewDidAppear` as we want to work with fully ready UI.
        let imageName = "rocket.jpg"
        let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
        let size = self.imageView.bounds.size
        let scale = UIScreen.main.scale
        
        let cgImage = self.downsample(imageAt: url, to: size, scale: scale)
        let image = UIImage(cgImage: cgImage)
        self.imageView.image = image
    }
}
```

And after this the memory consumption reduced to 27.8 mb

<img src="./Images/Screenshots/2/Screenshot 2020-03-08 at 23.14.54.png" width="30%" height="30%"/>

Check decompressed size of new downsampled image

```swift
fromByteCount: Int64(image.jpegData(compressionQuality: 1)!.count)
```

that shows only 0.9 mb, that is almost 10 times lower that the original! And there one more feature in downdampling, as downsampling works with CGImage we can schedule this on background thread, what is awesome!

```swift
override func viewDidAppear(_ animated: Bool) {
    super.viewDidAppear(animated)

    let imageName = "rocket.jpg"
    let url = Bundle.main.url(forResource: imageName, withExtension: nil)!
    let size = self.imageView.bounds.size
    let scale = UIScreen.main.scale
    
    DispatchQueue.global().async {
        let image = self.downsample(imageAt: url, to: size, scale: scale)
        DispatchQueue.main.async {
            let image = UIImage(cgImage: image)
            self.imageView.image = image
        }
    }
}
```

Okay, next takeaway is that for large images always use downsampling when possible.

### Crop image efficiently

Let's say we need to crop the rocket from the image. Check most popular answers on [stackoverflow.com](https://stackoverflow.com) here [how-do-you-crop-an-image-in-ios](https://stackoverflow.com/questions/10079701/how-do-you-crop-an-image-in-ios) and here [cropping-an-uiimage](https://stackoverflow.com/questions/158914/cropping-an-uiimage). I see these answers overcomplicated and all of them require us run cropping logic on UI thread, for large images the cropping can take some time that will freeze application's UI. As you remember it is allowed to work with CGImage on background thread, let's use this for image cropping.

First of all, we need to find real image orientation, when we preview the image on UIImageView it shows it to us in `CGImagePropertyOrientation.up` even if you check image EXIF info where `orientation` value is not `.up`. You can easily verify this by previewing photo from iOS gallery and then check its EXIF where orientation in most cases will be `CGImagePropertyOrientation.right`. There even is a documention about this behaviour [https://developer.apple.com/documentation/imageio/cgimagepropertyorientation](https://developer.apple.com/documentation/imageio/cgimagepropertyorientation)

> The pixel data for an image captured by an iOS device camera is encoded in the camera sensor's native landscape orientation. When the user captures a photo while holding the device in portrait orientation, iOS writes an orientation value of CGImagePropertyOrientation.right in the resulting image file. Software displaying the image can then, after reading that value from the file's metadata, apply a 90° clockwise rotation to the image data so that the image appears in the photographer's intended orientation.

That means UIImageView applies an orientation transformation to an image if EXIF info available for. Extracting the orienation value from EXIF

```swift
func orientation(imageURL: URL) -> CGImagePropertyOrientation {
    let options = [kCGImageSourceShouldCache as String: kCFBooleanFalse] as CFDictionary
    // default orientation if EXIF not available.
    var orientation: CGImagePropertyOrientation = CGImagePropertyOrientation.up
    if let source = CGImageSourceCreateWithURL(imageURL as CFURL, options),
       let exif: NSDictionary = CGImageSourceCopyPropertiesAtIndex(source, 0, options) {
        if let orientationRaw = exif[kCGImagePropertyOrientation] as? NSNumber,
           let orientationValue = CGImagePropertyOrientation(rawValue: orientationRaw.uint32Value) {
            orientation = orientationValue
        }
    }
    return orientation
}
```

Apply this to the image

```swift
func apply(orientation: CGImagePropertyOrientation, to imageURL: URL) -> CGImage {
    let options = [kCGImageSourceShouldCache as String: kCFBooleanFalse] as CFDictionary
    let source = CGImageSourceCreateWithURL(imageURL as CFURL, options)!
    let cgImageRaw = CGImageSourceCreateImageAtIndex(source, 0, nil)!
    // Apply new orientation.
    let ciImage = CIImage(cgImage: cgImageRaw).oriented(orientation)
    let ciContext = CIContext(options: nil)
    // Make a copy of original image in new orientation.
    return ciContext.createCGImage(ciImage, from: ciImage.extent)!
}
```

Now we are ready to crop the rocket from the image

```swift
let rocketFrame = CGRect(...)
let cropped = cgImage.cropping(to: rocketFrame)!
```  

Save the result to a file. Let's google how to write the image to a file. A lot of answers are here [saving-image-and-then-loading-it-in-swift-ios](https://stackoverflow.com/questions/37344822/saving-image-and-then-loading-it-in-swift-ios). You can see that almost all the answers suggest to use UIImage API

```swift
let image: UIImage = ...
let data = image.jpegData(compressionQuality: 1)!
try data.write(to: ...)
```

that again requires us to work on UI thread. There is a better way to do this, as always CGImage goes to rescue us

```swift
func save(cgImage: CGImage, to url: URL) {
    let destination = CGImageDestinationCreateWithURL(url as CFURL, kUTTypeJPEG, 1, nil)!
    CGImageDestinationAddImage(destination, cgImage, nil)
    CGImageDestinationFinalize(destination)
}
```

### Final thoughts

From all examples above we learned that application memory management is not easy task as it can look at first. Answers from [stackoverflow.com](https://stackoverflow.com) not always best way to solve a problem, but can help you to find right direction for solving your issue. Working with images also not easy task, cropping, transformations be sure always eats device CPU/GPU/RAM that is why it is important to move these kinds of tasks on background to not block UI.
