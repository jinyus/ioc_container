## A Dart Ioc Container
A simple IoC Container for Dart and Flutter. Use it for dependency injection or as a service locator. It has scoped, singleton, transient and async support.  


The library is eighty one lines of [source code](https://github.com/MelbourneDeveloper/ioc_container/blob/main/lib/ioc_container.dart) according to LCOV. That means you copy/paste it anywhere and it's simple enough for you to understand. 

If you've used Provider, you'll probably need an Ioc Container to compliment it. Provider and `InheritedWidgets` are good at passing dependencies through the widget tree, but Ioc Container is good at minting them in the first place. Return `get<>()` from your container to Provider's `create` builder method. Whenever Provider needs a dependency the Ioc Container will either create a new instance or grab one of the singletons/scoped objects.

You can do this. It's nice.

```dart
final a = A('a');
final builder = IocContainerBuilder();
builder
  //Singletons last for the lifespan of the app
  ..addSingletonService(a)
  ..add((i) => B(i.get<A>()))
  ..add((i) => C(i.get<B>()))
  ..add((i) => D(i.get<B>(), i.get<C>()));
final container = builder.toContainer();
var d = container.get<D>();
expect(d.c.b.a, a);
expect(d.c.b.a.name, 'a');
```

## Scoping
You can create a scoped container that will never create more than one instance of an object by type within the scope. You can check this example out in the tests. In this example, we create an instance of `D` but the object graph only has four object references. All instances of `A`, `B`, `C`, and `D` are the same instance. This is because the scoped container is only creating one instance of each type. When you are finished with the scoped instances, you can call `dispose()` to dispose everything.

```dart
final a = A('a');
final builder = IocContainerBuilder()
  ..addSingletonService(a)
  ..add((i) => B(i.get<A>()))
  ..add<C>(
    (i) => C(i.get<B>()),
    dispose: (c) => c.dispose(),
  )
  ..add<D>(
    (i) => D(i.get<B>(), i.get<C>()),
    dispose: (d) => d.dispose(),
  );
final container = builder.toContainer();
final scoped = container.scoped();
final d = scoped.get<D>();
scoped.dispose();
expect(d.disposed, true);
expect(d.c.disposed, true);
```    

## Async Initialization
You can do initialization work when instantiating an instance of your service. Just return a `Future<T>` instead of `T` (or use the `async` keyword). When you want an instance, call the `init()` method instead of `get()`

_Warning: you must put error handling inside singleton or scoped `async` factories. If a singleton/scoped async factory throws an error, that factory will continue to return a `Future` with an error for the rest of the lifespan of the container._

```dart
  test('Test Async', () async {
    final builder = IocContainerBuilder()
      ..add(
        (c) => Future<A>.delayed(
          //Simulate doing some async work
          const Duration(milliseconds: 10),
          () => A('a'),
        ),
      )
      ..add(
        (c) => Future<B>.delayed(
          //Simulate doing some async work
          const Duration(milliseconds: 10),
          () async => B(await c.init<A>()),
        ),
      );

    final container = builder.toContainer();
    final b = await container.init<B>();
    expect(b, isA<B>());
    expect(b.a, isA<A>());
  });
```

## Performance Comparison Benchmarks
This library is fast and holds up to comparable libraries in terms of performance. Check out the [benchmarks folder](https://github.com/MelbourneDeveloper/ioc_container/tree/benchmarks/benchmarks) of the GitHub repository to check out the benchmarks. 

_*Disclaimer: there is no claim that the methodology in these benchmarks is correct. It's possible that my benchmarks don't compare the same thing across libraries. I invite you and the library authors to check these and let me know if there are mistakes.*_

macOS - Mac Mini - 3.2 Ghz 6 Core Intel Core i7

Times in microseconds (μs)

|                  	| ioc_container         	| get_it                	| flutter_simple_DI     	| Riverpod             	|   	|
|------------------	|-----------------------	|-----------------------	|-----------------------	|----------------------	|---	|
| Get              	| 1.152956           	    | 1.6829909085045458 	    | 23.56929286888922  	    |                      	|   	|
| Get Async        	| 14.607701157643634 	    | 8.161859669070166  	    |                       	|                      	|   	|
| Get Scoped       	| 2.718096281903718  	    |                       	|                       	| 7.804826666666667 	  |   	|
| Register and Get 	| 3.6589533333333333 	    | 13.37688998488012  	    | 26.387617939769935 	    |                      	|   	|

- get_it: 7.2.0
- ioc_container: 1.0.0
- Riverpod: 2.0.2
- flutter_simple_dependency_injection: 2.0.0

## As a Service Locator
You can use an `IocContainer` as a service locator in Flutter and Dart. Just put an instance in a global space and use it to get your dependencies anywhere with scoping. 

_Note: there are many ways to avoid declaring the container globally. You should weigh up your options and make sure that declaring the container globally is the right choice for your app_. 

```dart
late final IocContainer container;

void main(List<String> arguments) {
  final builder = IocContainerBuilder()
    ..addSingletonService(A('A nice instance of A'))
    ..add((i) => B(i.get<A>()))
    ..add((i) => C(i.get<B>()))
    ..add((i) => D(i.get<B>(), i.get<C>()));
  container = builder.toContainer();

  final d = container.scoped().get<D>();
  // ignore: avoid_print
  print('Hello world: ${d.c.b.a.name}');
}
```

Install it like this:
> dart pub add ioc_container