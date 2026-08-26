---
title: "Abbey Of New Clairvaux AR App Tour Sprint 3"
description: "YOLO Model"
pubDate: "Apr 12 2026"
heroImage: "../../../assets/ar-tour-backdrop.jpg"
---

Sprint three is over and with it came some good work on the project! This sprint we were able to get an actual build out for testing. A lot was learned during this sprint, especially about dealing with lower performant platforms like web builds for Unity. Unfortunately, due to the complexity of the tasks assigned this sprint only three cards were completed. The main thing focused on during this sprint was implementing the main feature for this product which is the scanning feature.

The scanning feature uses a common machine learning model known as YOLO (You Only Look Once) which basically divides the screen space into a grid and trains a model uses neighboring pixels to build connections to identify classes. This is useful for us since a trained model can be used to pick out objects with incredible accuracy. Usually, YOLO is used in situations where you need to detect many objects that could slightly differ in shape/size; however, it still is great for general scanning purposes.

This scanning feature basically takes a snapshot of the current camera texture every ~30 frames and then runs it through the model that was trained using thousands of images. It will only go through roughly 5 layers every frame as to keep performance at around 30 frames per second just so we don't accidently crash the user's device. YOLO is designed to be performant; however, we are still doing billions of floating-point operations a second, so we also want to offload it to the GPU when necessary. Unfortunately, due to WebGL we don't have access to CPU multi-threading or GPU compute shaders, so we have to use GPU pixel shaders to transfer work to the GPU. This does come with a loss of performance, but it isn't really that noticeable in practice.

```c#
if (Texture == null)
{
    if (GameInstance.Get().GetGameService() != null)
      Texture = GameInstance.Get().GetGameService().WebCamTexture;
}
if (Texture != null && _model != null)
{
    _scanInProgress = true;
    //Scan
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
    var confidenceCopy = await confidenceTensor.ReadbackAndCloneAsync();
    var classCopy = await classTensor.ReadbackAndCloneAsync();
    float confidence = confidenceCopy[0];
    int classIndex = classCopy[0];
    if (confidence > 0.5f)
    {
        string name = _classLabels[classIndex];
        Debug.LogError($"Object detected! Class: {name} Confidence: {confidence}");
    }
    _scanInProgress = false;
}
```

After we take that snapshot, we will start up the worker using Unity Sentis which dynamically loads the model at runtime in a way that impacts performance the least. Using C# asynchronous methods allows us to clone each tensor off of the main thread to read and gather data from the GPU. Using this data, we could know various things like if it detected anything, the confidence level of the detection, the location of the detection, and the class of the detected object. We can also find even more data like calculated tensor values, but they aren't relevant for display.

```c#
var model = ModelLoader.Load(Asset);
var graph = new FunctionalGraph();
var input = graph.AddInput(model.inputs[0].dataType, model.inputs[0].shape);
var output = Functional.Forward(model, input);
var raw = output[0];
var scores = raw[0, 4..84, ..];
var classIdsForEachBox = Functional.ArgMax(scores, 0, false);
var maxConfForEachBox = Functional.ReduceMax(scores, new int[] { 0 }, false);
var bestBoxIndex = Functional.ArgMax(maxConfForEachBox, 0, false);
var tempBestBox = Functional.Unsqueeze(bestBoxIndex, 0);
var bestClassId = Functional.Gather(classIdsForEachBox, 0, tempBestBox);
var finalClassId = Functional.Squeeze(bestClassId);
var maxConfidence = Functional.ReduceMax(scores, new int[] { 0, 1 }, false);
_model = graph.Compile(new FunctionalTensor[] { maxConfidence, finalClassId });
_worker = new Worker(_model, BackendType.GPUPixel);
```

<figure class="gif-card">
  <img src="/gifs/Abbey_YOLO_Detection.gif" alt="POI Completion" />
  <figcaption>YOLO Object Detection</figcaption>
</figure>
