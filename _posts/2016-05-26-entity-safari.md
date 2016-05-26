---
categories: programming
date: 2016-05-26
layout: post
disqus_id: 2
title: "Entity Safari"
---

![](https://imgs.xkcd.com/comics/standards.png)

## Biodiversity

It's impossible to make a game without dealing with entities.
Aside from the game engine, the majority of your game lives in code that works with them.
You'd think that, after millions of attempts, we'd have found a perfect solution to implementing these omnipresent entities, but a quick look on the internet shows that the subject is still heavily contested.

The biggest two "warring camps" are what one might describe as "naive OOP", where you have an inheritance hierarchy of entity classes, and "pure ECS", where there are no entity classes, only nebulous amalgamations of components.
Even the ECS camp is undecided within itself.
Should entities be lists of components or indices into component tables?
Should components be full objects with their own methods or just plain old data with behaviour defined in procedures?

My view is that neither of these approaches are good.
Sure, the ECS camp may be right about the drawbacks of the "naive OOP" approach, particularly, but not limited to, the massive refactoring and/or poor design decisions associated with multiple inheritance.
I just think that the measures taken to fix these issues are too drastic and in the wrong direction.

Here's my proposal: "naive OOP" doesn't work because every entity is forced to derive from a single base class, which either ends up being too restrictive or too monolithic.
This forces "dynamic" workarounds like message systems, which introduce a lot of overhead on top of all of the virtual function calls.
"Pure ECS" does the same thing - instead of using the same structure for every entity, suddenly all of the structure is taken away.
You end up with a primordial soup of components, where genericity gone awry leads to an absurd amount of branching and memory usage.

With that waffle out of the way, let's start iterating towards a satisfying solution.

## Status quo

I'm going to start working from "naive OOP", which should hopefully be the most familiar.
It's based on a simple idea: that every entity derives from a common base class, and that inheritance is used to prevent code duplication:

{% highlight java %}
public abstract class Entity {
	protected Point position;

	public abstract void input();
	public abstract void update(float dt);
	public abstract void render(Graphics g);
}
{% endhighlight %}

Before actually needing to implement any entities, it looks very elegant and attractive.
The game loop is as straightforward as iterating through a singular collection of entities a few times, calling the methods of the base class:

{% highlight java %}
while(/* game is running */) {
	entities.forEach(e -> e.input());
	entities.forEach(e -> e.update(dt));
	entities.forEach(e -> e.render(g));
}
{% endhighlight %}

The only thing missing is our actual entities, so we're done with the boilerplate, right?
Not really.
It's inevitable that one comes up with two entity classes that share some state or behaviour.
Initially, this isn't a problem.
Just create a new class to serve as a base class for both of those, abstracting out the duplicated code.
But what if you have two separate base classes, and you want to combine their abstracted features?
In a language like C++, you *could* use multiple inheritance, but that puts major restrictions on how you structure your hierarchy and is generally difficult to work with.
In a language like Java, you either inherit one and deal with some duplication, or you refactor certain features out into components.
No matter what, you end up with lots of boilerplate, which somewhat negates the bonus of code reuse.

And to achieve what?
What benefit does putting every entity into a single collection have, other than making the game loop so simple?
Nothing, really, and that's what's so mind-boggling about it.
There's no real trade-off, and therefore no reason to do things this way (other than masochism), as far as I'm concerned.

## Seeing the forest for the trees

Right now, our entity class diagram would essentially look like a giant, gnarled tree.
There's a good metric to describe this called "depth of inheritance tree" (DIT).
The smaller your DIT (i.e., the flatter your hierarchy), the less chance you'll run into these situations, but you also have fewer opportunities to reuse code, except through composition.
As your DIT gets larger, you have more code to reuse through inheritance, but the chance of problems skyrockets.

So, if we've established that trying to use one base class is the root cause (heh) of our misery, let's just take that out of the equation.
Here, I've made base classes for two different entity "archetypes":

{% highlight java %}
public abstract class Prop {
	protected Point position;

	public abstract void render(Graphics g);
}

public abstract class Char {
	protected Point position;
	protected Stats stats;

	public abstract void update(float dt);
	public abstract void render(Graphics g);
}
{% endhighlight %}

A <tt>Prop</tt> is something that doesn't have any behaviour, except to render itself.
A <tt>Char</tt> is a character entity, which has some game logic and variables like health, level, etc.
You can imagine other possibilities for "archetypes" to serve as base classes.
The game loop is still quite straightforward:

{% highlight java %}
while(/* game is running */) {
	chars.forEach(c -> c.update(dt));
	props.forEach(p -> p.render(g));
	chars.forEach(c -> c.render(g));
}
{% endhighlight %}

Now, instead of a single giant inheritance tree of classes, we've split it into a "forest" of multiple smaller trees.
What's good about this?
No more singular root means no more shoehorning into a tiny interface or pushing functionality up into a kitchen sink class, and multiple inheritance situations will be rarer.
However, code reuse between these base classes is impossible without composition, which leads to entity methods that look something like this:

{% highlight java %}
public void update(float dt) {
	brain.update(dt);
	legs.update(brain, dt);
	position.update(legs, dt);
}
{% endhighlight %}

Is there a solution to this boilerplate mess?
Yes, by going fully compositional and procedural!

## Papermaking

Here are three example entity classes using pure composition:

{% highlight java %}
public class ModelProp {
	public Point position;
	public Model model;
}

public class BillboardProp {
	public Point position;
	public Image image;
}

public class StandingChar {
	public Point position;
	public Stats stats;
	public Model model;
}
{% endhighlight %}

Because there's no more inheritance, we can say we have a flat hierarchy.
This means that there's no possibility for multiple inheritance to become an issue, and that you can always add a new entity class, with full potential for code reuse, without disrupting any existing code.

But wait, where did the entity methods go?
Because we've already established that they accrue boilerplate under the presence of composition, we'll just do away with them and put behaviour in external systems (functions) that operate solely on components:

{% highlight java %}
public static void updateChar(Point position, Stats stats, float dt);
public static void renderModel(Point position, Model model, Graphics g);
public static void renderBillboard(Point position, Image image, Graphics g);
{% endhighlight %}

Now, we have total separation of data and logic.
When adding new entity classes that only mix and match existing components, the only other code that needs to be added is in the game loop, and the systems don't get touched:

{% highlight java %}
while(/* game is running */) {
	standingChars.forEach(c -> updateChar(c.position, c.stats, dt));
	modelProps.forEach(p -> renderModel(p.position, p.model, g));
	billboardProps.forEach(p -> renderBillboard(p.position, p.image, g));
	standingChars.forEach(c -> renderModel(c.position, c.model, g));
}
{% endhighlight %}

It's still quite straightforward.
The only difference between this game loop and the "forest" version is that, instead of calling entity methods, we call the appropriate systems using each entity's components.

Now, we could leave it at that, but I bet you're wondering how this approach scales.
It seems like we need to make a new class, and therefore a new collection and set of system calls in the game loop, every time we want to have a slightly different set of components together in an entity!

## Procedural possibilities

Here's where the nature of procedural programming really shines.
Instead of doing that, it's possible to add a component as an "optional" field:

{% highlight java %}
public class Char {
	public Point position;
	public Optional<Path> patrol;
	public Stats stats;
	public Model model;
}

public static void updatePatrolling(Point position, Path patrol, float dt);
{% endhighlight %}

Instead of having to make a <tt>PatrollingChar</tt> class, mostly identical to the <tt>StandingChar</tt> class except for an extra component, we compress both into a single class.
This requires some minor extra work in the game loop:

{% highlight java %}
while(/* game is running */) {
	for(Char c : chars) {
		if(c.patrol.isPresent()) {
			updatePatrolling(c.position, c.patrol.get(), dt);
		}
	}
}
{% endhighlight %}

Clearly, though, this is nicer than duplicating the system calls of <tt>StandingChar</tt> and adding one extra.
It will also look much nicer in a language with proper support for algebraic data types and pattern matching, such as Rust.
It's possible to do something similar with the <tt>ModelProp</tt> and <tt>BillboardProp</tt> classes, too:

{% highlight java %}
public class Prop {
	public Point position;
	// pretend for a moment that Java has Either...
	public Either<Model, Billboard> graphic;
}
{% endhighlight %}

The same principle would apply: inspect <tt>graphic</tt> to get either a <tt>Model</tt> or a <tt>Billboard</tt>, and call the corresponding system.
Finally, although it's not going to help in Java, it's easy to inject some data-oriented design to improve cache performance in the tight loops around system calls:

{% highlight java %}
public class Props {
	public Point[] positions;
	public Either<Model, Billboard>[] graphics;
}
{% endhighlight %}

Again, this will require a bit of extra code in the game loop, and better languages than Java will be able to deal with it cleanly.

Sure, with all of these tricks, there is a bit of extra branching and memory usage that we get in return for reducing even more boilerplate.
However, it's easy to pick and choose these trade-offs as you see fit, and not much code needs to be touched if profiling shows that you need to split a class up again to avoid the (slight) performance hits.
I think this is a good compromise between the overly-generic "pure ECS" designs and a model where every different set of components is a different type.
