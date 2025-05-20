# QP Quick Start Guide

By Leo Chen

ENGW3302

Published 2025-5-19

## Instructions for Viewing

Open this in a code editor such as VSCode or Zed and click on the preview button to view this file in its intended format or read it on GitHub.

## Introduction

It can be daunting to jump right into a codebase written using the QP/C framework, so this Quick Start Guide will walk you through a simple blinky example writtien in QP to showcase the specific quirks and features incorporated in QP! This guide will go over constructors, Active Objects, event posting, and general syntax. Shown below is the code for the QP logic, which was grabbed from this link: https://github.com/QuantumLeaps/qpc-examples/tree/3cce8da63383fec087f3e01a3c1fd922f7bea331/zephyr/blinky

```
//............................................................................
// Blinky class...
typedef struct {
// protected:
    QActive super;   // inherit QActive

// private:
    QTimeEvt timeEvt; // private time event generator
} Blinky;
extern Blinky Blinky_inst; // the Blinky active object

// protected:
static QState Blinky_initial(Blinky * const me, void const * const par);
static QState Blinky_off(Blinky * const me, QEvt const * const e);
static QState Blinky_on(Blinky * const me, QEvt const * const e);

//----------------------------------------------------------------------------
Blinky Blinky_inst;
QActive * const AO_Blinky = &Blinky_inst.super;

//............................................................................
void Blinky_ctor(void) {
    Blinky * const me = &Blinky_inst;
    QActive_ctor(&me->super, Q_STATE_CAST(&Blinky_initial));
    QTimeEvt_ctorX(&me->timeEvt, &me->super, TIMEOUT_SIG, 0U);
}

// HSM definition ----------------------------------------------------------
QState Blinky_initial(Blinky * const me, QEvt const * const e) {
    (void)e; // avoid compiler warning

    // arm the time event to expire in half a second and every half second
    QTimeEvt_armX(&me->timeEvt, BSP_TICKS_PER_SEC, BSP_TICKS_PER_SEC);

    return Q_TRAN(&Blinky_off);
}
//............................................................................
QState Blinky_off(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG: {
            BSP_ledOff();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG: {
            status = Q_TRAN(&Blinky_on);
            break;
        }
        default: {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
//............................................................................
QState Blinky_on(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG: {
            BSP_ledOn();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG: {
            status = Q_TRAN(&Blinky_off);
            break;
        }
        default: {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
```

## Part 1: Setting Up Blinky

Before QP can start anything, it needs to organize and intialize its variables. Starting with:

```
// Blinky class...
typedef struct {
// protected:
    QActive super;   // inherit QActive

// private:
    QTimeEvt timeEvt; // private time event generator
} Blinky;
extern Blinky Blinky_inst; // the Blinky active object

// protected:
static QState Blinky_initial(Blinky * const me, void const * const par);
static QState Blinky_off(Blinky * const me, QEvt const * const e);
static QState Blinky_on(Blinky * const me, QEvt const * const e);
```

### Inheritance
The struct member ```QActive super``` is essentially implementing inheritance in C. It allows the Blinky class to become a QActive object, which is what QP uses to enable event-driven programming by dividing the system into independent and concurrent components that pass messages to each other without sharing the same data. QActive objects are also commonly referred to as Active Objects (AOs). 

### Timeout Timer

The struct member ```QTimeEvt timeEvt``` is a variable that represents a timeout timer. When this timer expires, an event is posted with from within the QP framwork and it sends a timeout signal to the Blinky AO. It's up to the Blinky AO on how it wants to handle the timeout signal, but the implementation of the timeout is important in order to keep the QP app going because a complicated system with many AOs will be lefting waiting if an AO that receives a timeout signal isn't handled by that AO. 

### Initialization Preparation

Finally, the code snippet creates a Blinky AO instance for use and calls its states for use in the initialization. 

## Part 2: Initialization

```
//----------------------------------------------------------------------------
Blinky Blinky_inst;
QActive * const AO_Blinky = &Blinky_inst.super;

//............................................................................
void Blinky_ctor(void) {
    Blinky * const me = &Blinky_inst;
    QActive_ctor(&me->super, Q_STATE_CAST(&Blinky_initial));
    QTimeEvt_ctorX(&me->timeEvt, &me->super, TIMEOUT_SIG, 0U);
}

// HSM definition ----------------------------------------------------------
QState Blinky_initial(Blinky * const me, void const * const par) {
    Q_UNUSED_PAR(par);

    // arm the time event to expire in half a second and every half second
    QTimeEvt_armX(&me->timeEvt, BSP_TICKS_PER_SEC, BSP_TICKS_PER_SEC);

    return Q_TRAN(&Blinky_off);
}
```

### Globally Scoped Opaque Pointer

The code snippet starts out by calling a globally scoped opqaue pointer to the local Blinky AO instance. this is most useful in a more complicated codebase where there are many AOs that need to talk to each other through events and these globally scoped opaque pointers are how an AO can directly send an event to another AO. through the opaue pointer, one cannot access the elements inside of an AO struct. We will discuss why this pointer useful later. 

### Constructor

Next, the function ```Blinky_ctor(void)``` is a constructor for Blinky called during the project's setup. The constructor creates a local pointer to the Blinky instance called ```me```, which is often used to modify specific variables of the AO. For example, a Button AO instance could have a bolean member of the Button struct called ```pressed``` and the program would change the status by calling ```me->pressed = true;```. This boolean could then be used by multiple states inside of the Button AO or posted in an event to another AO. 

#### QActive Constructor

Moving onto the ```QActive_ctor();``` function, this is a built-in QP function that constructs the Blinky AO by passing the pointer to the Blinky QActive object to the QActive constructor. The QActive constructor then runs the Blinky AO's initial state with ```Q_STATE_CAST(&Blinky_initial)```. The state cast will run through the initial state of Blinky and then go into the Blinky_off state. 

#### Timeout Timer Constructor

After the QActive constructor is called, a constructor is called for the the timeout timer's ```QTimeEvt```. This constructor... 
1. Creates and initializes the time event object with the signal ```TIMEOUT_SIG``` for when the event is posted
2. Associates it with the Blinky AO (```&me->super```)
3. Sets the tick rate for timing (```0u```, which is the default tickrate)

### Initial State

Now we move onto the first state of the Blinky AO state machine. HSM stands for Hierarchical State Machine, which gives each state of a state machine a priority within a queue, which QP has in order to handle multiple AOs requesting system resources to perform their desired task.

#### Unused Parameter

Because this is the initialization state, there won't be any events to receive from the passed parameter ```QEvt const * const e```, so we assign the parameter to ```(void)e``` to avoid compiler warnings. 

#### Arm the Timeout Timer

```QTimeEvt_armX();``` is used to arm the timeout timer and doesn't return anything. Typically, QP functions that end in X are void functions. This function sets an initial timeout in its second parameter and sets a periodic interval for rearming in its third parameter. 

#### Transition

Now, the program is fully initialized and the initial state will return its next state, ```Blinky_off``` to the QActive constructor in ```Blinky_ctor();```. 

## Part 3: The State Machine

Now that the program is past the initialization phase, it's time to dissect how the individual states in the Blinky state machine are implemented!

### Blinky_off/Blinky_on Explained

View the sequence diagram here: https://imgur.com/a/7LbRo7z

The state first creates a QState to return called ```status```, which is returned to the QP event processor to handle the state transition. Then, based on the external signal from a posted event that Blinky_off receives, the Blinky_off state will go into the states described in the case statements. For example, if the Blinky AO is in the Blinky_off state and the timeout timer expires, the Blinky AO will receive an event with the signal ```TIMEOUT_SIG``` and then transition to Blinky_on. The ```Q_ENTRY_SIG``` and ```Q_EXIT_SIG``` (which isn't implemented in this example) cases are always entered when transitioning into and out of a state. So, in the example mentioned earlier, as the Blinky_off state will transition into Blinky_on and enter ```Q_ENTRY_SIG``` first and run the code there, then it will do whatever the Blinky_on state is designed to do. 

### Not In the Example: Event Posting Across AOs

The notion of event posting has been mentioned previously, but what would that look like if it were to come from another AO? For example, let's say a Button AO, which is an AO that manages a button on the PCB the Blinky program runs on, wants to send an event to tell Blinky to go to the off state. The Button AO would call: 

```
typedef struct{
    q_event_replyable_response_t super;
    bool pressed;
} blinky_off_event_t;

blinky_off_event_t * off_evt = Q_NEW(blinky_off_event_t, BLINKY_OFF_SIG);
QACTIVE_POST_REPLYABLE_REQUEST(AO_Blinky, 0u, off_evt, me)
```

Update to the state machine logic for this specific example with extra signals ```BLINKY_OFF_SIG``` and ```BLINKY_ON_SIG```:

```
QState Blinky_off(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG: 
        case BLINKY_OFF_SIG:
        {
            BSP_ledOff();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG:
        case BLINKY_ON_SIG:
        {
            status = Q_TRAN(&Blinky_on);
            break;
        }
        default:
        {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
//............................................................................
QState Blinky_on(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG:
        case BLINKY_ON_SIG:
        {
            BSP_ledOn();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG:
        case BLINKY_OFF_SIG:
        {
            status = Q_TRAN(&Blinky_off);
            break;
        }
        default:
        {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
```

You can see that the globally scoped pointer ```AO_Blinky``` is helpful here for posting the ```off_evt``` to the Blinky AO through the post replyable request function. The Blinky AO will receive the event with the ```BLINKY_OFF_SIG``` attached and whichever state it's in will handle that signal properly. For example, if the Blinky AO receives the ```BLINKY_OFF_SIG``` while in the Blinky_on state, it'll transition into the Blinky_off state and turn off the LED. 

### Not In the Example: Signal Subscribing

In the Event Posting section earlier, the ```BLINKY_OFF_SIG``` and ```BLINKY_ON_SIG``` from the Button AO could also be conveyed to the Blinky AO through publishing instead. Let's say you have a compliated project with many AOs, but there are multiple AOs that care about the Button AO's state. In this case, you could implement the code as so:

```
typedef struct{
    q_event_replyable_response_t super;
    bool pressed;
} blinky_off_event_t;

blinky_off_event_t * off_evt = Q_NEW(blinky_off_event_t, BUTTON_ON_SIG);
QF_PUBLISH(&off_evt, me);
```

Update to the state machine logic for this specific example with extra signals ```BUTTON_OFF_SIG``` and ```BUTTON_ON_SIG``` and the addition of signal subscribing in the initial setup of the Blinky AO.

```
QState Blinky_initial(Blinky * const me, void const * const par) {
    Q_UNUSED_PAR(par);

    // arm the time event to expire in half a second and every half second
    QTimeEvt_armX(&me->timeEvt, BSP_TICKS_PER_SEC, BSP_TICKS_PER_SEC);

    QActive_subscribe(&me->super, BUTTON_OFF_SIG);
    QActive_subscribe(&me->super, BUTTON_ON_SIG);

    return Q_TRAN(&Blinky_off);
}
//............................................................................
QState Blinky_off(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG: 
        case BUTTON_OFF_SIG:
        {
            BSP_ledOff();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG:
        case BUTTON_ON_SIG:
        {
            status = Q_TRAN(&Blinky_on);
            break;
        }
        default:
        {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
//............................................................................
QState Blinky_on(Blinky * const me, QEvt const * const e) {
    QState status;
    switch (e->sig) {
        case Q_ENTRY_SIG:
        case BUTTON_ON_SIG:
        {
            BSP_ledOn();
            status = Q_HANDLED();
            break;
        }
        case TIMEOUT_SIG:
        case BUTTON_OFF_SIG:
        {
            status = Q_TRAN(&Blinky_off);
            break;
        }
        default:
        {
            status = Q_SUPER(&QHsm_top);
            break;
        }
    }
    return status;
}
```

This change showcases how one could simply send out an event to all AOs that are subscribed to the ```BUTTON_ON_SIG``` and ```BUTTON_OFF_SIG``` and how those receiving AOs would handle those signals. Essentially, when an AO wants to send out a message to all AOs that care, this would be the better method of getting this message to all those AOs instead of posting directly to each one every time. 
