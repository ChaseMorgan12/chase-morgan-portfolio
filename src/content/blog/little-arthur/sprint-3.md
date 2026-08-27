---
title: "Little Arthur Sprint 3"
description: "Enemies"
pubDate: "Mar 03 2026"
heroImage: "../../../assets/little-arthur-image.png"
project: "little-arthur"
---

Sprint three has come to a close and with it came a lot of progress towards game completion! This sprint I was able to complete 13 cards and the entire team completed 194 points. This sets a new record for work completed in one sprint for this team and sets a good precedence for future sprints if this is our velocity throughout the project.

This sprint I mostly worked on enemy implementation with a bit of other miscellaneous tasks along the way. I am proud to say that all enemies were complete during this sprint and all that remains are the bosses which won't take that long after! Overall, I believe that this sprint went really well and cannot wait to see where the next few sprints take us.

The entire enemy system is built upon the strategy pattern which was defined last sprint and is used for pretty much everything the enemy does besides high level logic checks. The below code is the general logic loop for the base enemy which basically defines default behavior during each state. Each subsequent enemy has its own class that inherits from the base enemy to handle its own logic and flow.

```c#
protected virtual void FixedUpdate()
{
    transform.rotation = Quaternion.Lerp(transform.rotation, _goalRotation, Time.fixedDeltaTime * 10);

    _animatorController.SetIsMoving(_agent.velocity.magnitude > 0.125f);
    switch (_currentState)
    {
        case EnemyState.Idle:
            foreach (Player player in Player.s_connectedPlayers)
            {
                if (Vector3.Distance(transform.position, player.transform.position) <= _enemyData.aggroRange)
                {
                    Alert(player.transform);
                    return;
                }
            }

            if (_enemyData.allowWander && _shouldWander)
            {
                _shouldWander = false;
                Vector2 newWanderLocation = (UnityEngine.Random.insideUnitCircle * _enemyData.wanderRange);

                if (NavMesh.SamplePosition(new Vector3(newWanderLocation.x, 0, newWanderLocation.y) + _spawnPosition, out NavMeshHit wanderHit, _enemyData.wanderRange, NavMesh.AllAreas))
                {
                    _agent.destination = wanderHit.position;
                    WanderCooldown();
                }
            }
            break;
        case EnemyState.Aggro:
            _currentTarget = GetNearestPlayerTransform();

            if (_agent.enabled && CurrentTarget)
            {
                _agent.destination = CurrentTarget.position;
            }

            if (_enemyData.allowDeaggro && (!CurrentTarget || Vector3.Distance(_currentTarget.position, transform.position) > _enemyData.deaggroRange))
            {
                CurrentState = EnemyState.Idle;
                _agent.destination = _spawnPosition;
                _currentTarget = null;
            }
            break;
        case EnemyState.Fallback:
            Vector3 dir = (transform.position - CurrentTarget.transform.position).normalized;
            Vector3 fallbackPosition = transform.position + (dir * _enemyData.fallbackRadius);

            if (NavMesh.SamplePosition(new Vector3(fallbackPosition.x, 0, fallbackPosition.z), out NavMeshHit hit, _enemyData.fallbackRadius, NavMesh.AllAreas))
            {
                _agent.destination = hit.position;
            }
            break;
        case EnemyState.Dead:
            break;
        default:
            break;
    }

    Vector3 direction = ((_agent.enabled ? _agent.destination : CurrentTarget.position) - transform.position).normalized;
    _goalRotation = Quaternion.Euler(transform.eulerAngles.x, Quaternion.LookRotation(direction, Vector3.up).eulerAngles.y, transform.eulerAngles.z);
}
```

The first enemy I worked on this sprint was the barbarian which is a very good one to start as it is the simplest. All that the barbarian needs is the ability to attack which is a fairly simple strategy. The core idea of the enemy attack strategy is to wait until the enemy is in range and if they are, keep trying to attack until they are on cooldown (attack has a cooldown state), are out of range, or has changed strategies. This logic flow allows the barbarian to attack and still engage with the player at runtime.

The next enemy I worked on was the spearman. The spearman is like the barbarian; however, they have a lunge attack instead of a standard attack. To fulfill this requirement a lunge strategy must be made and because we are using Unity's navigation mesh, we need to get creative with the implementation. The way I get around the navigation mesh is to basically just disable it, and the lunge the enemy forward a bit and reenable the navigation mesh. The logic flow basically works like this: the spearman is alerted, goes to the player, attempt to attack, disable nav-mesh, lunge using a rigidbody, wait until the lunge is complete, re-enable nav-mesh, and repeat! This is of course an oversimplification of the system, but that is how it works on a high level.

The next enemy finished was the archer. By far the archer is the most complicated (and janky) enemy in Little Arthur. The archer unlike every other enemy in the game is a ranged enemy and that means we have to change a lot of logic, or do we? You see, an archer just basically sends a projectile and when the projectile hits down we tell the object it was damaged. What if we just use the standard melee attack, but move its attack area to the object being hit instead and put the cooldown to zero seconds? This approach allows us to define an attack that can dynamically move positions and thus we cut down on a lot of extra work that is unneeded. We, however, do still need a strategy for shooting as we want this decoupled from the standard attack strategy. Basically, how it works is that it checks where the player is, finds the velocity needed to shoot an arrow to that location, spawn a projectile, apply the velocity, and when the arrow collides with something, update the attack script location to represent the arrow hit location. This way allows us to leverage already made systems and apply new behavior to them! The archer also needs to also fallback if the player is too close. For this I basically just disable the attack strategy and set the enemy destination in the opposite direction of the player if the player comes too close.

```c#
_owningEnemy.AnimatorController.TriggerAttack();
_timeSinceLastProjectile = Time.time;
GameObject projectile = MonoBehaviour.Instantiate(_projectile, _owningEnemy.transform.position + _owningEnemy.transform.forward + Vector3.up, _owningEnemy.transform.rotation);

_activeProjectiles.Add(projectile);

CollisionListener listener = projectile.GetComponent();
Rigidbody body = projectile.GetComponent();

if (!listener)
{
    listener = projectile.AddComponent();
}

if (!body)
{
    body = projectile.AddComponent();
}

float distance = Vector3.Distance(_owningEnemy.CurrentTarget.position, projectile.transform.position);
float time = distance / _velocity();

Vector3 delta = _owningEnemy.CurrentTarget.position - projectile.transform.position;
float velX = new Vector3(delta.x, 0, delta.z).magnitude / time;
float velY = (delta.y - 0.5f * Physics.gravity.y * time * time);
Vector3 vel = new Vector3(delta.x, 0, delta.z).normalized * velX;
vel.y = velY;

body.linearVelocity = vel;

listener.OnTriggerEnterEvent.AddListener(col =>
{
    if (_activeProjectiles.Contains(col.gameObject))
    {
        return;
    }
    _attack.AttackOneShot(listener.transform.position, false);
    listener.OnTriggerEnterEvent.RemoveAllListeners();
    _activeProjectiles.Remove(projectile);
    MonoBehaviour.Destroy(listener.gameObject);
});
```

The next thing completed at this point was the ability to push enemies back with attacks. For this I literally just take the lunge strategy and apply a negative direction to it. Best to reuse code whenever possible!

The next enemy was the berserker which is very cool and unique; however, doesn't actually contain any new strategies. Basically, when the berserker is enraged, they will become faster and apply an attack sphere around them that hurts everything within. For this I just applied the attack on the enemy itself with a radius specified.

The second to last enemy was the shieldbearer which also has no unique strategies. This one actually turns into a barbarian when its shield is broken. The easiest way to implement this is to just spawn a barbarian when the original shieldbearer is killed. The last enemy was the housekarl which is just a beefed-up barbarian. No extra code was really needed for this enemy.

The last things finished this sprint included was a simple scene transition system, local player spawn areas, plugin addition, damage implementation, and audio implementation. The scene transition system just loads a new scene on top of the currently active scene just to avoid transitioning from one scene to the next. The local player spawn areas look to see if all players are located within itself and if so, allows for more players to load in and have a device assigned. Plugin addition included adding the Substance 3D Plugin for Unity which allows our level designer more freedom with materials. The damage implementation was just hooking up the pre-existing damage system to the player and having animations play for them. Lastly, the audio implementation just included having the player grunt when hurt.

Overall, this sprint went really well! I do hope that my team can keep up the velocity because if so, our game is going to turn out amazingly. Thank you for reading!
