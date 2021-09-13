---
title: Signaling and video calling
slug: Web/API/WebRTC_API/Signaling_and_video_calling
tags:
  - API
  - Audio
  - Calling
  - Example
  - Guide
  - Media
  - Signaling
  - Tutorial
  - Video
  - WebRTC
---
<div>{{WebRTCSidebar}}</div>

<p><a href="/en-US/docs/Web/API/WebRTC_API">WebRTC</a> allows real-time, peer-to-peer, media exchange between two devices. A connection is established through a discovery and negotiation process called <strong>signaling</strong>. This tutorial will guide you through building a two-way video-call.</p>

<p><a href="/en-US/docs/Web/API/WebRTC_API">WebRTC</a> is a fully peer-to-peer technology for the real-time exchange of audio, video, and data, with one central caveat. A form of discovery and media format negotiation must take place, <a href="/en-US/docs/Web/API/WebRTC_API/Session_lifetime#establishing_a_connection">as discussed elsewhere</a>, in order for two devices on different networks to locate one another. This process is called <strong>signaling</strong> and involves both devices connecting to a third, mutually agreed-upon server. Through this third server, the two devices can locate one another, and exchange negotiation messages.</p>

<p>In this article, we will further enhance the <a class="external external-icon" href="https://webrtc-from-chat.glitch.me/" rel="noopener">WebSocket chat</a> first created as part of our WebSocket documentation (this article link is forthcoming; it isn't actually online yet) to support opening a two-way video call between users. You can <a href="https://webrtc-from-chat.glitch.me/">try out this example on Glitch</a>, and you can <a href="https://glitch.com/edit/#!/remix/webrtc-from-chat">remix the example</a> to experiment with it as well. You can also <a href="https://github.com/mdn/samples-server/tree/master/s/webrtc-from-chat">look at the full project</a> on GitHub.</p>

<div class="note">
<p><strong>Note:</strong> If you try out the example on Glitch, please note that any changes made to the code will immediately reset any connections. In addition, there is a short timeout period; the Glitch instance is for quick experiments and testing only.</p>
</div>

<h2 id="The_signaling_server">The signaling server</h2>

<p>Establishing a WebRTC connection between two devices requires the use of a <strong>signaling server</strong> to resolve how to connect them over the internet. A signaling server's job is to serve as an intermediary to let two peers find and establish a connection while minimizing exposure of potentially private information as much as possible. How do we create this server and how does the signaling process actually work?</p>

<p>First we need the signaling server itself. WebRTC doesn't specify a transport mechanism for the signaling information. You can use anything you like, from <a href="/en-US/docs/Web/API/WebSockets_API">WebSocket</a> to {{domxref("XMLHttpRequest")}} to carrier pigeons to exchange the signaling information between the two peers.</p>

<p>It's important to note that the server doesn't need to understand or interpret the signaling data content. Although it's {{Glossary("SDP")}}, even this doesn't matter so much: the content of the message going through the signaling server is, in effect, a black box. What does matter is when the {{Glossary("ICE")}} subsystem instructs you to send signaling data to the other peer, you do so, and the other peer knows how to receive this information and deliver it to its own ICE subsystem. All you have to do is channel the information back and forth. The contents don't matter at all to the signaling server.</p>

<h3 id="Readying_the_chat_server_for_signaling">Readying the chat server for signaling</h3>

<p>Our <a href="https://github.com/mdn/samples-server/tree/master/s/websocket-chat">chat server</a> uses the <a href="/en-US/docs/Web/API/WebSockets_API">WebSocket API</a> to send information as {{Glossary("JSON")}} strings between each client and the server. The server supports several message types to handle tasks, such as registering new users, setting usernames, and sending public chat messages.</p>

<p>To allow the server to support signaling and ICE negotiation, we need to update the code. We'll have to allow directing messages to one specific user instead of broadcasting to all connected users, and ensure unrecognized message types are passed through and delivered, without the server needing to know what they are. This lets us send signaling messages using this same server, instead of needing a separate server.</p>

<p>Let's take a look at changes we need to make to the chat server to support WebRTC signaling. This is in the file <code><a href="https://github.com/mdn/samples-server/tree/master/s/webrtc-from-chat/chatserver.js">chatserver.js</a></code>.</p>

<p>First up is the addition of the function <code>sendToOneUser()</code>. As the name suggests, this sends a stringified JSON message to a particular username.</p>

<pre class="brush: js">function sendToOneUser(target, msgString) {
  var isUnique = true;
  var i;

  for (i=0; i &lt; connectionArray.length; i++) {
    if (connectionArray[i].username === target) {
      connectionArray[i].send(msgString);
      break;
    }
  }
}</pre>

<p>This function iterates over the list of connected users until it finds one matching the specified username, then sends the message to that user. The parameter <code>msgString</code> is a stringified JSON object. We could have made it receive our original message object, but in this example it's more efficient this way. Since the message has already been stringified, we can send it with no further processing. Each entry in <code>connectionArray</code> is a {{domxref("WebSocket")}} object, so we can just call its {{domxref("WebSocket.send", "send()")}} method directly.</p>

<p>Our original chat demo didn't support sending messages to a specific user. The next task is to update the main WebSocket message handler to support doing so. This involves a change near the end of the <code>"connection"</code> message handler:</p>

<pre class="brush: js">if (sendToClients) {
  var msgString = JSON.stringify(msg);
  var i;

  if (msg.target &amp;&amp; msg.target.length !== 0) {
    sendToOneUser(msg.target, msgString);
  } else {
    for (i=0; i &lt; connectionArray.length; i++) {
      connectionArray[i].send(msgString);
    }
  }
}</pre>

<p>This code now looks at the pending message to see if it has a <code>target</code> property. If that property is present, it specifies the username of the client to which the message is to be sent, and we call <code>sendToOneUser()</code> to send the message to them. Otherwise, the message is broadcast to all users by iterating over the connection list, sending the message to each user.</p>

<p>As the existing code allows the sending of arbitrary message types, no additional changes are required. Our clients can now send messages of unknown types to any specific user, letting them send signaling messages back and forth as desired.</p>

<p>That's all we need to change on the server side of the equation. Now let's consider the signaling protocol we will implement.</p>

<h3 id="Designing_the_signaling_protocol">Designing the signaling protocol</h3>

<p>Now that we've built a mechanism for exchanging messages, we need a protocol defining how those messages will look. This can be done in a number of ways; what's demonstrated here is just one possible way to structure signaling messages.</p>

<p>This example's server uses stringified JSON objects to communicate with its clients. This means our signaling messages will be in JSON format, with contents which specify what kind of messages they are as well as any additional information needed in order to handle the messages properly.</p>

<h4 id="Exchanging_session_descriptions">Exchanging session descriptions</h4>

<p>When starting the signaling process, an <strong>offer</strong> is created by the user initiating the call. This offer includes a session description, in {{Glossary("SDP")}} format, and needs to be delivered to the receiving user, which we'll call the <strong>callee</strong>. The callee responds to the offer with an <strong>answer</strong> message, also containing an SDP description. Our signaling server will use WebSocket to transmit offer messages with the type <code>"video-offer"</code>, and answer messages with the type <code>"video-answer"</code>. These messages have the following fields:</p>

<dl>
 <dt><code>type</code></dt>
 <dd>The message type; either <code>"video-offer"</code> or <code>"video-answer"</code>.</dd>
 <dt><code>name</code></dt>
 <dd>The sender's username.</dd>
 <dt><code>target</code></dt>
 <dd>The username of the person to receive the description (if the caller is sending the message, this specifies the callee, and vice-versa).</dd>
 <dt><code>sdp</code></dt>
 <dd>The SDP (Session Description Protocol) string describing the local end of the connection from the perspective of the sender (or the remote end of the connection from the receiver's point of view).</dd>
</dl>

<p>At this point, the two participants know which <a href="/en-US/docs/Web/Media/Formats/WebRTC_codecs">codecs</a> and <a href="/en-US/docs/Web/Media/Formats/codecs_parameter">codec parameters</a> are to be used for this call. They still don't know how to transmit the media data itself though. This is where {{Glossary('ICE', 'Interactive Connectivity Establishment (ICE)')}} comes in.</p>

<h3 id="Exchanging_ICE_candidates">Exchanging ICE candidates</h3>

<p>Two peers need to exchange ICE candidates to negotiate the actual connection between them. Every ICE candidate describes a method that the sending peer is able to use to communicate. Each peer sends candidates in the order they're discovered, and keeps sending candidates until it runs out of suggestions, even if media has already started streaming.</p>

<p>An {{domxref("RTCPeerConnection.icecandidate_event", "icecandidate")}} event is sent to the {{domxref("RTCPeerConnection")}} to complete the process of adding a local description using <code>pc.setLocalDescription(offer)</code>.</p>

<p>Once the two peers agree upon a mutually-compatible candidate, that candidate's SDP is used by each peer to construct and open a connection, through which media then begins to flow. If they later agree on a better (usually higher-performance) candidate, the stream may change formats as needed.</p>

<p>Though not currently supported, a candidate received after media is already flowing could theoretically also be used to downgrade to a lower-bandwidth connection if needed.</p>

<p>Each ICE candidate is sent to the other peer by sending a JSON message of type <code>"new-ice-candidate"</code> over the signaling server to the remote peer. Each candidate message include these fields:</p>

<dl>
 <dt><code>type</code></dt>
 <dd>The message type: <code>"new-ice-candidate"</code>.</dd>
 <dt><code>target</code></dt>
 <dd>The username of the person with whom negotiation is underway; the server will direct the message to this user only.</dd>
 <dt><code>candidate</code></dt>
 <dd>The SDP candidate string, describing the proposed connection method. You typically don't need to look at the contents of this string. All your code needs to do is route it through to the remote peer using the signaling server.</dd>
</dl>

<p>Each ICE message suggests a communication protocol (TCP or UDP), IP address, port number, connection type (for example, whether the specified IP is the peer itself or a relay server), along with other information needed to link the two computers together. This includes NAT or other networking complexity.</p>

<div class="note">
<p><strong>Note:</strong> The important thing to note is this: the only thing your code is responsible for during ICE negotiation is accepting outgoing candidates from the ICE layer and sending them across the signaling connection to the other peer when your {{domxref("RTCPeerConnection.onicecandidate", "onicecandidate")}} handler is executed, and receiving ICE candidate messages from the signaling server (when the <code>"new-ice-candidate"</code> message is received) and delivering them to your ICE layer by calling {{domxref("RTCPeerConnection.addIceCandidate()")}}. That's it.</p>

<p>The contents of the SDP are irrelevant to you in essentially all cases. Avoid the temptation to try to make it more complicated than that until you really know what you're doing. That way lies madness.</p>
</div>

<p>All your signaling server now needs to do is send the messages it's asked to. Your workflow may also demand login/authentication functionality, but such details will vary.</p>

<div class="note">
<p><strong>Note:</strong> The {{domxref("RTCPeerConnection.onicecandidate", "onicecandidate")}} Event and {{domxref("RTCPeerConnection.createAnswer", "createAnswer()")}} Promise are both async calls which are handled separately. Be sure that your signaling does not change order! For example {{domxref("RTCPeerConnection.addIceCandidate", "addIceCandidate()")}} with the server's ice candidates must be called after setting the answer with {{domxref("RTCPeerConnection.setRemoteDescription", "setRemoteDescription()")}}.</p>
</div>

<h3 id="Signaling_transaction_flow">Signaling transaction flow</h3>

<p>The signaling process involves this exchange of messages between two peers using an intermediary, the signaling server. The exact process will vary, of course, but in general there are a few key points at which signaling messages get handled:</p>

<p>The signaling process involves this exchange of messages among a number of points:</p>

<ul>
 <li>Each user's client running within a web browser</li>
 <li>Each user's web browser</li>
 <li>The signaling server</li>
 <li>The web server hosting the chat service</li>
</ul>

<p>Imagine that Naomi and Priya are engaged in a discussion using the chat software, and Naomi decides to open a video call between the two. Here's the expected sequence of events:</p>

<p><a href="/en-US/docs/Web/API/WebRTC_API/Signaling_and_video_calling/webrtc_-_signaling_diagram.svg"><img alt="Diagram of the signaling process" src="webrtc_-_signaling_diagram.svg"></a></p>

<p>We'll see this detailed more over the course of this article.</p>

<h3 id="ICE_candidate_exchange_process">ICE candidate exchange process</h3>

<p>When each peer's ICE layer begins to send candidates, it enters into an exchange among the various points in the chain that looks like this:</p>

<p><a href="webrtc_-_ice_candidate_exchange.svg"><img alt="Diagram of ICE candidate exchange process" src="webrtc_-_ice_candidate_exchange.svg"></a></p>

<p>Each side sends candidates to the other as it receives them from their local ICE layer; there is no taking turns or batching of candidates. As soon as the two peers agree upon one candidate that they can both use to exchange the media, media begins to flow. Each peer continues to send candidates until it runs out of options, even after the media has already begun to flow. This is done in hopes of identifying even better options than the one initially selected.</p>

<p>If conditions change (for example, the network connection deteriorates), one or both peers might suggest switching to a lower-bandwidth media resolution, or to an alternative codec. That triggers a new exchange of candidates, after which another media format and/or codec change may take place. In the guide <a href="/en-US/docs/Web/Media/Formats/WebRTC_codecs">Codecs used by WebRTC</a> you can learn more about the codecs which WebRTC requires browsers to support, which additional codecs are supported by which browsers, and how to choose the best codecs to use.</p>

<p>Optionally, see {{RFC(8445, "Interactive Connectivity Establishment")}}, <a href="https://datatracker.ietf.org/doc/html/rfc5245#section-2.3">section 2.3 ("Negotiating Candidate Pairs and Concluding ICE")</a> if you want greater understanding of how this process is completed inside the ICE layer. You should note that candidates are exchanged and media starts to flow as soon as the ICE layer is satisfied. This is all taken care of behind the scenes. Our role is to send the candidates, back and forth, through the signaling server.</p>

<h2 id="The_client_application">The client application</h2>

<p>The core to any signaling process is its message handling. It's not necessary to use WebSockets for signaling, but it is a common solution. You should, of course, select a mechanism for exchanging signaling information that is appropriate for your application.</p>

<p>Let's update the chat client to support video calling.</p>

<h3 id="Updating_the_HTML">Updating the HTML</h3>

<p>The HTML for our client needs a location for video to be presented. This requires video elements, and a button to hang up the call:</p>

<pre class="brush: html">&lt;div class="flexChild" id="camera-container"&gt;
  &lt;div class="camera-box"&gt;
    &lt;video id="received_video" autoplay&gt;&lt;/video&gt;
    &lt;video id="local_video" autoplay muted&gt;&lt;/video&gt;
    &lt;button id="hangup-button" onclick="hangUpCall();" disabled&gt;
      Hang Up
    &lt;/button&gt;
  &lt;/div&gt;
&lt;/div&gt;</pre>

<p>The page structure defined here is using {{HTMLElement("div")}} elements, giving us full control over the page layout by enabling the use of CSS. We'll skip layout detail in this guide, but <a href="https://github.com/mdn/samples-server/tree/master/s/webrtc-from-chat/chat.css">take a look at the CSS</a> on Github to see how we handled it. Take note of the two {{HTMLElement("video")}} elements, one for your self-view, one for the connection, and the {{HTMLElement("button")}} element.</p>

<p>The <code>&lt;video&gt;</code> element with the <code>id</code> "<code>received_video</code>" will present video received from the connected user. We specify the <code>autoplay</code> attribute, ensuring once the video starts arriving, it immediately plays. This removes any need to explicitly handle playback in our code. The "<code>local_video</code>" <code>&lt;video&gt;</code> element presents a preview of the user's camera; specifiying the <code>muted</code> attribute, as we don't need to hear local audio in this preview panel.</p>

<p>Finally, the "<code>hangup-button</code>" {{HTMLElement("button")}}, to disconnect from a call, is defined and configured to start disabled (setting this as our default for when no call is connected) and apply the function <code>hangUpCall()</code> on click. This function's role is to close the call, and send a signalling server notification to the other peer, requesting it also close.</p>

<h3 id="The_JavaScript_code">The JavaScript code</h3>

<p>We'll divide this code into functional areas to more easily describe how it works. The main body of this code is found in the <code>connect()</code> function: it opens up a {{domxref("WebSocket")}} server on port 6503, and establishes a handler to receive messages in JSON object format. This code generally handles text chat messages as it did previously.</p>

<h4 id="Sending_messages_to_the_signaling_server">Sending messages to the signaling server</h4>

<p>Throughout our code, we call <code>sendToServer()</code> in order to send messages to the signaling server. This function uses the <a href="/en-US/docs/Web/API/WebSockets_API">WebSocket</a> connection to do its work:</p>

<pre class="brush: js">function sendToServer(msg) {
  var msgJSON = JSON.stringify(msg);

  connection.send(msgJSON);
}</pre>

<p>The message object passed into this function is converted into a JSON string by calling {{jsxref("JSON.stringify()")}}, then we call the WebSocket connection's {{domxref("WebSocket.send", "send()")}} function to transmit the message to the server.</p>

<h4 id="UI_to_start_a_call">UI to start a call</h4>

<p>The code which handles the <code>"userlist"</code> message calls <code>handleUserlistMsg()</code>. Here we set up the handler for each connected user in the user list displayed to the left of the chat panel. This function receives a message object whose <code>users</code> property is an array of strings specifying the user names of every connected user.</p>

<pre class="brush: js">function handleUserlistMsg(msg) {
  var i;
  var listElem = document.querySelector(".userlistbox");

  while (listElem.firstChild) {
    listElem.removeChild(listElem.firstChild);
  }

  msg.users.forEach(function(username) {
    var item = document.createElement("li");
    item.appendChild(document.createTextNode(username));
    item.addEventListener("click", invite, false);

    listElem.appendChild(item);
  });
}</pre>

<p>After getting a reference to the {{HTMLElement("ul")}} which contains the list of user names into the variable <code>listElem</code>, we empty the list by removing each of its child elements.</p>

<div class="note">
<p><strong>Note:</strong> Obviously, it would be more efficient to update the list by adding and removing individual users instead of rebuilding the whole list every time it changes, but this is good enough for the purposes of this example.</p>
</div>

<p>Then we iterate over the array of user names using {{jsxref("Array.forEach", "forEach()")}}. For each name, we create a new {{HTMLElement("li")}} element, then create a new text node containing the user name using {{domxref("Document.createTextNode", "createTextNode()")}}. That text node is added as a child of the <code>&lt;li&gt;</code> element. Next, we set a handler for the {{event("click")}} event on the list item, that clicking on a user name calls our <code>invite()</code> method, which we'll look at in the next section.</p>

<p>Finally, we append the new item to the <code>&lt;ul&gt;</code> that contains all of the user names.</p>

<h4 id="Starting_a_call">Starting a call</h4>

<p>When the user clicks on a username they want to call, the <code>invite()</code> function is invoked as the event handler for that {{event("click")}} event:</p>

<pre class="brush: js">var mediaConstraints = {
  audio: true, // We want an audio track
  video: true // ...and we want a video track
};

function invite(evt) {
  if (myPeerConnection) {
    alert("You can't start a call because you already have one open!");
  } else {
    var clickedUsername = evt.target.textContent;

    if (clickedUsername === myUsername) {
      alert("I'm afraid I can't let you talk to yourself. That would be weird.");
      return;
    }

    targetUsername = clickedUsername;
    createPeerConnection();

    navigator.mediaDevices.getUserMedia(mediaConstraints)
    .then(function(localStream) {
      document.getElementById("local_video").srcObject = localStream;
      localStream.getTracks().forEach(track =&gt; myPeerConnection.addTrack(track, localStream));
    })
    .catch(handleGetUserMediaError);
  }
}</pre>

<p>This begins with a basic sanity check: is the user already connected? If there's already a {{domxref("RTCPeerConnection")}}, they obviously can't make a call. Then the name of the user that was clicked upon is obtained from the event target's {{domxref("Node.textContent", "textContent")}} property, and we check to be sure that it's not the same user that's trying to start the call.</p>

<p>Then we copy the name of the user we're calling into the variable <code>targetUsername</code> and call <code>createPeerConnection()</code>, a function which will create and do basic configuration of the {{domxref("RTCPeerConnection")}}.</p>

<p>Once the <code>RTCPeerConnection</code> has been created, we request access to the user's camera and microphone by calling {{domxref("MediaDevices.getUserMedia()")}}, which is exposed to us through the {{domxref("MediaDevices.getUserMedia")}} property. When this succeeds, fulfilling the returned promise, our <code>then</code> handler is executed. It receives, as input, a {{domxref("MediaStream")}} object representing the stream with audio from the user's microphone and video from their webcam.</p>

<div class="note">
<p><strong>Note:</strong> We could restrict the set of permitted media inputs to a specific device or set of devices by calling {{domxref("MediaDevices.enumerateDevices", "navigator.mediaDevices.enumerateDevices()")}} to get a list of devices, filtering the resulting list based on our desired criteria, then using the selected devices' {{domxref("MediaTrackConstraints.deviceId", "deviceId")}} values in the <code>deviceId</code> field of the <code>mediaConstraints</code> object passed into <code>getUserMedia()</code>. In practice, this is rarely if ever necessary, since most of that work is done for you by <code>getUserMedia()</code>.</p>
</div>

<p>We attach the incoming stream to the local preview {{HTMLElement("video")}} element by setting the element's {{domxref("HTMLMediaElement.srcObject", "srcObject")}} property. Since the element is configured to automatically play incoming video, the stream begins playing in our local preview box.</p>

<p>We then iterate over the tracks in the stream, calling {{domxref("RTCPeerConnection.addTrack", "addTrack()")}} to add each track to the <code>RTCPeerConnection</code>. Even though the connection is not fully established yet, you can begin sending data when you feel it's appropriate to do so. Media received before the ICE negotiation is completed may be used to help ICE decide upon the best connectivity approach to take, thus aiding in the negotiation process.</p>

<p>Note that for native apps, such as a phone application, you should not begin sending until the connection has been accepted at both ends, at a minimum, to avoid inadvertently sending video and/or audio data when the user isn't prepared for it.</p>

<p>As soon as media is attached to the <code>RTCPeerConnection</code>, a {{event("negotiationneeded")}} event is triggered at the connection, so that ICE negotiation can be started.</p>

<p>If an error occurs while trying to get the local media stream, our catch clause calls <code>handleGetUserMediaError()</code>, which displays an appropriate error to the user as required.</p>

<h4 id="Handling_getUserMedia_errors">Handling getUserMedia() errors</h4>

<p>If the promise returned by <code>getUserMedia()</code> concludes in a failure, our <code>handleGetUserMediaError()</code> function performs.</p>

<pre class="brush: js">function handleGetUserMediaError(e) {
  switch(e.name) {
    case "NotFoundError":
      alert("Unable to open your call because no camera and/or microphone" +
            "were found.");
      break;
    case "SecurityError":
    case "PermissionDeniedError":
      // Do nothing; this is the same as the user canceling the call.
      break;
    default:
      alert("Error opening your camera and/or microphone: " + e.message);
      break;
  }

  closeVideoCall();
}</pre>

<p>An error message is displayed in all cases but one. In this example, we ignore <code>"SecurityError"</code> and <code>"PermissionDeniedError"</code> results, treating refusal to grant permission to use the media hardware the same as the user canceling the call.</p>

<p>Regardless of why an attempt to get the stream fails, we call our <code>closeVideoCall()</code> function to shut down the {{domxref("RTCPeerConnection")}}, and release any resources already allocated by the process of attempting the call. This code is designed to safely handle partially-started calls.</p>

<h4 id="Creating_the_peer_connection">Creating the peer connection</h4>

<p>The <code>createPeerConnection()</code> function is used by both the caller and the callee to construct their {{domxref("RTCPeerConnection")}} objects, their respective ends of the WebRTC connection. It's invoked by <code>invite()</code> when the caller tries to start a call, and by <code>handleVideoOfferMsg()</code> when the callee receives an offer message from the caller.</p>

<pre class="brush: js">function createPeerConnection() {
  myPeerConnection = new RTCPeerConnection({
      iceServers: [     // Information about ICE servers - Use your own!
        {
          urls: "stun:stun.stunprotocol.org"
        }
      ]
  });

  myPeerConnection.onicecandidate = handleICECandidateEvent;
  myPeerConnection.ontrack = handleTrackEvent;
  myPeerConnection.onnegotiationneeded = handleNegotiationNeededEvent;
  myPeerConnection.onremovetrack = handleRemoveTrackEvent;
  myPeerConnection.oniceconnectionstatechange = handleICEConnectionStateChangeEvent;
  myPeerConnection.onicegatheringstatechange = handleICEGatheringStateChangeEvent;
  myPeerConnection.onsignalingstatechange = handleSignalingStateChangeEvent;
}
</pre>

<p>When using the {{domxref("RTCPeerConnection.RTCPeerConnection", "RTCPeerConnection()")}} constructor, we will specify an {{domxref("RTCConfiguration")}}-compliant object providing configuration parameters for the connection. We use only one of these in this example: <code>iceServers</code>. This is an array of objects describing STUN and/or TURN servers for the {{Glossary("ICE")}} layer to use when attempting to establish a route between the caller and the callee. These servers are used to determine the best route and protocols to use when communicating between the peers, even if they're behind a firewall or using {{Glossary("NAT")}}.</p>

<div class="note">
<p><strong>Note:</strong> You should always use STUN/TURN servers which you own, or which you have specific authorization to use. This example is using a known public STUN server but abusing these is bad form.</p>
</div>

<p>Each object in <code>iceServers</code> contains at least a <code>urls</code> field providing URLs at which the specified server can be reached. It may also provide <code>username</code> and <code>credential</code> values to allow authentication to take place, if needed.</p>

<p>After creating the {{domxref("RTCPeerConnection")}}, we set up handlers for the events that matter to us.</p>

<p>The first three of these event handlers are required; you have to handle them to do anything involving streamed media with WebRTC. The rest aren't strictly required but can be useful, and we'll explore them. There are a few other events available that we're not using in this example, as well. Here's a summary of each of the event handlers we will be implementing:</p>

<dl>
 <dt>{{domxref("RTCPeerConnection.onicecandidate")}}</dt>
 <dd>The local ICE layer calls your {{event("icecandidate")}} event handler, when it needs you to transmit an ICE candidate to the other peer, through your signaling server. See {{anch("Sending ICE candidates")}} for more information and to see the code for this example.</dd>
 <dt>{{domxref("RTCPeerConnection.ontrack")}}</dt>
 <dd>This handler for the {{event("track")}} event is called by the local WebRTC layer when a track is added to the connection. This lets you connect the incoming media to an element to display it, for example. See {{anch("Receiving new streams")}} for details.</dd>
 <dt>{{domxref("RTCPeerConnection.onnegotiationneeded")}}</dt>
 <dd>This function is called whenever the WebRTC infrastructure needs you to start the session negotiation process anew. Its job is to create and send an offer, to the callee, asking it to connect with us. See {{anch("Starting negotiation")}} to see how we handle this.</dd>
 <dt>{{domxref("RTCPeerConnection.onremovetrack")}}</dt>
 <dd>This counterpart to <code>ontrack</code> is called to handle the {{event("removetrack")}} event; it's sent to the <code>RTCPeerConnection</code> when the remote peer removes a track from the media being sent. See {{anch("Handling the removal of tracks")}}.</dd>
 <dt>{{domxref("RTCPeerConnection.oniceconnectionstatechange")}}</dt>
 <dd>The {{event("iceconnectionstatechange")}} event is sent by the ICE layer to let you know about changes to the state of the ICE connection. This can help you know when the connection has failed, or been lost. We'll look at the code for this example in {{anch("ICE connection state")}} below.</dd>
 <dt>{{domxref("RTCPeerConnection.onicegatheringstatechange")}}</dt>
 <dd>The ICE layer sends you the {{event("icegatheringstatechange")}} event, when the ICE agent's process of collecting candidates shifts, from one state to another (such as starting to gather candidates or completing negotiation). See {{anch("ICE gathering state")}} below.</dd>
 <dt>{{domxref("RTCPeerConnection.onsignalingstatechange")}}</dt>
 <dd>The WebRTC infrastructure sends you the {{event("signalingstatechange")}} message when the state of the signaling process changes (or if the connection to the signaling server changes). See {{anch("Signaling state")}} to see our code.</dd>
</dl>

<h4 id="Starting_negotiation">Starting negotiation</h4>

<p>Once the caller has created its  {{domxref("RTCPeerConnection")}}, created a media stream, and added its tracks to the connection as shown in {{anch("Starting a call")}}, the browser will deliver a {{event("negotiationneeded")}} event to the {{domxref("RTCPeerConnection")}} to indicate that it's ready to begin negotiation with the other peer. Here's our code for handling the {{event("negotiationneeded")}} event:</p>

<pre class="brush: js">function handleNegotiationNeededEvent() {
  myPeerConnection.createOffer().then(function(offer) {
    return myPeerConnection.setLocalDescription(offer);
  })
  .then(function() {
    sendToServer({
      name: myUsername,
      target: targetUsername,
      type: "video-offer",
      sdp: myPeerConnection.localDescription
    });
  })
  .catch(reportError);
}</pre>

<p>To start the negotiation process, we need to create and send an SDP offer to the peer we want to connect to. This offer includes a list of supported configurations for the connection, including information about the media stream we've added to the connection locally (that is, the video we want to send to the other end of the call), and any ICE candidates gathered by the ICE layer already. We create this offer by calling {{domxref("RTCPeerConnection.createOffer", "myPeerConnection.createOffer()")}}.</p>

<p>When <code>createOffer()</code> succeeds (fulfilling the promise), we pass the created offer information into {{domxref("RTCPeerConnection.setLocalDescription", "myPeerConnection.setLocalDescription()")}}, which configures the connection and media configuration state for the caller's end of the connection.</p>

<div class="note">
<p><strong>Note:</strong> Technically speaking, the string returned by <code>createOffer()</code> is an {{RFC(3264)}} offer.</p>
</div>

<p>We know the description is valid, and has been set, when the promise returned by <code>setLocalDescription()</code> is fulfilled. This is when we send our offer to the other peer by creating a new <code>"video-offer"</code> message containing the local description (now the same as the offer), then sending it through our signaling server to the callee. The offer has the following members:</p>

<dl>
 <dt><code>type</code></dt>
 <dd>The message type: <code>"video-offer"</code>.</dd>
 <dt><code>name</code></dt>
 <dd>The caller's username.</dd>
 <dt><code>target</code></dt>
 <dd>The name of the user we wish to call.</dd>
 <dt><code>sdp</code></dt>
 <dd>The SDP string describing the offer.</dd>
</dl>

<p>If an error occurs, either in the initial <code>createOffer()</code> or in any of the fulfillment handlers that follow, an error is reported by invoking our <code>reportError()</code> function.</p>

<p>Once <code>setLocalDescription()</code>'s fulfillment handler has run, the ICE agent begins sending {{event("icecandidate")}} events to the {{domxref("RTCPeerConnection")}}, one for each potential configuration it discovers. Our handler for the <code>icecandidate</code> event is responsible for transmitting the candidates to the other peer.</p>

<h4 id="Session_negotiation">Session negotiation</h4>

<p>Now that we've started negotiation with the other peer and have transmitted an offer, let's look at what happens on the callee's side of the connection for a while. The callee receives the offer and calls <code>handleVideoOfferMsg()</code> function to process it. Let's see how the callee handles the <code>"video-offer"</code> message.</p>

<h5 id="Handling_the_invitation">Handling the invitation</h5>

<p>When the offer arrives, the callee's <code>handleVideoOfferMsg()</code> function is called with the <code>"video-offer"</code> message that was received. This function needs to do two things. First, it needs to create its own {{domxref("RTCPeerConnection")}} and add the tracks containing the audio and video from its microphone and webcam to that. Second, it needs to process the received offer, constructing and sending its answer.</p>

<pre class="brush: js">function handleVideoOfferMsg(msg) {
  var localStream = null;

  targetUsername = msg.name;
  createPeerConnection();

  var desc = new RTCSessionDescription(msg.sdp);

  myPeerConnection.setRemoteDescription(desc).then(function () {
    return navigator.mediaDevices.getUserMedia(mediaConstraints);
  })
  .then(function(stream) {
    localStream = stream;
    document.getElementById("local_video").srcObject = localStream;

    localStream.getTracks().forEach(track =&gt; myPeerConnection.addTrack(track, localStream));
  })
  .then(function() {
    return myPeerConnection.createAnswer();
  })
  .then(function(answer) {
    return myPeerConnection.setLocalDescription(answer);
  })
  .then(function() {
    var msg = {
      name: myUsername,
      target: targetUsername,
      type: "video-answer",
      sdp: myPeerConnection.localDescription
    };

    sendToServer(msg);
  })
  .catch(handleGetUserMediaError);
}</pre>

<p class="brush: js">This code is very similar to what we did in the <code>invite()</code> function back in {{anch("Starting a call")}}. It starts by creating and configuring an {{domxref("RTCPeerConnection")}} using our <code>createPeerConnection()</code> function. Then it takes the SDP offer from the received <code>"video-offer"</code> message and uses it to create a new {{domxref("RTCSessionDescription")}} object representing the caller's session description.</p>

<p class="brush: js">That session description is then passed into {{domxref("RTCPeerConnection.setRemoteDescription", "myPeerConnection.setRemoteDescription()")}}. This establishes the received offer as the description of the remote (caller's) end of the connection. If this is successful, the promise fulfillment handler (in the <code>then()</code> clause) starts the process of getting access to the callee's camera and microphone using {{domxref("MediaDevices.getUserMedia", "getUserMedia()")}}, adding the tracks to the connection, and so forth, as we saw previously in <code>invite()</code>.</p>

<p class="brush: js">Once the answer has been created using {{domxref("RTCPeerConnection.createAnswer", "myPeerConnection.createAnswer()")}}, the description of the local end of the connection is set to the answer's SDP by calling {{domxref("RTCPeerConnection.setLocalDescription", "myPeerConnection.setLocalDescription()")}}, then the answer is transmitted through the signaling server to the caller to let them know what the answer is.</p>

<p>Any errors are caught and passed to <code>handleGetUserMediaError()</code>, described in {{anch("Handling getUserMedia() errors")}}.</p>

<div class="note">
<p><strong>Note:</strong> As is the case with the caller, once the <code>setLocalDescription()</code> fulfillment handler has run, the browser begins firing {{event("icecandidate")}} events that the callee must handle, one for each candidate that needs to be transmitted to the remote peer.</p>
</div>

<h5 id="Sending_ICE_candidates">Sending ICE candidates</h5>

<p>The ICE negotiation process involves each peer sending candidates to the other, repeatedly, until it runs out of potential ways it can support the <code>RTCPeerConnection</code>'s media transport needs. Since ICE doesn't know about your signaling server, your code handles transmission of each candidate in your handler for the {{event("icecandidate")}} event.</p>

<p>Your {{domxref("RTCPeerConnection.onicecandidate", "onicecandidate")}} handler receives an event whose <code>candidate</code> property is the SDP describing the candidate (or is <code>null</code> to indicate that the ICE layer has run out of potential configurations to suggest). The contents of <code>candidate</code> are what you need to transmit using your signaling server. Here's our example's implementation:</p>

<pre class="brush: js">function handleICECandidateEvent(event) {
  if (event.candidate) {
    sendToServer({
      type: "new-ice-candidate",
      target: targetUsername,
      candidate: event.candidate
    });
  }
}</pre>

<p>This builds an object containing the candidate, then sends it to the other peer using the <code>sendToServer()</code> function previously described in {{anch("Sending messages to the signaling server")}}. The message's properties are:</p>

<dl>
 <dt><code>type</code></dt>
 <dd>The message type: <code>"new-ice-candidate"</code>.</dd>
 <dt><code>target</code></dt>
 <dd>The username the ICE candidate needs to be delivered to. This lets the signaling server route the message.</dd>
 <dt><code>candidate</code></dt>
 <dd>The SDP representing the candidate the ICE layer wants to transmit to the other peer.</dd>
</dl>

<p>The format of this message (as is the case with everything you do when handling signaling) is entirely up to you, depending on your needs; you can provide other information as required.</p>

<div class="note">
<p><strong>Note:</strong> It's important to keep in mind that the {{event("icecandidate")}} event is <strong>not</strong> sent when ICE candidates arrive from the other end of the call. Instead, they're sent by your own end of the call so that you can take on the job of transmitting the data over whatever channel you choose. This can be confusing when you're new to WebRTC.</p>
</div>

<h5 id="Receiving_ICE_candidates">Receiving ICE candidates</h5>

<p>The signaling server delivers each ICE candidate to the destination peer using whatever method it chooses; in our example this is as JSON objects, with a <code>type</code> property containing the string <code>"new-ice-candidate"</code>. Our <code>handleNewICECandidateMsg()</code> function is called by our main <a href="/en-US/docs/Web/API/WebSockets_API">WebSocket</a> incoming message code to handle these messages:</p>

<pre class="brush: js">function handleNewICECandidateMsg(msg) {
  var candidate = new RTCIceCandidate(msg.candidate);

  myPeerConnection.addIceCandidate(candidate)
    .catch(reportError);
}</pre>

<p>This function constructs an {{domxref("RTCIceCandidate")}} object by passing the received SDP into its constructor, then delivers the candidate to the ICE layer by passing it into {{domxref("RTCPeerConnection.addIceCandidate", "myPeerConnection.addIceCandidate()")}}. This hands the fresh ICE candidate to the local ICE layer, and finally, our role in the process of handling this candidate is complete.</p>

<p>Each peer sends to the other peer a candidate for each possible transport configuration that it believes might be viable for the media being exchanged. At some point, the two peers agree that a given candidate is a good choice and they open the connection and begin to share media. It's important to note, however, that ICE negotiation does <em>not</em> stop once media is flowing. Instead, candidates may still keep being exchanged after the conversation has begun, either while trying to find a better connection method, or because they were already in transport when the peers successfully established their connection.</p>

<p>In addition, if something happens to cause a change in the streaming scenario, negotiation will begin again, with the {{event("negotiationneeded")}} event being sent to the {{domxref("RTCPeerConnection")}}, and the entire process starts again as described before. This can happen in a variety of situations, including:</p>

<ul>
 <li>Changes in the network status, such as a bandwidth change, transitioning from WiFi to cellular connectivity, or the like.</li>
 <li>Switching between the front and rear cameras on a phone.</li>
 <li>A change to the configuration of the stream, such as its resolution or frame rate.</li>
</ul>

<h5 id="Receiving_new_streams">Receiving new streams</h5>

<p>When new tracks are added to the <code>RTCPeerConnection</code>— either by calling its {{domxref("RTCPeerConnection.addTrack", "addTrack()")}} method or because of renegotiation of the stream's format—a {{event("track")}} event is set to the <code>RTCPeerConnection</code> for each track added to the connection. Making use of newly added media requires implementing a handler for the <code>track</code> event. A common need is to attach the incoming media to an appropriate HTML element. In our example, we add the track's stream to the {{HTMLElement("video")}} element that displays the incoming video:</p>

<pre class="brush: js">function handleTrackEvent(event) {
  document.getElementById("received_video").srcObject = event.streams[0];
  document.getElementById("hangup-button").disabled = false;
}</pre>

<p>The incoming stream is attached to the <code>"received_video"</code> {{HTMLElement("video")}} element, and the "Hang Up" {{HTMLElement("button")}} element is enabled so the user can hang up the call.</p>

<p>Once this code has completed, finally the video being sent by the other peer is displayed in the local browser window!</p>

<h5 id="Handling_the_removal_of_tracks">Handling the removal of tracks</h5>

<p>Your code receives a {{event("removetrack")}} event when the remote peer removes a track from the connection by calling {{domxref("RTCPeerConnection.removeTrack()")}}. Our handler for <code>"removetrack"</code> is:</p>

<pre class="brush: js">function handleRemoveTrackEvent(event) {
  var stream = document.getElementById("received_video").srcObject;
  var trackList = stream.getTracks();

  if (trackList.length == 0) {
    closeVideoCall();
  }
}</pre>

<p>This code fetches the incoming video {{domxref("MediaStream")}} from the <code>"received_video"</code> {{HTMLElement("video")}} element's {{htmlattrxref("srcObject", "video")}} attribute, then calls the stream's {{domxref("MediaStream.getTracks", "getTracks()")}} method to get an array of the stream's tracks.</p>

<p>If the array's length is zero, meaning there are no tracks left in the stream, we end the call by calling <code>closeVideoCall()</code>. This cleanly restores our app to a state in which it's ready to start or receive another call. See {{anch("Ending the call")}} to learn how <code>closeVideoCall()</code> works.</p>

<h4 id="Ending_the_call">Ending the call</h4>

<p>There are many reasons why calls may end. A call might have completed, with one or both sides having hung up. Perhaps a network failure has occurred, or one user might have quit their browser, or had a system crash. In any case, all good things must come to an end.</p>

<h5 id="Hanging_up">Hanging up</h5>

<p>When the user clicks the "Hang Up" button to end the call, the <code>hangUpCall()</code> function is called:</p>

<pre class="brush: js">function hangUpCall() {
  closeVideoCall();
  sendToServer({
    name: myUsername,
    target: targetUsername,
    type: "hang-up"
  });
}</pre>

<p><code>hangUpCall()</code> executes <code>closeVideoCall()</code> to shut down and reset the connection and release resources. It then builds a <code>"hang-up"</code> message and sends it to the other end of the call to tell the other peer to neatly shut itself down.</p>

<h5 id="Ending_the_call_2">Ending the call</h5>

<p>The <code>closeVideoCall()</code> function, shown below, is responsible for stopping the streams, cleaning up, and disposing of the {{domxref("RTCPeerConnection")}} object:</p>

<pre class="brush: js">function closeVideoCall() {
  var remoteVideo = document.getElementById("received_video");
  var localVideo = document.getElementById("local_video");

  if (myPeerConnection) {
    myPeerConnection.ontrack = null;
    myPeerConnection.onremovetrack = null;
    myPeerConnection.onremovestream = null;
    myPeerConnection.onicecandidate = null;
    myPeerConnection.oniceconnectionstatechange = null;
    myPeerConnection.onsignalingstatechange = null;
    myPeerConnection.onicegatheringstatechange = null;
    myPeerConnection.onnegotiationneeded = null;

    if (remoteVideo.srcObject) {
      remoteVideo.srcObject.getTracks().forEach(track =&gt; track.stop());
    }

    if (localVideo.srcObject) {
      localVideo.srcObject.getTracks().forEach(track =&gt; track.stop());
    }

    myPeerConnection.close();
    myPeerConnection = null;
  }

  remoteVideo.removeAttribute("src");
  remoteVideo.removeAttribute("srcObject");
  localVideo.removeAttribute("src");
  remoteVideo.removeAttribute("srcObject");

  document.getElementById("hangup-button").disabled = true;
  targetUsername = null;
}
</pre>

<p>After pulling references to the two {{HTMLElement("video")}} elements, we check if a WebRTC connection exists; if it does, we proceed to disconnect and close the call:</p>

<ol>
 <li>All of the event handlers are removed. This prevents stray event handlers from being triggered while the connection is in the process of closing, potentially causing errors.</li>
 <li>For both remote and local video streams, we iterate over each track, calling the {{domxref("MediaStreamTrack.stop()")}} method to close each one.</li>
 <li>Close the {{domxref("RTCPeerConnection")}} by calling {{domxref("RTCPeerConnection.close", "myPeerConnection.close()")}}.</li>
 <li>Set <code>myPeerConnection</code> to <code>null</code>, ensuring our code learns there's no ongoing call; this is useful when the user clicks a name in the user list.</li>
</ol>

<p>Then for both the incoming and outgoing {{HTMLElement("video")}} elements, we remove their {{htmlattrxref("src", "video")}} and {{htmlattrxref("srcObject", "video")}} attributes using their {{domxref("Element.removeAttribute", "removeAttribute()")}} methods. This completes the disassociation of the streams from the video elements.</p>

<p>Finally, we set the {{domxref("HTMLElement.disabled", "disabled")}} property to <code>true</code> on the "Hang Up" button, making it unclickable while there is no call underway; then we set <code>targetUsername</code> to <code>null</code> since we're no longer talking to anyone. This allows the user to call another user, or to receive an incoming call.</p>

<h4 id="Dealing_with_state_changes">Dealing with state changes</h4>

<p>There are a number of additional events you can set listeners for which notifying your code of a variety of state changes. We use three of them: {{event("iceconnectionstatechange")}}, {{event("icegatheringstatechange")}}, and {{event("signalingstatechange")}}.</p>

<h5 id="ICE_connection_state">ICE connection state</h5>

<p>{{event("iceconnectionstatechange")}} events are sent to the {{domxref("RTCPeerConnection")}} by the ICE layer when the connection state changes (such as when the call is terminated from the other end).</p>

<pre class="brush: js">function handleICEConnectionStateChangeEvent(event) {
  switch(myPeerConnection.iceConnectionState) {
    case "closed":
    case "failed":
      closeVideoCall();
      break;
  }
}</pre>

<p>Here, we apply our <code>closeVideoCall()</code> function when the ICE connection state changes to <code>"closed"</code> or <code>"failed"</code>. This handles shutting down our end of the connection so that we're ready start or accept a call once again.</p>

<div class="notecard note">
<p><strong>Note:</strong> We don't watch the <code>disconnected</code> signaling state here as it can indicate temporary issues and may go back to a <code>connected</code> state after some time. Watching it would close the video call on any temporary network issue.</p>
</div>

<h5 id="ICE_signaling_state">ICE signaling state</h5>

<p>Similarly, we watch for {{event("signalingstatechange")}} events. If the signaling state changes to <code>closed</code>, we likewise close the call out.</p>

<pre class="brush: js">function handleSignalingStateChangeEvent(event) {
  switch(myPeerConnection.signalingState) {
    case "closed":
      closeVideoCall();
      break;
  }
};</pre>

<div class="notecard note">
<p><strong>Note:</strong> The <code>closed</code> signaling state has been deprecated in favor of the <code>closed</code> {{domxref("RTCPeerConnection.iceConnectionState", "iceConnectionState")}}. We are watching for it here to add a bit of backward compatibility.</p>
</div>

<h5 id="ICE_gathering_state">ICE gathering state</h5>

<p>{{event("icegatheringstatechange")}} events are used to let you know when the ICE candidate gathering process state changes. Our example doesn't use this for anything, but it can be useful to watch these events for debugging purposes, as well as to detect when candidate collection has finished.</p>

<pre class="brush: js">function handleICEGatheringStateChangeEvent(event) {
  // Our sample just logs information to console here,
  // but you can do whatever you need.
}
</pre>

<h2 id="Next_steps">Next steps</h2>

<p>You can now <a href="https://webrtc-from-chat.glitch.me/">try out this example on Glitch</a> to see it in action. Open the Web console on both devices and look at the logged output—although you don't see it in the code as shown above, the code on the server (and on <a href="https://github.com/mdn/samples-server/tree/master/s/webrtc-from-chat">GitHub</a>) has a lot of console output so you can see the signaling and connection processes at work.</p>

<p>Another obvious improvement would be to add a "ringing" feature, so that instead of just asking the user for permission to use the camera and microphone, a "User X is calling. Would you like to answer?" prompt appears first.</p>

<h2 id="See_also">See also</h2>

<ul>
 <li><a href="/en-US/docs/Web/API/WebRTC_API">WebRTC API</a></li>
 <li><a href="/en-US/docs/Web/Media">Web media technologies</a></li>
 <li><a href="/en-US/docs/Web/Media/Formats">Guide to media types and formats on the web</a></li>
 <li><a href="/en-US/docs/Web/API/Media_Streams_API">Media Capture and Streams API</a></li>
 <li><a href="/en-US/docs/Web/API/Media_Capabilities_API">Media Capabilities API</a></li>
 <li><a href="/en-US/docs/Web/API/MediaStream_Recording_API">MediaStream Recording API</a></li>
 <li>The <a href="/en-US/docs/Web/API/WebRTC_API/Perfect_negotiation">Perfect Negotiation</a> pattern</li>
</ul>