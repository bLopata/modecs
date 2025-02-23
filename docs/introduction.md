# Introduction

## Philosophy

With ModECS it is best to follow the [Unix Philosophy](https://en.wikipedia.org/wiki/Unix_philosophy). In general, this means that you should strive to make your systems and components as simple as possible. If this means breaking one large feature into multiple smaller systems that pass information through components, generally that will be more maintainable, composable, and reusable than larger monolithic systems will be. That said, each system incurs one loop through all of its own entities, so if performance is an important aspect of the system it may be better to design it monolithically. Use your best discretion here.

## Runtime Composition

Not only can you register and add new components to entities during runtime, but new systems can be registered during runtime as well (after `engine.start()` has been called and the update loop is running). This creates an environment much like Unity3D or Unreal Engine provides, where the game world constantly runs while the user edits features in real-time without restarting the IDE. Re-registering something with the same name simply overwrites the previous registration.

## Events

The ModECS engine inherits [`eventemitter3`](https://github.com/primus/eventemitter3). The prime goal of this is to keep communication decoupled throughout all of the game's individual features, supplementing the fact that systems can be added and removed during runtime.

This creates a powerful paradigm in which you can create entirely "pluggable" features that can interact with systems both registered and yet to be registered.

## Guidelines

- Information should only enter a system via components.
  - Do not listen to events inside of system update loops. 
- Information can and should exit a system via both components and events. 
  - For advice on when to do so, see `Generic Logic vs Gamemode-Specific Logic`.
- Components should remain small, flat, and **should never change shape** to ensure performance and maintainability. 

## Modes

Modes are self-contained packages of code that register new systems and components that you can "plug in" to the engine by calling `engine.use()`.

E.g.

`JitterMode.js`
```javascript
export default engine => {

  engine.registerComponent('JITTER', {amount: 5})

  engine.registerSystem(
    'Jitter',
    ['POSITION', 'JITTER'],
    () => {
      const randomRange = (min, max) => Math.random() * (max - min) + min
      return (position, jitter) => {
        let xJitter = randomRange(-jitter.amount, jitter.amount)
        let yJitter = randomRange(-jitter.amount, jitter.amount)

        // jittered a little too much, emit an event
        if(xJitter >= jitter.amount/2) engine.emit('jitterbug-happened', position, xJitter)

        position.x += xJitter
        position.y += yJitter
      }
    }
  )

}
```

`main.js`
```javascript
engine.use(require('./JitterMode'))

engine.registerComponent('JITTERBUG', {amount:0})
engine.registerSystem(
  'JitterBugReactor',
  ['JITTERBUG'],
  () => {
    return (jitterbug) => {
      // what shall we do with this jitterbug?
    }
  }
)

// this is where game logic is best kept
engine.on('jitterbug-happened', (position, xJitter) => {
  const entityId = position.id

  // stop the jittering behavior
  engine.removeComponent(entityId, 'JITTER')

  // start the jitterbugging behavior, whatever that may be
  engine.addComponent(entityId, 'JITTERBUG', {amount: xJitter})
})
```

## Reusability

If you have developed a few games in the past, you may be familiar with the feeling of redundancy when re-implementing similar features over and over again in different games. ModECS, and ECS in general, attempts to provide a method to the madness that this creates.

### Generic Logic vs Gamemode-Specific Logic

Generic logic that doesn't necessarily apply only to your gamemode should be contained within a system. Conversely, gamemode-specific logic should be handled by an event listener, defined outside of the emitting system.

When breaking up features into smaller system pieces, the generic information should flow from system-to-system within components. Conversely, gamemode-specific information should be delegated and handled in an event listeners. The gamemode logic can then react appropriately to the vision of the game, meanwhile the system logic remains reusable for other types of gamemodes.

This is more or less a guideline to follow, and not necessarily a rule to abide by. Most of the time you will need to create systems with game-specific logic in them until you are able to discover what behavior is generic and what behavior is specific.

As time goes on, you may notice that things are more reusable and easier to debug once broken down into smaller systems with a focus. Runtime composition gives us the ability to stop and start any systems at-will while the game is running, without breaking any other systems or throwing any errors, because everything is communicating through components and events. Imagine a scenario where you can turn the gravity off for all entities in the game, simply by removing/disabling the gravity system!

### Declarative vs Imperative Logic

There are two levels of logic at-play in the ModECS framework.

System update loops and event listeners contain imperative code. Imperative logic describes the behavior and exactly how it should happen.

However, when emitting events from within system update loops, and when defining the event listeners that react to those events, declarative logic is in use. Declarative logic describes reactions, when and what should happen. We do not care how things are done at this level of logic.

Utilizing both of these paradigms of logic in-tandem with one another is the bread and butter of ModECS.