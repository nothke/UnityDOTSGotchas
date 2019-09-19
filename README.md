Various Nothke's DOTS gotchas. These are random and in no particular order, also used as a little cheatsheet. Hopefully, someone will find it useful.

Correct as of: 9/19/2019

## Be careful about obsolete resources

As DOTS is still in development, it is susceptible to frequent changes. The biggest problem however is that there are loads of resources and tutorials online showing no longer valid ways of doing things. Check [this page on Unity forums for a list of some of deprecated features](https://forum.unity.com/threads/api-deprecation-faq-0-0-23.636994/). If you encounter any of these, you are likely looking at an old tutorial.

## My VS DOTS snippets

(Warning: They are not updated to latest version of ECS! Will do that soon!)

Check out my snippets https://github.com/nothke/Unity-VS-Code-Snippets, which include quick-creating templates for systems, jobs, parallel jobs and more..

## Mathematics as a using static

It is useful to include Mathematics library as
```
using Unity.Mathematics;
using static Unity.Mathematics.math;
```
Because we can then create data by calling creation methods like `float3 position = float3(0,0,0)` without using the "new" keyword. This is beneficial because it is very similar to how it is written in shader code. As a matter of fact you can literally just copy the code to a HLSL function and it will behave in the same way.

## Mathematics swizzling

There is a hidden feature (doesn't show up in VS auto complete) in Mathematics library commonly referred to as "swizzling", where you can swap or convert values very easily, similar to how it's done in HLSL code.

A common swizzling example is when you want to convert 3D position into a horizontal 2D vector, so you want to put x and z into a float2's x and y:
```
float3 pos3d = float3(1, 2, 3);
float2 pos2d = pos3d.xz;
// pos2d is now (1, 3);

float4 vector = float4(1, 2, 3, 4);
vector.xw = vector.yz;
// vector is now (2, 2, 3, 3);
```

## Creating a renderable mesh entity

The entity archetype must at least contain:
- RenderMesh (from Unity.Rendering, found in "Hybrid Rendering" package)
- LocalToWorld (from Unity.Transforms)

Optionally, you can use Translation, Rotation, Scale or other transform component to place the entity.

Note: MeshRender is a ISharedComponentData and must be set with SetSharedComponentData

Important!: Material provided to the MeshRender must be GPU instanced (tick GPU instancing in the material inspector)

#### Example:

Create the archetype and RenderMesh at start:
```
var manager = World.Active.EntityManager;

EntityArchetype myRenderableArchetype = manager.CreateArchetype(
    typeof(RenderMesh),
    typeof(LocalToWorld),
    
    typeof(Translation));

RenderMesh myRenderMesh = new RenderMesh()
{
    mesh = myMesh,
    material = myInstancedMaterial
};
```

Then, create the entity and add components:

```
var entity = manager.CreateEntity(particleArch);

manager.SetSharedComponentData(entity, myRenderMesh);
manager.SetComponentData(entity, new Translation() { Value = myPosition });
```

You should now have a visible object in game!

## Creating or destroying Entities from within a JobComponentSystem

Since entities can only be created/destroyed on the main thread, their construction/destruction must be deferred until the job completes. You can issue a command to destroy an entity using an EntityCommandBuffer. The EntityCommandBuffer can be obtained from one of the EntityCommandBufferSystems (start typing EntityCommandBufferSystems, and you will get a bunch). You can obtain the system from World in OnCreateManager.

```
EndSimulationEntityCommandBufferSystem commandBufferSystem;

protected override void OnCreate()
{
    commandBufferSystem = World.GetOrCreateSystem<EndSimulationEntityCommandBufferSystem>();
}
```

Then, pass the command buffer to the job in OnUpdate. EDIT: Also, we need to tell the barrier system which job is using the command buffer so it can wait for it to finish.

```
var handle = new SystemJob()
{
    ecb = commandBufferSystem.CreateCommandBuffer().ToConcurrent();
}.Schedule(this, inputDeps);

// Tell the barrier system which job is using the ecb so it can complete it.
commandBufferSystem.AddJobHandleForProducer(handle);
```

The job must be IJobForEachWithEntity (since we need the entity) and should have:

```
public EntityCommandBuffer.Concurrent ecb;
```

In Job's Execute, if you wish to destroy the entity, use

```
ecb.DestroyEntity(index, entity);
```

Note: In the Unity example, the "index" provided to the EntityCommandBuffer is the same as the entity, but in the docs it says that it just needs to be a unique number as to not write to the same data

[Example from Unity samples](https://github.com/Unity-Technologies/EntityComponentSystemSamples/blob/master/ECSSamples/Assets/HelloCube/7.%20SpawnAndRemove/SpawnerSystem_SpawnAndRemove.cs)

## Running a system only on entities that contain a component, aka tagging

Since a certain Unity.Entities update, it is no longer recommended to include a component into a system if you are not using its data, that is, just for the sake of "tagging".

Instead, put `[RequireComponentTag(typeof(SomeComponentIRequire))]` above the system's job.

Alternatively, you can still pass data as tag, but use `[ReadOnly] ref SomeComponentIRequire` in Execute parameter.

## Don't automatically start a system on start

If you wish to prevent SOME systems from automatically starting, you can add `[DisableAutoCreation]` to the top of the system class.

If you wish to prevent ALL systems from automatically starting, you can add `UNITY_DISABLE_AUTOMATIC_SYSTEM_BOOTSTRAP` to your Project Settings > Player > Scripting Defines Symbols.

## Manual system start

If you wish to add systems to your world manually (considering [they are not auto-started](#dont-automatically-start-a-system-on-start)), first you need to put them in a system group, and then add them to the update list.

Example:

```
group = World.Active.GetOrCreateSystem<SimulationSystemGroup>();
var mySystem = World.Active.GetOrCreateSystem(typeof(MySystem));
group.AddSystemToUpdateList(mySystem);
```

This will now run at every invocation of the group's Update, respecting the [UpdateAfter/Before] sorting, just as if it was automatically started.

## Parallel writing to a NativeArray

Parallel writing is by default not allowed because of the race conditions of writing to the same index and Unity safety system will warn if you try to parallel write and will instead force single-threading.

BUT, you CAN write to an array in parallel if you make sure that you don't write to the same index. To do that you need to add `[NativeDisableParallelForRestriction]` in front of the NativeArray. Same goes for ComponentDataFromEntity and other collections. Note that you will now not be warned even if you are writing to the same spot, so, be very careful.

## Transferring data from entity to entity

Lets say we want to transfer some data from one entity to the other. First, we need to reference an entity in another entity. Then, you can get a `ComponentDataFromEntity<>` which is an array of component datas indexed by entity in a system using `GetComponentDataFromEntity<MyData>()`. You can use this to take data, pointed to by our stored Entity, from one entity to the other.

For example, we have a component data:

```
public struct MyData : IComponentData
{
    public float value;
}
```

..and we want to add to it's value from another entity. We create another component data that stores the linked entity and the amount we want to transfer:

```
public struct MyTransferingData : IComponentData
{
	public Entity entity;
	public float transferAmount;
}
```

In system OnUpdate we can fetch `ComponentDataFromEntity<MyData>`:
```
    protected override JobHandle OnUpdate(JobHandle inputDeps)
    {
        var myDatas = GetComponentDataFromEntity<MyData>();

        var job = new SystemJob()
        {
            myDatas = myDatas
        };

        return job.Schedule(this, inputDeps);
    }
```

In the System's Job, we can now get data by indexing the `ComponentDataFromEntity<MyData>`:
```
    [BurstCompile]
    struct SystemJob : IJobForEach<MyTransferingData>
    {
        [NativeDisableParallelForRestriction] // So we can parallel write
        public ComponentDataFromEntity<TileData> myDatas;

        public void Execute(ref MyTransferingData transfer)
        {
            var myDatas = tileDatas[transfer.entity];
            myDatas.value += transfer.dataToTransfer; // Where we add the value
            myDatas[transfer.entity] = tileData;
        }
    }
```

You can also mark as ReadOnly if you want to only read, making the job potentially faster.

## Tuples with auto layout are not supported by Burst

I've encountered this error when trying to return a tuple, for example:
```
public static (int, int) GetCoord(int index)
```
Instead, I had to use the out parameter to accomplish the same:
```
public static void GetCoord(int index, out int x, out int y)
```
I am not sure if it works by explicitly setting the layout (if that can even be done with C# tuples?). Alternatively you can always use a custom struct.

## Force ForEach system to run on a single thread

Use `job.ScheduleSingle(this, inputDeps);`

## Slowdown on start (Burst compilation)

You may notice that when starting the game in editor with ECS systems you encounter a few stutters that can last up to several seconds until the game starts running smoothly. This is because Unity compiles Burst code asynchronously every time you enter the play mode. Until the burst native code is ready, jobs will be run without burst.

You can force Unity to compile Burst code ahead of time by using `[BurstCompile(CompileSynchronously = true)]`, but I personally recommend not doing that as it is better to have a shorter time of entering the play mode.

But note that this only happens in editor! Rest assured, this will not happen in build, all burst code is compiled during the build process.

## Plenty of GC Allocations each frame when using ECS/Jobs

When using jobs or ECS you may notice a high amount of GC Allocations. There could be several reasons for that:
- Unity uses a managed object called a [DisposeSentinel](https://docs.unity3d.com/ScriptReference/Unity.Collections.LowLevel.Unsafe.DisposeSentinel.html) to track the native collection's lifetime. It is producing GC allocations when creating/disposing native collections. That is the expected behavior as the DisposeSentinel helps you track issues and memory leaks. Leak checks are by default not being tracked in build, therefore there won't be any DisposeSentinels nor GC allocations. You can also set Jobs > Leak Detection to off to turn it off in editor.

## Returning a single value from a Job

The only thing that "survives" a job are native collections, so if you want to return a value from a job, even if it's a single one, you must wrap it in a NativeArray.

For example, job for calculating a mean value might look like this:
```
public struct CalculateMeanValue : IJob
{
    [ReadOnly] public NativeArray<float> values;
    // This will be the NativeArray with a single element:
    [WriteOnly] public NativeArray<float> output;

    public void Execute()
    {
        float total = 0;
        for (int i = 0; i < values.Length; i++)
            total += values[i];

        output[0] = total / values.Length;
    }
}
```

You can even wrap BlobAssetReference into a NativeArray to return it, for example to build a Unity.Physics.Collider inside a job.

## Game fails to build with CreateEntity/AddComponent EntityCommandBuffer job

Creating entities/adding components is currently not supported by the Burst compiler. If you wish to add CreateEntity/AddComponent commands to the EntityCommandBuffer in a job, you should not `[BurstCompile]` it.

^ To be changed in the next Burst version.

## Dispose collections after the job has completed

Lets say we have a NativeArray with `Allocator.TempJob`, and we wish to dispose it when the job has completed, you can add a `[DeallocateOnJobCompletion]` on collection's definition inside a job. This is especially useful in systems since we don't know when the system's job will end.

Example:
```
public struct MyJob : IJobParallelFor
{
    [DeallocateOnJobCompletion]
    [ReadOnly] NativeArray<float> readFromThisArray;

    public void Execute(int i)
    {
    	float value = readFromThisArray[i];
        // Do something...
    }
}
```

## Unity Physics causes an InvalidCastException in Entity Debugger

Will probably be fixed, but in the meantime..

Add `"com.unity.properties": "0.6.4-preview"` (or [later versions](https://bintray.com/unity/unity/com.unity.properties) depending on your Unity version) to your packages file dependencies.

Solution [found in this post](https://forum.unity.com/threads/entity-debugger-feedback.522893/page-3#post-4853669)