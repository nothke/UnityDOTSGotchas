Various Nothke's DOTS Gotchas

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

## Destroying Entites from within a job

Since entities can only be created on the main thread, their destruction must be deferred until the job completes. You can issue a command to destroy an entity using EntityCommandBuffer. The EntityCommandBuffer can be obtained from one of the EntityCommandBufferSystems (start typing EntityCommandBufferSystems, and you will get a bunch). You can obtain the system from World in OnCreateManager.

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

Parallel writing is by default not allowed because of the race conditions of writing to the same index and Unity safety system will warn if you try to prallel write and will instead force the execution to not be parallel. 

BUT, you CAN write to an array in parallel if you make sure that you don't write to the same index, and you need to add `[NativeDisableParallelForRestriction]` in front of the NativeArray. Same goes for ComponentDataFromEntity and other collections. Note that you will now not be warned even if you are writing to the same spot, so, be very careful.