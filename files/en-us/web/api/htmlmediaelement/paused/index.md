---
title: HTMLMediaElement.paused
slug: Web/API/HTMLMediaElement/paused
tags:
- API
- HTML DOM
- HTMLMediaElement
- Property
- Read-only
browser-compat: api.HTMLMediaElement.paused
---
<div>{{APIRef("HTML DOM")}}</div>

<p>The read-only <strong><code>HTMLMediaElement.paused</code></strong> property
  tells whether the media element is paused.</p>

<h2 id="Syntax">Syntax</h2>

<pre
  class="brush: js">var <em>isPaused</em> = <em>audioOrVideo</em>.paused</pre>

<h3 id="Value">Value</h3>

<p>A boolean value. <code>true</code> is paused and <code>false</code> is not
  paused.</p>

<h2 id="Example">Example</h2>

<pre class="brush: js">var obj = document.createElement('video');
console.log(obj.paused); // true
</pre>

<h2 id="Specifications">Specifications</h2>

{{Specifications}}

<h2 id="Browser_compatibility">Browser compatibility</h2>

<p>{{Compat}}</p>

<h2 id="See_also">See also</h2>

<ul>
  <li>The interface defining it, {{domxref("HTMLMediaElement")}}.</li>
</ul>