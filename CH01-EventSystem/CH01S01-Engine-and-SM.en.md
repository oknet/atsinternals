# Engine and State Machine

We are able to put Event into this engine with 3 types of API: EventProcessor，EThread and Event

  - EventProcessor
    - Create an Event for State Machine
    - Select an EThread from the engine based on round-robin algorithm 
    - Put the Event into selected EThread
  - EThread
    - Put the Event into the EThread
  - Event
    - Put back the Event to the EThread that callback the SM

The engine implements different strategies upon types of event.

The engine calls Event->cont->handleEvent then the State Machine get the CPU to accomplish current work and turn to next state from current state.

The SM should suspend the operation and return back to the engine once it is blocked.

The SM will be callbacked again then continue the suspend task.

The engine is just like a car engine, each ignition will turn the wheel and trigger the air conditioning compressor but there is slight different: 

- Each ignition drives one function only
- Run the wheel or trigger the the air conditioning compressor

There is only one drive shaft of the engine, 

- the wheel have to move away after turning one time so that the drive shaft could run the next parts.
- However, most of time, turning once doesn't meet our requirements, we have to keep running the wheel to arrive our destination. 
- Meanwhile, passengers complain when we are running the wheel without having the air conditioner. 

In order to keep turning the wheel and air conditioner in rotation, we need

- Keep running the wheel until we arrived
- Trigger the the air conditioning compressor until we get comfortable temperature

But,

- How does the wheel know whether we arrived or not ? 
- How does the air conditioner know whether it is comfortable ?

Therefore, 

- We provide the accellerator to the drive to control the wheel
- We also provide button to set up the temperature.

We should design the sub-system in the followings:

- Wheel Sub-System
  - A system to keep the wheel running (Wheel SM)
  - A interface to start the system (Wheel Processor)
- Cooling Sub-System
  - A system to trigger the the air conditioning compressor (Cooling SM)
  - A interface to control the system (Cooling Processor)

At last, the passenger(s) and the driver control State Machine respectively
- The driver knowns we are still not arrived and step on the accelerator to run the wheel
- The passenger(s) turn on the air conditioning and set up the temperature

I will introduce NetHandler (a State Machine, as a part of Net Sub-System, it is drived by the engine) in the following chapters.
The engine drived State Machine is called "Users of the Engine".

In order to ensure the whole system running continuously and appropriately, it is necessary to limit the running time of each engine user.

As the user of the engine, should not wait for the incoming data (such as user input) or the change of external states, and should:

- Return to the engine immediately
- Wait for the next callback to State Machine from the engine
- Then check the whether latest data has arrived or the external state is changed

State Machine runs only without any blockings, it returns back to engine once there is blocking. 
It is similar to the car in neutral; we force the engine to drive another components to avoid stalling the engine.

Don't run State Machine in loop, for example:

- Is the temperature lower than 22 degree in the car?
- No, 
  - trigger the air conditioner once
  - and back to the beginning
- Yes, 
  - reschedule the State Machine to run again after 1 minute
  - return to the engine

This State Machine flow is loop without any blockings.

This design optimized the efficiency of air conditioner in maximum but it stops all the other components, which doesn't meet the users requirements.
Because it doesn't return to the engine until accomplished the works.
The correct design is:

- Is the temperature lower than 22 degree in the car?
- No, 
  - trigger the air conditioner once 
  - and reschedule the State Machine to run again immediately
  - return to the engine
- Yes, 
  - reschedule the State Machine to run again after 1 minute.
  - return to the engine

The above design, State Machine trigger the air conditioner once (limited operation) whenever it calls back by the Engine and it will notify the engine this State Machine needs to run again. 
In this way, the engine will allocate resources to drive this State Machine again which is running this State Machine repeatedly until the temperature in the car is lower than 22 degree.

At last, call State Machine to check the temperature inside the car in every minute.
