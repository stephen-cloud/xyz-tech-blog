[WebFlux](https://docs.spring.io/spring/docs/current/spring-framework-reference/web-reactive.html) is the Spring Framework's implementation of "Web on a Reactive Stack".

This "I'm Glad I Learned" story is about one engineer's journey learning the tech. It is _definitely not_ meant to be a substitute for the reference documentation above, which is astonishingly good documentation on any subject. But specifically about this topic.

## The gist

[The Reactive Manifesto](https://www.reactivemanifesto.org/) lists the reasons we need to get away from blocking monolithic code that doesn't scale well.

The Microservice pattern (see [Martin Fowler's take](https://martinfowler.com/articles/microservices.html)) looks like it's a direction that maps to how large engineering organizations can be more nimble and is a good match for how cloud-native architectures work best.

If you remember back in the day we ran clusters of JBoss servers on a mountain of hardware in data centers, you'll understand.

We needed big iron because all the software was lumped together and processed requests by blocking a thread. Threads are by no means free.

We can do better.

## 