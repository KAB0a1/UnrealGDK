# Non-Unreal server-worker types
<%(TOC)%>

By default, the GDK for Unreal uses a single Unreal server-worker type to handle all server-side computation. However, you can set up additional server-worker types that do not use Unreal or the GDK.

You can use these non-Unreal server-worker types to modularize your game’s functionality so you can re-use the functionality across different games. For example, you could use a non-Unreal server-worker type written in Python that interacts with a database or other third-party service, such as [Firebase](https://firebase.google.com/) or [PlayFab](https://playfab.com/).

## How to integrate non-Unreal server-worker types into your game
In order to interact with each other, Unreal and non-Unreal server-worker instances need to send and receive updates to and [commands](https://docs.improbable.io/reference/latest/shared/glossary#command) for the same SpatialOS components. We recommend that you define the SpatialOS components used by non-Unreal worker instances manually in [schema](https://docs.improbable.io/unreal/alpha/content/glossary#schema) files, separate from those automatically generated by the GDK.  This is because:

* SpatialOS command data in GDK-generated schema files is encoded as byte strings, making deserialization of commands in non-Unreal server-worker types more difficult.
* It aids portability; you can more easily re-use any non-Unreal server-worker types when you have an accompanying schema file.

Default single Unreal server-worker type development doesn’t accommodate schema files that haven’t been generated by the GDK, so you need to set your game up to handle this.

### How to set up interaction
To set up your game to interact with SpatialOS components defined outside the GDK (external SpatialOS components), you use the following:

* To send SpatialOS component updates and commands, use methods defined in the `SpatialWorkerConnection.h` file (described in the examples section below).
* To receive [network operations](https://docs.improbable.io/reference/latest/shared/design/operations) for external SpatialOS components, you must provide custom callbacks for specific component IDs and operation types. The GDK then forwards the operations to your callbacks. (This is described in the examples section below.)


>**TIP:** If you’re using schema from outside the GDK, you can customise [snapshot generation](https://docs.improbable.io/unreal/alpha/content/generating-a-snapshot#generating-a-snapshot) from the GDK toolbar’s  **Snapshot** button. This is to serialize additional entities with these external components which the default Unreal snapshot generation cannot currently do. See [Add to the snapshot](#add-to-the-snapshot) below for how to do this.

#### Send data
You set up your game to send SpatialOS component updates, command requests, and command responses directly using the `SpatialWorkerConnection.h` public methods:

* `void SendComponentUpdate(Worker_EntityId EntityId, const Worker_ComponentUpdate* ComponentUpdate);`
* `Worker_RequestId SendCommandRequest(Worker_EntityId EntityId, const Worker_CommandRequest* Request, uint32_t CommandId);`
* `void SendCommandResponse(Worker_RequestId RequestId, const Worker_CommandResponse* Response);`

You access these methods via a reference to the net driver, as shown below:

`SpatialWorkerConnection connection = Cast<USpatialNetDriver>(World->GetNetDriver())->Connection;`

There is a basic example in the _Examples_ section below. For more examples of how to construct component updates, command requests, and more, see the SpatialOS documentation on  [serialization in the Worker SDK in C’s API](https://docs.improbable.io/reference/latest/capi/serialization).

#### Receive data

>**Note:** Your external SpatialOS components must have an ID between 1000 and 2000 to be registered by the pipeline.

You set up your game to receive network operations using the `USpatialDispatcher::AddOpCallback` functions. These overloaded functions are parameterized with a `Worker_ComponentId` and a callback function const reference that takes one of the following network operation types as an argument:

* `Worker_AddComponentOp`
* `Worker_RemoveComponentOp`
* `Worker_AuthorityChangeOp`
* `Worker_ComponentUpdateOp`
* `Worker_CommandRequestOp`
* `Worker_CommandResponseOp`

`Worker_ComponentId` and each network operation type are defined in the [Worker SDK in C’s API](https://docs.improbable.io/reference/latest/capi/api-reference).

If you want to ensure that the SpatialOS worker connection receives your callback for initial network operations, you need to register the callbacks inside your game instance's `OnConnected` event callback.

Each `USpatialDispatcher::AddOpCallback` function returns a `CallbackId`. You can deregister your callbacks using the `USpatialDispatcher::RemoveOpCallback` function and passing the `CallbackId` parameter that was returned by the corresponding call to `USpatialDispatcher::AddOpCallback`.

There is a basic example in the _Examples_ section below. For more examples of how to deserialize the `Worker_Op` type see the SpatialOS documentation on [serialization in the Worker SDK in C’s API](https://docs.improbable.io/reference/latest/capi/serialization).

#### Add to the snapshot
You can customize snapshot generation by creating a class derived from the GDK `USnapshotGenerationTemplate` base class, and implementing the method below. You have the responsibility of incrementing the `NextEntityId` reference. If you don’t, snapshot generation will fail by attempting to add multiple entities to the snapshot with the same ID.
```  
    * Write to the snapshot generation output stream.
    * @param OutputStream the output stream for the snapshot being created.
    * @param NextEntityId the next available entity ID in the snapshot, this reference should be incremented appropriately.
    * @return bool the success of writing to the snapshot output stream, this is returned to the overall snapshot generation.
    **/
bool WriteToSnapshotOutput(Worker_SnapshotOutputStream* OutputStream, Worker_EntityId& NextEntityId);
```


There is a basic example in the _Examples_ section below. For more examples of how to deserialize see the SpatialOS documentation on  [serialization in the Worker SDK in C’s API](https://docs.improbable.io/reference/latest/capi/serialization).

### Examples
Below is a simple example schema file which a non-Unreal server-worker type could use to track player statistics:

```
package improbable.testing;

type UnrealRequest {
    string some_request_string = 1;
}
type UnrealResponse {
    string some_response_string = 1;
}

component NonUnrealAuthoritative {
    id = 1337;
    uint32 counter = 1;
}

component UnrealAuthoritative {
    id = 1338;
    uint32 other_counter = 1;
    command UnrealResponse test_command(UnrealRequest);
}
```

#### Send data
You could serialize and send a component update in your Unreal project code in the following way:

```
void SendSomeUpdate(Worker_EntityId TargetEntityId, Worker_ComponentId ComponentId)
{
    Worker_ComponentUpdate Update = {};
    Update.component_id = ComponentId;
    Update.schema_type = Schema_CreateComponentUpdate(ComponentId);
    Schema_Object* FieldsObject = Schema_GetComponentUpdateFields(Update.schema_type);
    Schema_AddInt32(FieldsObject, 1, ++UnrealCounter);
    Cast<USpatialNetDriver>(World->GetNetDriver())->Connection->SendComponentUpdate(TargetEntityId, &Update);
}
```


You could serialize and send a command response in your Unreal project code in the following way:


```
Worker_RequestId SendSomeCommandResponse(Worker_EntityId TargetEntityId, Worker_ComponentId ComponentId, Schema_FieldId CommandId) {
    Worker_CommandResponse Response = {};
    Response.component_id = ComponentId;
    Response.schema_type = Schema_CreateCommandResponse(ComponentId, CommandId);
    Schema_Object* ResponseObject = Schema_GetCommandResponseObject(Response.schema_type);
    const char* Text = "Hello World.";
    Schema_AddBytes(ResponseObject, 1, (const uint8_t*)Text, sizeof(char) * strlen(Text));
    Cast<USpatialNetDriver>(World->GetNetDriver())->Connection->SendCommandResponse(TargetEntityId, &Response);
}
```


#### Receive data
You could receive and deserialize a component update and command request in your Unreal project code in the following way:
```
void UTPSGameInstance::Init()
{
    // OnConnected is an event declared in USpatialGameInstance
    OnConnected.AddLambda([this]() {
        // On the client the world may not be completely set up, if s we can use the PendingNetGame
        USpatialNetDriver* NetDriver = Cast<USpatialNetDriver>(GetWorld()->GetNetDriver());
        if (NetDriver == nullptr)
        {
            NetDriver = Cast<USpatialNetDriver>(GetWorldContext()->PendingNetGame->GetNetDriver());
        }
        USpatialDispatcher* Dispatcher = NetDriver->Dispatcher;

        Dispatcher->AddOpCallback(1337, [this](const Worker_ComponentUpdateOp& Op) {
            // Example deserializing network operation
            uint32 PlayerCount = Schema_GetUint32(Schema_GetComponentUpdateFields(Op.update.schema_type), 1);

            // Example actor spawning
            const FVector Location = FVector::ZeroVector;
            const FRotator Rotation = FRotator::ZeroRotator;
            GetWorld()->SpawnActor(CubeClass, &Location, &Rotation);

            // Example serializing and sending another network operation
            Worker_ComponentUpdate Update = {};
            Update.component_id = 1338;
            Update.schema_type = Schema_CreateComponentUpdate(1338);
            Schema_Object* FieldsObject = Schema_GetComponentUpdateFields(Update.schema_type);
            Schema_AddInt32(FieldsObject, 1, PlayerCount);
            Cast<USpatialNetDriver>(GetWorld()->GetNetDriver())->Connection->SendComponentUpdate(Op.entity_id, &Update);

        });

        Dispatcher->AddOpCallback(1338, [this](const Worker_CommandRequestOp& Op) {
            // Example serializing and sending command response
            Worker_CommandResponse Response = {};
            Response.component_id = 1338;
            Response.schema_type = Schema_CreateCommandResponse(1338, 1);
            Schema_Object* response_object = Schema_GetCommandResponseObject(Response.schema_type);
            FString text = "Here's my response.";
            Schema_AddBytes(response_object, 1, (const uint8_t*)TCHAR_TO_ANSI(*text), sizeof(char) * strlen(TCHAR_TO_ANSI(*text)));
            Cast<USpatialNetDriver>(GetWorld()->GetNetDriver())->Connection->SendCommandResponse(Op.request_id, &Response);
        });
    });
}
```

#### Add to the snapshot
You could add a new entity with the given component in your Unreal project code in the following way:

```
UCLASS()
class UTestEntitySnapshotGeneration : public USnapshotGenerationTemplate
{
	GENERATED_BODY()

public:
	bool WriteToSnapshotOutput(Worker_SnapshotOutputStream* OutputStream, Worker_EntityId& NextEntityId) override {
		Worker_Entity TestEntity;
		TestEntity.entity_id = NextEntityId;

		TArray<Worker_ComponentData> Components;

		const WorkerAttributeSet TestWorkerAttributeSet{ TArray<FString>{TEXT("test_attribute")} };
		const WorkerRequirementSet TestWorkerPermission{ TestWorkerAttributeSet };
		const WorkerRequirementSet AnyWorkerPermission{ {SpatialConstants::UnrealClientAttributeSet, SpatialConstants::UnrealServerAttributeSet, TestWorkerAttributeSet } };

		WriteAclMap ComponentWriteAcl;
		ComponentWriteAcl.Add(SpatialConstants::POSITION_COMPONENT_ID, SpatialConstants::UnrealServerPermission);
		ComponentWriteAcl.Add(SpatialConstants::METADATA_COMPONENT_ID, SpatialConstants::UnrealServerPermission);
		ComponentWriteAcl.Add(SpatialConstants::PERSISTENCE_COMPONENT_ID, SpatialConstants::UnrealServerPermission);
		ComponentWriteAcl.Add(SpatialConstants::ENTITY_ACL_COMPONENT_ID, SpatialConstants::UnrealServerPermission);
		ComponentWriteAcl.Add(1337, TestWorkerPermission);
		ComponentWriteAcl.Add(1338, SpatialConstants::UnrealServerPermission);

		// Serialize NonUnrealAuthoritative component data
		Worker_ComponentData NonUnrealAuthoritativeComponentData{};
		NonUnrealAuthoritativeComponentData.component_id = 1337;
		NonUnrealAuthoritativeComponentData.schema_type = Schema_CreateComponentData(1337);
		Schema_Object* NonUnrealAuthoritativeComponentDataObject = Schema_GetComponentDataFields(NonUnrealAuthoritativeComponentData.schema_type);
		Schema_AddInt32(NonUnrealAuthoritativeComponentDataObject, 1, 1); // set counter field to 1 initially

		// Serialize FromUnreal component data
		Worker_ComponentData UnrealAuthoritativeComponentData{};
		UnrealAuthoritativeComponentData.component_id = 1338;
		UnrealAuthoritativeComponentData.schema_type = Schema_CreateComponentData(1338);
		Schema_Object* UnrealAuthoritativeComponentDataObject = Schema_GetComponentDataFields(UnrealAuthoritativeComponentData.schema_type);
		Schema_AddInt32(UnrealAuthoritativeComponentDataObject, 1, 1); // set other_counter field to 1 initially

		Components.Add(SpatialGDK::Position(SpatialGDK::Origin).CreatePositionData());
		Components.Add(SpatialGDK::Metadata(TEXT("TestEntity")).CreateMetadataData());
		Components.Add(SpatialGDK::Persistence().CreatePersistenceData());
		Components.Add(SpatialGDK::EntityAcl(AnyWorkerPermission, ComponentWriteAcl).CreateEntityAclData());
		Components.Add(NonUnrealAuthoritativeComponentData);
		Components.Add(UnrealAuthoritativeComponentData);

		TestEntity.component_count = Components.Num();
		TestEntity.components = Components.GetData();

		bool bSuccess = Worker_SnapshotOutputStream_WriteEntity(OutputStream, &TestEntity) != 0;
		if (bSuccess)
		{
			NextEntityId++;
		}

		return bSuccess;
	}
};
```

<br/>

------
_2019-04-11 Page updated with limited editorial review_<br/>
_2019-03-15 Page added with full editorial review_