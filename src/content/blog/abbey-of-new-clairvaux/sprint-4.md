---
title: "Abbey Of New Clairvaux AR App Tour Sprint 4"
description: "Fully Trained Model"
pubDate: "Apr 26 2026"
heroImage: "../../../assets/ar-tour-backdrop.jpg"
---

Sprint four is over and with it came a lot of progress towards finishing this project! The main goal for this sprint was to get a test build to show off to our client at New Clairvaux. For this we needed to take our pre-trained model and train it with the actual data needed. For this we had a few team members (including myself) go on-site to capture the necessary data. Very little programming was done this sprint as most programming was completed last sprint and now, we just need data. 14 cards were completed in total during sprint four.

<figure class="gif-card">
  <img src="/gifs/Abbey_Full_Object_Detection.gif" alt="Full Object Detection" />
  <figcaption>Full Object Detection</figcaption>
</figure>

The first thing completed this sprint was pretty much the only coding done which related to capturing bounding box data from a scan. This isn't too difficult but requires us to create more tensors and more channels in our model which does increase runtime complexity. The scanning service once a scan is complete will send the data to the scan user interface service. The scan user interface service then takes that data to give translation data to a pre-existing bounding box visual element to draw over the camera visual element.

```c#
if (Texture != null && _model != null)
{
    _scanInProgress = true;
    OnScanStarted?.Invoke();
    Tensor tensorIn = new Tensor(new TensorShape(1, 3, 640, 640));
    TextureConverter.ToTensor(Texture, tensorIn);
    var enumerator = _worker.ScheduleIterable(tensorIn);
    int layer = 0;
    while (enumerator.MoveNext())
    {
        layer++;
        if (layer % 5 == 0)
        {
            await Awaitable.NextFrameAsync();
        }
    }
    Tensor confidenceTensor = _worker.PeekOutput(0) as Tensor;
    Tensor classTensor = _worker.PeekOutput(1) as Tensor;
    Tensor coordsTensor = _worker.PeekOutput(2) as Tensor;
    var confidenceCopy = await confidenceTensor.ReadbackAndCloneAsync();
    var classCopy = await classTensor.ReadbackAndCloneAsync();
    var coordsCopy = await coordsTensor.ReadbackAndCloneAsync();
    float confidence = confidenceCopy[0];
    int classIndex = classCopy[0];
    if (confidence > k_ConfidenceThreshold)
    {
        string name = _classLabels[classIndex];
        Debug.Log($"Object detected! Class: {name} Confidence: {confidence}");
        BoundingBox box = new BoundingBox()
        {
            position = new Vector2(coordsCopy[0, 0] * 1 - 640 / 2, coordsCopy[0, 1] * 1 - 640 / 2),
            width = coordsCopy[0, 2] * 1,
            height = coordsCopy[0, 3] * 1
        };
        OnObjectScanned?.Invoke(name);
        ObjectScannedBox?.Invoke(box);
    }
    OnScanFinished?.Invoke(confidence > k_ConfidenceThreshold);
    _scanInProgress = false;
}
```

After that I made some visual display for when the user is scanning. Before, it was just scanning under the hood, but now it will tell the user when it is scanning, if it found something, and an error if nothing was found. This is all handled via the event in the scan service that invokes when a scan starts and is finished. A variable is then set in a data object which the UI binds to via data bindings.

<figure class="gif-card">
  <img src="/gifs/Abbey_Bounding_Boxes.gif" alt="Bounding Boxes" />
  <figcaption>Bounding Boxes</figcaption>
</figure>

After all this was done, we went on-site to take pictures of as many stops within the app as possible. After the pictures were taken, an application to label was searched for. We landed on Label Studio as it is the successor to Label IMG and is open source unlike most labelling software.

Next came the lengthy part, labelling and validating the data. Every image taken needs to be manually labelled one by one. Most of our stops have around 600 images (which is still on the smaller side), so it takes hours to label each stop. I was assigned the first two just to make sure our model was even close to being able to correctly identify each stop. Luckily, our model was able to successfully identify each stop with no trouble whatsoever.

Overall, this sprint started out stressful as we were still unsure if we would be able to meet our deadline. With this success it completely got rid of any worries I had about this project and now I believe we will make our May 14th deadline no problem! Thanks for reading!

<figure class="gif-card">
  <img src="/gifs/Abbey_Text_Display.gif" alt="Text Diplay" />
  <figcaption>Text Display</figcaption>
</figure>
