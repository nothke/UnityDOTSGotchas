Various Nothke's DOTS Gotchas

## My VS DOTS snippets

Check out my snippets https://github.com/nothke/Unity-VS-Code-Snippets, which include quick-creating templates for systems, jobs, parallel jobs and more..

## Creating a renderable mesh entity

The entity archetype must at least contain:
- RenderMesh (from Unity.Rendering, found in "Hybrid Rendering" package)
- LocalToWorld (from Unity.Transforms)

Note: MeshRender is a ISharedComponentData and must be set with SetSharedComponentData
Important!: Material provided to the MeshRender must be GPU instanced (tick GPU instancing in the material inspector)

Additionally, you can use Translation, Rotation, Scale or other transform component to place the entity.

#### Example:

Create the archetype and render mesh at start:
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

Then when you create the entity:

```
var entity = World.Active.EntityManager.CreateEntity(particleArch);

manager.SetSharedComponentData(entity, myRenderMesh);
manager.SetComponentData(entity, new Translation() { Value = myPosition });
```


## Destroying Entites from within a job

Since entities can be only created on the main thread, their destruction must be deferred until the job completes. You can issue a command to destroy an entity using EntityCommandBuffer. The EntityCommandBuffer can be obtained from one of the EntityCommandBufferSystems (start typing EntityCommandBufferSystems, and you will get a bunch). You can obtain the system from World in OnCreateManager.

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

Example: https://github.com/Unity-Technologies/EntityComponentSystemSamples/blob/master/ECSSamples/Assets/HelloCube/7.%20SpawnAndRemove/SpawnerSystem_SpawnAndRemove.cs

## Running a system only on entities that contain a component, aka tagging

Since a certain ECS update, it is no longer recommended to include a component into a system if you are not using its data, that is, just for the sake of "tagging".

Instead, put `[RequireComponentTag(typeof(SomeComponentIRequire))]` above the system's job.

Alternatively, you can still pass data as tag, but use `[ReadOnly] ref SomeComponentIRequire` in Execute parameter