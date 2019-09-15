Various Nothke's DOTS Gotchas

Correct as of: 9/15/2019

## My VS DOTS snippets

Check out my snippets https://github.com/nothke/Unity-VS-Code-Snippets, which include quick-creating templates for systems, jobs, parallel jobs and more..

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
EntityArchetype myRenderableArchetype = World.Active.EntityManager.CreateArchetype(
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
var entity = World.Active.EntityManager.CreateEntity(particleArch);

manager.SetSharedComponentData(entity, myRenderMesh);
manager.SetComponentData(entity, new Translation() { Value = myPosition });
```

You should now have a visible object in game!

## Creating or destroying Entites from within a JobComponentSystem

Since entities can only be created/destroyed on the main thread, their construction/destruction must be deferred until the job completes. You can issue a command to destroy an entity using an EntityCommandBuffer. The EntityCommandBuffer can be obtained from one of the EntityCommandBufferSystems (start typing EntityCommandBufferSystems, and you will get a bunch). You can obtain the system from World in OnCreateManager.

```
EndSimulationEntityCommandBufferSystem commandBufferSystem;

protected override void OnCreate()
{
    commandBufferSystem = World.GetOrCreateSystem<EndSimulationEntityCommandBufferSystem>();
}
```

Then, pass the command buffer to the job in OnUpdate

```
var job = new SystemJob()
{
    ecb = commandBufferSystem.CreateCommandBuffer().ToConcurrent();
};
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

Add `[DisableAutoCreation]` to the top of the system class.

## Parallel writing to a NativeArray

Parallel writing is by default not allowed because of the race conditions of writing to the same index and Unity safety system will warn if you try to parallel write and will instead force single-threading.

BUT, you CAN write to an array in parallel if you make sure that you don't write to the same index. To do that you need to add `[NativeDisableParallelForRestriction]` in front of the NativeArray. Same goes for ComponentDataFromEntity and other collections. Note that you will now not be warned even if you are writing to the same spot, so, be very careful.

## Trasfering data from entity to entity

Lets say we want to share transfer some data from one entity to the other. First, we need to reference an entity in another entity. Then, you can get a `ComponentDataFromEntity<>` which is an array of component datas indexed by entity in a system using `GetComponentDataFromEntity<MyData>()`. You can use this to take data, pointed to by our stored Entity, from one entity to the other.

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