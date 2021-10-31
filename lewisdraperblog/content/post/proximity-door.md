---
title: "Proximity Door"
date: 2021-10-31T11:49:06Z
featured_image: 'images/posts/proximity-door/door.jpg'
draft: false
---
When I first started working with unreal engine, it was apparent that creating a door was clearly a right of passage. But there are many ways of making a door in unreal, and I noticed that most of the tutorials I had seen used the Tick function. Don't get me wrong, using the tick function to create a door, especially in a super-beginner tutorial, is a great way of getting something working quickly. But longer term, you are going to want to learn how to avoid using the Tick wherever possible, and I think the earlier you start the better. So below is how I created an automaticaly opening door in UE4, with the help of Timelines.

---

## Class Setup

First we are going to create our Proximity Door actor

- Select **File > "New C++ Class..."** in the editor.
- Derive your class from **AActor**.
- Name your class something suitable. I chose **ProximityDoor**.
- Hit **Create Class**

After doing that, you should have 2 new files in your project, ProximityDoor.cpp & ProximityDoor.h, which will look like this...
##### ProximityDoor.h
```cpp
#include "CoreMinimal.h"
#include "GameFramework/Actor.h"
#include "ProximityDoor.generated.h"

UCLASS()
class TESTPROJECT_API AProximityDoor : public AActor
{
	GENERATED_BODY()	
public:	
	// Sets default values for this actor's properties
	AMyActor();
protected:
	// Called when the game starts or when spawned
	virtual void BeginPlay() override;
public:	
	// Called every frame
	virtual void Tick(float DeltaTime) override;

};
```
##### ProximityDoor.cpp

```cpp
#include "MyActor.h"
// Sets default values
AProximityDoor::AProximityDoor()
{
 	// Set this actor to call Tick() every frame.  You can turn this off to improve performance if you don't need it.
	PrimaryActorTick.bCanEverTick = true;
}
// Called when the game starts or when spawned
void AProximityDoor::BeginPlay()
{
	Super::BeginPlay();	
}
// Called every frame
void AProximityDoor::Tick(float DeltaTime)
{
	Super::Tick(DeltaTime);
}
```
Now the whole goal of this tutorial was to create a door without using Tick, so lets get rid of the Tick definition and implementation, and also set `PrimaryActorTick.bCanEverTick` to `false` in the constructor.

## Components

Lets add our components. We will need the following:

- `UStaticMeshComponent` for the door frame.
- `UStaticMeshComponent` for the door itself.
- `UBoxComponent` for the trigger volume.
- `UTimelineComponent` for the open/close animation.

Your header file should look like this:

```cpp
protected:
	//Components
	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	class UStaticMeshComponent* FrameMesh;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	class UStaticMeshComponent* DoorMesh;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	class UBoxComponent* DoorTrigger;

	UPROPERTY(VisibleAnywhere, BlueprintReadOnly)
	class UTimelineComponent* DoorTimeline;
```
Let us now create and attach these components in the constructor. I chose to use the door frame as the root of our actor. Note that the `UTimelineComponent` does not need to be attached to anything, as it is a `UActorComponent`, not a `USceneComponent`.

```cpp
AProximityDoor::AProximityDoor()
{
	PrimaryActorTick.bCanEverTick = false;

	FrameMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Frame Mesh"));
	RootComponent = FrameMesh;

	DoorMesh = CreateDefaultSubobject<UStaticMeshComponent>(TEXT("Door Mesh"));
	DoorMesh->SetupAttachment(RootComponent);

	DoorTrigger = CreateDefaultSubobject<UBoxComponent>(TEXT("Door Trigger"));
	DoorTrigger->SetupAttachment(RootComponent);

	DoorTimeline = CreateDefaultSubobject<UTimelineComponent>(TEXT("Door Timeline"));	
}
```
## Additional Properties

We will need two more variables in our class. The first is the curve which defines how our door will open. In our case, we want the angle of the door to smoothly change from 0° to 90°, so our curve is where we will define this. We will create and assign the curve later in the editor. The last variable we need is a delegate signature. We will later bind this to a function and pass it to our timeline, which the timeline will then use to give us the value on the curve at a specific time.

```cpp
	UPROPERTY(EditDefaultsOnly)
	UCurveFloat* DoorCurve;

	FOnTimelineFloat UpdateFunctionFloat;
```

## Creating the Door Blueprint

Now lets create the blueprint that we will be placing in the level. This is where we will specify the meshes for our mesh components, as well as adjusting our box component to a suitable size. To create the BP, right-click on the class in the C++ Classes folder in the content browser, and click on **Create Blueprint class based on ProximityDoor**.

{{< figure src="/images/posts/proximity-door/bpCreation.png" title="" >}}

Once opened the editor will look like this:

{{< figure src="/images/posts/proximity-door/meshSelection.png" title="" >}}

To set the door frames mesh, click on the frame mesh component on the top left, then select the mesh in the details panel on the right. I will be using the meshes provided in the Starter Content, which can be added to any project by going to **Add>Add Feature or Content Pack** in the content browser. After doing the same for the door mesh, your viewport should look something like this:

{{< figure src="/images/posts/proximity-door/meshesAdded.png" title="" >}}

**Note: By default, it seems the starter content door has no simple collision, so go into the mesh asset and set it to use complex as simple collision in the collision section on the right.**

Due to the doors origin being at it's corner (this will be useful for rotation), it will not be alligned with the frame by default, so we can adjust its transform to suit our needs, then save the blueprint. We will also adjust the box component so that it is large enough for a player to trigger it when nearby the door. Now is a good time to set the collision profile for the BoxComponent to **OverlapPawnOnly**. This will ennsure that your door will only open for the player. Your blueprint should look something like this:

{{< figure src="/images/posts/proximity-door/doorFinal.png" title="" >}}

## Timeline Curve

To define how the timeline will rotate the door, we need to create a curve. This is simply a curve going smoothly from 0 to 90, so our door can be rotated in a natural looking way. Right-click in the content browser and go to **Miscellaneous>Curve**. Select **CurveFloat** when prompted, then name your curve something like **DoorTimelineCurve**. Once opened you will see something like this:

{{< figure src="/images/posts/proximity-door/curveEmpty.png" title="" >}}

If you right click in the graph and select **Add Key**, a point will be added to the curve. Set its position to 0,0 at the top of the screen. Add a second point and set its location to (1,90). Set the view to **Normalised Mode** and you will see the whole curve like this: 

{{< figure src="/images/posts/proximity-door/normalisedview.png" title="" >}}

We can smooth out this curve to make the opening look more natural. Do this my selecting both points, then press the 2 key. This will flatten the tangents, and your curve should now look more curvy:

{{< figure src="/images/posts/proximity-door/curvycurve.png" title="" >}}

Lastly, let us assign this curve to the door blueprint:

{{< figure src="/images/posts/proximity-door/curveBP.png" title="" >}}

## Back to the code...

Now our blueprint is set up and contains our curve, lets set up the timeline in code to use this curve to rotate the door. First we must create some functions. We will need functions that will be called when we enter/exit the trigger volume, and we will need another function which the timeline will call every time it updates. 

```cpp
	UFUNCTION()
	void OnTriggerEnter(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& Hit);
	UFUNCTION()
	void OnTriggerExit(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, int32 OtherBodyIndex);
	UFUNCTION()
	void OnTimelineUpdate(float Value);
```

It is important that these functions are defined exactly as shown above so that we can bind them. In order for the trigger functions to be called during overlap events, we must subscribe to these events using `AddDynamic`. Additionally, we need to bind our delegate signature to our `OnTimelineUpdate` function using `BindDynamic` so that it can be used by the timeline. We will do these bindings in the `BeginPlay()` function. Then, we will call the `AddInterpFloat()` function on the timeline to specify the curve and the update function.

```cpp
void AProximityDoor::BeginPlay()
{
	Super::BeginPlay();

	//Delegate Bindings
	DoorTrigger->OnComponentBeginOverlap.AddDynamic(this, &AProximityDoor::OnTriggerEnter);
	DoorTrigger->OnComponentEndOverlap.AddDynamic(this, &AProximityDoor::OnTriggerExit);
	UpdateFunctionFloat.BindDynamic(this, &AProximityDoor::OnTimelineUpdate);

	if (DoorCurve)
	{	
		DoorTimeline->AddInterpFloat(DoorCurve, UpdateFunctionFloat);
	}
}
```
So now whenever something overlaps the trigger, `OnTriggerEnter` will be called. Similarly, when someone exits the trigger, `OnTriggerExit` will be called. Finally, every time the timeline updates, `UpdateFunctionFloat` will be called with the current curve value passed as a paramter.

All thats left to do is to start/reverse the timeline during the trigger events, and adjust the rotation of the door during the timeline events. It is a good idea to do a null-check on DoorCurve and DoorMesh just in case they haven't been set in the editor.

```cpp
void AProximityDoor::OnTriggerEnter(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, int32 OtherBodyIndex, bool bFromSweep, const FHitResult& Hit)
{
	if (DoorCurve)
	{
		DoorTimeline->Play();
	}	
}

void AProximityDoor::OnTriggerExit(UPrimitiveComponent* OverlappedComponent, AActor* OtherActor, UPrimitiveComponent* OtherComponent, int32 OtherBodyIndex)
{
	if (DoorCurve)
	{
		DoorTimeline->Reverse();
	}
}

void AProximityDoor::OnTimelineUpdate(float Value)
{
	if (DoorMesh)
	{
		FRotator NewRotation = FRotator(0.0f, Value, 0.0f);
		DoorMesh->SetRelativeRotation(NewRotation);
	}
}
```


## Final Result

If all goes well, you should end up with something like this...

{{< figure src="/images/posts/proximity-door/demo.gif" title="" >}}