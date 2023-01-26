---
title: "Features"
keywords: features
sidebar: htmlflow_sidebar
permalink: features
---

<div class="row">
<div class="col-md-7">

{% highlight java %}
HtmlFlow
  .doc(System.out)
    .html() // HtmlPage
      .head()
        .title().text("HtmlFlow").__()
      .__() // head
      .body()
        .div().attrClass("container")
          .h1().text("My first HtmlFlow page").__()
          .img().attrSrc("http://bit.ly/2MoHwrU").__()
          .p().text("Typesafe is awesome! :-)").__()
        .__() // div
      .__() // body
    .__(); // html
{% endhighlight %}

</div>
<div class="col-md-5">

{% highlight html %}
<html>
  <head>
    <title>HtmlFlow</title>
  </head>
  <body>
    <div class="container">
      <h1>My first HtmlFlow page</h1>
      <img src="http://bit.ly/2MoHwrU">
      <p>Typesafe is awesome! :-)</p>
    </div>
  </body>
</html>
{% endhighlight %}

</div>
</div>

Beyond `doc()`, the `HtmlFlow` API also provides `view()` and `viewAsync()`, which build
an `HtmlPage` with a `render(model)` or `renderAsync(model)` methods depending of a model
(asynchronous for the latter).

## Getting started

All builders (such as `body()`, `div()`, `p()`, etc) return the created element,
except `text()` which returns its parent element (e.g. `.h1().text("...")` returns
the `H1` parent object).
The same applies to attribute methods - `attr<attribute name>()` - that also
return their parent (e.g. `.img().attrSrc("...")` returns the `Img` parent object).

There is also a special method `__()` which returns the parent element.
This method is responsible for emitting the end tag of an element.

The HTML resulting from HtmlFlow respects all HTML 5.2 rules (e.g. `h1().div()`
gives a compilation error because it goes against the content
allowed by `h1` according to HTML5.2). So, whenever you type `.` after an element
the intelissense will just suggest the set of allowed elements and attributes.

The HtmlFlow API is according to HTML5.2 and is generated with the support
of an automated framework ([xmlet](https://github.com/xmlet/)) based on an [XSD
definition of the HTML5.2](https://github.com/xmlet/HtmlApiFaster/blob/master/src/main/resources/html_5_2.xsd)
syntax.
Thus, all attributes are strongly typed with enumerated types which restrict
the set of accepted values.

Finally, HtmlFlow also supports [_dynamic views_](#dynamic-views) with *data binders* that enable
the same HTML view to be bound with different object models.

## Output approaches

When you build an `HtmlPage` with `HtmlFlow.doc(Appendable out)` you may use any kind of
output compatible with `Appendable`, such as `Writer`, `PrintStream`, `StringBuilder`, or other
(notice some streams, such as `PrintStream`, are not buffered and may degrade performance).

HTML is emitted as builder methods are invoked (e.g. `.body()`, `.div()`, `.p()`, etc).

However, if you build an `HtmlView` with `HtmlFlow.view(view -> view.html().head()...)`
the HTML is only emitted when you call `render(model)` or `write(model)` on the resulting `HtmlView`.
Then, you can get the resulting HTML in two different ways:

```java
HtmlView view = HtmlFlow.view(view -> view
    .html()
        .head()
            ....
);
String html = view.render();        // 1) get a string with the HTML
view
    .setOut(System.out)
    .write();                       // 2) print to the standard output
```

Regardless the output approach you will get the same formatted HTML document.

`HtmlView` does a preprocessing of the provided function (e.g. `view -> ...`) computing
and storing all static HTML blocks for future render calls, avoiding useless concatenation 
of text and HTML tags and improving performance.


## Dynamic Views

`HtmlView` is a subclass of `HtmlPage`, built from a template function specified by the functional interface:

```java
interface HtmlTemplate { void resolve(HtmlPage page); }
```

Next we present an example of a view with a template (e.g. `taskDetailsTemplate`) that will be later
bound to a domain object `Task`.
Notice the use of the method `dynamic()` inside the `taskDetailsTemplate` whenever we need to 
access the domain object `Task` (i.e. the _model_).
This model will be passed later to the view through its method `render(model)` or `write(model)`.

``` java
HtmlView view = HtmlFlow.view(HtmlLists::taskDetailsTemplate);

public static void taskDetailsTemplate(HtmlPage view) {
    view
        .html()
            .head()
                .title().text("Task Details").__()
            .__() //head
            .body()
                .<Task>dynamic((body, task) -> body.text("Title:").text(task.getTitle()))
                .br().__()
                .<Task>dynamic((body, task) -> body.text("Description:").text(task.getDescription()))
                .br().__()
                .<Task>dynamic((body, task) -> body.text("Priority:").text(task.getPriority()))
            .__() //body
        .__(); // html
}
```

Next we present an example binding this same view with 3 different domain objects,
producing 3 different HTML documents.

``` java
List<Task> tasks = Arrays.asList(
    new Task(3, "ISEL MPD project", "A Java library for serializing objects in HTML.", Priority.High),
    new Task(4, "Special dinner", "Moonlight dinner!", Priority.Normal),
    new Task(5, "US Open Final 2018", "Juan Martin del Potro VS  Novak Djokovic", Priority.High)
);
for (Task task: tasks) {
    Path path = Paths.get("task" + task.getId() + ".html");
    Files.write(path, view.render(task).getBytes());
    Desktop.getDesktop().browse(path.toUri());
}
```

Finally, an example of a dynamic HTML table binding to a stream of tasks.
Notice, we  do not need any special templating feature to traverse the `Stream<Task>` and
we simply take advantage of Java Stream API.

``` java
static HtmlView tasksTableView = HtmlFlow.view(HtmlForReadme::tasksTableTemplate);

static void tasksTableTemplate(HtmlPage page) {
    page
        .html()
            .head()
                .title().text("Tasks Table").__()
            .__()
            .body()
                .table()
                    .attrClass("table")
                    .tr()
                        .th().text("Title").__()
                        .th().text("Description").__()
                        .th().text("Priority").__()
                    .__()
                    .tbody()
                        .<Stream<Task>>dynamic((tbody, tasks) ->
                            tasks.forEach(task -> tbody
                                .tr()
                                    .td().text(task.getTitle()).__()
                                    .td().text(task.getDescription()).__()
                                    .td().text(task.getPriority().toString()).__()
                                .__() // tr
                            ) // forEach
                        ) // dynamic
                    .__() // tbody
                .__() // table
            .__() // body
        .__(); // html
}
```

## Asynchronous HTML Views

`HtmlViewAsync` is another subclass of `HtmPage` also depending of an `HtmlTemplate` function, 
which can be bind with both synchronous, or asynchronous models.

Notice that calling `renderAsync()` returns immediately, without blocking, while the `HtmlTemplate`
function is still processing, maybe awaiting for the asynchronous model completion.
Thus, `renderAsync()` and `writeAsync()` return `CompletableFuture<String>` and 
`CompletableFuture<Void>` allowing to follow up processing and completion.

To ensure well-formed HTML, the HtmlFlow needs to observe the asynchronous models completion. 
Otherwise, the text or HTML elements following an asynchronous model binding maybe emitted before 
the HTML resulting from the asynchronous model.

Thus, to bind an asynchronous model we should use the builder
`.await(parent, model, onCompletion) -> ...)` where the `onCompletion` callback is used to signal 
HtmFlow that can proceed to the next continuation, as presented in next sample:

```java
static HtmlViewAsync tasksTableViewAsync = HtmlFlow.viewAsync(HtmlForReadme::tasksTableTemplateAsync);

static void tasksTableTemplateAsync(HtmlPage page) {
    page
        .html()
            .head()
                .title() .text("Tasks Table") .__()
            .__()
            .body()
                .table().attrClass("table")
                    .tr()
                        .th().text("Title").__()
                        .th().text("Description").__()
                        .th().text("Priority").__()
                    .__()
                    .tbody()
                    .<Flux<Task>>await((tbody, tasks, onCompletion) -> tasks
                        .doOnNext(task -> tbody
                            .tr()
                                .td().text(task.getTitle()).__()
                                .td().text(task.getDescription()).__()
                                .td().text(task.getPriority().toString()).__()
                            .__() // tr
                        )
                        .doOnComplete(onCompletion::finish)
                        .subscribe()
                    )
                    .__() // tbody
                .__() // table
            .__() // body
        .__(); // html
}
```

In previous example, the model is a
[Flux](https://projectreactor.io/docs/core/release/api/reactor/core/publisher/Flux.html),
which is a Reactive Streams
[`Publisher`](https://www.reactive-streams.org/reactive-streams-1.0.3-javadoc/org/reactivestreams/Publisher.html?is-external=true) with rx operators that emits 0 to N elements.

HtmlFlow _await_ feature works regardless the type of asynchronous model and can be used with
any kind of asynchronous API.


## Partial and Layout

HtmlFlow also enables the use of partial HTML blocks inside a template function.
This is useful whenever you want to reuse the same template with different HTML fragments.

The partial is constructed just like any template. Consider the following, which create a template, for a div containing a label and an input.

```java
public class InputField {
  public static class LabelValueModel {
        final String label;
        final String id;
        final Object value;

        private LabelValueModel(String label, String id, Object value) {
            this.label = label;
            this.id = id;
            this.value = value;
        }

        public static LabelValueModel of(String label, String id,  Object value) {
            return new LabelValueModel(label, id, value);
        }
    }

  public static HtmlView<LabelValueModel> view = DynamicHtml.view(InputField::template);

  static void template(DynamicHtml<LabelValueModel> view, LabelValueModel model) {
      view
          .div()
              .label()
              .dynamic(label -> label.text(model.label)).__() //label

              .input()
                  .dynamic(input -> input
                      .attrType(EnumTypeInputType.TEXT)
                      .attrId(model.id)
                      .attrName(model.id)
                      .attrValue(model.value.toString())
                  )
              .__()
          .__();
  }
}
```
Notice that we also introduce a model that is dedicated to this partial.
This will help us get data in the exact form needed to produce a complete, working and type-safe template.

This partial could be used inside another template.

```java
static void template(DynamicHtml<Pet> view, Pet pet) {
    view
      .div()
        .form().attrMethod(EnumMethodType.POST)
          .div().attrClass("form-group has-feedback")
            .dynamic(div -> view.addPartial(InputField.view, InputField.LabelValueModel.of("Date", "date", LocalDate.now())))
          .__() // div
        .__() // form
      .__() // div
}
```

This way of invoking partial is particularly useful when you need to use a smaller part (component) gathered together to produce a bigger template.
This is the most common usage of partials.

There is another way of using partials, it's to construct a layout. The layout is a normal template, but with a hole to be filed with partials.
Like we saw earlier, a partial has nothing special by itself. What is interesting is the layout, consider the following template.


```java
public class Layout {

    public static DynamicHtml<Object> view = (DynamicHtml<Object>) DynamicHtml.view(Layout::template).threadSafe();

    private static <T> void template(DynamicHtml<T> view, T model, HtmlView[] partials) {
        view
            .html()
                .head()
                    .meta().addAttr("http-equiv","Content-Type").attrContent("text/html; charset=UTF-8")
                    .__() //meta
                    .meta().attrCharset("utf-8")
                    .__() //meta
                    .meta().addAttr("http-equiv","X-UA-Compatible").attrContent("IE=edge")
                    .__() //meta
                    .meta().attrName("viewport").attrContent("width=device-width, initial-scale=1")
                    .__() //meta
                    .link().addAttr("rel","shortcut icon").addAttr("type","image/x-icon").attrHref("/resources/images/favicon.png")
                    .__() //link
                    .title()
                        .text("My awesome templating system")
                    .__() //title
                  .__() //head
                .body()
                    .nav().attrClass("navbar navbar-default").addAttr("role","navigation")
                      ..dynamic(__ -> view.addPartial(partials[0]) )
                    .__() //nav
                    .div().attrClass("container-fluid")
                        .div().attrClass("container xd-container")
                          .dynamic(__ ->  view.addPartial(partials[1], model) )
                        .__() //div
                    .__() //div
                .__() //body
            .__(); //html
    }
}
```

Notice the third argument to the function `template`, this array of partials is the place where we receive the partials to fill the holes of our layout.
To use them we called two distinct signatures of `view.addPartial`, one with only the partial, and one with a partial and a model. Depending on the type of
templated hidden behind `partials[0]`` we would use one signature or the other.

So we have defined our layout, let's use it to create a template with much less clutter.

```java
public class APage {

    public static DynamicHtml<Object> view = (DynamicHtml<Object>) DynamicHtml.view(Layout::template, Object model, new DynamicHtml<Object>[]{Menu::template, APage::template]).threadSafe();

    private static <T> void template(DynamicHtml<T> view, T model, HtmlView[] partials) {
        view
            .div().text("Type safety feel so cozy !!")
    }
}
```

Notice the member `view` which call the layout's template, and passing the menu and the page component as an array.


<p>&nbsp;</p>
