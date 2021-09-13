---
title: MediaTrackConstraints.latency
slug: Web/API/MediaTrackConstraints/latency
tags:
- API
- Audio
- Media
- Media Capture and Streams API
- Media Streams API
- MediaTrackConstraints
- Property
- Reference
- WebRTC
- getusermedia
- latency
browser-compat: api.MediaTrackConstraints.latency
---
<div>{{APIRef("Media Capture and Streams")}}</div>

<p>The {{domxref("MediaTrackConstraints")}} dictionary's
  <code><strong>latency</strong></code> property is a <a href="/en-US/docs/Web/API/MediaTrackConstraints#ConstrainDouble"><code>ConstrainDouble</code></a>
  describing the requested or mandatory constraints placed upon the value of the
  {{domxref("MediaTrackSettings.latency", "latency")}} constrainable property.</p>

<p>If needed, you can determine whether or not this constraint is supported by checking
  the value of {{domxref("MediaTrackSupportedConstraints.latency")}} as returned by a call
  to {{domxref("MediaDevices.getSupportedConstraints()")}}. However, typically this is
  unnecessary since browsers will ignore any constraints they're unfamiliar with.</p>

<p>Because {{Glossary("RTP")}} doesn't include this information, tracks associated with a
  <a href="/en-US/docs/Web/API/WebRTC_API">WebRTC</a> {{domxref("RTCPeerConnection")}}
  will never include this property.</p>

<h2 id="Syntax">Syntax</h2>

<pre class="brush: js">var <em>constraintsObject</em> = { latency: <em>constraint</em> };

<em>constraintsObject</em>.latency = <em>constraint</em>;
</pre>

<h3 id="Value">Value</h3>

<p>A <a href="/en-US/docs/Web/API/MediaTrackConstraints#ConstrainDouble"><code>ConstrainDouble</code></a> describing the acceptable or required value(s) for an
  audio track's latency, with values specified in seconds. In audio processing, latency is
  the time between the start of processing (when sound occurs in the real world, or is
  generated by a hardware device) and the data being made available to the next step in
  the audio input or output process. In most cases, low latency is desirable for
  performance and user experience purposes, but when power consumption is a concern, or
  delays are otherwise acceptable, higher latency might be acceptable.</p>

<p>If this property's value is a number, the user agent will attempt to obtain media whose
  latency tends to be as close as possible to this number given the capabilities of the
  hardware and the other constraints specified. Otherwise, the value of this
  <a href="/en-US/docs/Web/API/MediaTrackConstraints#ConstrainDouble"><code>ConstrainDouble</code></a> will guide the user agent in its efforts to provide an
  exact match to the required latency (if <code>exact</code> is specified or both
  <code>min</code> and <code>max</code> are provided and have the same value) or to a
  best-possible value.</p>

<div class="note">
  <p><strong>Note:</strong> Latency is always prone to some variation due to hardware usage demands, network
    constraints, and so forth, so even in an "exact" match, some variation should be
    expected.</p>
</div>

<h2 id="Example">Example</h2>

<p>See {{SectionOnPage("/en-US/docs/Web/API/Media_Streams_API/Constraints", "Example:
  Constraint exerciser")}} for an example.</p>

<h2 id="Specifications">Specifications</h2>

{{Specifications}}

<h2 id="Browser_compatibility">Browser compatibility</h2>

<p>{{Compat}}</p>

<h2 id="See_also">See also</h2>

<ul>
  <li><a href="/en-US/docs/Web/API/Media_Streams_API">Media Capture and Streams API</a>
  </li>
  <li><a href="/en-US/docs/Web/API/Media_Streams_API/Constraints">Capabilities,
      constraints, and settings</a></li>
  <li>{{domxref("MediaTrackConstraints")}}</li>
  <li>{{domxref("MediaDevices.getSupportedConstraints()")}}</li>
  <li>{{domxref("MediaTrackSupportedConstraints")}}</li>
  <li>{{domxref("MediaStreamTrack")}}</li>
</ul>