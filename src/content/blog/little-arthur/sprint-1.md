---
title: "Little Arthur Sprint 1"
description: "Base Systems"
pubDate: "Feb 05 2026"
heroImage: "../../../assets/little-arthur-image.png"
project: "little-arthur"
---

Sprint 1 has already been completed and with it came with many of our base systems done. This project I am more or less taking a backseat to implementation of most frontend systems and instead focusing on backend systems. This sprint had me completing a custom multi-player input manager, player camera systems, project setup, and the framework for our enemy system.

I've always had a love-hate relationship with Unity's built-in input manager and try to avoid using it as much as possible. I really hate systems that act as a black box where you cannot see what is happening nor get components it is creating. Thus, I decided to make my own input manager. Is it overkill? Yes! Is it cool and more controllable? Also, yes! The main pain point that I come across with the input manager is the spawning and de-spawning of players. The custom input manager uses a custom settings asset that is fully editable in the inspector as seen to the left of this text!

Something I do when programming new systems like this is to try different programming language features. I have been a longtime fan of some of C#'s functional patterns like pattern matching, so I decided to make the logic of getting a spawn position into a positional pattern. This is pretty much a switch statement that returns a type at the end. Here is an example of what it looks like in our code. Note: this is the code shortened without input checking or comments.

```c#
private Vector3 GetSpawnPosition(int index) => _settings.SpawnType switch
{
    PlayerSpawnType.World =>
    _settings.PlayerSpawnPoints[index],

    PlayerSpawnType.RelativeToCamera =>
    Physics.Raycast(Camera.main.transform.position, Camera.main.transform.forward, out RaycastHit hit) ?
    (hit.point + _settings.PlayerSpawnPoints[index]) :
    _settings.PlayerSpawnPoints[index],

    PlayerSpawnType.RelativeToPlayerOne =>
    _settings.PlayerSpawnPoints[index] +
    (Player.s_connectedPlayers.Count > 0 ?
    Player.s_connectedPlayers[0].transform.position :
    Vector3.zero),

    _ => _settings.PlayerSpawnPoints[index]
};
```

The part of that input system that took the longest was actually the systems that allow the manager to run. I have had the recent experience of getting acquainted with Unreal C++ programming and through it really enjoyed the Subsystems that are built into the engine. I decided to completely port over that system into Unity C# just to have a better time managing my systems. This was a daunting task that took many hours to get working, but I believe it was well worth it. The main way of how it works is it has a main GameInstance that is the only singleton in the game. It then will spawn in any subsystems it finds in the assembly of the game and initialize them accordingly. There are three types of subsystems, a IGameService which has the life cycle of the game, ISceneService which has the lifetime of the scene it was created in, and IGameLoopService which provides common MonoBehaviour functions like Update, FIxedUpdate, and LateUpdate. All of these allow any class to have a particular lifetime and exist outside of the main game loop for complete control over the game systems. It also has a Bootstrapper class that initializes everything that is necessary for the system to function. Note: the reason for them being called Services is to make it in line with C# naming conventions instead of C++. Here is some code that is a part of main GameInstance that spawns in all GameServices right before the first scene is loaded.

```c#
public void Initialize_Internal()
{
    if (_instance == null)
    {
        _instance = this;
    }
    else
    {
        return;
    }

    //Initialize all Game Services

    Assembly assembly = AppDomain.CurrentDomain.GetAssemblies().First(assembly => assembly.GetName().Name == "Assembly-CSharp");

    List types = assembly.GetTypes().ToList();

    types = types.Where(type => type.GetInterface(nameof(IGameService)) != null).ToList();

    foreach (Type type in types)
    {
        IGameService gameService = (IGameService)Activator.CreateInstance(type);

        gameService?.Initialize();

        BindGameLoopService(type, gameService);

        _gameServices.Add(gameService);
    }

    SceneManager.activeSceneChanged += OnSceneLoad;

    GameObject bridge = new GameObject("gameLoopBridge");
    bridge.AddComponent();
    UnityEngine.Object.DontDestroyOnLoad(bridge);

    Initialize();
}
```

The next thing I worked on was the camera system which proved to be a lot easier to implement than I originally thought. I have a lot of experience with making these types of systems and know they can sometimes be a nightmare; however, due to the framework that I made before it was a breeze. What I basically did was create a new IGameService which might be changed to a ISceneService later on, but for now IGameService works. With that I added the IGameLoopService interface which gave me access to the LateUpdate function in Unity. In this loop it basically looks to see if its frustrum size, position, or rotation needs updates and linearly interpolates each value necessary. Everything is also data driven so if the designer wants a change, they can add it! Here is some of the code that handles the resizing.

```c#
if (_canResize)
{
    if (_camera.orthographicSize != _cameraSizeGoal)
    {
        _camera.orthographicSize =
            Mathf.Lerp
            (
                _camera.orthographicSize,
                _cameraSizeGoal,
                Time.deltaTime * _settings.CameraResizeLerpSpeed
            );
    }
}
```

The last real implementation that I did during this sprint was the Enemy framework. I based it off of what I did during the development of Steel Specter, but with less anonymous/lambda expression bloat. A big pain point of that system was too generic and caused a bunch of impossible to debug errors. This system will be a lot more specialized to just handling strategies instead of everything under the sun. It is a basic strategy pattern where one strategy can be active at once. Here is some of that code that makes it work!

```c#
public abstract class Enemy : MonoBehaviour
{
    protected IStrategy _currentStrategy;

    public IStrategy CurrentStrategy => _currentStrategy;

    public virtual void ApplyStrategy(T strategyInstance) where T : IStrategy
    {
        if (strategyInstance != null && (IStrategy)strategyInstance != _currentStrategy)
        {
            _currentStrategy?.Cancel();
            _currentStrategy = strategyInstance;
            _currentStrategy.Execute();
        }
    }

    public virtual void CancelStrategy()
    {
        _currentStrategy?.Cancel();
        _currentStrategy = null;
    }
}
```

The rest of this sprint was spent on initial project set up and the Wwise integration in Unity. Overall, this sprint was a bit slow in terms of my previous work; however, I am fulfilling the role of a background programmer as I unfortunately have a lot more duties to attend to and was told to take on less work due to this. However, I always like to experiment and when I start programming it can be very hard to stop! Thanks for reading!
