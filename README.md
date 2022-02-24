# Selfie Segmentation with MediaPipe and React
With the increase in remote work and everyone taking calls at home. Most popular video conferencing apps have now implemented virtual background replacement since it improves the overall exprience and provide privacy enhancment in their homes or whevever they may be. 

In this post we will work on implementing virtual background for an app that you might be working on. We will be using WebRTC APIs to capture a stream from a camera and MediaPipe to use Machine Learning and WebAssembly to remove our background.

### MediaPipe


MediaPipe is a cross-platform ML solution for live and streaming media that was developed and open sourced by Google. Background replacement is just one of many feature including real time hand, iris and body pose tracking. For more information on what the library does, [read here](https://ai.googleblog.com/2020/10/background-features-in-google-meet.html)

![image](https://1.bp.blogspot.com/-JDQHZZxbu8k/X5s7TInMUOI/AAAAAAAAGv4/M3l9IpoBh2w515cN6TUCIC0kim2sdm3twCLcBGAsYHQ/s16000/image8%2B%25281%2529.jpg)

### Getting Started


Before we go further, let's start with the basics of getting a stream from the camera. To do this, lets first create a new React App or you can simply use an existing app

`npx create-react-app virtual-background` 

`cd virtual-background`

### Capturing video stream from a User's Camera


We'll do all our work in `App.js`. Clear out any unnecessary code so that you `App.js` looks similar to below

`import './App.css';  
function App() {  
return (  
   
); }  
export default App;`

If you run `yarn start` you should just see a white screen.

Next we want to add a useEffect that requests users media on app load. Add the following code before the `return`  statement.

`useEffect(() => {  
const contraints = {  
video: { width: { min: 1280 }, height: { min: 720 } }  
}  
navigator.mediaDevices.getUserMedia(constraints).then(stream => {  
console.log(stream)  
})  
}, [])`

`navigator.mediaDevices.getUserMedia` is the API that allows us to request for either `audio`  or `video`  devices that the browser has access to. For simplicity, we will only be using video and providing some requirements such as min width and height. As soon as the browser reloads, you should see an alert asking for access to the camera

![image](https://res.craft.do/user/full/bc447247-b9e6-8215-f294-4590c33eea48/doc/8050312F-8E3F-4E91-8BBA-736968B01323/5DEC6DC5-5A0C-4A2C-B1B4-669FD4D38C41_2/FDqKQ68e9QuLv0N50nnHWSKwufhFmxUEGlwPmzu312Ez/Image.png)

Since we are simply logging the stream, you should see a `MediaStream` in your developer console

![image](https://res.craft.do/user/full/bc447247-b9e6-8215-f294-4590c33eea48/doc/8050312F-8E3F-4E91-8BBA-736968B01323/5A0ACDA9-E611-4A0D-A836-E68B3E7C755C_2/0eNj94iXJkeoXEVODJkmmbV0YljDHvfxTL3IywlSoyMz/Image.png)

### Viewing our camera stream


Now we can create a `video` element and set its source to our media stream. Since we are using React and need access to the video element, we will use the `useRef` hook. We'll first delacre `inputVideoRef` and assign `useRef()` to it. Next we add `<video>`  element inside the top level `div` and give it the `inputVideoRef`. We are setting the srcObject of the video element to the stream we get from the camera when we get the stream. We will also need a canvas to draw our new view. so we create 2 more refs `canvasRef` and `contextRef` . Lastly, in our useEffect we will set the contextRef to a 2D context. You `App` component should look like below. 

  
`import { useEffect, useRef } from 'react';  
import './App.css';  
function App() {  
const inputVideoRef = useRef()  
const canvasRef = useRef()  
const contextRef = useRef()  
  
useEffect(() => {  
contextRef.current = canvasRef.current.getContext("2d")  
const constraints = {  
video: { width: { min: 1280 }, height: { min: 720 } }  
}  
navigator.mediaDevices.getUserMedia(constraints).then(stream => {  
inputVideoRef.current.srcObject = stream  
})  
}, [])  
return (  
<div className="App>  
<video autoPlay ref={inputVideoRef} />  
<canvas ref={canvasRef} width={1280} height={720} />  
</div>  
); }  
export default App;`

Once you save and refresh the app, you should see your pretty face in your browser.

### Installing Dependencies


We will now install MediaPipe's `selfie_segmentation`  library 

`npm install @mediapipe/selfie_segmentation` 

### Flow of Bytes


Before we go further, lets talk about our flow of bytes in our app a little bit. Our camera sends a stream of data to our browser, which gets passed to the Video element. Then a frame/image at a point in time from the video element is sent to Mediapipe (locally), which is then run through the ML model to provide 2 types of results, a segmentation mask and an image to draw on top of the segmentation mask. So what that means is that we will have to constantly feel Mediapipe with data and MediaPipe will keep returning results

### Selfie Segmentation


Selfie Segmentation has a `constructor` , a `setOptions`  method and an `onResults` method. We will continue building our app by utilizing each of these.

Selfie Segmentation constructor - Inside our existing useEffect, we instanciate an instance of  selfieSegmentation

	`const selfieSegmentation = new SelfieSegmentation({  
locateFile: (file) =>  
https://cdn.jsdelivr.net/npm/@mediapipe/selfie_segmentation/${file},  
});`

Next, we `setOptions` also inside the same useEffect

	`selfieSegmentation.setOptions({  
modelSelection: 1,  
selfieMode: true,  
});`

`modelSelection` is `0`  or `1`. `0` uses general model and `1` is landscape model. Google Meets uses a variant of landscape.  `selfieMode` will simple flip the image horizontially. 

### Send a Frame and Loop


Lastly, we want to create a `sendToMediaPipe` (also inside the useEffect). This method will call the `send` method on selfieSegmentation sending it a frame of `inputVideoRef`  and contantly call itself via `requestAnimationFrame` . One thing to know is that we cannot feed empty frames to the send method and in the first couple renders of the app component, the inputVideoRef hasn't received video information from the camera yet. So we have to check and ensure that the `videoWidth` is 0. 

const sendToMediaPipe = async () => {  
if (!inputVideoRef.current.videoWidth) {  
console.log(inputVideoRef.current.videoWidth);  
requestAnimationFrame(sendToMediaPipe);  
} else {await selfieSegmentation.send({ image: inputVideoRef  
.current });  
requestAnimationFrame(sendToMediaPipe);  
}  
};

### Let's deal with the result!


Next, when we get a result back, we want to do something with those results. Let's create a method called `onResults`

This can be inside or ourside the useEffect.  
`const onResults = (results) => {  
contextRef.current.save();  
contextRef.current.clearRect(  
0,  
0,  
canvasRef.current.width,  
canvasRef.current.height  
);  
contextRef.current.drawImage(  
results.segmentationMask,  
0,  
0,  
canvasRef.current.width,  
canvasRef.current.height  
);  
// Only overwrite existing pixels.  
contextRef.current.globalCompositeOperation = "source-out";  
contextRef.current.fillStyle = "#00FF00";  
contextRef.current.fillRect(  
0,  
0,  
canvasRef.current.width,  
canvasRef.current.height  
);  
// Only overwrite missing pixels.  
contextRef.current.globalCompositeOperation = "destination-atop";  
contextRef.current.drawImage(  
results.image,  
0,  
0,  
canvasRef.current.width,  
canvasRef.current.height  
);  
contextRef.current.restore();  
};`

Here we are doing a couple of things. First clearing the canvas, then we draw on the results we get from the segmentationMask, which are a set of pixes anywhere on our 1280 x 720 canvas that aren't determinted to be human. Next we are gonna turn all those pixels green. We are then gonna draw on the pixels we get from the image result on top. 

 We then pass the onResults method to selfieSegmentation.onResults (inside the useEffect)

`selfieSegmentation.onResults(onResults);` 


For more information, checkout [MediaPipe](https://google.github.io/mediapipe/solutions/selfie_segmentation#javascript-solution-api)

If you have any questions, feel free to reach out [@_tkoriginal](https://twitter.com/_tkoriginal) on twitter

