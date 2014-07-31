Web, messages and protocols
===========================

In the last chapter, we talked a little bit about why composability is
valuable, now let's flesh a little bit of terminology to get more
precise understanding.

So, again, what does it mean to compose objects?
------------------------------------------------

Basically it means that an object has obtained a reference to another
object and is able to invoke methods on it. By being composed together, two
objects form a small system that can be expanded with more objects as
needed. Thus, a bigger object oriented system forms something similar to
a web:

![Web of objects](images/WebOfObjects.png)

where the circles are the objects and the arrows are methods invocations
from one object on another.

If we take the web metaphor a little bit further, we can note some
similarities to e.g. a TCP network:

1.  An object can send **messages** to other objects (i.e. call methods
    on them - arrows on the above diagram) using **interfaces**. Each
    message has a **sender** and a **recipient**.
2.  In order to send a message to a recipient, a sender has to acquire
    an **address** of the recipient, which, in object oriented world, we
    call a reference (actually, prior to virtual machines, references
    were just that - addresses in memory. With virtual machines around,
    this may be a little more complicated).
3.  A communication between sender and recipients has to obey certain
    **protocol** (don't worry if you do not see th analogy now - I'll 
    follow up with more explanation of this topic later).

Let us try to apply this terminology to an example. Let's say that we 
have an anti-fire alarm system in an office that, when triggered, makes 
all lifts go to bottom floor, opens them and then disables them. Among 
others, the office contains automatic lifts, that contain their own
control systems and mechanical lifts, that are controlled from the 
outside by a special custom mechanism. Now to write some code! We will
have some objects like alarm, automatic lift and mechanical lift.

First, we do not want the alarm to have to distinguish between automatic 
and mechanical lifts - this would only add complexity to alarm system. 
Thus, we need a special **interface** (let's call it `Lift`) to communicate 
with both `AutoLift` and `MechanicalLift`. Through this interface, 
an alarm will be able to send messages to both types of lifts without 
having to know a difference between them.

{lang="csharp"}
~~~
public interface Lift
{
  ...
}

public class AutoLift : Lift
{
  ...
}

public class MechanicalLift : Lift
{
  ...
}
~~~

Next, to be able to communicate with specific lifts through the `Lift` 
interface, an alarm object has to acquire **"addresses"** of the lift objects
 (i.e. references to them). We can pass them e.g. through a constructor:

{lang="csharp"}
~~~
public class Alarm
{
  //obtain "addresses" through here
  public Alarm(IEnumerable<Lift> lifts)
  {
    //store the "addresses" for later use
    _lifts = lifts;
  }

  private readonly IEnumerable<Lift> _lifts;

}
~~~

Then, the alarm can send three kinds of **messages**: `GoToBottomFloor()`,
`OpenDoor()`, and `DisablePower()` to any of the lifts through the
`Lift` interface:

{lang="csharp"}
~~~
public interface Lift
{
  void GoToBottomFloor();
  void OpenDoor();
  void DisablePower();
}
~~~

and, as a matter of fact, it sends all these messages when triggered:

{lang="csharp"}
~~~
public void Trigger()
{
  foreach(var lift in _lifts)
  {
    lift.GoToBottomFloor();
    lift.OpenDoor();
    lift.DisablePower();
  }
}
~~~

By the way, note that the order in which the messages are sent **does** 
matter. For example, if we disabled the power first, asking the powerless 
lift to go anywhere would be impossible.

In this communication, `Alarm` is a **sender** - it knows what it is
sending, it knows why, but does not know exactly what are the recipients
going to do when they receive the message - its knowledge ends on the 
fact that it is communicating through the `Lift` interface. 
The rest is left to objects that implement `Lift` (namely, `AutoLift` 
and `MechanicalLift`). They are **recipients** - they do not know who 
they got the message from (unless they are told in the content of the message somehow - but even then they can be cheated), but they know how to react, based on who they are, what the kind of the message they received  and what's the message content (i.e. method arguments - in this simplistic example there are none, actually).

To illustrate that this separation between a sender and a recipient does, 
in fact, exist, it is sufficient to say that we could even write an 
implementation of `Lift` interface that would just ignore the messages 
it got from the alarm (or fake that it did what it was asked for)
and the alarm will not even notice.

![Sender, interface, and recipient](images/SenderRecipientMessage.png)

Ok, I hope we got that part straight. Time for some new requirements.
It has been decided that whenever any malfunction happens in the
lift when it is executing the alarm emergency procedure, the lift object
should report this by throwing an exception called
`LiftUnoperationalException`. This affects both `Alarm` and implementations
of `Lift`:

1.  The `Lift` implementations need to know that when a malfunction 
    happens, they should report it by throwing the said exception.
2.  The `Alarm` must be ready to handle the exception thrown from lifts
    and act accordingly (e.g. still try to secure other lifts).

Here is an exemplary code of `Alarm` handling the malfunction reports:

{lang="csharp"}
~~~
public void Trigger()
{
  foreach(var lift in _lifts)
  {
    try
    {
      lift.GoToBottomFloor();
      lift.OpenDoor();
      lift.DisablePower();
    }
    catch(LiftUnoperationalException e)
    {
      report.ThatCannotSecure(lift);
    }
  }
}
~~~

In other words, there is a **protocol** between `Alarm` and `Lift` that 
must be adhered to by both sides. 

## Summary

Each of the objects in the web can receive messages and most of them
send messages to other objects. Throughout the next chapters, I will
refer to an object sending a message as sender and an object receiving a
message as recipient.