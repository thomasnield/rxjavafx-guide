# 9. Switching, Throttling, and Buffering

In the previous chapter, we learned that RxJava makes concurrency accessible and fairly trivial to accomplish. But being able to compose concurrency easily enables us to do much more with RxJava.

In UI development, users will inevitably click things that kick off long-running processes. Even if you have concurrency in place, users that rapidly select UI inputs can kick of expensive processes, and those processes will start to queue up undesirably. Other times, we may want to group up rapid emissions because they logically are a single unit, such as typing keystrokes. There are tools to effectively overcome all these problems, and we will cover them in this chapter.


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

This is a pretty quick computation which hardly keeps the JavaFX thread busy. But in the real world, running database queries or HTTP requests can take awhile. The last thing we want is for these rapid inputs to create a queue of requests that will quickly make the application unusable as it works through the queue. Let's emulate this by using the `delay()` operator. Remember that the `delay()` operator already specifies a `subscribeOn()` internally, but we can specify an argument which `Scheduler` it uses. Let's put it in the IO Scheduler. The `Subscriber` must receive each emission on the JavaFX thread, so be sure to `observeOn()` the JavaFX Scheduler before the emission goes to the `Subscriber`.

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

Now if we click several items on the top `ListView`, you will notice a 3-second lag before the letters show up on the bottom `ListView`. This emulates long-running requests for each click, and now we have these requests queuing up and causing the bottom `ListView` to go berserk, trying to display each previous request before it gets to the current one. Obviously, this is undesirable. We likely want to kill previous requests when a new one comes in. This is simple to do. Just change the `flatMap()` that emits the `List<String>` of characters to a `switchMap()`.

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

This makes the application much more responsive. The `switchMap()` works identically to the `flatMap()`, but it will only chase after the latest user input and kill any previous requests. In other words, it is only chasing after the latest `Observable` for the latest emission, and unsubscribing any previous requests. The `switchMap()` is a powerful utility to create responsive and resilient UI's, and is the perfect way to handle users who compulsively click on anything that kicks off long processes!

Youc an also use the `switchMap()` to cancel long-running or infinite processes using a neat little trick with `Observable.empty()`. For instance, a `ToggleButton` has a true/false state depending on whether it is selected. When you emit its `false` state, you can return an empty `Observable` to kill the previous processing `Observable`, as shown below. When the `ToggleButton` is selected, it will kick off an `Observable.interval()` that emits a consecutive integer every 10 milliseconds. But unselecting the `ToggleButton` will cause the `flatMap()` to switch to an `Observable.empty()`, killing and unsubscribing from the `Observable.interval()` (Figure 9.2).

**Java**

```java
public class JavaApp extends Application {
    @Override
    public void start(Stage stage) throws Exception {
        VBox vBox = new VBox();

        ToggleButton toggleButton = new ToggleButton("START");
        Label timerLabel = new Label("0");

        JavaFxObservable.fromObservableValue(toggleButton.selectedProperty())
                .switchMap(selected -> {
                    if (selected) {
                        toggleButton.setText("STOP");
                        return Observable.interval(10, TimeUnit.MILLISECONDS);
                    } else {
                        toggleButton.setText("START");
                        return Observable.empty();
                    }
                })
                .observeOn(JavaFxScheduler.getInstance())
                .subscribe(i -> timerLabel.setText(i.toString()));

        vBox.getChildren().setAll(toggleButton,timerLabel);
        stage.setScene(new Scene(vBox));
        stage.show();
    }
}
```

**Kotlin**

```kotlin
class MyView : View() {

    override val root = vbox {

        val toggleButton = togglebutton("START")
        val timerLabel = label("0")

        toggleButton.selectedProperty().toObservable()
            .switchMap { selected ->
                if (selected) {
                    toggleButton.text = "STOP"
                    Observable.interval(10, TimeUnit.MILLISECONDS)
                } else {
                    toggleButton.text = "START"
                    Observable.empty()
                }
            }.observeOnFx()
            .subscribe {
                timerLabel.text = it.toString()
            }
    }
}
```

**Figure 9.2**

![](http://i.imgur.com/RfQY61B.png)


The `switchMap()` can come in handy for any situation where you want to switch from one `Observable` source to another.


## Buffering
