# SwiftUI Animation Patterns Reference

## Core Concepts

### How SwiftUI Animations Work

State changes are the only way to trigger view updates. By default, changes aren't animated, but SwiftUI provides mechanisms to animate them.

```swift
struct ContentView: View {
    @State private var flag = false

    var body: some View {
        Rectangle()
            .frame(width: flag ? 100 : 50, height: 50)
            .onTapGesture {
                withAnimation(.linear) { flag.toggle() }
            }
    }
}
```

**Animation Process:**
1. State change triggers view tree re-evaluation
2. SwiftUI compares new view tree to current render tree
3. Animatable properties are identified
4. Timing curve generates progress values (0 to 1)
5. Values interpolate smoothly (~60 fps)

**Key Characteristics:**
- Animations are additive and cancelable
- Always start from current render tree state
- Blend smoothly when interrupted

## Property Animations vs Transitions

### Property Animations

Property animations interpolate changed properties of views that exist before and after state change:

```swift
struct ContentView: View {
    @State private var flag = false

    var body: some View {
        Rectangle()
            .frame(width: flag ? 100 : 50, height: 50)
            .animation(.default, value: flag)
            .onTapGesture { flag.toggle() }
    }
}
```

### Transitions

Transitions are animations for views being inserted or removed from the render tree:

```swift
var body: some View {
    VStack {
        Button("Toggle") {
            withAnimation { flag.toggle() }
        }
        if flag {
            Rectangle()
                .frame(width: 100, height: 100)
                .transition(.scale)
        }
    }
}
```

**Key Insight:** Different branches in conditionals have different identities, triggering transitions instead of property animations.

## Controlling Animations

### 1. Implicit Animations

Use `.animation(_:value:)` to animate when a specific value changes:

```swift
Rectangle()
    .frame(width: flag ? 100 : 50, height: 50)
    .animation(.default, value: flag)
```

**Important:** Always use the value parameter (the version without it is deprecated).

**Placement Matters:**
```swift
// Animation applies to frame and rectangle
Rectangle()
    .frame(width: flag ? 100 : 50, height: 50)
    .animation(.default, value: flag)
```

### 2. Explicit Animations

Use `withAnimation` to wrap state changes that should be animated:

```swift
Button("Animate") {
    withAnimation(.spring) {
        flag.toggle()
    }
}
```

**Characteristics:**
- Animates all changes from state changes in the closure
- Scope defined by closure, not view tree position
- Good for event-driven animations

### 3. Binding Animations

Apply animation when a binding's value is set:

```swift
struct ContentView: View {
    @State private var flag = false

    var body: some View {
        ToggleRectangle(flag: $flag.animation(.default))
    }
}
```

### iOS 17+: Scoped Animations

Scope animations to specific modifiers:

```swift
Text("Hello World")
    .opacity(flag ? 1 : 0)
    .animation(.default) {
        $0.rotationEffect(flag ? .zero : .degrees(90))
    }
```

This animates only the rotation, not the opacity.

### When to Use Which

**Implicit Animations:**
- Animations tied to specific value changes
- Precise control over view tree scope

**Explicit Animations:**
- Event-driven animations (button taps, gestures)
- Easy to distinguish model updates from user interactions

## Timing Curves

### Built-in Curves

- `.linear` - Constant speed
- `.easeIn` - Starts slow, ends fast
- `.easeOut` - Starts fast, ends slow
- `.easeInOut` - Slow-fast-slow
- `.default` - System default
- Spring curves - Physics-based
- `.bouncy` (iOS 17+) - Overshoots target

### Animation Modifiers

```swift
// Speed
.animation(.default.speed(2.0), value: flag)

// Delay
.animation(.default.delay(1.0), value: flag)

// Repeat
.animation(.default.repeatCount(3, autoreverses: true), value: flag)
```

### Custom Timing Curves (iOS 17+)

```swift
struct MyCustomAnimation: CustomAnimation {
    func animate<V>(value: V, time: TimeInterval, context: inout AnimationContext<V>) -> V? where V : VectorArithmetic {
        // Custom interpolation logic
        // Return nil when animation is complete
    }
}
```

## Transactions

### Understanding Transactions

Transactions are the underlying mechanism for all animations. Every view update is wrapped in a transaction carrying animation information.

```swift
// Using withAnimation (convenient)
withAnimation(.default) { flag.toggle() }

// Using withTransaction (explicit)
var transaction = Transaction(animation: .default)
withTransaction(transaction) { flag.toggle() }
```

### The .transaction Modifier

```swift
Rectangle()
    .frame(width: flag ? 100 : 50, height: 50)
    .transaction { t in
        t.animation = .default
    }
```

### Animation Precedence

**Implicit animations take precedence over explicit animations:**

```swift
Button("Tap") {
    withAnimation(.linear) { flag.toggle() }
}
.animation(.bouncy, value: flag)  // This wins!
```

### Disabling Animations

```swift
.transaction { t in
    t.disablesAnimations = true
}
```

## Completion Handlers (iOS 17+)

### Basic Usage

```swift
Button("Animate") {
    withAnimation(.default) {
        flag.toggle()
    } completion: {
        print("Animation complete!")
    }
}
```

### Common Pitfall: Completion Not Firing

```swift
// BAD - completion only fires once
.transaction {
    $0.addAnimationCompletion { print("Done!") }
}

// GOOD - use value parameter for reexecution
.transaction(value: flag) {
    $0.addAnimationCompletion { print("Done!") }
}
```

## The Animatable Protocol

### Understanding Animatable

The `Animatable` protocol enables property interpolation:

```swift
protocol Animatable {
    associatedtype AnimatableData : VectorArithmetic
    var animatableData: Self.AnimatableData { get set }
}
```

### Custom Animatable Modifier

```swift
struct MyOpacity: ViewModifier, Animatable {
    var animatableData: Double

    init(_ opacity: Double) {
        animatableData = opacity
    }

    func body(content: Content) -> some View {
        content.opacity(animatableData)
    }
}
```

### Creating Custom Animations: Shake Effect

```swift
struct Shake: ViewModifier, Animatable {
    var numberOfShakes: Double

    var animatableData: Double {
        get { numberOfShakes }
        set { numberOfShakes = newValue }
    }

    func body(content: Content) -> some View {
        content
            .offset(x: -sin(numberOfShakes * 2 * .pi) * 30)
    }
}

// Usage
struct ContentView: View {
    @State private var shakes = 0

    var body: some View {
        Button("Shake!") { shakes += 1 }
            .modifier(Shake(numberOfShakes: Double(shakes)))
            .animation(.default, value: shakes)
    }
}
```

### Multiple Animatable Properties

Use `AnimatablePair` to compose multiple values:

```swift
struct ComplexAnimation: ViewModifier, Animatable {
    var offset: CGFloat
    var rotation: Double

    var animatableData: AnimatablePair<CGFloat, Double> {
        get { AnimatablePair(offset, rotation) }
        set {
            offset = newValue.first
            rotation = newValue.second
        }
    }

    func body(content: Content) -> some View {
        content
            .offset(x: offset)
            .rotationEffect(.degrees(rotation))
    }
}
```

**Pitfall:** Default `animatableData` implementation does nothing. Always explicitly implement it.

## Transitions

### Default Behavior

Without specifying a transition, SwiftUI applies `.opacity`:

```swift
if flag {
    Rectangle()
        .frame(width: 100, height: 100)
}
```

### Transition States

- **Active State**: Appearance when inserting/removing begins
- **Identity State**: Normal, at-rest appearance

### Built-in Transitions

```swift
.transition(.opacity)                      // Fade
.transition(.scale)                        // Scale
.transition(.slide)                        // Slide from edge
.transition(.move(edge: .leading))         // Move from specific edge
.transition(.offset(x: 100, y: 0))         // Offset by amount
```

### Combining Transitions

```swift
// Parallel
.transition(.slide.combined(with: .opacity))

// Asymmetric
.transition(
    .asymmetric(
        insertion: .slide,
        removal: .scale
    )
)
```

### Custom Transitions (Pre-iOS 17)

```swift
struct Blur: ViewModifier {
    var radius: CGFloat

    func body(content: Content) -> some View {
        content.blur(radius: radius)
    }
}

extension AnyTransition {
    static func blur(radius: CGFloat) -> Self {
        .modifier(
            active: Blur(radius: radius),
            identity: Blur(radius: 0)
        )
    }
}

// Usage
.transition(.blur(radius: 5))
```

### Custom Transitions (iOS 17+)

```swift
struct BlurTransition: Transition {
    var radius: CGFloat

    func body(content: Content, phase: TransitionPhase) -> some View {
        content
            .blur(radius: phase.isIdentity ? 0 : radius)
    }
}
```

### Critical: Transitions Require Animations

```swift
// BAD - animation is in the subtree being removed
if flag {
    Rectangle()
        .transition(.blur(radius: 5))
        .animation(.default, value: flag)
}

// GOOD - animation is outside the conditional
VStack {
    if flag {
        Rectangle()
            .transition(.blur(radius: 5))
    }
}
.animation(.default, value: flag)

// ALSO GOOD - explicit animation
Button("Toggle") {
    withAnimation(.default) { flag.toggle() }
}
```

### Identity Changes Trigger Transitions

```swift
Rectangle()
    .id(flag)  // Different identity when flag changes
    .transition(.scale)
```

## Phase-Based Animations (iOS 17+)

Phase animations cycle through discrete phases automatically:

```swift
struct Sample: View {
    @State private var shakes = 0

    var body: some View {
        Button("Shake") {
            shakes += 1
        }
        .phaseAnimator([0, -20, 20], trigger: shakes) { content, offset in
            content.offset(x: offset)
        }
    }
}
```

**Infinite Loop (no trigger):**
```swift
.phaseAnimator([0, -20, 20]) { content, offset in
    content.offset(x: offset)
}
```

**Custom Timing Per Phase:**
```swift
.phaseAnimator([0, -20, 20], trigger: shakes) { content, offset in
    content.offset(x: offset)
} animation: { phase in
    switch phase {
    case -20: return .bouncy
    case 20: return .linear
    default: return .smooth
    }
}
```

**Using Enum Phases:**
```swift
enum AnimationPhase: Equatable {
    case initial
    case expanded
    case rotated
}

.phaseAnimator([.initial, .expanded, .rotated]) { content, phase in
    switch phase {
    case .initial:
        content
    case .expanded:
        content.scaleEffect(1.5)
    case .rotated:
        content.scaleEffect(1.5).rotationEffect(.degrees(45))
    }
}
```

## Keyframe-Based Animations (iOS 17+)

Keyframe animations provide fine-grained control with exact values at specific times:

```swift
struct ShakeSample: View {
    @State private var trigger = 0

    var body: some View {
        Button("Shake") {
            trigger += 1
        }
        .keyframeAnimator(
            initialValue: 0,
            trigger: trigger
        ) { content, offset in
            content.offset(x: offset)
        } keyframes: { value in
            CubicKeyframe(-30, duration: 0.25)
            CubicKeyframe(30, duration: 0.5)
            CubicKeyframe(0, duration: 0.25)
        }
    }
}
```

### Keyframe Types

- **CubicKeyframe**: Smooth interpolation
- **LinearKeyframe**: Straight-line interpolation
- **MoveKeyframe**: Instant jump (no interpolation)

### Multiple Tracks

```swift
struct ShakeData {
    var offset: CGFloat = 0
    var rotation: Angle = .zero
}

.keyframeAnimator(
    initialValue: ShakeData(),
    trigger: trigger
) { content, data in
    content
        .offset(x: data.offset)
        .rotationEffect(data.rotation)
} keyframes: { value in
    KeyframeTrack(\.offset) {
        CubicKeyframe(-30, duration: 0.25)
        CubicKeyframe(30, duration: 0.5)
        CubicKeyframe(0, duration: 0.25)
    }

    KeyframeTrack(\.rotation) {
        LinearKeyframe(.degrees(20), duration: 0.1)
        LinearKeyframe(.degrees(-20), duration: 0.2)
        LinearKeyframe(.degrees(0), duration: 0.1)
    }
}
```

**Tracks run in parallel**, each animating one property.

### KeyframeTimeline

Query animation values directly:

```swift
let timeline = KeyframeTimeline(initialValue: ShakeData()) {
    KeyframeTrack(\.offset) {
        CubicKeyframe(-30, duration: 0.25)
        CubicKeyframe(30, duration: 0.5)
        CubicKeyframe(0, duration: 0.25)
    }
}

let valueAt50Percent = timeline.value(time: 0.5)
```

## Best Practices

### Animation Guidelines

1. **Apply Animations Locally**
   - Place animations close to what's being animated
   - Prevents unintended side effects

2. **Always Use Value Parameter**
   - Use `.animation(_:value:)` not deprecated `.animation(_:)`
   - Prevents unexpected animation triggers

3. **Understand View Identity**
   - Property animations require stable view identity
   - Changing identity triggers transitions

4. **Choose the Right Type**
   - Property animations: interpolating values on existing views
   - Transitions: inserting/removing views
   - Phase animations: multi-step sequences returning to start
   - Keyframe animations: complex, precisely-timed animations

### Performance Considerations

1. **Narrow Animation Scope**
   - Animate only what needs to change
   - Consider breaking views into smaller pieces

2. **Prefer Transforms Over Layout**
   - Animate `offset`, `scale`, `rotation` instead of `frame`
   - Layout animations are more expensive

3. **Spring Animation Awareness**
   - Can be more expensive than simple curves
   - Complete later than "logical" completion

### Debugging Animations

```swift
// Print interpolated values
struct MyModifier: ViewModifier, Animatable {
    var value: Double
    var animatableData: Double {
        get { value }
        set {
            value = newValue
            print("Animation value: \(newValue)")
        }
    }
}

// Slow down for inspection
.animation(.linear(duration: 3.0).speed(0.2), value: flag)
```

## Common Pitfalls

1. **Animations Without State Changes**
   - Animations only work with state changes
   - Can't animate constants directly

2. **Transition Without Animation**
   - Always pair transitions with animations
   - Place animations outside conditional structure

3. **Forgetting animatableData**
   - Default implementation does nothing
   - No compiler error, but animation won't work

4. **Completion Handlers Not Reexecuting**
   - Use `.transaction(value:)` variant
   - Ensure closure depends on state

5. **Animation Precedence Confusion**
   - Implicit animations override explicit ones
   - Use `disablesAnimations` when needed

## Summary Checklist

- [ ] Using `.animation(_:value:)` with value parameter (not deprecated version)
- [ ] Transitions paired with animations outside conditional structure
- [ ] Custom `Animatable` implementations have explicit `animatableData`
- [ ] Completion handlers use `.transaction(value:)` for reexecution
- [ ] Animations placed close to animated content
- [ ] Preferring transforms over layout changes for performance
- [ ] Using phase animations for multi-step sequences (iOS 17+)
- [ ] Using keyframe animations for precise timing control (iOS 17+)
- [ ] View identity stable for property animations
- [ ] Explicit animations for event-driven changes
- [ ] Implicit animations for value-dependent changes
