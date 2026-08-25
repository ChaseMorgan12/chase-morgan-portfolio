---
title: "Little Arthur Sprint 2"
description: "Player Abilities, Movement and Enemies"
pubDate: "Feb 18 2026"
heroImage: "../../../assets/little-arthur-image.png"
---

Sprint 2 is finished and with it came 18 points of completion! A lot was completed this sprint, and I am extremely proud of the team with how much we pushed through especially with sprint 1 being as slow as it was. A lot of the work this sprint was related to player abilities, movement, and enemies. I also did do a bit of audio, animation, and model implementation.

This sprint I started with a couple enemy related cards starting with a simple prefab spawner for the enemies. The main functionality of this prefab spawner is to instantiate a prefab when triggered. It has three different ways to trigger a spawn with those being InvokeRepeating, OnTrigger, or Custom. A common use case for Custom is to use the animator to trigger an animation event to spawn the prefab, but any class or method can invoke the spawning of the prefab as Custom mode just means it won't automatically spawn the prefab. It serves as a one stop shop for everything related to spawning prefabs!

The other enemy related thing I did was give the enemy the ability of traversal! It is very rudimentary; however, it is coded perfectly to allow us to build on top of this base with enemy states so we can apply strategies in that way!

```c#
public void SpawnPrefab()
{
    if (_trackAliveObjects)
    {
        //Leverage the Linq library to automatically delete items that are null from the spawn array.
        PrefabsSpawned = PrefabsSpawned.Where(item => item != null).ToList();
    }

    _count = PrefabsSpawned.Count;

    bool notValid = _maxSpawn >= 1 && _maxSpawn <= _count;

    notValid |= !CanSpawn;
    notValid |= (Time.time - _lastSpawnTime) < _spawnRate;

    if (notValid)
    {
        return;
    }

    int spawnTransform = UnityEngine.Random.Range(0, _spawningTransforms.Length);
    GameObject spawnedPrefab = Instantiate(_prefab, _spawningTransforms[spawnTransform].position, _spawningTransforms[spawnTransform].rotation);

    OnSpawn?.Invoke(spawnedPrefab);

    if (_destroyOnSpawn)
    {
        CanSpawn = false;
        Destroy(gameObject);
        return;
    }

    PrefabsSpawned.Add(spawnedPrefab);
    _lastSpawnTime = Time.time;
}
```

Pretty much the rest of this sprint time was spent on perfecting our players. I started by allowing the player to die which was as simple as adding a function that gets called when their health hits zero. As a temporary death we just unload the player from the world and wait until they respawn to spawn them back in. In the future we will most likely play a death animation and have some sort of way to bring them back.

The next thing was making all of the special abilities for the player. I started with making a base class that leverages the PlayerAttack class I made last sprint which pretty much has all the logic and timers needed to make these special attacks. I then made a class for each attack that has unique implementation except for King Arthur. King Arthur is just a larger attack, so I just use the default SpecialAttack class with different starting parameters. Sir Gawain was next which is just a spherical attack which I solved by adding a boolean flag to the base PlayerAttack class so spherical attacks work. Sir Percival has my personal favorite attack where they slam their weapon into the ground sending anything in their path flying! I accomplished this by binding a function to the OnEnemyHit event within the PlayerAttack script to know when the player hits something that needs to be launched. I then get the direction between the player and object and launch them depending on the magnitude set in the inspector. Lastly, I programmed Sir Lancelot's special ability which is just a dash with invincibility frames. I basically check if they can attack and if they can, apply an impulse force to give a forward thrust.

```c#
public class SirPercivalAbility : SpecialAbility
{
    private float _launchMagnitude = 5, _upVelocity = 0.5f;

    private void ApplyLaunch(Collider collider, float _)
    {
        if (collider.TryGetComponent(out Rigidbody rigidbody))
        {
            Vector3 direction = (collider.transform.position - _owningPlayer.PlayerMeshObject.transform.position).normalized;

            direction.y = 0;

            rigidbody.AddForce((direction * _launchMagnitude) + (Vector3.up * _upVelocity), ForceMode.Impulse);
        }
    }

    public SirPercivalAbility(Player player) : base(player)
    {
        _specialAttack.OnEnemyHit += ApplyLaunch;
    }

    public override void Attack()
    {
        base.Attack();
    }

    protected override void SetupAbility()
    {
        if (_owningPlayer.Class != PlayerClass.None)
        {
            _specialAttackParamAsset = s_assetDictionary[_owningPlayer.Class];
            _specialAttack = new PlayerAttack(_owningPlayer, _specialAttackParamAsset.AttackParameters, true);

            _launchMagnitude = _specialAttackParamAsset.CustomParameters.First(pair => pair.Value1 == "launchMagnitude").Value2;
            _upVelocity = _specialAttackParamAsset.CustomParameters.First(pair => pair.Value1 == "upVelocity").Value2;
        }
    }
}
```

Next on the to-do list was importing the player alongside implementing the animations. For this I took a page out of Unreal's book and created a dedicated animation controller similar to Unreal Engine's AnimationInstance class. This allows me to have a base class that automatically binds animation hashes with their name so I can easily vet any parameters. I then made a PlayerAnimatorController that will act as a global controller instance for all players. Basically, I instantiate this alongside the player so that whenever I want the player to move, I can just call it instead of getting reference to the animator component, checking if the parameter exists, and then setting the value. This in my opinion just makes things simpler with one unified call.

I retroactively fit all animations related to moving into the codebase. I also made all animations calls generic so that when we add more in for certain characters their animations will automatically play with zero extra code! The only thing this system doesn't do is lock the Animator so that only the class can edit like Unreal, but with Unity's architecture this is just not really feasible. The system actually went through a couple revisions where the first one worked; however, was way over-engineered. It was meant to be a onetime create and forget class to just handle everything. This approach works but just didn't really make sense with the architecture of our own game. I decided to take the simple approach next and just have it bind parameters on start and have a couple helper functions. All other implementation is on the specific instance of the controller.

```c#
[RequireComponent(typeof(Animator))]
public AnimatorController : MonoBehaviour
{
    protected Animator _animator;

    protected IDictionary _parameters = new Dictionary();

    protected virtual void Awake()
    {
        _animator = GetComponent();

        foreach (var parameter in _animator.parameters)
        {
            _parameters.Add(parameter.name, parameter.nameHash);
        }
    }
}
```

The last few things I did this sprint involved implementing a dash with invincibility frames, create a movement controller, and music implementation. The dash was easy as it was already implemented through Sir Lancelot's special ability, but I did add a few extra parameters so that the designer can fine tune it! The movement controller was originally a job for another programmer; however, it did not work and we needed one really quickly for a build, so I added it which is great because before I didn't have any of the animations linked to actual movement. Lastly, the music implementation was added so that the level designer can easily bind a AKAudioEvent (Wwise sound) to a specific scene to be loaded. I decided to go by name only because if going on indexes the scene needs to be included in the build which isn't always the case.

Overall, I feel like this sprint was a lot better than the first one and if we stay on track, we will be able to finish this game by the deadline no problem! Thank you for reading!
