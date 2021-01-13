---
layout: single
title: How to show thumbnails of XPS documents.
tags: [c#, coding, xps]
category: [blog]
comments: false
#JANUARY 25, 2010 BY PRAVESH SONI

---

Recently I came across the requirement for displaying thumbnails of XPS document in SharePoint document library. As a hardcore developer, I’m not in favor of readymade third -party components unless and until you really need them. Hence, I’ve decided to create an image generator myself. I explored the native .net methods for reading the XPS documents. After exploring I found that in .net framework 3.0, there is a managed dll called ReachFramework.dll is having all the necessary classes and methods for reading and writing of XPS document.

### Classes
![Class diagram - XPS document thumbnail generation](/assets/images/ClassDiagram.jpg "Class diagram - XPS document thumbnail generation")

### XpsThumbnail class

This is the one and only class which is actually generates the thumbnails from XPS documents using GenerateThumbnail method.

```csharp
converter.GenerateThumbnail();
```

### Properties

| Property          | Description                                   |
| ----------------- | --------------------------------------------  |
| OutputFormat	    | Output format for the generated image.        |
| OutputQuality	    | Quality for the generated image.              |
| OutputStream	    | MemoryStream object to be returned            |
| XpsFileName	    | XPS document whose image needs to be gerated. |



### Complete Class Listing

```csharp
///
/// Provides methods for converting XPS document in to various image format
///
public class XpsImage
{
    private BitmapEncoder bitmapEncoder = null;
    ///
    /// Default constructor
    ///
    public XpsImage()
    {
    }

    ///
    /// Sets the XPS file to be read
    ///
    public String XpsFileName { private get; set; }

    ///
    /// Gets or Sets the image format for thumbnail
    ///
    public OutputFormat OutputFormat {get; set; }

    ///
    /// Gets or Sets the image quality for thumbnail
    ///
    public OutputQuality OutputQuality { private get; set; }

    ///
    /// Returns the Memory stream of generated thumbnail
    ///
    public MemoryStream OutputStream { get; private set; }

    ///
    /// Generate the thumbnail of given document and populates the ThumbnailStream property
    ///
    public void GenerateThumbnail()
    {
        XpsDocument xpsDocument = new XpsDocument(this.XpsFileName, FileAccess.Read);
        FixedDocumentSequence documentPageSequence = xpsDocument.GetFixedDocumentSequence();

        string fileNameWithoutExtension = Path.GetFileNameWithoutExtension(this.XpsFileName);
        string fileExtension = string.Empty;

        switch (this.OutputFormat)
        {
            case OutputFormat.Jpeg:
                fileExtension = ".jpg";
                bitmapEncoder = new JpegBitmapEncoder();
                break;
            case OutputFormat.Png:
                fileExtension = ".png";
                bitmapEncoder = new PngBitmapEncoder();
                break;
            case OutputFormat.Gif:
                fileExtension = ".gif";
                bitmapEncoder = new GifBitmapEncoder();
                break;
            default:
                fileExtension = ".jpg";
                bitmapEncoder = new JpegBitmapEncoder();
                break;
        }

        double imageQualityRatio = 1.0;
        switch (this.OutputQuality)
        {
            case OutputQuality.Low:
                imageQualityRatio /= 2.0;
                break;

            case OutputQuality.Good:
                imageQualityRatio *= 2.0;
                break;

            case OutputQuality.Super:
                imageQualityRatio *= 3.0;
                break;
            default:
                imageQualityRatio *= 1.0;
                break;
        }

        DocumentPage documentPage = documentPageSequence.DocumentPaginator.GetPage(0);
        RenderTargetBitmap targetBitmap = new RenderTargetBitmap((int)(documentPage.Size.Width * imageQualityRatio), (int)(documentPage.Size.Height * imageQualityRatio), 96.0 * imageQualityRatio, 96.0 * imageQualityRatio, PixelFormats.Pbgra32);
        targetBitmap.Render(documentPage.Visual);

        bitmapEncoder.Frames.Add(BitmapFrame.Create(targetBitmap));
        string str4 = string.Format("{0}{1}", fileNameWithoutExtension, fileExtension);
        MemoryStream memoryStream = new MemoryStream();
        bitmapEncoder.Save(memoryStream);
        this.OutputStream = memoryStream;
        xpsDocument.Close();
    }
}
```
### Using Code

The class itself is the simple class and made for simplicity. Below is small snippet how to utilize the class.

```csharp
converter.XpsFileName = “XPSdocument.xps”;
converter.OutputFormat = OutputFormat.Png;
converter.OutputQuality = OutputQuality.Super;
converter.GenerateThumbnail();
```

### Conclusion

The sample application provided with this article just demonstrated the utilization of the code. In the demo application you need to save the image by invoking context menu from the thumbnail to see the output as I’m using the in memory image stream to display image in to picture box.

And please don’t hesitate to provide feedback, good or bad! I hope you enjoyed this article.
