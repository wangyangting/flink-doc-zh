---
title: "FlinkCEP - Flink 中的复杂事件处理"
nav-title: 事件处理（CEP）
nav-parent_id: libs
nav-pos: 1
---
<!--
Licensed to the Apache Software Foundation (ASF) under one
or more contributor license agreements.  See the NOTICE file
distributed with this work for additional information
regarding copyright ownership.  The ASF licenses this file
to you under the Apache License, Version 2.0 (the
"License"); you may not use this file except in compliance
with the License.  You may obtain a copy of the License at

  http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing,
software distributed under the License is distributed on an
"AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY
KIND, either express or implied.  See the License for the
specific language governing permissions and limitations
under the License.
-->

FlinkCEP 是在 Flink 之上所实现的 CEP（复杂事件处理）库。
它可以让你在不断的事件流中检测事件模式，模式匹配之后，就可以处理数据中的重要信息。

本页面描述了 Flin CEP 中可用的 API 调用。
我们首先介绍 [Pattern API](#the-pattern-api)，它可以让你在流中使用自己的方式来检测想要的模式，在介绍它之前，我们可以先了解一下 [对匹配的事件序列进行检测和操作](#detecting-patterns).
然后，我们介绍 CEP 库在事件时间内的 [延迟处理](#handling-lateness-in-event-time) 所做的假设，以及如何从老的 Flink 版本 [迁移任务](#migrating-from-an-older-flink-version) 到 Flink-1.3 版本。

* This will be replaced by the TOC
{:toc}

## 入门指南

如果你想快速入门，[设置 Flink 程序]({{ site.baseurl }}/dev/linking_with_flink.html)，以及添加 FlinkCEP 依赖到你项目的 `pom.xml` 文件中去。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight xml %}
<dependency>
  <groupId>org.apache.flink</groupId>
  <artifactId>flink-cep-scala{{ site.scala_version_suffix }}</artifactId>
  <version>{{site.version }}</version>
</dependency>
{% endhighlight %}
</div>
</div>

{% info %} FlinkCEP 不是二进制发行版中的一部分. 请参阅 [这里]({{site.baseurl}}/dev/linking.html) 了解如何将它与 cluster execution 相关联起来.

现在你可以开始使用 Pattern API 来编写第一个 CEP 程序啦。

你想要去应用模式匹配的 `DataStream` 中的事件必须实现正确的 `equals()` 和 `hashCode()` 方法，因为 FlinkCEP 使用它们来比较和匹配事件。 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Event> input = ...

Pattern<Event, ?> pattern = Pattern.<Event>begin("start").where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getId() == 42;
            }
        }
    ).next("middle").subtype(SubEvent.class).where(
        new SimpleCondition<Event>() {
            @Override
            public boolean filter(SubEvent subEvent) {
                return subEvent.getVolume() >= 10.0;
            }
        }
    ).followedBy("end").where(
         new SimpleCondition<Event>() {
            @Override
            public boolean filter(Event event) {
                return event.getName().equals("end");
            }
         }
    );

PatternStream<Event> patternStream = CEP.pattern(input, pattern);

DataStream<Alert> result = patternStream.select(
    new PatternSelectFunction<Event, Alert> {
        @Override
        public Alert select(Map<String, List<Event>> pattern) throws Exception {
            return createAlertFrom(pattern);
        }
    }
});
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val input: DataStream[Event] = ...

val pattern = Pattern.begin("start").where(_.getId == 42)
  .next("middle").subtype(classOf[SubEvent]).where(_.getVolume >= 10.0)
  .followedBy("end").where(_.getName == "end")

val patternStream = CEP.pattern(input, pattern)

val result: DataStream[Alert] = patternStream.select(createAlert(_))
{% endhighlight %}
</div>
</div>

## Pattern API

该 pattern API 允许您定义要从 input stream（输入流）中提取的复杂模式序列。

每一个复杂模式序列由多个简单的模式组成，例如. 模式寻找具有相同属性的个别事件.
从现在开始，我们将称这些简单的模式为 **patterns**，以及我们在流中搜索的最终复杂模式序列为 **pattern sequence**。
您可以将 pattern sequence（模式序列）看作是这些模式的 graphx（图），它们从用户指定的 *conditions* 从一个模式转换到下一个模式，例如. `event.getName().equals("start")`.
一个 **match** 是一系列的输入事件，它通过一系列有效的模式转换来访问了复杂模式图的所有模式。

{% warn Attention %} 每个模式必须有一个唯一的名字，后面你会使用它来识别所匹配的事件。

{% warn Attention %} 模式名 **CANNOT** 包含字符 `":"` 。

在本章的其余部分中，我们首先来介绍如何定义 [单独模式](#individual-patterns)，然后就可以将这些单独的模式联合起来作为 [复杂模式](#combining-patterns) 。

### Individual Patterns（单个模式）

一个 **Pattern** 可以是一个 *singleton* 或一个 *looping* 模式。
Singleton patterns 接收一个单独的事件，然而 looping patterns 可以接收更多的事件。
在模式匹配的符号中，模式 `"a b+ c? d"` (或 `"a"`, 随后是 *一个或更多的* `"b"`'s, 后面跟一个可选的 `"c"`, 随后再是 `"d"`), `a`, `c?`, 和 `d` 是 singleton patterns，然而 `b+` 是一个 looping one。
默认情况下，一个模式是一个 singleton pattern，并且可以使用 [Quantifiers](#quantifiers) 将它转换成为一个 looping one。
每个模式都有一个或多个基于它事件上的 [Conditions](#conditions)。

#### Quantifiers（数量）

在 FlinkCEP 中，您可以使用方法: `pattern.oneOrMore()` 来指定 looping patterns，对于那些期望一个或多个事件发生的模式（例如，前面提及的 `b+`）;
以及 `pattern.times(#ofTimes)`，该模式期望一个指定事件类型发生的次数，例如. 4 `a`'s;
以及 `pattern.times(#fromTimes, #toTimes)`，该模式期待一个指定事件类型中的最小数量发生和最大数量发生的次数，例如. 2-4 `a`s.

您可以使用 `pattern.greedy()` 方法来让 looping pattern 变得 greedy（贪婪），但是您不能使得 group patterns 变得贪婪。
您可以让所有的 patterns，looping or not，可选的使用 `pattern.optional()` 方法。

对于一个名为 `start` 的模式，下面是有效的 quantifiers（数量）:

 <div class="codetabs" markdown="1">
 <div data-lang="java" markdown="1">
 {% highlight java %}
 // expecting 4 occurrences
 start.times(4);

 // expecting 0 or 4 occurrences
 start.times(4).optional();

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4);

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy();

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional();

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy();

 // expecting 1 or more occurrences
 start.oneOrMore();

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy();

 // expecting 0 or more occurrences
 start.oneOrMore().optional();

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy();

 // expecting 2 or more occurrences
 start.timesOrMore(2);

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy();

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy();
 {% endhighlight %}
 </div>

 <div data-lang="scala" markdown="1">
 {% highlight scala %}
 // expecting 4 occurrences
 start.times(4)

 // expecting 0 or 4 occurrences
 start.times(4).optional()

 // expecting 2, 3 or 4 occurrences
 start.times(2, 4)

 // expecting 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).greedy()

 // expecting 0, 2, 3 or 4 occurrences
 start.times(2, 4).optional()

 // expecting 0, 2, 3 or 4 occurrences and repeating as many as possible
 start.times(2, 4).optional().greedy()

 // expecting 1 or more occurrences
 start.oneOrMore()

 // expecting 1 or more occurrences and repeating as many as possible
 start.oneOrMore().greedy()

 // expecting 0 or more occurrences
 start.oneOrMore().optional()

 // expecting 0 or more occurrences and repeating as many as possible
 start.oneOrMore().optional().greedy()

 // expecting 2 or more occurrences
 start.timesOrMore(2)

 // expecting 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).greedy()

 // expecting 0, 2 or more occurrences
 start.timesOrMore(2).optional()

 // expecting 0, 2 or more occurrences and repeating as many as possible
 start.timesOrMore(2).optional().greedy()
 {% endhighlight %}
 </div>
 </div>

#### Conditions（条件）

在每一个模式中，从一个模式到达下一个模式，您可以指定一个额外的 **conditions（条件）**.

 1. [传入事件的属性](#conditions-on-properties)。例如，它的值应该大于 5，或者比以前所接收时间的平均值要大。

 2. [匹配时间的连续性](#conditions-on-contiguity)。例如，检测模式 `a,b,c` 在任何匹配的事件之间没有不匹配的事件。

后面一种方式涉及了 "looping" patterns（循环模式），也就是说，模式可以接收更多的事件。例如，`a b+ c` 中的 `b+`，它搜索一个或多个 `b`。

##### Conditions on Properties（属性上的条件）

您可以通过 `pattern.where()`, `pattern.or()` 或 `pattern.until()` 方法在事件属性上指定条件。
它们可以是 `IterativeCondition` 或 `SimpleCondition`。

**Iterative Conditions（迭代条件）:** 这是最常见是一种条件。这就是您可以如何指定接收后续事件的条件，该条件基于先前接收的事件的属性或其子集上的统计量。

下面是一个迭代条件的代码，如果 name 以 "foo" 开头，则接收名为 "middle" 模式的下一个事件，并且如果此模式的先前接收事件的价格总和加上当前价格的事件不会超出 5.0.
迭代条件是非常强大的，特别是与循环模式联合起来的时候。例如，`oneOrMore()` 。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
middle.oneOrMore().where(new IterativeCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value, Context<SubEvent> ctx) throws Exception {
        if (!value.getName().startsWith("foo")) {
            return false;
        }

        double sum = value.getPrice();
        for (Event event : ctx.getEventsForPattern("middle")) {
            sum += event.getPrice();
        }
        return Double.compare(sum, 5.0) < 0;
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
middle.oneOrMore().where(
    (value, ctx) => {
        lazy val sum = ctx.getEventsForPattern("middle").asScala.map(_.getPrice).sum
        value.getName.startsWith("foo") && sum + value.getPrice < 5.0
    }
)
{% endhighlight %}
</div>
</div>

{% warn Attention %} 调用 `context.getEventsForPattern(...)` 为一个给定的潜在匹配来查找所有先前接收的事件。
此操作的成本可能会有所不同，所以在实现您的自定义条件时，该尽量减少其使用。

**Simple Conditions（简单条件）:** 这种类型的条件扩展了上述提及的 `IterativeCondition` 类，并且会根据事件本身的属性上的 *only* 来决定是否接收事件。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
start.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return value.getName().startsWith("foo");
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
start.where(event => event.getName.startsWith("foo"))
{% endhighlight %}
</div>
</div>

最后，您还可以通过 `pattern.subtype(subClass)` 方法将接收事件的类型限制为初始事件类型（这里是 `Event`）的子类型。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
start.subtype(SubEvent.class).where(new SimpleCondition<SubEvent>() {
    @Override
    public boolean filter(SubEvent value) {
        return ... // some condition
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
start.subtype(classOf[SubEvent]).where(subEvent => ... /* some condition */)
{% endhighlight %}
</div>
</div>

**Combining Conditions（联合条件）:** 如上所述，您可以联合 `subtype` 条件和其它的条件。
这适用于每种情况。
您可以通过顺序调用 `where()` 方法来任意组合条件。
最终结果将是个别条件结果的逻辑 **AND**。
要使用 **OR** 来联合条件，您可以使用 `or()` 方法，如下所述。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
pattern.where(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // some condition
    }
}).or(new SimpleCondition<Event>() {
    @Override
    public boolean filter(Event value) {
        return ... // or condition
    }
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
pattern.where(event => ... /* some condition */).or(event => ... /* or condition */)
{% endhighlight %}
</div>
</div>


**Stop condition（停止条件）:** 在循环模式（`oneOrMore()` 和 `oneOrMore().optional()`）的场景中，您也可以指定一个停止条件。
例如，接受值大于 5 的事件，直到它们的合小雨 50。

为了更好的理解它，让我们来看看下面的示例。

* 像 `"(a+ until b)"` 这样的模式 (一个或多个 `"a"` 直到 `"b"`)

* 传入事件的序列 `"a1" "c" "a2" "b" "a3"`

* 操作的输出结果: `{a1 a2} {a1} {a2} {a3}`.

正如您所见，由于停止条件，`{a1 a2 a3}` 或 `{a2 a3}` 不会被返回。

##### Conditions on Contiguity（邻近条件）

FlinkCEP 支持以下事件之间的连续形式 : 

 1. **Strict Contiguity（严格连续性）**: 期望所有匹配的事件都严格依次出现，而不存在任何不匹配的事件。

 2. **Relaxed Contiguity（宽松连续性）**: 忽略出现在匹配项之间的不匹配事件。

 3. **Non-Deterministic Relaxed Contiguity（非确定性的宽松连续性）**: 进一步放宽连续性，允许忽略一些匹配事件的其他匹配。

用一个例子来说明上面的套路，假设一个模式序列是 `"a+ b"`（一个或多个 `"a"` 后面紧跟着 `"b"`），输入是 `"a1", "c", "a2", "b"` ，那么将会出现下面的结果 : 

 1. **Strict Contiguity（严格连续性）**: `{a2 b}` -- 由于 `"c"` 在 `"a1"` 后面，所以造成了 `"a1"` 被丢弃。

 2. **Relaxed Contiguity（宽松连续性）**: `{a1 b}` 和 `{a1 a2 b}` -- `c` 被忽略。

 3. **Non-Deterministic Relaxed Contiguity（非确定性的宽松连续性）**: `{a1 b}`, `{a2 b}` 和 `{a1 a2 b}`.

对于循环模式（例如，`oneOrMore()` and `times()`）默认是 **Relaxed Contiguity（宽松连续性）** 。
如果您想要 **Strict Contiguity（严格连续性）** ，您必须通过调用 `consecutive()` 方法来明确的指定，并且如果您想要 **Non-Deterministic Relaxed Contiguity（非确定性的宽松连续性）** ，则可以调用 `allowCombinations()` 方法。

{% warn Attention %}
在本节中，我们正在讨论 *单个循环模式中的连续性* ，并且需要在该上下文中理解 `consecutive()` 和 `allowCombinations()` 方法的调用。
稍候，我们将学习 [Combining Patterns](#combining-patterns) 时再来讨论其它的调用。例如，`next()` 和 `followedBy()` ，这些调用用于指定在两个模式之间的邻近条件。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
       <tr>
            <td><strong>where(condition)</strong></td>
            <td>
                <p>Defines a condition for the current pattern. To match the pattern, an event must satisfy the condition.
                 Multiple consecutive where() clauses lead to their conditions being ANDed:</p>
{% highlight java %}
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
});
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>or(condition)</strong></td>
            <td>
                <p>Adds a new condition which is ORed with an existing one. An event can match the pattern only if it
                passes at least one of the conditions:</p>
{% highlight java %}
pattern.where(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // some condition
    }
}).or(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
{% endhighlight %}
                    </td>
       </tr>
              <tr>
                 <td><strong>until(condition)</strong></td>
                 <td>
                     <p>Specifies a stop condition for a looping pattern. Meaning if event matching the given condition occurs, no more
                     events will be accepted into the pattern.</p>
                     <p>Applicable only in conjunction with <code>oneOrMore()</code></p>
                     <p><b>NOTE:</b> It allows for cleaning state for corresponding pattern on event-based condition.</p>
{% highlight java %}
pattern.oneOrMore().until(new IterativeCondition<Event>() {
    @Override
    public boolean filter(Event value, Context ctx) throws Exception {
        return ... // alternative condition
    }
});
{% endhighlight %}
                 </td>
              </tr>
       <tr>
           <td><strong>subtype(subClass)</strong></td>
           <td>
               <p>Defines a subtype condition for the current pattern. An event can only match the pattern if it is
                of this subtype:</p>
{% highlight java %}
pattern.subtype(SubEvent.class);
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>oneOrMore()</strong></td>
          <td>
              <p>Specifies that this pattern expects at least one occurrence of a matching event.</p>
              <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
              internal contiguity see <a href="#consecutive_java">consecutive</a>.</p>
              <p><b>NOTE:</b> It is advised to use either <code>until()</code> or <code>within()</code> to enable state clearing</p>
{% highlight java %}
pattern.oneOrMore();
{% endhighlight %}
          </td>
       </tr>
           <tr>
              <td><strong>timesOrMore(#times)</strong></td>
              <td>
                  <p>Specifies that this pattern expects at least <strong>#times</strong> occurrences
                  of a matching event.</p>
                  <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
                  internal contiguity see <a href="#consecutive_java">consecutive</a>.</p>
{% highlight java %}
pattern.timesOrMore(2);
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>times(#ofTimes)</strong></td>
          <td>
              <p>Specifies that this pattern expects an exact number of occurrences of a matching event.</p>
              <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
              internal contiguity see <a href="#consecutive_java">consecutive</a>.</p>
{% highlight java %}
pattern.times(2);
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>times(#fromTimes, #toTimes)</strong></td>
          <td>
              <p>Specifies that this pattern expects occurrences between <strong>#fromTimes</strong>
              and <strong>#toTimes</strong> of a matching event.</p>
              <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
              internal contiguity see <a href="#consecutive_java">consecutive</a>.</p>
{% highlight java %}
pattern.times(2, 4);
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>optional()</strong></td>
          <td>
              <p>Specifies that this pattern is optional, i.e. it may not occur at all. This is applicable to all
              aforementioned quantifiers.</p>
{% highlight java %}
pattern.oneOrMore().optional();
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>greedy()</strong></td>
          <td>
              <p>Specifies that this pattern is greedy, i.e. it will repeat as many as possible. This is only applicable
              to quantifiers and it does not support group pattern currently.</p>
{% highlight java %}
pattern.oneOrMore().greedy();
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>consecutive()</strong><a name="consecutive_java"></a></td>
          <td>
              <p>Works in conjunction with <code>oneOrMore()</code> and <code>times()</code> and imposes strict contiguity between the matching
              events, i.e. any non-matching element breaks the match (as in <code>next()</code>).</p>
              <p>If not applied a relaxed contiguity (as in <code>followedBy()</code>) is used.</p>

              <p>E.g. a pattern like:</p>
{% highlight java %}
Pattern.<Event>begin("start").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("c");
  }
})
.followedBy("middle").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("a");
  }
}).oneOrMore().consecutive()
.followedBy("end1").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("b");
  }
});
{% endhighlight %}
              <p>Will generate the following matches for an input sequence: C D A1 A2 A3 D A4 B</p>

              <p>with consecutive applied: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}</p>
              <p>without consecutive applied: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
          </td>
       </tr>
       <tr>
       <td><strong>allowCombinations()</strong><a name="allow_comb_java"></a></td>
       <td>
              <p>Works in conjunction with <code>oneOrMore()</code> and <code>times()</code> and imposes non-deterministic relaxed contiguity
              between the matching events (as in <code>followedByAny()</code>).</p>
              <p>If not applied a relaxed contiguity (as in <code>followedBy()</code>) is used.</p>

              <p>E.g. a pattern like:</p>
{% highlight java %}
Pattern.<Event>begin("start").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("c");
  }
})
.followedBy("middle").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("a");
  }
}).oneOrMore().allowCombinations()
.followedBy("end1").where(new SimpleCondition<Event>() {
  @Override
  public boolean filter(Event value) throws Exception {
    return value.getName().equals("b");
  }
});
{% endhighlight %}
               <p>Will generate the following matches for an input sequence: C D A1 A2 A3 D A4 B</p>

               <p>with combinations enabled: {C A1 B}, {C A1 A2 B}, {C A1 A3 B}, {C A1 A4 B}, {C A1 A2 A3 B}, {C A1 A2 A4 B}, {C A1 A3 A4 B}, {C A1 A2 A3 A4 B}</p>
               <p>without combinations enabled: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
       </td>
       </tr>
  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
	    </thead>
    <tbody>

        <tr>
            <td><strong>where(condition)</strong></td>
            <td>
              <p>Defines a condition for the current pattern. To match the pattern, an event must satisfy the condition.
                                  Multiple consecutive where() clauses lead to their conditions being ANDed:</p>
{% highlight scala %}
pattern.where(event => ... /* some condition */)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>or(condition)</strong></td>
            <td>
                <p>Adds a new condition which is ORed with an existing one. An event can match the pattern only if it
                passes at least one of the conditions:</p>
{% highlight scala %}
pattern.where(event => ... /* some condition */)
    .or(event => ... /* alternative condition */)
{% endhighlight %}
                    </td>
                </tr>
<tr>
          <td><strong>until(condition)</strong></td>
          <td>
              <p>Specifies a stop condition for looping pattern. Meaning if event matching the given condition occurs, no more
              events will be accepted into the pattern.</p>
              <p>Applicable only in conjunction with <code>oneOrMore()</code></p>
              <p><b>NOTE:</b> It allows for cleaning state for corresponding pattern on event-based condition.</p>
{% highlight scala %}
pattern.oneOrMore().until(event => ... /* some condition */)
{% endhighlight %}
          </td>
       </tr>
       <tr>
           <td><strong>subtype(subClass)</strong></td>
           <td>
               <p>Defines a subtype condition for the current pattern. An event can only match the pattern if it is
               of this subtype:</p>
{% highlight scala %}
pattern.subtype(classOf[SubEvent])
{% endhighlight %}
           </td>
       </tr>
       <tr>
          <td><strong>oneOrMore()</strong></td>
          <td>
               <p>Specifies that this pattern expects at least one occurrence of a matching event.</p>
                            <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
                            internal contiguity see <a href="#consecutive_scala">consecutive</a>.</p>
                            <p><b>NOTE:</b> It is advised to use either <code>until()</code> or <code>within()</code> to enable state clearing</p>
{% highlight scala %}
pattern.oneOrMore()
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>timesOrMore(#times)</strong></td>
          <td>
              <p>Specifies that this pattern expects at least <strong>#times</strong> occurrences
              of a matching event.</p>
              <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
              internal contiguity see <a href="#consecutive_scala">consecutive</a>.</p>
{% highlight scala %}
pattern.timesOrMore(2)
{% endhighlight %}
           </td>
       </tr>
       <tr>
                 <td><strong>times(#ofTimes)</strong></td>
                 <td>
                     <p>Specifies that this pattern expects an exact number of occurrences of a matching event.</p>
                                   <p>By default a relaxed internal contiguity (between subsequent events) is used.
                                   For more info on internal contiguity see <a href="#consecutive_scala">consecutive</a>.</p>
{% highlight scala %}
pattern.times(2)
{% endhighlight %}
                 </td>
       </tr>
       <tr>
         <td><strong>times(#fromTimes, #toTimes)</strong></td>
         <td>
             <p>Specifies that this pattern expects occurrences between <strong>#fromTimes</strong>
             and <strong>#toTimes</strong> of a matching event.</p>
             <p>By default a relaxed internal contiguity (between subsequent events) is used. For more info on
             internal contiguity see <a href="#consecutive_java">consecutive</a>.</p>
{% highlight scala %}
pattern.times(2, 4)
{% endhighlight %}
         </td>
       </tr>
       <tr>
          <td><strong>optional()</strong></td>
          <td>
             <p>Specifies that this pattern is optional, i.e. it may not occur at all. This is applicable to all
                           aforementioned quantifiers.</p>
{% highlight scala %}
pattern.oneOrMore().optional()
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>greedy()</strong></td>
          <td>
             <p>Specifies that this pattern is greedy, i.e. it will repeat as many as possible. This is only applicable
             to quantifiers and it does not support group pattern currently.</p>
{% highlight scala %}
pattern.oneOrMore().greedy()
{% endhighlight %}
          </td>
       </tr>
       <tr>
          <td><strong>consecutive()</strong><a name="consecutive_scala"></a></td>
          <td>
            <p>Works in conjunction with <code>oneOrMore()</code> and <code>times()</code> and imposes strict contiguity between the matching
                          events, i.e. any non-matching element breaks the match (as in <code>next()</code>).</p>
                          <p>If not applied a relaxed contiguity (as in <code>followedBy()</code>) is used.</p>

      <p>E.g. a pattern like:</p>
{% highlight scala %}
Pattern.begin("start").where(_.getName().equals("c"))
  .followedBy("middle").where(_.getName().equals("a"))
                       .oneOrMore().consecutive()
  .followedBy("end1").where(_.getName().equals("b"))
{% endhighlight %}

            <p>Will generate the following matches for an input sequence: C D A1 A2 A3 D A4 B</p>

                          <p>with consecutive applied: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}</p>
                          <p>without consecutive applied: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
          </td>
       </tr>
       <tr>
              <td><strong>allowCombinations()</strong><a name="allow_comb_java"></a></td>
              <td>
                <p>Works in conjunction with <code>oneOrMore()</code> and <code>times()</code> and imposes non-deterministic relaxed contiguity
                     between the matching events (as in <code>followedByAny()</code>).</p>
                     <p>If not applied a relaxed contiguity (as in <code>followedBy()</code>) is used.</p>

      <p>E.g. a pattern like:</p>
{% highlight scala %}
Pattern.begin("start").where(_.getName().equals("c"))
  .followedBy("middle").where(_.getName().equals("a"))
                       .oneOrMore().allowCombinations()
  .followedBy("end1").where(_.getName().equals("b"))
{% endhighlight %}

                      <p>Will generate the following matches for an input sequence: C D A1 A2 A3 D A4 B</p>

                      <p>with combinations enabled: {C A1 B}, {C A1 A2 B}, {C A1 A3 B}, {C A1 A4 B}, {C A1 A2 A3 B}, {C A1 A2 A4 B}, {C A1 A3 A4 B}, {C A1 A2 A3 A4 B}</p>
                      <p>without combinations enabled: {C A1 B}, {C A1 A2 B}, {C A1 A2 A3 B}, {C A1 A2 A3 A4 B}</p>
              </td>
              </tr>
  </tbody>
</table>
</div>

</div>

### Combining Patterns（联合模式）

现在您已经学习了单个模式的套路，是时候来看看如何将它们联合在一起组成完整的模式序列了。

模式序列必须以 initial pattern（初始模式）来开始，如下所示 : 

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin("start");
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val start : Pattern[Event, _] = Pattern.begin("start")
{% endhighlight %}
</div>
</div>

现在，您可以通过在它们之间指定所描述的 *contiguity conditions（邻近条件）* 来添加更多的模式到你的模式序列。
在 [邻近条件](#conditions-on-contiguity) 中我们描述了 Flink 所支持的不同的邻近模式，它们分别为 *strict*, *relaxed* 和 *non-deterministic relaxed*，并且还介绍了如何将它们应用到循环模式中去。
要中 consecutive patterns（连续模式）中应用它们, 则可以使用 : 

1. `next()`, 针对 *strict*,
2. `followedBy()`, 针对 *relaxed*, 以及
3. `followedByAny()`, 针对 *non-deterministic relaxed* 邻近模式.

或

1. `notNext()`, 如果你不想让一个事件类型直接跟随另一个
2. `notFollowedBy()`, 如果你不希望事件类型在两个其它事件类型之间的任何地方

{% warn Attention %} 模式序列不能以 `notFollowedBy()` 方法来结束。

{% warn Attention %} 一个 `NOT` 模式不能够在前面加一个可选的模式。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

// strict contiguity
Pattern<Event, ?> strict = start.next("middle").where(...);

// relaxed contiguity
Pattern<Event, ?> relaxed = start.followedBy("middle").where(...);

// non-deterministic relaxed contiguity
Pattern<Event, ?> nonDetermin = start.followedByAny("middle").where(...);

// NOT pattern with strict contiguity
Pattern<Event, ?> strictNot = start.notNext("not").where(...);

// NOT pattern with relaxed contiguity
Pattern<Event, ?> relaxedNot = start.notFollowedBy("not").where(...);

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

// strict contiguity
val strict: Pattern[Event, _] = start.next("middle").where(...)

// relaxed contiguity
val relaxed: Pattern[Event, _] = start.followedBy("middle").where(...)

// non-deterministic relaxed contiguity
val nonDetermin: Pattern[Event, _] = start.followedByAny("middle").where(...)

// NOT pattern with strict contiguity
val strictNot: Pattern[Event, _] = start.notNext("not").where(...)

// NOT pattern with relaxed contiguity
val relaxedNot: Pattern[Event, _] = start.notFollowedBy("not").where(...)

{% endhighlight %}
</div>
</div>

宽松的邻近性意味着只有第一个以后的事件会被匹配，当使用非确定性的宽松邻近性时，在相同的开始将会发出多个匹配。
例如，假设一个模式是 `a b`，给定的事件序列是 `"a", "c", "b1", "b2"` ，将会给出以下结果:

1. Strict Contiguity（严格连续性）: `a` 和 `b`: `{}` (不匹配), `"c"` 在 `"a"` 之后造成了 `"a"` 被丢弃.

2. Relaxed Contiguity（宽松连续性）: `a` 和 `b`: `{a b1}`, 由于宽松连续性的原因，所以跳过了不匹配的事件，直到匹配为止.

3. Non-Deterministic Relaxed Contiguity（非确定性宽松连续性） 在 `a` 和 `b` 之间 : `{a b1}`, `{a b2}`, 因为这是最常见的形式.

也可以为模式定义一个时态约束来使其有效。
例如，您可以通过 `pattern.within()` 方法定义一个应该在 10 秒内的模式。
Temporal patterns（时态模式）支持 [处理和事件时间]({{site.baseurl}}/dev/event_time.html) 。

{% warn Attention %} 一个模式序列只能有一个时态约束。如果在不同的单个模式上定义了这样的时态约束，那么就会应用最小的时态约束。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
next.within(Time.seconds(10));
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
next.within(Time.seconds(10))
{% endhighlight %}
</div>
</div>

还可以将模式序列定义为 `begin`, `followedBy`, `followedByAny` 和 `next` 的条件.
模式序列在逻辑上将被视为匹配条件，并且返回一个 `GroupPattern`，以及它可以应用 `oneOrMore()`, `times(#ofTimes)`, `times(#fromTimes, #toTimes)`, `optional()`, `consecutive()`, `allowCombinations()` 到 `GroupPattern` 上。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}

Pattern<Event, ?> start = Pattern.begin(
    Pattern.<Event>begin("start").where(...).followedBy("start_middle").where(...)
);

// strict contiguity
Pattern<Event, ?> strict = start.next(
    Pattern.<Event>begin("next_start").where(...).followedBy("next_middle").where(...)
).times(3);

// relaxed contiguity
Pattern<Event, ?> relaxed = start.followedBy(
    Pattern.<Event>begin("followedby_start").where(...).followedBy("followedby_middle").where(...)
).oneOrMore();

// non-deterministic relaxed contiguity
Pattern<Event, ?> nonDetermin = start.followedByAny(
    Pattern.<Event>begin("followedbyany_start").where(...).followedBy("followedbyany_middle").where(...)
).optional();

{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}

val start: Pattern[Event, _] = Pattern.begin(
    Pattern.begin[Event, _]("start").where(...).followedBy("start_middle").where(...)
)

// strict contiguity
val strict: Pattern[Event, _] = start.next(
    Pattern.begin[Event, _]("next_start").where(...).followedBy("next_middle").where(...)
).times(3)

// relaxed contiguity
val relaxed: Pattern[Event, _] = start.followedBy(
    Pattern.begin[Event, _]("followedby_start").where(...).followedBy("followedby_middle").where(...)
).oneOrMore()

// non-deterministic relaxed contiguity
val nonDetermin: Pattern[Event, _] = start.followedByAny(
    Pattern.begin[Event, _]("followedbyany_start").where(...).followedBy("followedbyany_middle").where(...)
).optional()

{% endhighlight %}
</div>
</div>

<br />

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><strong>begin(#name)</strong></td>
            <td>
            <p>Defines a starting pattern:</p>
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin("start");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>begin(#pattern_sequence)</strong></td>
            <td>
            <p>Defines a starting pattern:</p>
{% highlight java %}
Pattern<Event, ?> start = Pattern.<Event>begin(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#name)</strong></td>
            <td>
                <p>Appends a new pattern. A matching event has to directly succeed the previous matching event
                (strict contiguity):</p>
{% highlight java %}
Pattern<Event, ?> next = start.next("middle");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#pattern_sequence)</strong></td>
            <td>
                <p>Appends a new pattern. A sequence of matching events have to directly succeed the previous matching event
                (strict contiguity):</p>
{% highlight java %}
Pattern<Event, ?> next = start.next(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#name)</strong></td>
            <td>
                <p>Appends a new pattern. Other events can occur between a matching event and the previous
                matching event (relaxed contiguity):</p>
{% highlight java %}
Pattern<Event, ?> followedBy = start.followedBy("middle");
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#pattern_sequence)</strong></td>
            <td>
                 <p>Appends a new pattern. Other events can occur between a sequence of matching events and the previous
                 matching event (relaxed contiguity):</p>
{% highlight java %}
Pattern<Event, ?> followedBy = start.followedBy(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedByAny(#name)</strong></td>
            <td>
                <p>Appends a new pattern. Other events can occur between a matching event and the previous
                matching event, and alternative matches will be presented for every alternative matching event
                (non-deterministic relaxed contiguity):</p>
{% highlight java %}
Pattern<Event, ?> followedByAny = start.followedByAny("middle");
{% endhighlight %}
             </td>
        </tr>
        <tr>
             <td><strong>followedByAny(#pattern_sequence)</strong></td>
             <td>
                 <p>Appends a new pattern. Other events can occur between a sequence of matching events and the previous
                 matching event, and alternative matches will be presented for every alternative sequence of matching events
                 (non-deterministic relaxed contiguity):</p>
{% highlight java %}
Pattern<Event, ?> followedByAny = start.followedByAny(
    Pattern.<Event>begin("start").where(...).followedBy("middle").where(...)
);
{% endhighlight %}
             </td>
        </tr>
        <tr>
                    <td><strong>notNext()</strong></td>
                    <td>
                        <p>Appends a new negative pattern. A matching (negative) event has to directly succeed the
                        previous matching event (strict contiguity) for the partial match to be discarded:</p>
{% highlight java %}
Pattern<Event, ?> notNext = start.notNext("not");
{% endhighlight %}
                    </td>
                </tr>
                <tr>
                    <td><strong>notFollowedBy()</strong></td>
                    <td>
                        <p>Appends a new negative pattern. A partial matching event sequence will be discarded even
                        if other events occur between the matching (negative) event and the previous matching event
                        (relaxed contiguity):</p>
{% highlight java %}
Pattern<Event, ?> notFollowedBy = start.notFollowedBy("not");
{% endhighlight %}
                    </td>
                </tr>
       <tr>
          <td><strong>within(time)</strong></td>
          <td>
              <p>Defines the maximum time interval for an event sequence to match the pattern. If a non-completed event
              sequence exceeds this time, it is discarded:</p>
{% highlight java %}
pattern.within(Time.seconds(10));
{% endhighlight %}
          </td>
       </tr>
  </tbody>
</table>
</div>

<div data-lang="scala" markdown="1">
<table class="table table-bordered">
    <thead>
        <tr>
            <th class="text-left" style="width: 25%">Pattern Operation</th>
            <th class="text-center">Description</th>
        </tr>
    </thead>
    <tbody>
        <tr>
            <td><strong>begin()</strong></td>
            <td>
            <p>Defines a starting pattern:</p>
{% highlight scala %}
val start = Pattern.begin[Event]("start")
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#name)</strong></td>
            <td>
                <p>Appends a new pattern. A matching event has to directly succeed the previous matching event
                (strict contiguity):</p>
{% highlight scala %}
val next = start.next("middle")
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>next(#pattern_sequence)</strong></td>
            <td>
                <p>Appends a new pattern. A sequence of matching events have to directly succeed the previous matching event
                (strict contiguity):</p>
{% highlight scala %}
val next = start.next(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#name)</strong></td>
            <td>
                <p>Appends a new pattern. Other events can occur between a matching event and the previous
                matching event (relaxed contiguity) :</p>
{% highlight scala %}
val followedBy = start.followedBy("middle")
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedBy(#pattern_sequence)</strong></td>
            <td>
                <p>Appends a new pattern. Other events can occur between a sequence of matching events and the previous
                matching event (relaxed contiguity) :</p>
{% highlight scala %}
val followedBy = start.followedBy(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
            </td>
        </tr>
        <tr>
            <td><strong>followedByAny(#name)</strong></td>
            <td>
                <p>Appends a new pattern. Other events can occur between a matching event and the previous
                matching event, and alternative matches will be presented for every alternative matching event
                (non-deterministic relaxed contiguity):</p>
{% highlight scala %}
val followedByAny = start.followedByAny("middle")
{% endhighlight %}
            </td>
         </tr>
         <tr>
             <td><strong>followedByAny(#pattern_sequence)</strong></td>
             <td>
                 <p>Appends a new pattern. Other events can occur between a sequence of matching events and the previous
                 matching event, and alternative matches will be presented for every alternative sequence of matching events
                 (non-deterministic relaxed contiguity):</p>
{% highlight scala %}
val followedByAny = start.followedByAny(
    Pattern.begin[Event]("start").where(...).followedBy("middle").where(...)
)
{% endhighlight %}
             </td>
         </tr>

                <tr>
                                    <td><strong>notNext()</strong></td>
                                    <td>
                                        <p>Appends a new negative pattern. A matching (negative) event has to directly succeed the
                                        previous matching event (strict contiguity) for the partial match to be discarded:</p>
{% highlight scala %}
val notNext = start.notNext("not")
{% endhighlight %}
                                    </td>
                                </tr>
                                <tr>
                                    <td><strong>notFollowedBy()</strong></td>
                                    <td>
                                        <p>Appends a new negative pattern. A partial matching event sequence will be discarded even
                                        if other events occur between the matching (negative) event and the previous matching event
                                        (relaxed contiguity):</p>
{% highlight scala %}
val notFollowedBy = start.notFollowedBy("not")
{% endhighlight %}
                                    </td>
                                </tr>

       <tr>
          <td><strong>within(time)</strong></td>
          <td>
              <p>Defines the maximum time interval for an event sequence to match the pattern. If a non-completed event
              sequence exceeds this time, it is discarded:</p>
{% highlight scala %}
pattern.within(Time.seconds(10))
{% endhighlight %}
          </td>
      </tr>
  </tbody>
</table>
</div>

</div>

### After Match Skip Strategy（匹配后的跳过策略）

针对一个给定的模式，相同的事件可能本分配到多个成功的匹配中。
为了控制一个事件将被分配到多少个匹配的情况，您需要通过调用 `AfterMatchSkipStrategy` 方法来指定跳过策略。
一共有四种类型的跳过策略，如下所述:

* <strong>*NO_SKIP*</strong>: 每一种可能的事件都会被发出。
* <strong>*SKIP_PAST_LAST_EVENT*</strong>: 丢弃每一个包含匹配事件的部分匹配。
* <strong>*SKIP_TO_FIRST*</strong>: 丢弃包含在第一个 *PatternName* 之前的匹配事件的每一个的部分匹配。
* <strong>*SKIP_TO_LAST*</strong>: 丢弃包含在最后个 *PatternName* 之前的匹配事件的每一个的部分匹配。.

请注意，当使用 SKIP_TO_FIRST* 和 *SKIP_TO_LAST* 跳过策略时，应该指定一个有效的 *PatternName* 。

例如，给定一个模式 `a b{2}` 和一个数据流 `ab1, ab2, ab3, ab4, ab5, ab6` ，这四种跳过策略之间的差异如下 : 

<table class="table table-bordered">
    <tr>
        <th class="text-left" style="width: 25%">Skip Strategy</th>
        <th class="text-center" style="width: 25%">Result</th>
        <th class="text-center"> Description</th>
    </tr>
    <tr>
        <td><strong>NO_SKIP</strong></td>
        <td>
            <code>ab1 ab2 ab3</code><br>
            <code>ab2 ab3 ab4</code><br>
            <code>ab3 ab4 ab5</code><br>
            <code>ab4 ab5 ab6</code><br>
        </td>
        <td>After found matching <code>ab1 ab2 ab3</code>, the match process will not discard any result.</td>
    </tr>
    <tr>
        <td><strong>SKIP_PAST_LAST_EVENT</strong></td>
        <td>
            <code>ab1 ab2 ab3</code><br>
            <code>ab4 ab5 ab6</code><br>
        </td>
        <td>After found matching <code>ab1 ab2 ab3</code>, the match process will discard all started partial matches.</td>
    </tr>
    <tr>
        <td><strong>SKIP_TO_FIRST</strong>[<code>b</code>]</td>
        <td>
            <code>ab1 ab2 ab3</code><br>
            <code>ab2 ab3 ab4</code><br>
            <code>ab3 ab4 ab5</code><br>
            <code>ab4 ab5 ab6</code><br>
        </td>
        <td>After found matching <code>ab1 ab2 ab3</code>, the match process will discard all partial matches containing <code>ab1</code>, which is the only event that comes before the first <code>b</code>.</td>
    </tr>
    <tr>
        <td><strong>SKIP_TO_LAST</strong>[<code>b</code>]</td>
        <td>
            <code>ab1 ab2 ab3</code><br>
            <code>ab3 ab4 ab5</code><br>
        </td>
        <td>After found matching <code>ab1 ab2 ab3</code>, the match process will discard all partial matches containing <code>ab1</code> and <code>ab2</code>, which are events that comes before the last <code>b</code>.</td>
    </tr>
</table>

要指定使用的跳过策略，通过调用如何方法之后，创建一个 `AfterMatchSkipStrategy` 即可:

<table class="table table-bordered">
    <tr>
        <th class="text-left" width="25%">Function</th>
        <th class="text-center">Description</th>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.noSkip()</code></td>
        <td>Create a <strong>NO_SKIP</strong> skip strategy </td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipPastLastEvent()</code></td>
        <td>Create a <strong>SKIP_PAST_LAST_EVENT</strong> skip strategy </td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipToFirst(patternName)</code></td>
        <td>Create a <strong>SKIP_TO_FIRST</strong> skip strategy with the referenced pattern name <i>patternName</i></td>
    </tr>
    <tr>
        <td><code>AfterMatchSkipStrategy.skipToLast(patternName)</code></td>
        <td>Create a <strong>SKIP_TO_LAST</strong> skip strategy with the referenced pattern name <i>patternName</i></td>
    </tr>
</table>

也可以通过调用如下方法，将跳过策略应用到一个模式中:

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
AfterMatchSkipStrategy skipStrategy = ...
Pattern.begin("patternName", skipStrategy);
{% endhighlight %}
</div>
<div data-lang="scala" markdown="1">
{% highlight scala %}
val skipStrategy = ...
Pattern.begin("patternName", skipStrategy)
{% endhighlight %}
</div>
</div>

## Detecting Patterns（检测模式）

指定要查找的模式序列后，是时候将其应用于输入流以检测潜在的匹配了。
要根据您的模式序列运行事件流，您必须创建一个 `PatternStream`。
给定一个输入流 `input`，一个模式 `pattern` 和一个可选的比较器 `comparator` 用于在 EventTime（事件时间）或同时到达的情况下用相同的 timestamp（时间戳）排序事件，
您可以通过调用以下方法来创建 `PatternStream` :

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
DataStream<Event> input = ...
Pattern<Event, ?> pattern = ...
EventComparator<Event> comparator = ... // optional

PatternStream<Event> patternStream = CEP.pattern(input, pattern, comparator);
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val input : DataStream[Event] = ...
val pattern : Pattern[Event, _] = ...
var comparator : EventComparator[Event] = ... // optional

val patternStream: PatternStream[Event] = CEP.pattern(input, pattern, comparator)
{% endhighlight %}
</div>
</div>

输入流可以根据您的情况是 *keyed* 或 *non-keyed* 。

{% warn Attention %} 在一个 non-keyed 的流上应用您的模式将导致 job 的并行度等于 1。

### Selecting from Patterns（从模式中选择）

一旦您获得了一个 `PatternStream`，您就可以通过 `select` 或 `flatSelect` 方法从检测到的事件序列中进行选择。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
The `select()` method requires a `PatternSelectFunction` implementation.
A `PatternSelectFunction` has a `select` method which is called for each matching event sequence.
It receives a match in the form of `Map<String, List<IN>>` where the key is the name of each pattern in your pattern
sequence and the value is a list of all accepted events for that pattern (`IN` is the type of your input elements).
The events for a given pattern are ordered by timestamp. The reason for returning a list of accepted events for each
pattern is that when using looping patterns (e.g. `oneToMany()` and `times()`), more than one event may be accepted for a given pattern. The selection function returns exactly one result.

{% highlight java %}
class MyPatternSelectFunction<IN, OUT> implements PatternSelectFunction<IN, OUT> {
    @Override
    public OUT select(Map<String, List<IN>> pattern) {
        IN startEvent = pattern.get("start").get(0);
        IN endEvent = pattern.get("end").get(0);
        return new OUT(startEvent, endEvent);
    }
}
{% endhighlight %}

A `PatternFlatSelectFunction` is similar to the `PatternSelectFunction`, with the only distinction that it can return an
arbitrary number of results. To do this, the `select` method has an additional `Collector` parameter which is
used to forward your output elements downstream.

{% highlight java %}
class MyPatternFlatSelectFunction<IN, OUT> implements PatternFlatSelectFunction<IN, OUT> {
    @Override
    public void flatSelect(Map<String, List<IN>> pattern, Collector<OUT> collector) {
        IN startEvent = pattern.get("start").get(0);
        IN endEvent = pattern.get("end").get(0);

        for (int i = 0; i < startEvent.getValue(); i++ ) {
            collector.collect(new OUT(startEvent, endEvent));
        }
    }
}
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
The `select()` method takes a selection function as argument, which is called for each matching event sequence.
It receives a match in the form of `Map[String, Iterable[IN]]` where the key is the name of each pattern in your pattern
sequence and the value is an Iterable over all accepted events for that pattern (`IN` is the type of your input elements).

The events for a given pattern are ordered by timestamp. The reason for returning an iterable of accepted events for each pattern is that when using looping patterns (e.g. `oneToMany()` and `times()`), more than one event may be accepted for a given pattern. The selection function returns exactly one result per call.

{% highlight scala %}
def selectFn(pattern : Map[String, Iterable[IN]]): OUT = {
    val startEvent = pattern.get("start").get.next
    val endEvent = pattern.get("end").get.next
    OUT(startEvent, endEvent)
}
{% endhighlight %}

The `flatSelect` method is similar to the `select` method. Their only difference is that the function passed to the
`flatSelect` method can return an arbitrary number of results per call. In order to do this, the function for
`flatSelect` has an additional `Collector` parameter which is used to forward your output elements downstream.

{% highlight scala %}
def flatSelectFn(pattern : Map[String, Iterable[IN]], collector : Collector[OUT]) = {
    val startEvent = pattern.get("start").get.next
    val endEvent = pattern.get("end").get.next
    for (i <- 0 to startEvent.getValue) {
        collector.collect(OUT(startEvent, endEvent))
    }
}
{% endhighlight %}
</div>
</div>

### Handling Timed Out Partial Patterns（处理超时的部分模式）

无论何时一个模式通过 `within` 关键字附加窗口长度，部分事件序列可能因为超过窗口长度而被丢弃。
为了对这些超时的部分匹配做出反应，可以使用 `select` 和 `flatSelect` API 调用来指定一个超时处理的程序。
该超时处理程序针对每个超时的部分事件序列进行调用。
超时处理程序接收模式中迄今为止已匹配的所有事件，以及检测到超时当时的时间戳。

为了处理部分模式，`select` 和 `flatSelect` API 调用提供了一个重载的版本并且附带了一个参数

 * `PatternTimeoutFunction`/`PatternFlatTimeoutFunction`
 * [OutputTag]({{ site.baseurl }}/dev/stream/side_output.html) for the side output in which the timed out matches will be returned
 * and the known `PatternSelectFunction`/`PatternFlatSelectFunction`.

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">

~~~java
PatternStream<Event> patternStream = CEP.pattern(input, pattern);

OutputTag<String> outputTag = new OutputTag<String>("side-output"){};

SingleOutputStreamOperator<ComplexEvent> result = patternStream.select(
    new PatternTimeoutFunction<Event, TimeoutEvent>() {...},
    outputTag,
    new PatternSelectFunction<Event, ComplexEvent>() {...}
);

DataStream<TimeoutEvent> timeoutResult = result.getSideOutput(outputTag);

SingleOutputStreamOperator<ComplexEvent> flatResult = patternStream.flatSelect(
    new PatternFlatTimeoutFunction<Event, TimeoutEvent>() {...},
    outputTag,
    new PatternFlatSelectFunction<Event, ComplexEvent>() {...}
);

DataStream<TimeoutEvent> timeoutFlatResult = flatResult.getSideOutput(outputTag);
~~~

</div>

<div data-lang="scala" markdown="1">

~~~scala
val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)

val outputTag = OutputTag[String]("side-output")

val result: SingleOutputStreamOperator[ComplexEvent] = patternStream.select(outputTag){
    (pattern: Map[String, Iterable[Event]], timestamp: Long) => TimeoutEvent()
} {
    pattern: Map[String, Iterable[Event]] => ComplexEvent()
}

val timeoutResult: DataStream<TimeoutEvent> = result.getSideOutput(outputTag)
~~~

The `flatSelect` API call offers the same overloaded version which takes as the first parameter a timeout function and as second parameter a selection function.
In contrast to the `select` functions, the `flatSelect` functions are called with a `Collector`. You can use the collector to emit an arbitrary number of events.

~~~scala
val patternStream: PatternStream[Event] = CEP.pattern(input, pattern)

val outputTag = OutputTag[String]("side-output")

val result: SingleOutputStreamOperator[ComplexEvent] = patternStream.flatSelect(outputTag){
    (pattern: Map[String, Iterable[Event]], timestamp: Long, out: Collector[TimeoutEvent]) =>
        out.collect(TimeoutEvent())
} {
    (pattern: mutable.Map[String, Iterable[Event]], out: Collector[ComplexEvent]) =>
        out.collect(ComplexEvent())
}

val timeoutResult: DataStream<TimeoutEvent> = result.getSideOutput(outputTag)
~~~

</div>
</div>

## Handling Lateness in Event Time（事件时间中处理延迟）

在 `CEP` 中，处理元素的顺序非常重要。
为了确保元素在事件时间内以正确的顺序处理，一个传入元素最初被放入一个缓冲区，其中元素根据其时间戳进行升序 *排序*，并且当 watermark（水位）到达时，该缓冲区的所有元素处理时间戳小于水位的时间戳。
这意味着水位之间的元素按事件时间顺序来进行处理。

{% warn Attention %} 该库假设了事件时间发生时水位的正确性。

为了保证水位之间的元素是按事件时间的顺序来处理的，Flink 的 CEP 库假设了 *correctness of the watermark（水位的正确性）* ，并且认为 *late* 元素的时间小于最后所见的水位（and considers as *late* elements whose timestamp is smaller than that of the last
seen watermark.）。
Late elements（）不会被进一步处理。

## Examples（示例）

以下示例在 `Events` 的 keyed data stream 上检测模式 `start, middle(name = "error") -> end(name = "critical")` 。
这些事件是通过它们的 `id` 来关联的，并且该有效模式必须在 10 秒以内。
整个处理过程都是在事件时间内完成的。

<div class="codetabs" markdown="1">
<div data-lang="java" markdown="1">
{% highlight java %}
StreamExecutionEnvironment env = ...
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime);

DataStream<Event> input = ...

DataStream<Event> partitionedInput = input.keyBy(new KeySelector<Event, Integer>() {
	@Override
	public Integer getKey(Event value) throws Exception {
		return value.getId();
	}
});

Pattern<Event, ?> pattern = Pattern.<Event>begin("start")
	.next("middle").where(new SimpleCondition<Event>() {
		@Override
		public boolean filter(Event value) throws Exception {
			return value.getName().equals("error");
		}
	}).followedBy("end").where(new SimpleCondition<Event>() {
		@Override
		public boolean filter(Event value) throws Exception {
			return value.getName().equals("critical");
		}
	}).within(Time.seconds(10));

PatternStream<Event> patternStream = CEP.pattern(partitionedInput, pattern);

DataStream<Alert> alerts = patternStream.select(new PatternSelectFunction<Event, Alert>() {
	@Override
	public Alert select(Map<String, List<Event>> pattern) throws Exception {
		return createAlert(pattern);
	}
});
{% endhighlight %}
</div>

<div data-lang="scala" markdown="1">
{% highlight scala %}
val env : StreamExecutionEnvironment = ...
env.setStreamTimeCharacteristic(TimeCharacteristic.EventTime)

val input : DataStream[Event] = ...

val partitionedInput = input.keyBy(event => event.getId)

val pattern = Pattern.begin("start")
  .next("middle").where(_.getName == "error")
  .followedBy("end").where(_.getName == "critical")
  .within(Time.seconds(10))

val patternStream = CEP.pattern(partitionedInput, pattern)

val alerts = patternStream.select(createAlert(_)))
{% endhighlight %}
</div>
</div>

## Migrating from an older Flink version（从旧的 Flink 版本进行迁移）

Flink-1.3 中的 CEP 库附带了许多新功能，这些功能导致 API 发生了一些变化。
在这里，我们描述您需要对旧 CEP 作业进行的更改，以便能够使用 Flink-1.3 运行它们。
在做出这些更改并重新编译工作之后，您将能够从旧版本作业的保存点恢复执行，即无需重新处理过去的数据。

所需的更改是 :

1. 更改您的条件 (`where(...)` 子句中的那些) 来扩展 `SimpleCondition` 类，而不是实现 `FilterFunction` 接口.

2. Change your functions provided as arguments to the `select(...)` and `flatSelect(...)` methods to expect a list of
events associated with each pattern (`List` in `Java`, `Iterable` in `Scala`). This is because with the addition of
the looping patterns, multiple input events can match a single (looping) pattern.

3. Flink 1.1 和 1.2 中的 `followedBy()` 意味着 `non-deterministic relaxed contiguity（非确定宽松连续性）` (请参阅
[这里](#conditions-on-contiguity)). 在 Flink 1.3 中这个已经发生了变化，`followedBy()` 意味着 `relaxed contiguity（宽松连续性）,
而如果需要 `non-deterministic relaxed contiguity（非确定宽松连续性）` 的话应该使用 `followedByAny()` 。

{% top %}
