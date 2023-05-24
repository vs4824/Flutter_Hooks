# Flutter Hooks

Hooks are a new kind of object that manage the life-cycle of a Widget. They exist for one reason: increase the code-sharing between widgets by removing duplicates.

## Motivation

StatefulWidget suffers from a big problem: it is very difficult to reuse the logic of say initState or dispose. An obvious example is AnimationController:

   ```
   class Example extends StatefulWidget {
  final Duration duration;

  const Example({Key? key, required this.duration})
      : super(key: key);

  @override
  _ExampleState createState() => _ExampleState();
}

class _ExampleState extends State<Example> with SingleTickerProviderStateMixin {
  AnimationController? _controller;

  @override
  void initState() {
    super.initState();
    _controller = AnimationController(vsync: this, duration: widget.duration);
  }

  @override
  void didUpdateWidget(Example oldWidget) {
    super.didUpdateWidget(oldWidget);
    if (widget.duration != oldWidget.duration) {
      _controller!.duration = widget.duration;
    }
  }

  @override
  void dispose() {
    _controller!.dispose();
    super.dispose();
  }

  @override
  Widget build(BuildContext context) {
    return Container();
  }
}
   ```

All widgets that desire to use an AnimationController will have to reimplement almost all of this logic from scratch, which is of course undesired.

Dart mixins can partially solve this issue, but they suffer from other problems:

1. A given mixin can only be used once per class.

2. Mixins and the class share the same object.

3. This means that if two mixins define a variable under the same name, the result may vary between compilation fails to unknown behavior.

This library proposes a third solution:

   ```
   class Example extends HookWidget {
  const Example({Key? key, required this.duration})
      : super(key: key);

  final Duration duration;

  @override
  Widget build(BuildContext context) {
    final controller = useAnimationController(duration: duration);
    return Container();
  }
}
   ```

This code is functionally equivalent to the previous example. It still disposes the AnimationController and still updates its duration when Example.duration changes. But you're probably thinking:

Where did all the logic go?
That logic has been moved into useAnimationController, a function included directly in this library (see Existing hooks) - It is what we call a Hook.

Hooks are a new kind of object with some specificities:

1. They can only be used in the build method of a widget that mix-in Hooks.

2. The same hook can be reused arbitrarily many times. The following code defines two independent AnimationController, and they are correctly preserved when the widget rebuild.

   ```
   Widget build(BuildContext context) {
   final controller = useAnimationController();
   final controller2 = useAnimationController();
   return Container();
   }
   ```
   
3. Hooks are entirely independent of each other and from the widget. This means that they can easily be extracted into a package and published on pub for others to use.

## Principle

Similar to State, hooks are stored in the Element of a Widget. However, instead of having one State, the Element stores a List<Hook>. Then in order to use a Hook, one must call Hook.use.

The hook returned by use is based on the number of times it has been called. The first call returns the first hook; the second call returns the second hook, the third call returns the third hook and so on.

If this idea is still unclear, a naive implementation of hooks could look as follows:

   ```
   class HookElement extends Element {
  List<HookState> _hooks;
  int _hookIndex;

  T use<T>(Hook<T> hook) => _hooks[_hookIndex++].build(this);

  @override
  performRebuild() {
    _hookIndex = 0;
    super.performRebuild();
  }
}
   ```
