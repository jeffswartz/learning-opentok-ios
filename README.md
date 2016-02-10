Learning OpenTok iOS Sample App
===============================

This sample app shows how to accomplish basic tasks using the OpenTok iOS SDK.
It connects the user with another client so that they can share an OpenTok audio-video
chat session. 

The step-0 branch includes a basic Xcode project.  Before you can test the application,
you need to make some settings in Xcode and set up a web service to handle some
OpenTok-related API calls.

1. Download the [OpenTok iOS SDK] [1].

2. Locate the LearningOpenTok.xcodeproj file and open it in Xcode.

3. Include the OpenTok.framework in the list of frameworks used by the app.
   From the OpenTok iOS SDK, you can drag the OpenTok.framework file into the list of
   frameworks in the Xcode project explorer for the app.

4. In the ViewController.h file, edit the following values with credentials for your
   OpenTok session: `_apiKey`, `_sessionId`, `_token`.

**Session ID** -- Each client that connects to the session needs the session ID, which identifies
the session. Think of a session as a room, in which clients meet. Depending on the requirements of your application, you will either reuse the same session (and session ID) repeatedly or generate
new session IDs for new groups of clients.

**Token** -- The client also needs a token, which grants them access to the session. Each client is
issued a unique token when they connect to the session. Since the user publishes an audio-video stream to the session, the token generated must include the publish role (the default). For more
information about tokens, see the OpenTok [Token creation overview] [2].

**API key** -- The API key identifies your OpenTok developer account.

For testing, you can set these values using the OpenTok dashboard] [3].

In a shipping app, generate tokens and sessions using an [OpenTok server SDK] [4].

## Connecting to the session

Upon starting up, the app calls the `[self doConnect]` method to initialize an OTSession object
and connect to the OpenTok session:

    - (void)doConnect
    {
        // Initialize a new instance of OTSession and begin the connection process.
        _session = [[OTSession alloc] initWithApiKey:_apiKey
                                           sessionId:_sessionId
                                            delegate:self];
        OTError *error = nil;
        [_session connectWithToken:_token error:&error];
        if (error)
        {
            NSLog(@"Unable to connect to session (%@)",
                  error.localizedDescription);
        }
    }

The OTSession object (`_session`), defined by the OpenTok iOS SDK, represents the OpenTok session
(which connects users).

The `[OTSession connectWithToken:error]` method connects the iOS app to the OpenTok session.
You must connect before sending or receiving audio-video streams in the session (or before
interacting with the session in any way).

This app sets `self` to implement the `[OTSessionDelegate]` interface to receive session-related
messages. These messages are sent when other clients connect to the session, when they send
audio-video streams to the session, and upon other session-related events, which we will look
at in the following sections.

## Publishing an audio video stream to the session

1. In Xcode, launch the app in a connected iOS device or in the iOS simulator.

2. On first run, the app asks you for access to the camera:

     LearningOpenTok would like to Access the Camera: Don't Allow / OK

   iOS OS requires apps to automatically ask the user to grant camera permission to an app.

   The published stream appears in the lower-lefthand corner of the video view. (The main storyboard
   of the app defines many of the views and UI controls used by the app.)

3. Now close the app and find the test.html file in the root of the project. You will use the
   test.html file (in located in the root directory of this project), to connect to the OpenTok
   session and publish an audio-video stream from a web browser:

   * Edit the test.html file and set the `sessionCredentialsUrl` variable to match the
     `ksessionCredentialsUrl` property used in the iOS app. Or -- if you are using hard-coded
     session ID, token, and API key settings -- set the `apiKey`,`sessionId`, and `token` variables.

   * Add the test.html file to a web server. (You cannot run WebRTC videos in web pages loaded
     from the desktop.)

   * In a browser, load the test.html file from the web server.

4. Run the iOS app again. The app will send an audio-video stream to the web client and receive
   the web client's stream.

5. Click the mute mic button (below the video views).

   This mutes the microphone and prevents audio from being published. Click the button again to
   resume publishing audio.

6. Click the mute mic button in the subscribed stream view.

   This mutes the local playback of the subscribed stream.

7. Click the swap camera button (below the video views).

   This toggles the camera used (between front and back) for the published stream.

Upon successfully connecting to the OpenTok session (see the previous section), the
`[OTSessionDelegate session:didConnect:]` message is sent. The ViewController.m code implements
this delegate method:

    - (void)sessionDidConnect:(OTSession*)session
    {
        // We have successfully connected, now start pushing an audio-video stream
        // to the OpenTok session.
        [self doPublish];
    }

The method calls the `[self doPublish]` method, which first initializes an OTPublisher object,
defined by the OpenTok iSO SDK:

    _publisher = [[OTPublisher alloc]
                  initWithDelegate:self];

The code calls the `[OTSession publish:error:]` method to publish an audio-video stream
to the session:

    OTError *error = nil;
    [_session publish:_publisher error:&error];
    if (error)
    {
        NSLog(@"Unable to publish (%@)",
              error.localizedDescription);
    }

It then adds the publisher's view, which contains its video, as a subview of the
`_publisherView` UIView element, defined in the main storyboard.

    [_publisher.view setFrame:CGRectMake(0, 0, _publisherView.bounds.size.width,
                                       _publisherView.bounds.size.height)];
    [_publisherView addSubview:_publisher.view];

This app sets `self` to implement the OTPublisherDelegate interface and receive publisher-related
events.

Upon successfully publishing the stream, the implementation of the
`[OTPublisherDelegate publisher:streamCreated]`  method is called:

    - (void)publisher:(OTPublisherKit *)publisher
    streamCreated:(OTStream *)stream
    {
        NSLog(@"Now publishing.");
    }

If the publisher stops sending its stream to the session, the implementation of the
`[OTPublisherDelegate publisher:streamDestroyed]` method is called:

    - (void)publisher:(OTPublisherKit*)publisher
    streamDestroyed:(OTStream *)stream
    {
        [self cleanupPublisher];
    }

The `[self cleanupPublisher:]` method removes the publisher's view (its video) from its
superview:

    - (void)cleanupPublisher {
        [_publisher.view removeFromSuperview];
        _publisher = nil;
    }

## Subscribing to another client's audio-video stream

The [OTSessionDelegate session:streamCreated:] message is sent when a new stream is created in
the session. The app implements this delegate method with the following:

    - (void)session:(OTSession*)session
    streamCreated:(OTStream *)stream
    {
        NSLog(@"session streamCreated (%@)", stream.streamId);

        if (nil == _subscriber)
        {
            [self doSubscribe:stream];
        }
    }

The method is passed an OTStream object (defined by the OpenTok iOS SDK), representing the stream
that another client is publishing. Although this app assumes that only one other client is
connecting to the session and publishing, the method checks to see if the app is already
subscribing to a stream (if the `_subscriber` property is set). If not, the session calls `[self doSubscribe:stream]`, passing in the OTStream object (for the new stream):

    - (void)doSubscribe:(OTStream*)stream
    {
        _subscriber = [[OTSubscriber alloc] initWithStream:stream
                                                  delegate:self];
        OTError *error = nil;
        [_session subscribe:_subscriber error:&error];
        if (error)
        {
            NSLog(@"Unable to publish (%@)",
                  error.localizedDescription);
        }
    }

The method initializes an OTSubscriber object (`_subscriber`), used to subscribe to the stream,
passing in the OTStream object to the initialization method. It also sets `self` to implement the
OTSubscriberDelegate interface, which is sent messages related to the subscriber.

It then calls `[OTSession subscribe:error:]` to have the app to subscribe to the stream.

When the app starts receiving the subscribed stream, the
`[OTDSubscriberDelegate subscriberDidConnectToStream:]` message is sent. The implementation of the
delegate method adds view of the subscriber stream (defined by the `view` property of the OTSubscriber object) as a subview of the `_subscriberView` UIView object, defined in the main
storyboard:

    - (void)subscriberDidConnectToStream:(OTSubscriberKit*)subscriber
    {
        NSLog(@"subscriberDidConnectToStream (%@)",
              subscriber.stream.connection.connectionId);
        [_subscriber.view setFrame:CGRectMake(0, 0, _subscriberView.bounds.size.width,
                                              _subscriberView.bounds.size.height)];
        [_subscriberView addSubview:_subscriber.view];
        _subscriberAudioBtn.hidden = NO;

        _chatTextInputView.hidden = NO;
    }

It also displays the input text field for the text chat. The app hides this field until
you start viewing the other client's audio-video stream.

If the subscriber's stream is dropped from the session (perhaps the client chose to stop publishing
or to the implementation of the `[OTSession session:streamDestroyed]` method is called:

    - (void)session:(OTSession*)session
    streamDestroyed:(OTStream *)stream
    {
        NSLog(@"session streamDestroyed (%@)", stream.streamId);
        if ([_subscriber.stream.streamId isEqualToString:stream.streamId])
        {
            [self cleanupSubscriber];
        }
    }

The `[self cleanupSubscriber:]` method removes the publisher's view (its video) from its
superview:

    - (void)cleanupPublisher {
        [_subscriber.view removeFromSuperview];
        _subscriber = nil;
    }



[1]: https://tokbox.com/opentok/libraries/client/ios/
[2]: https://tokbox.com/opentok/tutorials/create-token/
[3]: https://dashboard.tokbox.com
[4]: https://tokbox.com/developer/sdks/server/
