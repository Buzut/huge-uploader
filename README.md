# huge uploader

`huge-uploader` is a JavaScript module designed to handle huge file uploads by chunking them in the browser. Uploads are resumable, fault tolerent, offline aware and mobile ready.

HTTP and especially HTTP servers have limits and were not designed to transfer large files. In addition, network connexion can be unreliable. No one wants an upload to fail after hours… Sometimes we even need to pause the upload, and HTTP doesn't allow that.

The best way to circumvent these issues is to chunk the file and send it in small pieces. If a chunk fails, no worries, it's small and fast to re-send it. Wanna pause? Ok, just start where you left off when ready.

That's what `huge-uploader` does. It:
* chunks the file in pieces of your chosen size,
* retries to upload a given chunk when transfer failed,
* auto pauses transfer when device is offline and resumes it when back online,
* allow you to pause and resume the upload,
* obviously allows you to set custom headers and post parameters.

## Installation & usage
```javascript
npm install huge-uploader --save
```

```javascript
// require using commonJS
const uploader = require('huge-uploader');

// or in es6, using a module bundler like webpack
import uploader from 'huge-uploader';

// instanciate the module with a settings object
const uploader = new HugeUploader({ endpoint: '//where-to-send-files.com/upload/', file: fileObject });

// subscribe to events
uploader.on('error', (err) => {
	console.error('Something bad happened', err.detail);
});

uploader.on('progress', (progress) => {
    console.log(`The upload is at ${progress.detail}%`);
});

uploader.on('finish', () => {
    console.log('yeahhh');
});

// if you want to pause/resume the upload
uploader.togglePause();
```

### Constructor settings object
The constructor takes a settings object. Available options are:
* `endpoint { String }` – where to send the chunks (__required__)
* `file { Object }` – a [File object](https://developer.mozilla.org/en-US/docs/Web/API/File) representing the file to upload (__required__)
* `headers { Object }` – custom headers to send with each request
* `postParams { Object }` – post parameters that __will be sent with the last chunk__
* `chunkSize { Number }` – size of each chunk in MB (default is 10MB)
* `retries { Number }` – number of total retries (total, not per chunk) after which upload fails (default is 5)
* `delayBeforeRetry { Number }` – how long to wait (in seconds) after a failure before next try (default is 5s)


### Events
Events handling informations are instances of the [`CustomEvent` constructor](https://developer.mozilla.org/en-US/docs/Web/API/CustomEvent/CustomEvent). Hence, message is available in the `detail` property.

#### `error`
Either server responds with an error code that isn't going to change.
Success response codes are `200`, `201`, `204`. All error codes apart from `408`, `502`, `503`, `504` are considered not susceptible to change with a retry.

Or there were too many retries already.
```javascript
uploader.on('error', err => console.log(err.detail)); // A string explaining the error
```

#### `fileRetry`
```javascript
uploader.on('fileRetry', (msg) => {
    /** msg.detail is an object like:
    * {
    * 	  message: 'An error occured uploading chunk 243. 6 retries left',
    *     chunk: 243,
    *     retriesLeft: 6
    * }
    */
});
```

#### `progress`
```javascript
uploader.on(progress, progress => console.log(progress.detail)); // Number between 0 and 100
```

#### `finish`
```javascript
uploader.on('finish', () => console.log('🍾'));
```

#### `offline`
Notifies that browser is offline, hence the uploader paused itself. Nevertheless, it's paused internally, it has nothing to do with paused triggered with `.togglePause()` method nor does it interact with user pause state.

```javascript
uploader.on('offline', () => console.log('no problem, wait and see…'));
```

### `online`
Notifies that browser is back online and uploader is going to resume the upload (if not paused by `.togglePause()`).

```javascript
uploader.on('offline', () => console.log('😎'));
```

### Method
There is only one method: `.togglePause()`. As its name implies, it pauses and resumes the upload. If you need to abort an upload, you can use this method to stop it and then destroy the instance's variable.


## How to set up with the server
This module has a twin [Node.js module](https://github.com/Buzut/huge-uploader-nodejs) to handle uploads with a Node.js server as a backend. Neverthless it's easy to implement the server side in your preferred language (if you develop a module, tell me about it so I can add it to this README).


Files are sent with `POST` requests containing the following headers:
* `uploader-file-id` unique file id based on file size, upload time and a random generated number (so it's really unique),
* `uploader-chunks-total`the total numbers of chunk that will be sent,
* `uploader-chunk-number` the current chunk number (0 based index, so last chunk is `uploader-chunks-total - 1`).

`POST` parameters are sent with the last chunk if any (as set in constructor's options object).

The typical server implementation is to create a directory (name it after `uploader-file-id`) when chunk 0 is received and write all chunks into it. When last chunk is received, grab the `POST` parameters if any, concatenate all the files into a single file and remove the temporary directory.

Also, don't forget that you might never receive the last chunk if upload is abandonned, so don't forget to clean your upload directory from time to time.

In case you are sending to another domain or subdomain than the current site, you'll have to setup [`CORS`](https://developer.mozilla.org/en-US/docs/Web/HTTP/CORS) accordingly. That is, set the following `CORS` headers:
* `Access-Control-Allow-Origin: https://origin-domain.com` (here you can set a wildcard or the domain from whitch you upload the file,
* `Access-Control-Allow-Methods: POST,OPTIONS`,
* `Access-Control-Allow-Headers: uploader-chunk-number,uploader-chunks-total,uploader-file-id`,
* `Access-Control-Max-Age: 86400`.

These parameters tell your browser that it can use `OPTIONS` (the [preflight request](https://developer.mozilla.org/en-US/docs/Glossary/Preflight_request)) and `POST` methods on the target domain and that the custom headers are allowed to be sent. The last header tells the browser than it can cache the result of the preflight request (here for 24hrs) so that it doesn't need to re-send a preflight before each `POST` request.


## Under the hood

The library works around the [HTML5 `File` API](https://developer.mozilla.org/en-US/docs/Web/API/File), the rather new [`Fetch` API](https://developer.mozilla.org/en-US/docs/Web/API/Fetch_API) and the new [`EventTarget` constructor](https://developer.mozilla.org/en-US/docs/Web/API/EventTarget/EventTarget).

`EventTarget` constructor is polyfilled so it won't be a problem. Nevertheless, your target browsers have to support HTML5 [`File` API](https://caniuse.com/#feat=fileapi) and [`Fetch` API](https://caniuse.com/#feat=fetch). It means that all browsers in their recent versions apart from Internet Explorer can run this without a problem.

You can polyfill [`Fetch`](https://github.com/github/fetch) if you want to support IE.

## Contributing

There's sure room for improvement, so feel free to hack around and submit PRs!
Please just follow the style of the existing code, which is [Airbnb's style](http://airbnb.io/javascript/) with [minor modifications](.eslintrc).
