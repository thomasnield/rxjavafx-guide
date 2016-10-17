# 9. Switching, Throttling, and Buffering

In the previous chapter, we learned that RxJava makes concurrency accessible and fairly trivial to accomplish. But being able to compose concurrency easily enables us to do much more with RxJava. Switching with `switchMap()` allows us to quickly unsubscribe from a previous `Observable` and move on to the next, and this enables some powerful patterns in JavaFX UI's. Throttling and buffering provides us with tools to cope with rapidly-firing Observables, such as those emitting keystrokes. Each of these in isolation are useful, but you can achieve amazing functionality by combining them together.

## Switching with `switchMap()`

Let's emulate a situation where rapid user inputs could overwhelm your application with requests. Say you have two `ListView<T>` controls. The top one has `String` values, and the bottom one will always display the individual characters for each selected `String`. When you select "Alpha" on the top one, the bottom one will contain items "A","l","p","h", and "a" (Figure 9.1).

**Java**

```java

public final class JavaFxApp extends Application {

    @Override
    public void start(Stage stage) throws Exception {

        VBox root = new VBox();

        ListView<String> listView = new ListView<>();

        listView.getItems().setAll("Alpha","Beta","Gamma",
                "Delta","Epsilon","Zeta","Eta");

        ListView<String> itemsView = new ListView<>();

        JavaFxObservable.fromObservableList(listView.getSelectionModel().getSelectedItems())
                .flatMap ( list -> Observable.from(list)
                    .flatMap (s -> Observable.from(s.split("(?!^)")))
                    .toList()
                ).subscribe(l -> itemsView.getItems().setAll(l));

        root.getChildren().addAll(listView, itemsView);

        stage.setScene(new Scene(root));

        stage.show();
    }
}
```
**Kotlin**

```kotlin
class MyView : View() {

    val items = listOf("Alpha","Beta","Gamma",
            "Delta","Epsilon","Zeta","Eta").observable()

    override val root = vbox {

        val listView = listview(items)

        listview<String> {
            listView.selectionModel.selectedItems.onChangedObservable().filterNotNull()
                .flatMap { it.toObservable()
                        .delay(3, TimeUnit.SECONDS, Schedulers.io())
                        .flatMap { it.toCharArray().map(Char::toString).toObservable() }
                        .toList()
                }.observeOnFx().subscribe { items.setAll(it) }
        }
    }
}
```
**Figure 9.1**

![](http://i.imgur.com/dk3VmWp.png)

This is a pretty quick computation and hardly keeps the JavaFX thread busy. But in the real world, running database queries or HTTP requests can take awhile. The last thing you want is for these rapid inputs to create a queue of requests which will quickly make the application unusable as it works through the queue. Let's emulate this by using the `delay()` operator. Remember that the `delay()` operator already specifies a `subscribeOn()` internally, but we can specify an argument which `Scheduler` it uses. Let's put it in the IO Scheduler. Remember also the `Subscriber` must receive each emission on the JavaFX thread, so be sure to `observeOn()` it before the emission goes to the `Subscriber`.

**Java**

```java
public final class JavaFxApp extends Application {

    @Override
    public void start(Stage stage) throws Exception {

        VBox root = new VBox();

        ListView<String> listView = new ListView<>();

        listView.getItems().setAll("Alpha","Beta","Gamma",
                "Delta","Epsilon","Zeta","Eta");

        ListView<String> itemsView = new ListView<>();

        JavaFxObservable.fromObservableList(listView.getSelectionModel().getSelectedItems())
                .flatMap ( list -> Observable.from(list)
                    .delay(3, TimeUnit.SECONDS, Schedulers.io())
                    .flatMap (s -> Observable.from(s.split("(?!^)")))
                    .toList()
                ).observeOn(JavaFxScheduler.getInstance())
                .subscribe(l -> itemsView.getItems().setAll(l));

        root.getChildren().addAll(listView, itemsView);

        stage.setScene(new Scene(root));

        stage.show();
    }
}
```

**Kotlin**

```kotlin
class MyView : View() {

    val items = listOf("Alpha","Beta","Gamma",
            "Delta","Epsilon","Zeta","Eta").observable()

    override val root = vbox {

        val listView = listview(items)

        listview<String> {
            listView.selectionModel.selectedItems.onChangedObservable().filterNotNull()
                .flatMap { it.toObservable()
                        .delay(3, TimeUnit.SECONDS, Schedulers.io())
                        .flatMap { it.toCharArray().map(Char::toString).toObservable() }
                        .toList()
                }.observeOnFx().subscribe { items.setAll(it) }
        }
    }
}
```

Now if we click several items on the top `ListView`, you will notice a 3-second lag before the letters show up on the bottom `ListView`. This emulates long-running requests for each click, and now we have these requests queuing up and causing the bottom `ListView` to go berserk as it trys to catch up and display each request. Obviously, this is undesirable. We likely want to kill previous requests when a new one comes in. This is simple to do. Just change the `flatMap()` that emits the `List<String>` of characters to a `switchMap()`.

**Java**

```java
public final class JavaFxApp extends Application {

    @Override
    public void start(Stage stage) throws Exception {

        VBox root = new VBox();

        ListView<String> listView = new ListView<>();

        listView.getItems().setAll("Alpha","Beta","Gamma",
                "Delta","Epsilon","Zeta","Eta");

        ListView<String> itemsView = new ListView<>();

        JavaFxObservable.fromObservableList(listView.getSelectionModel().getSelectedItems())
                .switchMap ( list -> Observable.from(list)
                    .delay(3, TimeUnit.SECONDS, Schedulers.io())
                    .flatMap (s -> Observable.from(s.split("(?!^)")))
                    .toList()
                ).observeOn(JavaFxScheduler.getInstance())
                .subscribe(l -> itemsView.getItems().setAll(l));

        root.getChildren().addAll(listView, itemsView);

        stage.setScene(new Scene(root));

        stage.show();
    }
}
```

**Kotlin**

```kotlin
class MyView : View() {

    val items = listOf("Alpha","Beta","Gamma",
            "Delta","Epsilon","Zeta","Eta").observable()

    override val root = vbox {

        val listView = listview(items)

        listview<String> {
            listView.selectionModel.selectedItems.onChangedObservable().filterNotNull()
                .switchMap { it.toObservable()
                        .delay(3, TimeUnit.SECONDS, Schedulers.io())
                        .flatMap { it.toCharArray().map(Char::toString).toObservable() }
                        .toList()
                }.observeOnFx().subscribe { items.setAll(it) }
        }
    }
}
```

This makes the application much more responsive. The `switchMap()` will only chase after only the latest user input and kill any previous requests. In other words, it is only chasing after the latest `Observable` for the latest emission, and unsubscribing from any previous requests. The `switchMap()` is a powerful utility to create responsive and resilient UI's, and is the perfect way to handle users who rapidly click on things that kick off long processes!

The `switchMap()` can surprisingly be used for other UI situations, such as switching from one infinite hot `Observable` to another. 
