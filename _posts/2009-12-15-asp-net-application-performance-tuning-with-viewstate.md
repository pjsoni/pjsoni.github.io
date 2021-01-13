---
layout: single
title: ASP.net application performance tuning with ViewState
tags: [asp.net, c#, coding, performance tuning]
category: [blog]
comments: false

---

In past, I’ve started developing asp.net applications with the default settings of asp.net. Sometimes I found that the application is not responding quickly as it should be. I’ve dig in to this matter and figured out that the entire page is heavily loaded with the hidden page viewstate. Due to this length, the application is taking much long time to load in browser. To overcome this issue, the viewstate needs to be compressed using some kind of the compression.


![ASP.net application viewstate size after compression](//images/Viewstate1.png)


Below is the methods which I’ve used in my application for compressing the viewstate data.

```csharp
public static class Zip
{
    private const int BUFFER_SIZE = 65536;
    static int viewStateCompression = Deflater.NO_COMPRESSION;

    public static  int ViewStateCompression
    {
        get { return viewStateCompression; }
        set { viewStateCompression = value; }
    }

    public static byte[] Compress(byte[] bytes)
    {
        using (MemoryStream memoryStream = new MemoryStream(BUFFER_SIZE))
        {
            Deflater deflater = new Deflater(ViewStateCompression);
            using (Stream stream = new DeflaterOutputStream(memoryStream, deflater, BUFFER_SIZE))
            {
                stream.Write(bytes, 0, bytes.Length);
            }
            return memoryStream.ToArray();
        }
    }

    public static byte[] Decompress(byte[] bytes)
    {
        using (MemoryStream byteStream = new MemoryStream(bytes))
        {
            using (Stream stream = new InflaterInputStream(byteStream))
            {
                using (MemoryStream memory = new MemoryStream(BUFFER_SIZE))
                {
                    byte[] buffer = new byte[BUFFER_SIZE];
                    while (true)
                    {
                        int size = stream.Read(buffer, 0, BUFFER_SIZE);
                        if (size <= 0)
                            break;

                        memory.Write(buffer, 0, size);
                    }
                    return memory.ToArray();
                }
            }
        }
    }
}
```

`System.Web.UI.Page`  is having two methods `SavePageStateToPersistenceMedium(Object state)`  and `LoadPageStateFromPersistenceMedium()`  which are responsible for state management via viewstate. We need to override these methods in to our custom base class for compressing the viewstate.



```csharp
protected override void SavePageStateToPersistenceMedium(Object state)
{
    if (ViewStateCompression == Deflater.NO_COMPRESSION)
    {
        base.SavePageStateToPersistenceMedium(state);
        return;
    }

    Object viewState = state;
    if (state is Pair)
    {
        Pair statePair = (Pair)state;
        PageStatePersister.ControlState = statePair.First;
        viewState = statePair.Second;
    }

    using (StringWriter writer = new StringWriter())
    {
        new LosFormatter().Serialize(writer, viewState);
        string base64 = writer.ToString();
        byte[] compressed = Zip.Compress(Convert.FromBase64String((base64)));
        PageStatePersister.ViewState = Convert.ToBase64String(compressed);
    }
    PageStatePersister.Save();
}

protected override Object LoadPageStateFromPersistenceMedium()
{
    if (viewStateCompression == Deflater.NO_COMPRESSION)
        return base.LoadPageStateFromPersistenceMedium();

    PageStatePersister.Load();
    String base64 = PageStatePersister.ViewState.ToString();
    byte[] state = Zip.Decompress(Convert.FromBase64String(base64));
    string serializedState = Convert.ToBase64String(state);

    object viewState = new LosFormatter().Deserialize(serializedState);
    return new Pair(PageStatePersister.ControlState, viewState);
}
```


To enable compression we need to override OnInit method from `System.Web.UI.Page` class.

```csharp
protected override void OnInit(EventArgs e)
{
    Zip.ViewStateCompression = Deflater.BEST_COMPRESSION;
    base.OnInit(e);
}
```


I’ve used Sharp ZipLib for the compression and found the interesting results. On an average I found that the entire viewstate is compressed nearly up to 30% of the original viewstate size.


![ASP.net application viewstate size after compression](/assets/images/viewstate2.png)

Below is a small piece of code which is showing the viewstate data size.


```csharp
public class ViewstateSizeControl : Label
{
    private const string SCRIPT = "document.getElementById('{0}').innerText = 'Size of Viewstate: ' + document.getElementById('__VIEWSTATE').value.length;";

    protected override void OnLoad(EventArgs e)
    {
        if (Visible)
        {
            Page.ClientScript.RegisterStartupScript(
                typeof(Page),
                UniqueID,
                string.Format(SCRIPT, ClientID),
                true);
        }
        base.OnLoad(e);
    }
}
```


To use the control, simply put the control on the aspx page and compare the size of viewstate before and after compression.

Hope this will help you all in your upcoming/current assignment.

Happy Programming..
