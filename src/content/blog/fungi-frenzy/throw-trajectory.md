---
title: "Fast and Accurate Throw Trajectory Arc using Unity Jobs and Burst"
description: "Fast throw trajectory calcuator and displayer"
pubDate: "Jul 10 2026"
heroImage: "../../../assets/ff-throw-trajectory.gif"
project: "fungi-frenzy"
---

One of the main elements of <i>Fungi Frenzy</i> is the throwing mechanic. Thus a lot of care needs to be put into making the feature as easy to use and interpret as possible. A magnitude bar works but can lead to a lot of issues with the player needing to guess what each interval does. This solution using Jobs and Burst aims to solve this while providing no loss in performance.

Early renditions of this feature handled everything on the main thread within the update loop of Unity and was becoming an issue optimizing performance over accuracy. It maintained solid FPS by only simulating the physics at various steps turning 1000 operations into just around 10. This works but comes at the cost of accuracy, and in a game like this that is not a compromise we want to make. It then gets even more complicated when we throw mesh generation along the arc. It was obvious we needed to start from scratch and implement a new system for this, so that is what we did.

<h5 style="text-align: center;">Decoupling</h5>
The first order of business was to decouple the logic as it originally ran on the PlayerController class which is fine; however, is better suited as its own self-contained system where it is handled by two separate classes, one for the calculations and one for the display. This way we obfuscate code and only expose what is necessary like the start and cancel methods.

The two classes used are ThrowTrajectoryCalculator and ThrowTrajectoryDisplayer and in those classes include a couple private structs which are the jobs themselves. The classes themselves hand off data initialized with them to the jobs themselves and schedules the completion of the job. The PlayerController should really not handle data and is more of a class that handles input and relays it to other classes. This way we can just create and forget about it at the start of our game.

<h5 style="text-align: center;">Calculation</h5>
Calculation is actually not too advanced as it just needs to calculate the difference in position from a to b within an set array of positions. To do this we just use a base job that is not running in parallel and instead is just running on a separate thread. We set the index to the position value stored and then operate a few calculations and bam! we are done and ready to update the local position value for the next index.

There isn't much going on here as we want physics calculations to follow how Unity calculates them using rigibodies to make our arc as accurate as possible. This part is just calculating the arc with a set step count usually at around 1000 which means 1000 positions are calculated.

```c#
[BurstCompile]
private struct PhysicsPredictionJob : IJob
{
    [WriteOnly] public NativeArray<float3> positions;

    public float3 start;
    public float3 velocity;
    public float mass;
    public float drag;
    public float gravityY;
    public float step;

    public void Execute()
    {
        float3 position = start;
        float3 currentVelocity = velocity;

        for (int index = 0; index < positions.Length; index++)
        {
            positions[index] = position;

            float dragFactor = drag * step;
            float dampening = 1f - math.clamp(dragFactor, 0, 1);

            currentVelocity *= dampening;
            currentVelocity.y += gravityY * step;

            position += currentVelocity * step;
        }
    }
}
```

<h5 style="text-align: center;">Display</h5>
Displaying is actually the harder thing to calculate as we need to first pass in the positions via a read only native array and calculate the arc with collisions so we aren't wasting precious GPU and CPU cycles calculating matrix data for positions that the object would never hit. To start we take in the positions just calculated and throw them into a job running in parallel where we convert the delta positions into a RaycastCommand which is Unity's native raycast that works with burst and jobs. Here is how that works in code:

```c#
[BurstCompile]
private struct PhysicsPredictionRaycastJob : IJobParallelFor
{
    [ReadOnly] public NativeArray<float3> positions;
    [WriteOnly] public NativeArray<RaycastCommand> commands;
    public QueryParameters queryParameters;

    public void Execute(int index)
    {
        float3 start = positions[index];
        float3 end = positions[index + 1];
        float3 direction = end - start;
        float distance = math.length(direction);

        if (distance > 0.0001f)
        {
            direction = math.normalize(direction);
        }

        commands[index] = new RaycastCommand(start, direction, queryParameters, distance);
    }
}
```

After this is calculated we take those raycast commands and schedule them to be complete. This is just done by running JobHandle RaycastCommand.ScheduleBatch(commands, results, minCommandsPerJob, [dependsOn = default]). For the purposes of this display we can safely stop any further calculation if a raycast hits anything. After all of that, we can finally start converting these positions into a 4x4 matrix to give to the GPU to instance.

For converting the positions to matrices we take everything that has been calculated so far (positions/raycasts) and calculate a transform matrix with a specified spacing parameter. This is done by taking the current position and last position's distance value and making sure no collision happened between them. We have a local float for total distance from last step so we can discard values too close to each other. If a distance is greater than the spacing we create a matrix at that point and move on.

After all of these calculations we take the positions back to the handler class (ThrowTrajectorDisplayer) and render each mesh as an instance on the GPU. Doing this we can render high volumes of meshes with custom materials and topology with little to no drawbacks as it is all rendered directly to the scene.

```c#
[BurstCompile(CompileSynchronously = true)]
private struct PositionsToMatricesJob : IJob
{
    [ReadOnly] public NativeArray<float3> positions;
    [ReadOnly] public NativeArray<RaycastHit> raycastHits;
    public NativeArray<float4x4> matrices;

    public NativeReference<int> meshCount;

    public float scale;
    public float collisionScaleFactor;
    public float spacing;

    public void Execute()
    {
        if (positions.Length < 2) return;

        int index = 0;
        int max = matrices.Length;

        RaycastHit hit = raycastHits[0];

        if (hit.colliderEntityId != 0)
        {
            return;
        }

        matrices[index] = CreateMatrix(0, index, max);

        index++;

        float totalDistance = 0.0f;

        for (int posIndex = 1; posIndex < positions.Length; posIndex++)
        {
            float3 pos = positions[posIndex];
            float3 lastPos = positions[posIndex - 1];

            float distance = math.distance(pos, lastPos);
            totalDistance += distance;

            if (posIndex < positions.Length - 2)
            {
                hit = raycastHits[posIndex];

                if (hit.colliderEntityId != 0)
                {
                    if (index > 0)
                    {
                        matrices[index - 1] = CreateMatrix(posIndex, index, max, collisionScaleFactor);
                    }

                    break;
                }
            }

            if (totalDistance >= spacing)
            {
                if (index >= max) break;

                matrices[index] = CreateMatrix(posIndex, index, max);

                index++;

                totalDistance = 0;
            }
        }

        meshCount.Value = index;
    }

    private float4x4 CreateMatrix(int posIndex, int matrixIndex, int count, float scaleFactor = 1)
    {
        float3 position = positions[posIndex];
        quaternion rotation = quaternion.identity;
        float3 forward = float3.zero;

        if (posIndex < positions.Length - 1)
        {
            forward = math.normalize(positions[posIndex + 1] - position);
        }
        else if (posIndex > 0)
        {
            forward = math.normalize(position - positions[posIndex - 1]);
        }

        if (math.lengthsq(forward) > 0.001f)
        {
            rotation = quaternion.LookRotation(forward, math.up());
        }

        float progress = (float)matrixIndex / count;
        float currentScale = math.lerp(scale * scaleFactor, (scale * scaleFactor) * 0.3f, progress);

        return float4x4.TRS(position, rotation, new float3(currentScale));
    }
}
```

<h5 style="text-align: center;">Conclusion</h5>
Overall, this was a very optimized to draw potentially 100s to 1000s of meshes as trajectory points along a physics based arc. Utilizing similar math to Unity we can make the arc follow the exact same line that the object would realistically take! Thanks for reading! 
Here is the final code:

```c#
[Serializable]
public class ThrowTrajectoryCalculator : IDisposable
{
    #region Jobs

    [BurstCompile]
    private struct PhysicsPredictionJob : IJob
    {
        [WriteOnly] public NativeArray<float3> positions;

        public float3 start;
        public float3 velocity;
        public float mass;
        public float drag;
        public float gravityY;
        public float step;

        public void Execute()
        {
            float3 position = start;
            float3 currentVelocity = velocity;

            for (int index = 0; index < positions.Length; index++)
            {
                positions[index] = position;

                float dragFactor = drag * step;
                float dampening = 1f - math.clamp(dragFactor, 0, 1);

                currentVelocity *= dampening;
                currentVelocity.y += gravityY * step;

                position += currentVelocity * step;
            }
        }
    }

    #endregion

    [SerializeField] private int _maxStepCount = 500;
    public int MaxStepCount => _maxStepCount;

    private JobHandle _jobHandle;
    public JobHandle JobHandle => _jobHandle;

    private NativeArray<float3> _positions;
    public NativeArray<float3> Positions => _positions;

    public bool IsCreated { get; private set; }

    public ThrowTrajectoryCalculator() { }

    public void StartCalculation(float3 start, float3 velocity, float mass, float drag)
    {
        if (!IsCreated)
        {
            Debug.LogError("Tried to start a calculation on a yet to be created trajectory calculator!");
            return;
        }

        var predictionJob = new PhysicsPredictionJob
        {
            positions = _positions,
            start = start,
            velocity = velocity,
            mass = mass,
            drag = drag,
            gravityY = Physics.gravity.y,
            step = Time.fixedDeltaTime
        };

        _jobHandle = predictionJob.Schedule();
    }

    public NativeArray<float3> FinishCalculation()
    {
        _jobHandle.Complete();

        return _positions;
    }

    public void Create()
    {
        _positions = new NativeArray<float3>(_maxStepCount, Allocator.Persistent);

        IsCreated = true;
    }

    public void Dispose()
    {
        if (_positions.IsCreated) _positions.Dispose();
    }
}

[Serializable]
public class ThrowTrajectoryDisplayer : IDisposable
{
    #region Jobs

    [BurstCompile]
    private struct PhysicsPredictionRaycastJob : IJobParallelFor
    {
        [ReadOnly] public NativeArray<float3> positions;
        [WriteOnly] public NativeArray<RaycastCommand> commands;
        public QueryParameters queryParameters;

        public void Execute(int index)
        {
            float3 start = positions[index];
            float3 end = positions[index + 1];
            float3 direction = end - start;
            float distance = math.length(direction);

            if (distance > 0.0001f)
            {
                direction = math.normalize(direction);
            }

            commands[index] = new RaycastCommand(start, direction, queryParameters, distance);
        }
    }

    [BurstCompile(CompileSynchronously = true)]
    private struct PositionsToMatricesJob : IJob
    {
        [ReadOnly] public NativeArray<float3> positions;
        [ReadOnly] public NativeArray<RaycastHit> raycastHits;
        public NativeArray<float4x4> matrices;

        public NativeReference<int> meshCount;

        public float scale;
        public float collisionScaleFactor;
        public float spacing;

        public void Execute()
        {
            if (positions.Length < 2) return;

            int index = 0;
            int max = matrices.Length;

            RaycastHit hit = raycastHits[0];

            if (hit.colliderEntityId != 0)
            {
                return;
            }

            matrices[index] = CreateMatrix(0, index, max);

            index++;

            float totalDistance = 0.0f;

            for (int posIndex = 1; posIndex < positions.Length; posIndex++)
            {
                float3 pos = positions[posIndex];
                float3 lastPos = positions[posIndex - 1];

                float distance = math.distance(pos, lastPos);
                totalDistance += distance;

                if (posIndex < positions.Length - 2)
                {
                    hit = raycastHits[posIndex];

                    if (hit.colliderEntityId != 0)
                    {
                        if (index > 0)
                        {
                            matrices[index - 1] = CreateMatrix(posIndex, index, max, collisionScaleFactor);
                        }

                        break;
                    }
                }

                if (totalDistance >= spacing)
                {
                    if (index >= max) break;

                    matrices[index] = CreateMatrix(posIndex, index, max);

                    index++;

                    totalDistance = 0;
                }
            }

            meshCount.Value = index;
        }

        private float4x4 CreateMatrix(int posIndex, int matrixIndex, int count, float scaleFactor = 1)
        {
            float3 position = positions[posIndex];
            quaternion rotation = quaternion.identity;
            float3 forward = float3.zero;

            if (posIndex < positions.Length - 1)
            {
                forward = math.normalize(positions[posIndex + 1] - position);
            }
            else if (posIndex > 0)
            {
                forward = math.normalize(position - positions[posIndex - 1]);
            }

            if (math.lengthsq(forward) > 0.001f)
            {
                rotation = quaternion.LookRotation(forward, math.up());
            }

            float progress = (float)matrixIndex / count;
            float currentScale = math.lerp(scale * scaleFactor, (scale * scaleFactor) * 0.3f, progress);

            return float4x4.TRS(position, rotation, new float3(currentScale));
        }
    }

    #endregion

    private JobHandle _jobHandle;
    private ThrowTrajectoryCalculator _calculator;

    [SerializeField] private Mesh _displayMesh;
    [SerializeField] private Material _displayMaterial;
    private Material _displayMaterialInstance;
    private RenderParams _meshParams;

    [SerializeField] private float _meshScale = 0.25f;
    [SerializeField] private float _meshSpacing = 0.75f;
    [SerializeField, Tooltip("Trajectory final point scale if collided with geometry.")] private float _meshCollisionScaleFactor = 3.0f;

    [SerializeField] private LayerMask _layerMask;

    private NativeArray<RaycastCommand> _raycastCommands;
    private NativeArray<RaycastHit> _raycastHits;
    private NativeArray<float4x4> _meshMatrices;
    private NativeReference<int> _meshCount = new(0, Allocator.Persistent);

    public bool IsCreated { get; private set; }

    public ThrowTrajectoryDisplayer() { }

    public void Create(ThrowTrajectoryCalculator calculator)
    {
        if (!_displayMesh || !_displayMaterial || calculator == null)
        {
            Debug.LogError("Mesh or Material not provided for trajectory display!");
            return;
        }

        _calculator = calculator;

        _displayMaterialInstance = UnityEngine.Object.Instantiate(_displayMaterial);

        _meshParams = new RenderParams(_displayMaterialInstance)
        {
            worldBounds = new Bounds(Vector3.zero, Vector3.one * 1000f)
        };

        _raycastCommands = new NativeArray<RaycastCommand>(_calculator.MaxStepCount - 1, Allocator.Persistent);
        _raycastHits = new NativeArray<RaycastHit>(_calculator.MaxStepCount - 1, Allocator.Persistent);
        _meshMatrices = new NativeArray<float4x4>(_calculator.MaxStepCount, Allocator.Persistent);

        IsCreated = true;
    }

    public void StartDisplay()
    {
        if (!IsCreated || !_calculator.IsCreated)
        {
            Debug.LogError("Trajectory Displayer or Calculator was not created! Create before attempting to display!");
        }

        QueryParameters parameters = new QueryParameters(_layerMask, false, QueryTriggerInteraction.Ignore, false);

        var raycastJob = new PhysicsPredictionRaycastJob
        {
            positions = _calculator.Positions,
            commands = _raycastCommands,
            queryParameters = parameters
        };

        JobHandle raycastHandle = raycastJob.Schedule(_calculator.Positions.Length - 1, 64, _calculator.JobHandle);

        JobHandle raycastCommandHandle = RaycastCommand.ScheduleBatch(_raycastCommands, _raycastHits, 64, raycastHandle);

        var matrixJob = new PositionsToMatricesJob()
        {
            positions = _calculator.Positions,
            raycastHits = _raycastHits,
            matrices = _meshMatrices,
            meshCount = _meshCount,
            scale = _meshScale,
            spacing = _meshSpacing,
            collisionScaleFactor = _meshCollisionScaleFactor
        };

        _jobHandle = matrixJob.Schedule(raycastCommandHandle);
    }

    public void CompleteDisplay()
    {
        if (!IsCreated)
        {
            return;
        }

        _jobHandle.Complete();

        int meshDrawCount = _meshCount.Value;

        if (meshDrawCount > 0)
        {
            NativeArray<Matrix4x4> view = _meshMatrices.Reinterpret<Matrix4x4>();

            Graphics.RenderMeshInstanced(_meshParams, _displayMesh, 0, view, meshDrawCount);

            _meshCount.Value = 0;
        }
    }

    public void SetColor(Color color, bool overwriteAlpha = false)
    {
        if (!overwriteAlpha)
        {
            color.a = _displayMaterialInstance.color.a;
        }

        _displayMaterialInstance.color = color;
    }

    public void Dispose()
    {
        if (_raycastCommands.IsCreated) _raycastCommands.Dispose();
        if (_raycastHits.IsCreated) _raycastHits.Dispose();
        if (_meshMatrices.IsCreated) _meshMatrices.Dispose();
        if (_meshCount.IsCreated) _meshCount.Dispose();

        UnityEngine.Object.Destroy(_displayMaterialInstance);
    }
}
```
