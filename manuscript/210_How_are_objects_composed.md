Composing a web of objects
==========================

Three important questions
-------------------------

Ok, I told you that there is such a thing as a web of objects exists, 
that there are connections, protocols and such, but there is one thing 
I left out: how does a web of objects come into existence?

This is, of course, a fundamental question, because if we are not able 
to build a web, we do not have a web. The question itself is a little 
more tricky that you may think and it contains three other questions 
that we need to answer:

1.  How does an object obtain a reference to another one in the web?
2.  When are objects composed?
3.  Where are objects composed?

For now, you may have some trouble understanding why these questions 
are important, but the good news is that you won't have to trust me 
too long, because these questions are the topic of this chapter.

Before we take a deep dive, let's try to answer these questions for a 
really simple example code of a console application:

{lang="csharp"}
~~~
public static void Main(string[] args)
{
  var sender = new Sender(new Recipient());
}
~~~

1.  How does an object (`Sender`) obtain a reference to another one 
(`Recipient`)? The reference is passed through constructor.
2.  When are objects composed? During application startup.
3.  Where are objects composed? At application entry point (`Main()` 
method)

Depending on circumstances, we have sets of best answers to these 
questions. But first, let us take the questions on one by one.

How does sender obtain a reference to recipient?
------------------------------------------------

There are few ways, each of them useful in certain circumstances:

### Pass as constructor parameter

Two objects can be composed by passing one into the constructor of
another:

{lang="csharp"}
~~~
sender = new Sender(recipient);
~~~

A sender is composed with a recipient and saves a reference to
the recipient in a private field for later, like this:

{lang="csharp"}
~~~
public Sender(Recipient recipient)
{
  this._recipient = recipient;
}
~~~

Composing using constructors has one significant advantage. The code 
that passes `Recipient` to `Sender` through constructor is most often 
in a totally different place than the code using `Sender`.
Thus, the code using `Sender` is not aware that `Sender` stores a 
reference to `Recipient` inside it. This
basically means that when `Sender` is used, e.g. like this:

{lang="csharp"}
~~~
sender.DoSomething()
~~~

`Sender` may send message to `Recipient` internally, but the code
invoking the `DoSomething()` method is completely unaware of that. 
Which is good, because "what you hide, you can change" - if we decide 
that the `Sender` needs not use the `Recipient` to do its duty, the 
code that uses `Sender` does not need to change at all - it still uses 
the `Sender` the same way:

{lang="csharp"}
~~~
sender.DoSomething()
~~~

All we have to change is the composition code:

{lang="csharp"}
~~~
//no need to pass a reference to Recipient anymore
sender = new Sender();
~~~

Passing into constructor is a great way to compose sender with a recipient
permanently. In order to be able to do this, a `Recipient` must, of
course, exist before a `Sender` does. Another less obvious requirement
for this composition is that `Recipient` must be usable at least as long
as `Sender` is usable. In other words, the following is nonsense:

{lang="csharp"}
~~~
sender = new Sender(recipient);
recipient.Dispose(); //but sender is unaware of it 
                     //and may still use recipient in:
sender.DoSomething();
~~~

### Pass inside a message (i.e. as a method parameter)

Another common way of composing objects together is passing one object
as a parameter of another object's method call:

{lang="csharp"}
~~~
sender.DoSomethingWithHelpOf(recipient);
~~~

In such case, the objects are most often composed temporarily, just for 
the time of execution of this single method:

{lang="csharp"}
~~~
public void DoSomethingWithHelpOf(Recipient recipient)
{
  //... perform some logic
  
  recipient.HelpMe();
  
  //... perform some logic
}
~~~

This is a great way to compose objects when we want to use the same
sender with different recipients at different times (most often from
different parts of the code):

{lang="csharp"}
~~~
//in one place
sender.DoSomethingWithHelpOf(recipient);

//in another place:
sender.DoSomethingWithHelpOf(anotherRecipient);

//in yet another place:
sender.DoSomethingWithHelpOf(yetAnotherRecipient);
~~~

### Return recipient from a method

This method of composing objects uses another intermediary object - a
factory (that creates new recipient instance on each call) or a cache 
(that usually yields the same object many times when asked with the 
same arguments). Most often, the sender is given a reference to this 
intermediary object as a constructor parameter (an approach we already 
discussed):

{lang="csharp"}
~~~
sender = new Sender(factory);
~~~

and then the intermediary object is used to deliver other objects:

{lang="csharp"}
~~~
public class Sender
{
  //...

  public DoSomething() 
  {
    var recipient = _factory.CreateRecipient();
    recipient.DoSomethingElse();
  }
}
~~~

This kind of composition is beneficial when a different recipient is 
needed each time `DoSomething()` is called (much like in case of 
 previously discussed approach of passing recipient as a method 
 parameter), but at the same time (contrary to passing recipient as a 
 method parameter), the code using the `Sender` should not (or cannot) be 
 responsible for supplying a recipient.

To be more clear, here is a comparison of two approaches: passing 
recipient inside a message:

{lang="csharp"}
~~~
//user of Sender passes a recipient:
public DoSomething(Recipient recipient) 
{
  recipient.DoSomethingElse();
}
~~~

and obtaining from factory:

{lang="csharp"}
~~~
//user of Sender does not know about Recipient
public DoSomething() 
{
  var recipient = _factory.CreateRecipient();
  recipient.DoSomethingElse();
}
~~~

### "Register" a recipient with already created sender

This means passing a recipient to an **already created** sender 
(contrary to passing as constructor parameter where recipient was 
passed **during** creation) as a parameter of a method that stores the 
reference for later use. This may be a "setter" method, although I do 
not like naming it according to convention "setWhatever()" - after Kent 
Beck (Implementation Patterns book) I find this convention too much 
implementaion-focused instead of purpose-focused. Thus, I pick 
different names based on what domain concept is modeled by the 
registration method or what is its purpose.

Note that this is similar to "pass inside a message" approach, only 
this time, the passed recipient is not used immediately and forgotten, 
but rather remembered for later use.

Anyway, a quick example. Suppose we have a temperature sensor that can
report its current and historically mean value for the current date to 
whoever registers with it. If no one registers, it still does its job, 
because mean value is based on historical data and someone might 
register for it at any time, right? So, part of the definition of such 
a sensor could look like this:

{lang="csharp"}
~~~
public class TemperatureSensor
{
  private TemperatureObserver _observer 
    = new NullObserver(); //ignores values by default
  private Temperature _meanValue = Temperature.Zero();
  //+ maybe more fields related to storing historical data

  public void Run()
  {
    while(/* needs to run */)
    {
      var currentValue = /* get current value somehow */;
      _meanValue = /* update mean value somehow */;

      _observer.NotifyOn(currentValue, _meanValue);
      
      WaitUntilTheNextMeasurementTime();
    } 
  }
}
~~~

As you can see, by default, the sensor reports its values to nowhere
(`NullObserver`), which is a safe default value (using a `null` 
instead would cause exceptions or force us to put an ugly null check 
inside the `Run()` method). Still, we want to be able to supply our own 
observer one day, when we start caring about the measured and 
calculated values. This means we need to have a method inside the 
`TemperatureSensor` class to overwrite this default "do-nothing"
observer with one that we provide:

{lang="csharp"}
~~~
public void FromNowOnReportTo(TemperatureObserver observer)
{
  _observer = observer;
}
~~~

This lets us overwrite the observer with a new one should we ever need 
to do it. 

Time for a general remark. Allowing registering recipients after a 
sender is created is a way of saying: "the recipient is optional - if 
you provide one, fine, if not, I will do my work without it". Please, 
do not use this kind of mechanism for required recipients - these 
should all be passed through constructor, making it harder to create 
invalid objects that are only partially ready to work. Placing a 
recipient in a constructor signature is effectively saying that "I will 
not work without it". 

Now, the observer API we just skimmed over gives us the possibility to
have a single observer at any given time. When we register new observer,
the reference to the old one is overwritten. This is not really useful 
in our context, is it? With real sensors, we often want them to report 
their measurements to multiple places (e.g. we want the measurements 
printed on screen, saved to database, used as part of more complex 
calculations). This can be achieved in two ways.

The first way would be to just hold a collection of observers in our 
sensor, and add to this collection whenever a new observer is registered:

{lang="csharp"}
~~~
IList<TemperatureObserver> _observers 
  = new List<TemperatureObserver>();

public void FromNowOnReportTo(TemperatureObserver observer)
{
  _observers.Add(observer);
}
~~~

In such case, reporting would mean iterating over the observers list:

{lang="csharp"}
~~~
...
foreach(var observer in _observers)
{
  observer.NotifyOn(currentValue, meanValue);
}
...
~~~

Another, more flexible option, is not to introduce this collection in 
the sensor, but instead, create a special kind of "broadcasting 
observer" that would hold collection of other observers (welcome 
composability!) and broadcast the values to them every time it itself 
receives those values. It could be created and registered like this:

{lang="csharp"}
~~~
var broadcastingObserver 
  = new BroadcastingObserver(
      new DisplayingObserver(),
      new StoringObserver(),
      new CalculatingObserver());

sensor.FromNowOnReportTo(broadcastingObserver);
~~~

This would let us change the broadcasting implementation without
touching either the sensor code or the other observers. For example, we 
might introduce `ParallelBroadcastObserver` that would notify each observer
asynchronously instead of sequentially and put it to use by changing 
the composition code only:

{lang="csharp"}
~~~
var broadcastingObserver 
  = new ParallelBroadcastObserver(
      new DisplayingObserver(),
      new StoringObserver(),
      new CalculatingObserver());

sensor.FromNowOnReportTo(broadcastingObserver);
~~~

Anyway, as I said, use registering instances very wisely and only if you
specifically need it. Also, if you do use it, evaluate how allowing
changing observers at runtime is affected by multithreading scenarios.
This is because maintaining a changeable field (or a collection) 
throughout the object lifetime means that multiple thread might access 
it and get in each other's way.

Where are objects composed?
---------------------------

Ok, we went over some ways of passing a recipient to a sender. The big 
question is: which code is going to pass the reference?

For almost all of the approaches described above there is no 
limitation - you pass the reference where you need to pass it.

There is one approach, however, that is more limited, and this approach 
is **passing as constructor parameter**.

Why is that? Well, remember we were talking about separating objects 
usage from construction, right? And invoking constructor 
imples creating an object, right? Which implies that we can assemble 
objects using constructors only in the places we separated creation of 
objects to.

Typically, there are two such types of places: **composition root** and 
**factories**. Let us take them one by one.

### Composition Root

A composition root is a location near application entry point where 
you compose the part of the system on which you invoke your `Run()`, 
`Execute()`,  `Go()` or whatever.

For simplification, let's take an example of a console application. 
Usually, when not using any framework, your application is symbolized 
as a class:

{lang="csharp"}
~~~
public static void Main(string[] args)
{
  var myApplication = new MyApplication();
  myApplication.RunWith(args);
}
~~~

The above is a simple composition root.

TODO

### Factories

Factories are objects responsible for creating other objects. They are 
a layer of abstraction over constructors. A lot of times I talk with 
people, they do not understand the benefits of using factories "since 
we already have the `new` operator". But factories present a number of 
benefits:

TODO what a factory is? a small example?

#### They allow creating objects polymorphically

Typically, a return type of a factory is an interface or, at worst, an 
abstract class. This means that whatever uses the factory, knows only 
that it receives an object of that type.

Let's say we have a factory like this:

{lang="csharp"}
~~~
public class Version1ProtocolMessageFactory
{
  public Message createFrom(MessageData rawData)
  {
    switch(rawData.MessageType)
    {
      case Messages.SessionInit:
        return new SessionInit(rawData);
      case Messages.SessionEnd:
        return new SessionEnd(rawData);
      case Messages.SessionPayload:
        return new SessionPayload(rawData);
      default:
        throw new UnknownMessage(rawData);
    }
  }
}
~~~

Note that the factory can create many different types of messages 
depending on what is in the raw data, but for the user of the factory, 
this is irrelevant. All that is seen by the user is this:

{lang="csharp"}
~~~
var message = _messageFactory.NewInstanceFrom(rawData);
message.ValidateUsing(_primitiveValidations);
message.ApplyTo(_sessions);
~~~

So, whatever is going on in the factory, it is hidden from the users 
of the factory. 

TODO no need to change if rule changes

#### They are themselves polymorphic

So far, I keep talking about composability over and over again so much 
that you're probably already sick of the word. But hey, here comes 
another benefit of the factory - in contrast to constructors, they are 
composable.

We may have a class that uses a factory:

{lang="csharp"}
~~~
public class MessageInbound
{
  private MessageFactory _factory;
  
  public MessageInbound(MessageFactory factory)
  {
    _factory = factory;
  }
  
  public void Process(MessageData data)
  {
    Message message = _factory.NewMessageFrom(data);
    message.ValidateUsing(_primitiveValidations);
    message.ApplyTo(_system);
  }
}
~~~

Object of this class needs to be composed with factory implementing 
the `MessageFactory` interface in a composition root like this:

{lang="csharp"}
~~~
new MessageInbound(new BinaryMessageFactory());
~~~

Suppose that one day we need to reuse the logic of `MessageInbound` in 
another place, but make it create XML messages instead of binary 
messages. If we used a constructor to create messages, we would need 
to either copy-paste the `MessageInbound` class code and change the 
line where message is created, or we would need to make an `if` 
statement inside the `MessageInbound` to choose which message we are 
required to create.

This is not reuse - this is hacking.

Thankfully, thanks to the factory, we can use the `MessageInbound` 
just as it is, but compose it with a different factory. So, we will 
have to instances of `MessageInbound` class in our composition root:

{lang="csharp"}
~~~
var binaryMessageInbound
  = new MessageInbound(
      new BinaryMessageFactory(), 
      BinListenAddress); 

var xmlMessageInbound
  = new MessageInbound(
      new XmlMessageFactory(), 
      XmlListenAddress);
~~~

TODO make each a subchapter of L4

1.  They allow creating objects polymorphically (hiding of 
created type):
1.  They are themselves polymorphic (hiding of object creation rule)
1.  They allow putting more complex creation logic in one place 
(reduce redundancy)
1.  They allow decoupling from some of the dependencies of created 
objects (lower coupling)
1.  They allow naming rulesets for object creation (better readability)

When are objects composed?
--------------------------

Most of our system is assembled up-front when the application
starts and stays this way until the application finishes executing. But 
not all of it. Let's call this part the static part of the web. We 
will talk about **composition root** pattern that guides definition of 
that part.

Apart from that, there's the dynamic part. The first thing that leads 
to this dynamism is the lifetime of the objects that are connected. 
Some objects represent requests that arrive during the application 
runtime, are processed and then discarded. When there is no need to 
process such a request anymore, the objects representing it are 
discarded as well. Other objects represent items in the cache that live 
for some time and then expire etc. so it's impossible to define these 
objects up-front. Soon, we will talk about the **Factory** pattern that 
will let us handle creation and composition of such short-lived objects.

Another thing affecting the dynamic part of the web is the temporary 
character of connections themselves. The objects may have a lifespan 
as long as the application itself, but be connected only the needs of a 
single interaction (e.g. when one object is passed to a method of 
another as an argument) or at some point during the application 
runtime. We will talk about doing these things by passing objects 
inside messages and by using the **Observer** pattern.

Where are objects composed?
---------------------------




First, let's take a look at different ways of acquiring an object by
another object:


Now that we discussed how to pass a reference to recipient into a
sender, let's take a look at places it can be done:

TODO factory and composition root