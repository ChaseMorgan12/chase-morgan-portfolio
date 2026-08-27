---
title: "Asynchronous Generic Save System"
description: "Save System"
pubDate: "Jun 22 2026"
heroImage: "../../../assets/ff-save-system.png"
project: "fungi-frenzy"
---

Save systems are essential to keeping track of user data from each play session and can have a lot of issues with scalability. Writing to disk can be a very expensive operation and when run from the main thread, can introduce some annoying stutters to a game session. Unity Awaitables can be used to aid with this in Unity 6+ to offload this work to background worker threads extremely easily.

<h5 style="text-align: center;">Awaitables</h5>
Starting with Unity 2023.1 Awaitables came to Unity C# to align with C#'s asynchronous coding model backed by C# Tasks. This is a huge improvement as coroutines were the main way to run code at different times. With Awaitables you can now run code without having a MonoBehaviour required for Coroutines and you can even return values with them. For example this code spawns an object asynchronously and returns the object after spawned:

```c#
//Unity methods can also be marked as async the same as IEnumerator for coroutines!
private async void Start()
{
    GameObject[] objects = await SpawnInstances(prefab, 10);
    Debug.Log("Spawning finished!");
}

public Awaitable<GameObject[]> SpawnInstances(GameObject prefab, int count) => InstantiateAsync(prefab, count);
```

This allows Unity to spawn each prefab and you can get the exact moment it is finished!

<h5 style="text-align: center;">Utilizing Awaitables for a File System</h5>
Awaitables are incredibly useful and a big part is being able to offload work on a background thread and come back to the main thread when that work is finished. This is backbone of how the file system for <i>Fungi Frenzy</i> works! When a save command is sent to a save game object it will do its surface level validation like checking if the data is null and cloning the data in memory. After this it goes to a background thread where it waits asynchronously for the file it needs to edit to be free. This is to prevent accidentally corrupting the file since we could be writing to it from multiple threads. A semaphore is used to lock the file as C# has a built-in await function for them. Once the file is verified as being safe to write to it will attempt to write to it atomically which means to fully write or not at all. Here is an example of what that code could look like:

```c#
private bool WriteToDiskAtomic(TData data)
{
    string uniqueTempLocation = $"{_fileLocation}.{Guid.NewGuid()}.tmp";

    try
    {
        string jsonString = JsonConvert.SerializeObject(data, _jsonSerializer);
        File.WriteAllText(uniqueTempLocation, jsonString);

        if (File.Exists(uniqueTempLocation))
        {
            if (File.Exists(_fileLocation))
            {
                long currentDiskTimestamp = PeekDiskTimestamp(_fileLocation);

                if (data.DataTimestamp < currentDiskTimestamp)
                {
                    return false;
                }

                File.Replace(uniqueTempLocation, _fileLocation, null);
            }
            else
            {
                File.Move(uniqueTempLocation, _fileLocation);
            }
        }

        return true;
    }
    catch (Exception ex)
    {
        Debug.LogError($"Write failed: {ex.Message}");
        return false;
    }
    finally
    {
        if (File.Exists(uniqueTempLocation))
        {
            try
            {
                File.Delete(uniqueTempLocation);
            }
            catch{}
        }
    }
}
```

Note the finally section as it is a very useful feature of C# that defers code to run at the end of the try catch block. This is very useful for cleaning up any allocations or releasing file locks. For this code block it deletes the tmp file we create.

Now that we have an actual way to write data atomically we need an interface to actually save. There are two methods that we use is one to force a save and another to save asynchronously when available. The latter is the standard save that can be used throughout normal gameplay while the former should only be reserved for last minute or emergency saves. Here is some of that code for the standard saving, <b>NOTE:</b> we have a delegate that automatically does a memberwise clone of the data which is what \_clone(data) is doing.

```c#
public async Awaitable<bool> SaveAsync(TData data)
{
    if (data == null)
    {
        Debug.LogError("Tried to save data that was null!");
        return false;
    }

    TData snapshot = _clone(data);

    long timestamp = DateTime.UtcNow.Ticks;
    snapshot.DataTimestamp = timestamp;
    snapshot.DataVersion = Application.version;

    Debug.Log($"Saving {data.GetType()}... Waiting for {_fileLocation} to be safe to write to...");

    await Awaitable.BackgroundThreadAsync();

    await _filelock.WaitAsync();

    try
    {
        return WriteToDiskAtomic(snapshot);
    }
    finally
    {
        _filelock.Release();
        Debug.Log($"{data.GetType()} saved... File lock released...");
    }
}
```

For saving asynchronously, taking a snapshot is crucial as that data could be updated throughout the save process leading to corrupted data. In <i>Fungi Frenzy</i> we use record classes to store data as they come pre-equipped with a memberwise clone method which can be performed with the code below:

```c#
public record Data
{
    public int val = -1;
}

static int main()
{
    Data data = new();

    //This performs a memberwise clone!
    Data copy = data with {};

    return 0;
}
```

For this save system we just pass in this clone method as we cannot use a record as a constraint for generics to enforce the clone method. This does mean we can pass in normal classes with a generic clone method though!

<h5 style="text-align: center;">File Formatting and Flow</h5>

For <i>Fungi Frenzy</i> we track data using JSON format which has the benefit of easy editing, especially for settings; however, does come at the cost of increased file size and increased cost in performance. This is why this system is incredibly useful as our player settings needs to include a lot of different data types.

Games are also not straightforward with flow as crashes can happen or a user can choose to force end your own game. For this async methods can not write fast enough so it is highly recommended to have a force method that attempts to save as soon as possible. for this you should provide an overwrite boolean flag to force itself to overwrite with the newest data so you aren't waiting for the file to unlock.

<h5 style="text-align: center;">Conclusion</h5>
Overall, this save system allows very dynamic saving which is necessary for a lot of games. For <i>Fungi Frenzy</i> we save data asynchronously every minute of in game time, but will do an emergency save whenever game focus is also lost. Thanks for reading!
