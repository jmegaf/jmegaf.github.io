---
categories: programming
date: 2016-05-25
layout: post
disqus_id: 1
title: "Mean, Clean State Machine"
---

![](http://assets.amuniversal.com/3d0a9fd0a040012f2fe600163e41dd5b)

## What's in a state machine?

State machines are everywhere in game programming.
Your player character might be a state machine, with different states for running, jumping, shooting, and swimming.
Your menu system can be a state machine, with different states for the main menu, gameplay, pause screen, etc.
Coroutines are state machines, too, and they're useful for cutscenes, turn-based gameplay, and many other parts of game logic that transcend ticks or frames.

Distilled to the essentials, what does a state machine do?
You have "something" that can be in multiple states, each with their own behaviours.
These states can also have transitions from one to another.
Sounds pretty simple, right?
One would expect that it shouldn't involve much code, then, and I agree.

Now, a lot of the time when you hear about a "state machine", it's as part of the term "finite state machine".
I'm not a big fan of FSMs, because they imply having a separate object that "manages" the states.
The caveats involved also tend to lead people to come up with convoluted workarounds, like hierarchical state machines or state machines with a state stack.
Instead, I found that it was very effective to avoid using a manager in the first place.

## Firing the manager

So, how do we go about getting rid of this state manager, if it's so bad?
Let's dig right into it by implementing this really simple state machine for a turnstile.

![](https://upload.wikimedia.org/wikipedia/commons/thumb/9/9e/Turnstile_state_machine_colored.svg/1000px-Turnstile_state_machine_colored.svg.png)

First of all, there are two possible behaviours: reacting to a push, and reacting to a coin being inserted.
Most state machine tutorials would get you to write a corresponding interface, like so:

{% highlight java %}
public interface State {
	public void push();
	public void coin();
}
{% endhighlight %}

The states would be passed a reference to the state manager when they're created, so they can notify it when they want a transition.
This turns out to be very ugly, and breaks encapsulation.
Why should the states be coupled to their manager?
Instead, we can use the return values of the behaviours to send a signal to the state manager:

{% highlight java %}
public enum StateID {
	LOCKED,
	UNLOCKED
}

public interface State {
	public StateID push();
	public StateID coin();
}
{% endhighlight %}

Now, the states are decoupled and completely self-contained.
However, we still have a FSM, and we still end up with a manager that is nearly all boilerplate.
Can we do better?
Yes, we can, with one simple tweak to the interface:

{% highlight java %}
public interface State {
	public State push();
	public State coin();
}
{% endhighlight %}

The way this works is: each state will return a new instance of the state that was transitioned to, or itself, if there was no transition.
Let's implement the concrete states to get an idea:

{% highlight java %}
public class Locked implements State {
	public State push() {
		System.out.println("The turnstile refuses to budge.");
		return this; // no transition
	}

	public State coin() {
		System.out.println("The mechanism unlocks.");
		return new Unlocked(); // unlock
	}
}

public class Unlocked implements State {
	public State push() {
		System.out.println("The turnstile locks behind you.");
		return new Locked(); // lock
	}

	public State coin() {
		System.out.println("The coin falls into a return tray.");
		return this; // no transition
	}
}
{% endhighlight %}

It's straightforward to use - no manager required:

{% highlight java %}
State current = new Locked();

while(/* program is running */) {
	if(/* "push" was input */) {
		current = current.push();
	}

	if(/* "coin" was input */) {
		current = current.coin();
	}
}
{% endhighlight %}

The important thing is that the current state is updated with the result of the methods.

## Creative variation

So, how does this scale to situations where, with a FSM, a stack or some other more advanced system would be required?
For example, consider a state machine for screens in an RPG.
There would be an "overworld" state, a "battle" state, and a "pause" state.
Either of the "overworld" and "battle" states can be paused, and the game has to be able to resume to its previous state, unchanged.

The solution is to encode the structure by holding references to other states within the state objects themselves, and possibly returning them instead of the current state or a new one.  In this case, the "pause" state would take a reference to the previous state in its constructor, and return it when the game should be resumed:

{% highlight java %}
public interface Screen {
	public Screen update();
}

public class Overworld implements Screen {
	public Screen update() {
		if(/* pause button was pressed */) {
			return new Pause(this);
		}

		return this;
	}
}

public class Battle implements Screen {
	// same pausing logic as Overworld
}

public class Pause implements Screen {
	private Screen previous;

	public Pause(Screen previous) {
		this.previous = previous;
	}

	public Screen update() {
		if(/* pause button was pressed */) {
			return previous;
		}

		return this;
	}
}
{% endhighlight %}

It also turns out that chain-of-responsibility behaviours are very easy to implement with this approach.
If the "pause" screen wanted to draw some text and blur on top of the paused screen, it can simply call the method on <tt>previous</tt>:

{% highlight java %}
// method would be added to the Screen interface
public void draw(Graphics g) {
	previous.draw(g);
	g.blur();
	g.drawText("Paused");
}
{% endhighlight %}

Using an external stack, each state would need to expose flags like <tt>doRenderThrough</tt>, and more boilerplate would need to go into the state manager.

## Appendix

For comparison, here's the rest of the code for the first approach, where the states call methods on the state manager directly:

{% highlight java %}
public class Turnstile {
	private State current;

	public Turnstile() {
		current = new Locked(this);
	}

	public void push() {
		current.push();
	}

	public void coin() {
		current.coin();
	}

	public void lock() {
		current = new Locked();
	}

	public void unlock() {
		current = new Unlocked();
	}
}

public class Locked implements State {
	private Turnstile manager;

	public Locked(Turnstile manager) {
		this.manager = manager;
	}

	public void push() {
		System.out.println("The turnstile refuses to budge.");
	}

	public void coin() {
		System.out.println("The mechanism unlocks.");
		manager.unlock();
	}
}

public class Unlocked implements State {
	private Turnstile manager;

	public Unlocked(Turnstile manager) {
		this.manager = manager;
	}

	public void push() {
		System.out.println("The turnstile locks behind you.");
		manager.lock();
	}

	public void coin() {
		System.out.println("The coin falls into a return tray.");
	}
}
{% endhighlight %}

And the second, where they return the state ID to transition to:

{% highlight java %}
public class Turnstile {
	private StateID currentID;
	private State current;

	public Turnstile() {
		currentID = StateID.LOCKED;
		current = new Locked(this);
	}

	private void transition(StateID newID) {
		if(newID != currentID) {
			if(newID == StateID.LOCKED) {
				current = new Locked();
			} else if(newID == StateID.UNLOCKED) {
				current = new Unlocked();
			}

			currentID = newID;
		}
	}

	public void push() {
		transition(current.push());
	}

	public void coin() {
		transition(current.coin());
	}
}

public class Locked implements State {
	public StateID push() {
		System.out.println("The turnstile refuses to budge.");
		return StateID.LOCKED;
	}

	public StateID coin() {
		System.out.println("The mechanism unlocks.");
		return StateID.UNLOCKED;
	}
}

public class Unlocked implements State {
	public StateID push() {
		System.out.println("The turnstile locks behind you.");
		return StateID.LOCKED;
	}

	public StateID coin() {
		System.out.println("The coin falls into a return tray.");
		return StateID.UNLOCKED;
	}
}
{% endhighlight %}

We saved ourselves from a lot of nasty boilerplate code, not even considering the extra work needed to add a stack.
