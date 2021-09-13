---
title: Using Touch Events
slug: Web/API/Touch_events/Using_Touch_Events
tags:
  - Guide
  - TouchEvent
  - touch
---
<p>{{DefaultAPISidebar("Touch Events")}}</p>

<p>Today, most Web content is designed for keyboard and mouse input. However, devices with touch screens (especially portable devices) are mainstream and Web applications can either directly process touch-based input by using {{domxref("Touch_events","Touch Events")}} or the application can use <em>interpreted mouse events</em> for the application input. A disadvantage to using mouse events is that they do not support concurrent user input, whereas touch events support multiple simultaneous inputs (possibly at different locations on the touch surface), thus enhancing user experiences.</p>

<p>The touch events interfaces support application specific single and multi-touch interactions such as a two-finger gesture. A multi-touch interaction starts when a finger (or stylus) first touches the contact surface. Other fingers may subsequently touch the surface and optionally move across the touch surface. The interaction ends when the fingers are removed from the surface. During this interaction, an application receives touch events during the start, move, and end phases. The application may apply its own semantics to the touch inputs.</p>

<h2 id="Interfaces">Interfaces</h2>

<p>Touch events consist of three interfaces ({{domxref("Touch")}}, {{domxref("TouchEvent")}} and {{domxref("TouchList")}}) and the following event types:</p>

<ul>
 <li>{{domxref("Document/touchstart_event", "touchstart")}} - fired when a touch point is placed on the touch surface.</li>
 <li>{{event("touchmove")}} - fired when a touch point is moved along the touch surface.</li>
 <li>{{event("touchend")}} - fired when a touch point is removed from the touch surface.</li>
 <li>{{event("touchcancel")}} - fired when a touch point has been disrupted in an implementation-specific manner (for example, too many touch points are created).</li>
</ul>

<p>The {{domxref("Touch")}} interface represents a single contact point on a touch-sensitive device. The contact point is typically referred to as a <em>touch point</em> or just a <em>touch</em>. A touch is usually generated by a finger or stylus on a touchscreen, pen or trackpad. A touch point's <a href="/en-US/docs/Web/API/Touch#properties">properties</a> include a unique identifier, the touch point's target element as well as the <em>X</em> and <em>Y</em> coordinates of the touch point's position relative to the viewport, page, and screen.</p>

<p>The {{domxref("TouchList")}} interface represents a <em>list</em> of contact points with a touch surface, one touch point per contact. Thus, if the user activated the touch surface with one finger, the list would contain one item, and if the user touched the surface with three fingers, the list length would be three.</p>

<p>The {{domxref("TouchEvent")}} interface represents an event sent when the state of contacts with a touch-sensitive surface changes. The state changes are starting contact with a touch surface, moving a touch point while maintaining contact with the surface, releasing a touch point and canceling a touch event. This interface's attributes include the state of several <em>modifier keys</em> (for example the <kbd>shift</kbd> key) and the following touch lists:</p>

<ul>
 <li>{{domxref("TouchEvent.touches","touches")}} - a list of all of the touch points currently on the screen.</li>
 <li>{{domxref("TouchEvent.targetTouches","targetTouches")}} - a list of the touch points on the <em>target</em> DOM element.</li>
 <li>{{domxref("TouchEvent.changedTouches","changedTouches")}} - a list of the touch points whose items depend on the associated event type:
  <ul>
   <li>For the {{domxref("Document/touchstart_event", "touchstart")}} event, it is a list of the touch points that became active with the current event.</li>
   <li>For the {{event("touchmove")}} event, it is a list of the touch points that have changed since the last event.</li>
   <li>For the {{event("touchend")}} event, it is a list of the touch points that have been removed from the surface (that is, the set of touch points corresponding to fingers no longer touching the surface).</li>
  </ul>
 </li>
</ul>

<p>Together, these interfaces define a relatively low-level set of features, yet they support many kinds of touch-based interaction, including the familiar multi-touch gestures such as multi-finger swipe, rotation, pinch and zoom.</p>

<h2 id="From_interfaces_to_gestures">From interfaces to gestures</h2>

<p>An application may consider different factors when defining the semantics of a gesture. For instance, the distance a touch point traveled from its starting location to its location when the touch ended. Another potential factor is time; for example, the time elapsed between the touch's start and the touch's end, or the time lapse between two <em>consecutive</em> taps intended to create a double-tap gesture. The directionality of a swipe (for example left to right, right to left, etc.) is another factor to consider.</p>

<p>The touch list(s) an application uses depends on the semantics of the application's <em>gestures</em>. For example, if an application supports a single touch (tap) on one element, it would use the {{domxref("TouchEvent.targetTouches","targetTouches")}} list in the {{domxref("Document/touchstart_event", "touchstart")}} event handler to process the touch point in an application-specific manner. If an application supports two-finger swipe for any two touch points, it will use the {{domxref("TouchEvent.changedTouches","changedTouches")}} list in the {{event("touchmove")}} event handler to determine if two touch points had moved and then implement the semantics of that gesture in an application-specific manner.</p>

<p>Browsers typically dispatch <em>emulated</em> mouse and click events when there is only a single active touch point. Multi-touch interactions involving two or more active touch points will usually only generate touch events. To prevent the emulated mouse events from being sent, use the {{domxref("Event.preventDefault()","preventDefault()")}} method in the touch event handlers. For more information about the interaction between mouse and touch events, see {{domxref("Touch_events.Supporting_both_TouchEvent_and_MouseEvent", "Supporting both TouchEvent and MouseEvent")}}.</p>

<h2 id="Basic_steps">Basic steps</h2>

<p>This section contains a basic usage of using the above interfaces. See the {{domxref("Touch_events","Touch Events Overview")}} for a more detailed example.</p>

<p>Register an event handler for each touch event type.</p>

<pre class="brush: js">// Register touch event handlers
someElement.addEventListener('touchstart', process_touchstart, false);
someElement.addEventListener('touchmove', process_touchmove, false);
someElement.addEventListener('touchcancel', process_touchcancel, false);
someElement.addEventListener('touchend', process_touchend, false);
</pre>

<p>Process an event in an event handler, implementing the application's gesture semantics.</p>

<pre class="brush: js">// touchstart handler
function process_touchstart(ev) {
  // Use the event's data to call out to the appropriate gesture handlers
  switch (ev.touches.length) {
    case 1: handle_one_touch(ev); break;
    case 2: handle_two_touches(ev); break;
    case 3: handle_three_touches(ev); break;
    default: gesture_not_supported(ev); break;
  }
}
</pre>

<p>Access the attributes of a touch point.</p>

<pre class="brush: js">// Create touchstart handler
someElement.addEventListener('touchstart', function(ev) {
  // Iterate through the touch points that were activated
  // for this element and process each event 'target'
  for (var i=0; i &lt; ev.targetTouches.length; i++) {
    process_target(ev.targetTouches[i].target);
  }
}, false);
</pre>

<p>Prevent the browser from processing <em>emulated mouse events</em>.</p>

<pre class="brush: js">// touchmove handler
function process_touchmove(ev) {
  // Set call preventDefault()
  ev.preventDefault();
}
</pre>

<h2 id="Best_practices">Best practices</h2>

<p>Here are some <em>best practices</em> to consider when using touch events:</p>

<ul>
 <li>Minimize the amount of work that is done in the touch handlers.</li>
 <li>Add the touch point handlers to the specific target element (rather than the entire document or nodes higher up in the document tree).</li>
 <li>Add {{event("touchmove")}}, {{event("touchend")}} and {{event("touchcancel")}} event handlers within the {{domxref("Document/touchstart_event", "touchstart")}}.</li>
 <li>The target touch element or node should be large enough to accommodate a finger touch. If the target area is too small, touching it could result in firing other events for adjacent elements.</li>
</ul>

<h2 id="Implementation_and_deployment_status">Implementation and deployment status</h2>

<p>The <a href="/en-US/docs/Web/API/Touch_events#browser_compatibility">touch events browser compatibility data</a> indicates touch event support among mobile browsers is relatively broad, with desktop browser support lagging although additional implementations are in progress.</p>

<p>Some new features regarding a touch point's <a href="/en-US/docs/Web/API/Touch#touch_area">touch area</a> - the area of contact between the user and the touch surface - are in the process of being standardized. The new features include the <em>X</em> and <em>Y</em> radius of the ellipse that most closely circumscribes a touch point's contact area with the touch surface. The touch point's <em>rotation angle</em> - the number of degrees of rotation to apply to the described ellipse to align with the contact area - is also be standardized as is the amount of pressure applied to a touch point.</p>

<h2 id="What_about_Pointer_Events">What about Pointer Events?</h2>

<p>The introduction of new input mechanisms results in increased application complexity to handle various input events, such as key events, mouse events, pen/stylus events, and touch events. To help address this problem, the <a href="https://www.w3.org/TR/pointerevents/">Pointer Events standard</a> <em>defines events and related interfaces for handling hardware agnostic pointer input from devices including a mouse, pen, touchscreen, etc.</em>. That is, the abstract <em>pointer</em> creates a unified input model that can represent a contact point for a finger, pen/stylus or mouse. See the <a href="/en-US/docs/Web/API/Pointer_events">Pointer Events MDN article</a>.</p>

<p>The pointer event model can simplify an application's input processing since a pointer represents input from any input device. Additionally, the pointer event types are very similar to mouse event types (for example, <code>pointerdown</code> <code>pointerup</code>) thus code to handle pointer events closely matches mouse handling code.</p>

<p>The implementation status of pointer events in browsers is <a href="https://caniuse.com/#search=pointer">relatively high</a> with Chrome, Firefox, IE11 and Edge having complete implementations.</p>

<h2 id="Examples_and_demos">Examples and demos</h2>

<p>The following documents describe how to use touch events and include example code:</p>

<ul>
 <li>{{domxref("Touch_events","Touch Events Overview")}}</li>
 <li><a href="https://developers.google.com/web/fundamentals/design-and-ui/input/touch/touch-events">Implement Custom Gestures</a></li>
 <li><a href="https://www.codicode.com/art/easy_way_to_add_touch_support_to_your_website.aspx">Add touch screen support to your website (The easy way)</a></li>
</ul>

<p>Touch event demonstrations:</p>

<ul>
 <li><a href="https://rbyers.github.io/paint.html">Paint Program (by Rick Byers)</a></li>
 <li><a href="https://patrickhlauke.github.io/touch/">Touch/pointer tests and demos (by Patrick H. Lauke)</a></li>
</ul>

<h2 id="Community">Community</h2>

<ul>
 <li><a href="https://github.com/w3c/touch-events">Touch Events Community Group</a></li>
 <li><a href="https://lists.w3.org/Archives/Public/public-touchevents/">Mail list</a></li>
 <li><a href="irc://irc.w3.org:6667/">W3C #touchevents IRC channel</a></li>
</ul>

<h2 id="Related_topics_and_resources">Related topics and resources</h2>

<ul>
 <li><a href="https://www.w3.org/TR/pointerevents/">Pointer Events Standard</a></li>
</ul>