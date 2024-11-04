_<p align="center">Unissey Confidential</p>_
![logo](https://user-images.githubusercontent.com/2079561/134871554-4682d336-60a0-48d1-9dd3-6e8330e6e013.png)

# Unissey JavaScript SDK

The official Unissey Javascript SDK for the browser. This library handles video capture needed for Unissey API. 

## Motivation

**Unissey SDK.js** helps you handle video acquisition and provides all the materials requested by Unissey biometric API. These materials consist of compressed video and metadata encrypted string. SDK.js is designed to be easy to integrate. It performs all the processings and gives feedbacks to the calling application (acquisition status, face position info, recording progression and potential issues).

## Demo

Here is an example of a basic use of the sdk for recording a video and displaying it on the same web page.
[https://github.com/unissey/sdk-web-examples/tree/master/sdk-web-js](https://github.com/unissey/sdk-web-examples/tree/master/sdk-web-js)

[Try it here](https://astounding-jelly-e183ec.netlify.app/)

## Installation
The package is currently not public. To gain access, please follow the steps below:
- Send your GitHub account name to the Unissey team, so that we grant you an access permission.
- Get a Personal Access Token from Github (see below).
- On your build system, add a "NPM_TOKEN" environement variable and set its value to the Personal Access Token (see below).

#### Get a personnal access token

To generate and get a Github Personal Access Token follow the instructions [here](https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line). 
Make sure to include at least the **`repo`** && **`read:packages`** permissions to the access token.

(Don't forget to send your github account name to the Unissey team.)

#### Authenticate to the registry

Add your token in the `NPM_TOKEN` environement variables:

```shell
export NPM_TOKEN=<your-token>
```

Then reference the new registry in a `.npmrc` in your project directory

**npm 6:**
```shell
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
registry=https://npm.pkg.github.com/unissey
```
**npm 7 (and higher):**
```shell
//npm.pkg.github.com/:_authToken=${NPM_TOKEN}
@unissey:registry=https://npm.pkg.github.com
```

#### Install the dependency

(compatible with most build systems: Webpack, Rollup, etc...)

**using npm:**

```shell
npm install @unissey/sdk-web-js
```

**or using yarn:**

```shell
$ yarn add @unissey/sdk-web-js
```

**Note**: if you don't use a build system that handles modules (pure js for instance) and want to reference *sdk-web-js* in a `<script>` tag, the exported symbols are accessible through the "UnisseySDK" namespace, like this :

```html
<script src=".../sdk-web-js@0.2.0/index.js" />
...
const unisseySession = await UnisseySDK.UnisseySdk.createSession(videoElem, UnisseySdk.AcquisitionPreset.SELFIE_FAST);
```
Contact us (support@unissey.com) if you have any question regarding your installation and this specific case.

## Getting started

The following example sets up handlers for events, creates an acquisition session and starts the processing.

```javascript
// We have defined somewhere an HTML Video element and a canvas to draw the overlay over it
// <div>
//   <canvas id="canvasElem" style="z-index: 2; position: absolute; width: 600px"></canvas>
//   <video id="videoElem" style="z-index: 1; position: absolute; width: 600px" autoplay muted playsinline></video>
// </div>
import {UnisseySdk, AcquisitionEvent, AcquisitionPreset} from '@unissey/sdk-web-js';

let unisseySession;

// 1. Set callbacks for SDK events

// attach listeners for the events we are interested in.
//    SDK status listener
UnisseySdk.addListener(AcquisitionEvent.STATUS, (status) =>
	// status : StatusEvent (string)
    //  StatusEvent.NO_SESSION ("no-session") --> No (or no more) UnisseySession. Any reference to previous session should be removed.
    //  StatusEvent.READY ("ready") --> the session is ready to start a capture
    //  StatusEvent.STARTING ("starting") --> waiting for position validation
    //  StatusEvent.RUNNING ("running") --> recording is in progress
    //  StatusEvent.ABORTING ("aborting") --> aborting is in progress
    console.log(`SDK status changed to: ${status}`);
    if (status === StatusEvent.NO_SESSION) {
      // release reference to unisseySession
      unisseySession = null;
    }
);

//    Issue listener
UnisseySdk.addListener(AcquisitionEvent.ISSUE, (type, value) => {
	// type : IssueType (string)
    // IssueType.NO_FACE ("no-face") --> No face is detected after 3 seconds (default). (The delay can be configured with noFaceIssueDelayMs in FaceCheckerConfig) ) 
    //        value can be ignored
    // IssueType.FORBIDDEN_ACTION ("forbidden-action") --> The user has made an illicit action
    //            value = 1 : loose visibility (another window become active)
    //            value = 2 : loose focus (another window become active)
    //            value = 3 : a kekboard key has been pressed
    // IssueType.MOVE (“move”)  Only used if FaceCheckerOptions.check = "whileRecording" mode
    //                      This mode is currently not supported  
    //    value : 0=Back, 1=Front, 2=Left, 3=Right, 4=Down, 5=Up, 6=Face is lost while recording
    // IssueType.CAMERA_ERROR ("camera-error”)  An error occurs while trying to open the camera video stream.
    //                      This may happens while calling createSession(). 
    //                      (A log message is displayed in the console with potentially more information) 
    //    value : 0=Unspecified error, 1=Permission denied, 2=Fail to open camera (may be already opened by another app)
	console.log(`An issue is reported: ${type} : ${value}`);
	if (unisseySession) unisseySession.abort();
});

//    Acquisition progression listener
UnisseySdk.addListener(AcquisitionEvent.PROGRESS, (progress) => {
	// progress : progression value in range [0, 1]
	console.log(`${(progress * 100).toFixed(0)}%`);
});

//    Face position information listener
UnisseySdk.addListener(AcquisitionEvent.FACE_INFO, (type, value) => {
	// type : FaceInfoType (string)
	//      FaceInfoType.FACE_VALID ('faceValid')  -> the face position is valid and stable. We start the acquisition. 
    //                                    Status is going to be StatusEvent.RUNNING
	//      FaceInfoType.GOOD_POS ('goodFacePos') -> the face Position is good. We now wait for a few frames to check if the position is stable before running.
	//      FaceInfoType.FACE_LOST ('faceLost') -> the face is lost (before recording).
	//      FaceInfoType.TOO_FAR ('faceTooFar') -> The face size is too small. The user should get closer to the camera for capture.
	//      FaceInfoType.BAD_POS ('badFacePos') -> the face is not correctly centered 
	//         value (FaceMoveDirection) : value gives the direction to go to correct the position
	//              FaceMoveDirection.LEFT (2), FaceMoveDirection.RIGHT (3), FaceMoveDirection.DOWN (4), FaceMoveDirection.UP (5)
	console.log(`Face info : ${type} : ${value}`);
});

// 2. Create a UnisseySession with  HTML video element (videoElem), an AcquisitionPreset and a canvas (canvasElem) to render the overlay
let unisseySession = await UnisseySdk.createSession(videoElem, AcquisitionPreset.SELFIE_FAST, canvasElem).catch((e)=> {
        console.warn("Cannot create Unissey session", e);
      });

// Start verification
try {
	const { media, metadata, error } = await unisseySession.capture();
  if (error) console.log(error);

} catch (err) {
    console.error(`Verification aborted: ${err.message}`);
}

await unisseySession.release(); 
// The unisseySession is not usable anymore. Let's release the reference.
// Note : release() emits a StatusEvent.NO_SESSION event that will release the reference to unisseySession in the AcquisitionEvent.STATUS handler. Thus, unisseySession is already null and the following line actually do nothing
unisseySession = null;

```

# Reference

## UnisseySdk

`UnisseySdk` is the SDK top level class. It is a static class (with only static methods) and cannot be instantiated. It manages `UnisseySession` objects. Note that only one session can be created at a time. You have to release previously created sessions before creating a new one. Don't forget to release the session when you don't need it anymore, in order to release the resource and close the camera.

Here's a schematic view of `UnisseySession` **lifecycle** :
```
                                ____________________________________________                  
                                v                                           |
      createSession() --> UnisseySession --> .capture() --> media + metadata
    ^                            ^                       |                    |
    |                            |                     .abort()               |
    |                            |_______________________|                .release()
    |_________________________________________________________________________|
```

A new session is created with the `createSession` static method. The `capture` method is the main method of a `UnisseySession` . It returns a media (video or image) and some metadata embedded into an encrtypted string. After the `capture` completion, the session is ready to start a new capture. The `abort`method can be used to cancel the current capture and to reset the session (that is in a ready state to start a new capture).`abort` simply stops the current capture but the video stream remains open. Lastly, the release method can be called to close the session, free resources and close the camera. In practice, the release method call should occur after receiving the media and metadata, but it can be called as well at any time.

The `SessionStatus` reflects the lifecycle's stage. Here's a replication of the scheme with the status evolution (in parentheses)
```
                                                  _____________________________________________________________________         
                                                  v                                                                    |                  
    (NO_SESSION)  createSession() --> UnisseySession (READY) --> .capture() --(STARTING + RUNNING)-> media + metadata
          ^                                       ^                                     |                                   |
          |                                       |                                   .abort()                              |
          |                                       |__________(ABORTING)_________________|                               .release()
          |_________________________________________________________________________________________________________________|
```

  The initial status is `NO_SESSION`. When createSession completes, the session status becaomes `READY` (i.e. 'ready to capture'). When starting a new capture, the status changes to `STARTING` and switches to `RUNNING` when recording is in progress (after the validation of the face position). When the capture is completed, status becomes `READY` again. If a current capture is aborted, the status is temporarily set to `ABORTING` and becomes `READY` when the abort is completed. After release, the status is reset to `NO_SESSION`. 
  

**1. Summary**
```javascript
class UnisseySdk {
  static addListener(eventType: AcquisitionEvent, f: (...args: any[]) => void): typeof UnisseySdk;
  static removeListener(eventType: AcquisitionEvent, f: (...args: any[]) => void): typeof UnisseySdk;
  static setLogLevel(logLevel: LogLevel = LogLevel.WARNING);

  static async createSession(
    videoElem: HTMLVideoElement,
    acquisitionPreset = AcquisitionPreset.SELFIE_FAST,
    overlayCanvas?: HTMLCanvasElement,
    config?: Partial<SessionConfig>
  ): Promise<UnisseySession>;

  static getVersion():string;
}
```

**2. Details**

### `static async createSession`
```javascript
async function createSession(
    videoElem: HTMLVideoElement, 
    acquisitionPreset = AcquisitionPreset.SELFIE_FAST, 
    overlayCanvas?: HTMLCanvasElement, 
    config?: Partial<SessionConfig>
): Promise<UnisseySession>
```

#### **Basic usage:**
```javascript
const videoElem = document.getElementById('my-video');
const unisseySession = await UnisseySdk.createSession(videoElem).catch ((e: Error) => {
  console.warn("Cannot create Unissey session", e);
});
```
It creates a [UnisseySession](#unisseysession) object, the main object of UnisseySDK that handles camera and video acquisition.
The method is asynchronous and returns a Promise resolving with a `UnisseySession` object.

Only one `UnisseySession` object can exist at a time (even if you may have several UnisseySdk objects, which is not recommended). An exception is thrown if you call `createSession` while a session already exists or is under construction.

You should always release the current session before creating a new one.

#### **Parameters:**

- `videoElem`: The HTML `<video>` element to use to do the video acquisition
- `preset`: The acquisition mode to use. It defines the parameters used for the video capture, like video settings (resolution, frame rate), recording format, or recording length (see below).
- `overlayCanvas`: Optional HTML `<canvas>` element. It enables the overlay drawing, that helps focus on the area of interest (i.e. the face for the selfie video). If not provided, the drawing of the overlay is disabled.
- `unisseySessionConfig`: This optional parameter allows for specific customisation of the acquisition. Note that the presets (see above) are designed to fit many use cases and generally don't require any additional settings. *SessionConfig* still allows for further customisation if needed. If you have specific needs, do not hesitate to contact us so we can help you for this configuration. More details are provided below.

#### **Presets**

The following presets are currently available:

```javascript
enum AcquisitionPreset {
  // A fast and fluid acquisition mode to record a selfie video for Unissey algorithms (<1 second, with 600x600 resolution and mjpeg compression).
  // This is the default value.
  SELFIE_FAST = "selfie-fast",

  // A longer acquisition with higher video resolution (3 seconds at 25 fps with 720p image size and h264 compression)
  SELFIE_SUBSTANTIAL = "selfie-substantial",

  // A preset that combines a short video for PAD algorithm and a longer video
  SELFIE_OPTIMIZED = "selfie-optimized",

  // A preset that records nothing
  NO_RECORD = "no-record",

  // A preset for recording a video of a document (like an ID card).
  DOC_VIDEO = "doc-video",

  // A preset for recording an image of a document (like an ID card).
  DOC_IMAGE = "doc-image",
}
```

Default: `SELFIE_FAST`
Utility import: `import {AcquisitionPreset} from '@unissey/sdk-web-js';`
Presets attributes :
 
| Preset | Video Length | Format | Resolution |Frame Rate | BitRate | Camera | Overlay | Position Cheching |
| ------ | ------------ | ------ | ---------- | ---------- | ------- | ------ | ------- | ----------- |
| SELFIE_FAST | 1/4 s   | MJPG   | 600x600    | 25         | N/A     | Front  | OVAL    | Yes |
| SELFIE_SUBSTANTIAL | 3 s | WEBM| 1280*720   | 25         | 5000    | Front  | OVAL    | Yes |
| SELFIE_OPTIMIZED | 3 s | WEBM + MJPG| 1280*720   | 25         | 5000    | Front  | OVAL    | Yes |
| NO_RECORD | 0 s | | 1280*720   | 25         | N/A    | Front  | None    | No |
| DOC_VIDEO | 3 s       | WEBM   | 1280*720   | 25         | 2000    | Rear   | ID_DOCUMENT | No |
| DOC_IMAGE | 1 frame   | JPG    | 1280*720   | N/A        | N/A    |  Rear   | ID_DOCUMENT | No |

The table above shows the default values. All of them can be overridden with `SessionConfig`, except  'Position Cheching' that can be overridden with `CaptureOptions` and 'Format' that cannot be modified.

If you're not sure which preset to use, do not hesitate to contact us.


#### **`SessionConfig`**

```js
import { SessionConfig } from '@unissey/sdk-web-js';
```

The *SessionConfig* is described below:
*Note that all fields are optional (as explicitly defined by the question mark after each property name below.)* 

```typescript
type SessionConfig = {
  // Enable the overlay drawing and specify optional parameters. If not provided, drawing of the overlay is disabled.
  overlayConfig?: OverlayConfig;
    {
      displayMode?: OverlayDisplayMode; // The overlay shape, chosen among:  OVAL = "oval" (default), RECT_PORTRAIT = "rect-portrait": a portrait-oriented rectangle, RECT_LANDSCAPE = "rect-landscape": a landscape-oriented rectangle, ID_DOCUMENT = "id-document": similar to RECT_LANDSCAPE with thicker corners. The default is OVAL for "selfie" acquisition presets and ID_DOCUMENT for document acquisition presets.
      colors?: OverlayColors;    // Optional color customisation.
        {
          background: [R,G,B,Alpha],  // The background / overlay color and transparency (default : [255, 255, 255, 0.8])
          innerBorder: [R,G,B,Alpha],  // The oval/rect border color (default : [255, 255, 255, 1])
          progressColor: [R,G,B,Alpha],  // The progress bar (running over the inner border) color (default : [0, 188, 212, 1])
        };
        filter: string; // a CSS-like filter string, that applies an effect to the video preview. Example : "blur(5px)"
    }; 
  cameraConfig?: CameraConfig;    // To modify the camera parameters. You generally don't have to change these values, as the presets define standard default values for each use case. 
      {
      preferredFps?: number;    // The frame rate, in fps (frame per second). You generally don't have to change it.
      preferredResolution?: VideoResolutionPreset;  // The preferred video resolution (width, height). The default value depends on the preset. Possible values are listed below.
      preferredOrientation?: VideoOrientation;  // 2 values are officially supported. VideoOrientation.SAME_AS_SCREEN ("same-as-screen") (-> follow the orientation of the device), VideoOrientation.LANDSCAPE ("landscape") (But this is not recommended by now. Some versions of mobile browsers may badly handle orientation change for this mode). The default value is SAME_AS_SCREEN.
      // Note : 2 other modes are implemented (VideoOrientation.LANDSCAPE and VideoOrientation.PERPENDICULAR_TO_SCREEN). They are really not recommended by now because portrait orientation is poorly handled by most of browsers on computers
      facingMode?: FacingMode; // The camera to use, for mobile devices. FacingMode.FRONT = "user" (the 'selfie' mode), FacingMode.BACK = "environment" (the 'rear' camera). The default value depends on the preset (FRONT for "selfie" presets, BACK for "document" presets)
    }
  recordingConfig?: RecordingConfig;    // To modify the recording parameters. You generally don't have to change these values, as the presets define standard default values for each use case. 
    {
    audio?: boolean;      // 'true' to enable audio recording (default: 'false').
    bitRateKbps?: number; // The bitrate (in kbps) used by video recorder.
    length?: VideoLength; // Used to change the preset's video length, indicated as a duration or number of frames
        //      VideoLength is defined like this :
        //    {
        //      | { type: "duration"; durationMs: number }      // duration in milliseconds
        //      | { type: "frame-count"; frameCount: number };  // number of frames to record
        //    }

    faceCheckerConfig?: FaceCheckerConfig;  // may be used to override preset default and configure the face position checker for the session.
        // When disabled ({ check: "disabled" }), the face Checker is not present in the session and it will be impossible to do a capture() with  face Checking.
        //      In this case, recording will start immediately, without any face position prompt.
        // When enabled  ({ check: "beforeRecording" }), the face checker is mounted into the session (even if the preset won't enable it by default)
        //      In this case, actual recording will start after face position validation, except if the client explicitly disable it with captureOptions parameter.
        //      Note that it is not equivalent to disable the face checker with faceCheckerConfig (at session creation) and with faceCheckerOptions (at capture() time)
        //      The behavior is not the same :
        //        If the face checker is disabled with faceCheckerConfig (at session creation), recording will start immediately, without any face position prompt.
        //        If the face checker is disabled with faceCheckerOptions (at capture() time), recording will start just after a quick prompt that simulate the validation of the correct position.
        //      FaceCheckerConfig is defined like this :
        //        | { check: "disabled" }
        //        | { check: "beforeRecording" | "whileRecording"; customFaceArea?: FaceRect; noFaceIssueDelayMs?:number };
        // default: { check: "beforeRecording"} for selfie presets or { check: "disabled" } for document presets.
        // Note: { check: "whileRecording"} is limited to very specific use cases and is currently disabled.
        // The optional **'customFaceArea'** parameter can be used **if the application draws its own overlay** and wants the user to be positionned at a specific position (and not the default central position).
        // The optional **'noFaceIssueDelayMs'** parameter can be used to customize the delay before sending IssueType.NO_FACE event when no face is detected. A negative value means that the issue will be never emitted. The default is 3000 ms.
        
    iadConfig?: IadConfig;  // To configure Injection Attack Detection (IAD is disabled by default if iadConfig is not defined)
    //      IadConfig is defined like this :
    //        {
    //          mode: IadMode; // (default: IadMode.DISABLED). Possible values are : 
    //                  DISABLED = "disabled", // No IAD (equivalent to keep 'iadConfig' undefined)
    //                  MEDIA_INTEGRITY = "media-integrity", // No IAD measures as such, but compute a hash to ensure that the media is not corrupted after capture
    //                  PASSIVE = "passive", // Passive IAD
    //                  PASSIVE_LT = "passive-lt",  // only does IAD measures that don't need randomness (thus don't need IAD preparation data)
    //                  ACTIVE_FALLBACK = "active-fallback", // Passive IAD if strong method can be used else fallback to an active challenge
    //                  ACTIVE = "active", // just an active challenge (with no passive IAD)
    //          data?: string; // the data get from 'iad-prepare' end-point (requested except for challenge active or passive ( except in sequenceAllWithDurations use case)
    //          retryCount?: number; // the retry number. By retry, we mean another session with the same 'data' (i.e. without a new call to 'iad-prepare' end-point)
    //          activeChallengeConfig?: ActiveChallengeConfig; // used if mode is ACTIVE_FALLBACK or ACTIVE. (Required in this case.)
    //              {
    //                numberOfActions: number; // Number of actions the user must perform (must be >= 1 except if sequenceAllWithDurations is defined)
    //                maxSecondsBetweenActions: number; // The maximum duration (seconds) between two action requests
    //                numberOfInstructions: number; // number of possible action (SDK.js doesn't render messages. It just sends index of instruction to be displayed). SDK.web is in charge of displaying messages.
    //                actionValidation: { type: ActionValidationType }; 
    //                      NONE,     // timer only. A new new action is triggered after maxSecondsBetweenActions. 
    //                      LOOSE_FACE, // use face detector.  A new new action is triggered when the face is not detected anymore (or after maxSecondsBetweenActions). 
    //                additionalRecord: boolean;   // true to trig a video recording (additional clip) during the challenge
    //                selfieBeforeAction?: number; // Specifies where to insert the capture of the selfie video for PAD. 0 means at the beginning. numberOfActions means at the end. The default is randomly selected between 2 actions (i.e. in the range [1, numberOfActions - 1]). Ignored if sequenceAllWithDurations is defined.
    //                sequenceAllWithDurations?: number[]; // optional specific duration in seconds for   (if defined, numberOfActions is ignored. The actual number of action is min(sequenceAllWithDurations.length, instructionMessages.length))
    //              };
    //        }

  versions?: { [name: string]: string }; // For Unissey Upper level SDK only. This is an optional key-value version structure that is included in the metadata (and that only Unissey can read)

  // An utility function that returns a new SessionConfig object that clones another one.
  export function cloneSessionConfig(sessionConfig: SessionConfig): SessionConfig
}
```
*Example:*
The example provided at the top of this documentation shows how to implement a selfie video capture.
Here's another example on how to implement a document capture:
```javascript
const sessionConfig = {
    overlayConfig: {
      displayMode: OverlayDisplayMode.ID_DOCUMENT, // Display the rectangle (document-shaped) overlay hole
    },
  };
const preset = AcquisitionPreset.DOC_VIDEO;    // or AcquisitionPreset.DOC_IMAGE for a jpeg image
await  unisseySession = await UnisseySdk.createSession(videoElem, preset, canvasElem, sessionConfig).catch ((e: Error) => {
  console.warn("Cannot create Unissey session", e);
});
const captureResult = await unisseySession.capture();
const reference = captureResult.media;
```

 *VideoResolutionPreset*:
The video resolution within the `cameraConfig` field can be used to override the preset's default value. The available resolution settings are provided below.
Please note that the presets are designed to work for many different use cases, so you probably don't have to change default resolution value.

``` javascript
  NO_SPECIFIC_RESOLUTION = "no-specific-resolution",  // The default system resolution

  // 16/9 (aspectRatio: 1.777777778)
  STD_480P = "720x480",    // 720 x 480   (DVD)
  STD_720P = "1280x720",   // 1280 x 720  (HD Reasy)
  STD_1080P = "1920x1080", // 1920 x 1080 (Full HD)
  STD_2160P = "3840x2160", // 3840 x 2160 (UHD 4K)
  STD_4320P = "7680x4320", // 7680 x 4320 (UHD 8K)

  // 4/3 (aspectRatio: 1.333333333)
  STD_VGA = "640x480",   // 640 x 480
  STD_SVGA = "800x600",  // 800 x 600
  STD_XVGA = "1024x768", // 1024 x 768

  // square (aspectRatio: 1.)
  SQUARE_600P = "600x600", // 600 x 600
```

#### Return value: **`UnisseySession`**
A promise resolved with a [UnisseySession](#unisseysession) object that allows you to record the video (see below). 
An exception is thrown (and the promise is rejected) if the session cannot be created in case the promise is rejected

#### Exception
An exception is thrown if the session cannot be created. For instance, if you call `createSession` while a session already exists or is under construction, or if the camera cannot be opened, or if any unexpected error occurs.

#### Issue
An ISSUE IssueType.CAMERA_ERROR is raised (via the EventEmitter) if the camera fails to open. 
The associated value is : 0=Unspecified error, 1=Permission denied, 2=Fail to open camera (may be already opened by another app)
*Note* : In these cases, an exception is thrown (and a log message is displayed in the console) with potentially more information.


### `static getVersion()`

```javascript
getVersion(): string;
```
#### usage:
```javascript
const version = UnisseySdk.getVersion();
```
returns the SDK version

#### Return value: **`string`**


### `static setLogLevel`
```javascript
setLogLevel(logLevel: LogLevel = LogLevel.WARNING) 
```
#### usage:
```javascript
UnisseySdk.setLogLevel(LogLevel.INFO);   // override logLevel to get more logs
```

#### parameters:
##### `logLevel`: `LogLevel`
*The default value (`LogLevel.WARNING`) is suitable for most use cases including production. You probably don't want to set a specific value*

Valid values are:

```javascript
enum LogLevel {
  QUIET = 0,
  WARNING = 1,
  INFO = 2,
  VERBOSE = 3,
  PERFORMANCE = 4,
}
```

Default: `WARNING`


### `static addListener`
This static method allows to attach listeners to Unissey SDK Acquisition Events in order to get information about the current session.
(We recommend to detach listeners when you don't use UnisseySDK anymore, to avoir any possible leak.)

```javascript
addListener(eventType: AcquisitionEvent, f: (...args: any[]) => void)
```
#### parameters:
##### `eventType`: `AcquisitionEvent`
see [UnisseySdk Events](#unisseyevents) for more information on events that can be listened to.
##### `f`: a function
see [UnisseySdk Events](#unisseyevents) for more information on events that can be listened to.

#### usage:
```javascript
    const sdkJsListener_status = (status: StatusEvent) => {...};
    const sdkJsListener_faceInfo = (type: FaceInfoType, value: number /* value?:FaceMoveDirection*/) => {...};
    UnisseySdk.addListener(AcquisitionEvent.STATUS, sdkJsListener_status)
      .addListener(AcquisitionEvent.FACE_INFO, sdkJsListener_faceInfo);
```

#### Return value: **`UnisseySdk`**
  return reference to UnisseySdk class to allow chaining


### `static removeListener`
This static method allows to detach listeners.
(It is recommended to detach listeners when you don't use UnisseySDK anymore, to avoir any possible leak.)

```javascript
removeListener(eventType: AcquisitionEvent, f: (...args: any[]) => void)
```
#### parameters:
##### `eventType`: `AcquisitionEvent`
see [UnisseySdk Events](#unisseyevents) for more information on events that can be listened to.
##### `f`: a function
see [UnisseySdk Events](#unisseyevents) for more information on events that can be listened to.

#### usage:
```javascript
    ...
    UnisseySdk.removeListener(AcquisitionEvent.STATUS, sdkJsListener_status)
      .removeListener(AcquisitionEvent.FACE_INFO, sdkJsListener_faceInfo);
```

#### Return value: **`UnisseySdk`**
  return reference to UnisseySdk class to allow chaining


## <a id="unisseyevents"></a>UnisseySdk Events

**1. Summary**
UnisseySdk sends feedbacks to the calling application and reports useful information about the acquisition process 
You should listen to these events, by registering listeners (with `addListener()`  - please refer to the example at the top of this documentation).

Utility values: `import {AcquisitionEvent, StatusEvent, IssueType, FaceInfoType, FaceMoveDirection} from '@unissey/sdk-web-js';`


**2. Details**

### `AcquisitionEvent.STATUS`
The **_AcquisitionEvent.STATUS'_** ("status") event gives feedbacks on the status of the current session.

Listener implementation:
```javascript
UnisseySdk.addListener(AcquisitionEvent.STATUS, (status:StatusEvent) => {...});
```

Possible values are:
```javascript
enum StatusEvent {
  NO_SESSION = "no-session", // No (or no more) UnisseySession. Any reference to previous sessions should be removed.
  READY = "ready", // The session is ready to start a capture.
  STARTING = "starting", // Waiting for face position validation. AcquisitionEvent.FACE_INFO events are being sent.
  RUNNING = "running",   // Recording is in progress. AcquisitionEvent.PROGRESS events are being sent.
  ABORTING = "aborting", // Aborting is in progress.
}
```

### `AcquisitionEvent.PROGRESS`
The **_AcquisitionEvent.PROGRESS'_**("progress") event notifies listener about the recording progression.
Listener implementation:
```javascript
UnisseySdk.addListener(AcquisitionEvent.PROGRESS, (progress:number) => {...});
```
`progress` is in the range [0, 1].


### `AcquisitionEvent.FACE_INFO`
The **_AcquisitionEvent.FACE_INFO'_**("face-info") event notifies listener about the face position. 
A face detector is integrated to the SDK to provide information on the user's face position to guide him during the capture. 
The event is sent when UnisseySession.capture() has started and if FaceCheckerOptions.check = "beforeRecording" mode (the default mode for "selfie" session).

Listener implementation:
```javascript
UnisseySdk.addListener(AcquisitionEvent.FACE_INFO, (type:FaceInfoType, value:number) => {...});
```

Possible issue types:
```javascript
enum FaceInfoType {
  TOO_FAR = "faceTooFar", // The face size is too small. The user should get closer to the camera. 'value' can be ignored.
  BAD_POS = "badFacePos", // The face is not correctly centered. 'value' (FaceMoveDirection) gives what direction the user should take to better position himself: FaceMoveDirection.LEFT (2), FaceMoveDirection.RIGHT (3), FaceMoveDirection.DOWN (4), FaceMoveDirection.UP (5)
  GOOD_POS = "goodFacePos", // The face position is good. We now wait for a few frames to check if the position is stable before running.
  FACE_VALID = "faceValid", // The face position is valid and stable. We can start the acquisition. Status is going to be StatusEvent.RUNNING
  FACE_LOST = "faceLost", // The face is lost (before recording).
}
```


### `AcquisitionEvent.ISSUE`
The **_AcquisitionEvent.ISSUE'_**("issue") event is sent when an error occurs.

Listener implementation:
```javascript
UnisseySdk.addListener(AcquisitionEvent.ISSUE, (type:IssueType, value:number) => {...});
```
Possible issue types are:
```javascript
enum IssueType {
  NO_FACE = "no-face",    // No face is detected after 3 seconds (default) (can be configured with noFaceIssueDelayMs in FaceCheckerConfig). 'value' can be ignored.
  FORBIDDEN_ACTION = "forbidden-action",  // The final user has made an illicit action. Possible values are:
    //            value = 1 : loose visibility (another window becomes active)
    //            value = 2 : loose focus (another window becomes active)
    //            value = 3 : a kekboard key has been pressed 
  MOVE = "move",    // Only used if FaceCheckerOptions.check = "whileRecording" mode
    //    This mode is currently not supported. Possible values are:
    //    0=Back, 1=Front, 2=Left, 3=Right, 4=Down, 5=Up, 6=Face is lost while recording
  CAMERA_ERROR = "camera-error",    //  An error occurs while trying to open the camera video stream.   
    //                      This may happens while calling createSession(). 
    //                      Additionnaly, an exception is thrown (and a log message is displayed in the console) with potentially more information 
    //    value : 0=Unspecified error, 1=Permission denied, 2=Fail to open camera (may be already opened by another app)
}
```
When receiving an Issue, the recommended behaviour is to display a meaningful message to the user and abort the current acquisition (using `unisseySession.abort();`) if you want to keep the video stream open and let the user start a new capture, or release the session (using `unisseySession.release();`) if you want to close the video stream and start another session later.
Note that in case of ISSUE.FORBIDDEN_ACTION, the session will be automatically released just after you receive the message.


### `AcquisitionEvent.ACTIVE_CHALLENGE`
The **_AcquisitionEvent.ACTIVE_CHALLENGE_**("active-challenge") allows client to drive the flow for active challenge.

Listener implementation:
```javascript
UnisseySdk.addListener(UnisseySDK.AcquisitionEvent.ACTIVE_CHALLENGE, (activeChallengeEventType:ActiveChallengeEventType, value:number) => {...});
```
Possible activeChallengeEventType types are:
```javascript
export enum ActiveChallengeEventType {
  ACTION = "action", // Trigger a new action. 'value' is the index of the instruction message to be displayed. if 'value' is negative, just erase the instruction message currently displayed.
  REMAINING_SECONDS = "remaining-seconds", // sent at each second.  'value' is the max remaining delay (in seconds) for the current action.
  COMPLETED = "completed", // The active challenge is completed.  'value' is '1' in case of normal completion or 0 in case of interruption. 
}
```


##  <a id="unisseysession"></a>UnisseySession
UnisseySession objects basically manages camera and video acquisition, and can be configured to handle many use cases.

**1. Summary**
```javascript
interface UnisseySession {
  capture(captureOptions?: Partial<CaptureOptions>): Promise<Capture>;
  abort(): Promise<void>;
  getInfo(): SessionInfo;
  release(): Promise<void>;
}
```

**2. Details**

### `async capture`
```javascript
async capture(captureOptions?: Partial<CaptureOptions>): Promise<Capture> 
```
#### usage:
```javascript
const { media, metadata, error } = await unisseySession.capture();
```
or, to disable face position checking:
```javascript
const { media, metadata, error } = await unisseySession.capture({faceCheckerOptions: { check: "disabled" }});
```

Starts the acquisition process. The function is _async_  and returns a promise that is resolved when the acquisition is over by providing the acquired media (generally a video), a `metadata` encrypted string, and an optional `error` string. By default, the face position checker is enabled, and the recording actually starts when the user's face is detected at the correct position. (**AcquisitionEvent.FACE_INFO** messages are sent to the listener, so that the application can give some instruction to the user for him to correctly positioned himself). If the session is aborted (by calling `abort` method) or if an unexpected error occurs, the received `media` is an empty blob and an explicit `error` message is set.

Utility import: `import {CaptureOptions, Capture} from '@unissey/sdk-web-js';`

#### parameters:
#### `captureOptions`: `CaptureOptions` (optional)
```javascript
interface CaptureOptions {
  faceCheckerOptions?: FaceCheckerConfig;
}

type FaceCheckerConfig =
  | { check: "disabled" }
  | { check: "beforeRecording" | "whileRecording"; customFaceArea?: FaceRect; noFaceIssueDelayMs?:number };
```
'captureOptions' is used to override face position checker configuration. For instance, this can be used to disable face position checking without having to create a new session.
When face position checking is disabled ({ check: "disabled" }), recording starts immediately. 
If enabled ({ check: "beforeRecording"}), the position will be checked and the capture will only start when the user is correctly positioned.
default: { check: "beforeRecording"} for selfie presets or { check: "disabled" } for document presets.
Note: { check: "whileRecording"} is limited to very specific use cases and is currently disabled.
The optional **'customFaceArea'** parameter can be used **if the application draws its own overlay** and wants the user to be positionned at a specific position (and not the default central position).
The optional **'noFaceIssueDelayMs'** parameter can be used to customize the delay before sending IssueType.NO_FACE event when no face is detected. A negative value means that the issue will be never emitted. The default is 3000 ms. 

#### Return value: **`Capture`**

An object that holds the _media_ blob and a _metadata_ obfuscated string.
Both the _media_ blob and _metadata_ object must be provided when calling Unissey **_analyze_** API endpoint.

```javascript
 type Capture = {
  media: Blob;        // The captured media (generally a video, but it can be an image when using the DOC_IMAGE preset). It can be an empty blob if an error occurs, or if the capture is  aborted before recording actually starts. 
  metadata: string;   // A string to send to the Unissey API, in addition to the 'media' itself
  error?: string;     // An optional error message provided if an error occurs, or if the capture is aborted before recording actually starts.
                      //  Possible error messages are : "Acquisition is aborted", "IAD Configuration error : ..." , "Unexpected Error"
};
```


### `async abort`
```javascript
async abort(): Promise<void>
```
usage:
```javascript
await unisseySession.abort();
```
To abort the current capture session. This method can be called when 'capture' has started and has not yet finished. The capture promise is rejected and a new capture is ready to start again.


### getInfo()

```javascript
getInfo(): SessionInfo;
```

#### usage:
```javascript
const info = unisseySession.getInfo()
console.log({info});
```

Returns an object with information about the acquisition session.
By now, it only contains the acquisition length.

Utility import: `import {SessionInfo} from '@unissey/sdk-web-js';`

#### Return value: **`SessionInfo`**

```javascript
type SessionInfo = {
  acquisition_length: number; // The acquisition duration in seconds
};
```


### `async release`
```javascript
  async release()
```

#### usage:
```javascript
await unisseySession.release();  
// The unisseySession is not usable anymore. Let's release the reference.
unisseySession = null;
```
Free the resources (close the camera, free workers memory, etc...). This emits a `StatusEvent.NO_SESSION` event.
After `release()`, the reference to the UnisseySession object is not usable anymore and should be released (unisseySession = null).
This should be done by your AcquisitionEvent.STATUS handler when handling `StatusEvent.NO_SESSION`.
If not, you should at least release the reference to `unisseySession`  after  `release()` promise resolutiuon.
- `release()` should be called when the session is not used anymore, either before creating a new session (with a different preset or configuration), or before exiting the video acquisition page.

## Enabling Injection Attack Detection (IAD)

An injection attack attempts to replace the video stream of the physical camera by another one, e.g by using a virtual camera. Unissey provides a solution to prevent against such attacks, which requires a few additional steps:

```javascript
// 1. Get random data from the server.
const iadPreparationResponse = await fetch("https://api-analyze.unissey.com/api/v3/iad/prepare", {
  method: "POST",
  headers: { "Content-Type": "text/plain" },
});
const iadPreparationData = await response.text();

// 2. Create a session.
const videoElem = document.getElementById("my-video");
const preset = AcquisitionPreset.SELFIE_FAST;
const overlayCanvas = document.getElementById("my-canvas");
const sessionConfig = {
  iad: {
    mode: IadMode.PASSIVE,    // Enable IAD
    data: iadPreparationData,    // Pass the preparation IAD preparation data here
  }
};
const unisseySdk = new UnisseySdk();
const unisseySession = await unisseySDK.createSession(videoElem, preset, overlayCanvas, sessionConfig);

// 3. Do the video capture. The `metadata` field contains data that the server will use to determine if there was an injection suspicion.
// It must be passed to the next api-analyze call.
const { media, metadata } = await unisseySession.capture();

// 4. Do the actual analyze call.
const formData = new FormData();
formData.append("selfie", media);             // Specify the selfie video
formData.append("processings", "liveness");   // Specify the processings
formData.append("metadata", metadata);        // Specify the metadata. If is required to perform IAD

const response = await fetch("https://api-analyze.unissey.com/api/v3/analyze", {
  method: "POST",
  headers: {
    ...formData.getHeaders(),
    Authorization: "<API-KEY>",
  },
  body: formData.getBuffer()
});

// 5. Interpret the results
const padSuccess = response.data.is_genuine;
const iadBasicLevelThreshold = 2;
const iadSuccess = response.data.details.injection.selfie.trust_score >= iadBasicLevelThreshold;
if (padSuccess && iadSuccess) {
  // The session is accepted
}
else {
  // The session is rejected
}

// If an injection attack was detected, the response will contain a negative result with no further explanation, in order not to give potential attackers any clue.
// It means there is no way to know if a user is rejected specifically because an injection attack was detected.
```

## Versions

| Version | Date       | Description                                           |
| ------- | ---------- | ----------------------------------------------------- |
| 0.3.5   | 2024-10-25 | New listening mechanics                               |
| 0.3.4   | 2024-10-08 | Handle more IAD use cases                             |
| 0.3.x   | 2023-11-28 | Many improvements and fixes. Remove unisseySdk object |
| 0.2.x   | 2023-01-01 | SDK.js V2                                             |


## Compatibility

| Chrome   | Firefox  | Edge     | IE  | Opera    | Safari   | Brave    |
| -------- | -------- | -------- | --- | -------- | -------- | -------- |
| &#10004; | &#10004; | &#10004; | -   | &#10004; | &#10004; | &#10004; |

## Support

`support@unissey.com`

## License / Copyright

This SDK is distributed under Unissey license agreement.
