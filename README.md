# Beam Coder

[Node.js](https://nodejs.org/) native bindings to [FFmpeg](https://www.ffmpeg.org/) with support for asynchronous processing via streams and promises.

The aim of this module is to facilitate access to the capabilities of FFmpeg - including media muxing, demuxing, encoding, decoding and filtering - from Node.js applications. Rather than using the filesystem and controlling the FFmpeg as an external command line process, the beam coder executes functions of the FFmpeg _libav*_ libraries directly. Work is configured with Javascript objects and jobs execute over data buffers that are shared between Javascript and C. Long running media processing operations are asynchronous, running as promises that execute native code on a separate thread from the main event loop.

### Example

View JPEGs in a browser of any key frame in the current folder, e.g. `.MP4` media files on a camera memory card, with the following code snippet. For example, to access a key frame around 42.5 seconds from the start of a file called `GOPR9502.MP4`:

    http://localhost:3000/GOPR9502.MP4/42.5

Beam coder uses promises and so the code for the server works well with Javascript's `async`/`await` feature and [koa](https://koajs.com) contexts:

```javascript
const beamcoder = require('beamcoder');
const Koa = require('koa');
const app = new Koa();

app.use(async (ctx) => { // Assume HTTP GET with path /<file_name>/<time_in_s>
  let parts = ctx.path.split('/'); // Split the path into filename and time
  if ((parts.length < 3) || (isNaN(+parts[2]))) return; // Ignore favicon etc..
  let dm = await beamcoder.demuxer('file:' + parts[1]); // Probe the file
  await dm.seek({ time: +parts[2] }); // Seek to the closest keyframe to time
  let packet = await dm.read(); // Find the next video packet (assumes stream 0)
  for ( ; packet.stream_index !== 0 ; packet = await dm.read() );
  let dec = beamcoder.decoder({ demuxer: dm, stream_index: 0 }); // Create a decoder
  let decResult = await dec.decode(packet); // Decode the frame
  if (decResult.frames.length === 0) // Frame may be buffered, so flush it out
    decResult = await dec.flush();
  // Filtering could be used to transform the picture here, e.g. scaling
  let enc = beamcoder.encoder({ // Create an encoder for JPEG data
    name : 'mjpeg', // FFmpeg does not have an encoder called 'jpeg'
    width : dec.width,
    height: dec.height,
    pix_fmt: dec.pix_fmt.indexOf('422') >= 0 ? 'yuvj422p' : 'yuvj420p',
    time_base: [1, 1] });
  let jpegResult = await enc.encode(decResult.frames[0]); // Encode the frame
  await enc.flush(); // Tidy the encoder
  ctx.type = 'image/jpeg'; // Set the Content-Type of the data
  ctx.body = jpegResult.packets[0].data; // Return the JPEG image data
});

app.listen(3000); // Start the server on port 3000
```

### Scope

The developers created beam coder to enable development of highly-scalable frame-by-frame, packet-by-packet, web-fit clustered microservices and lambda functions. Beam coder enables the combination of media IO and processing functions with the comprehensive library of scalable IO modules for Node.js, such as [express](https://expressjs.com/), [koa](https://koajs.com/), [ioredis](https://www.npmjs.com/package/ioredis), [AWS SDK for Javascript](https://aws.amazon.com/sdk-for-node-js/) etc..

If you are looking to write your own frame-by-frame transcoder, media mangler or muxer, you are in the right place. However, if you want to control FFmpeg as a command line application over complete files or piped streams from a Node.js application, many other projects are available, such as [fluent-ffmpeg](https://www.npmjs.com/package/fluent-ffmpeg).

Does beam coder support X, Y or Z protocol / format / codec / file type / stream type / hardware etc.? If FFmpeg supports it, its possible and likely. Beam coder is self-describing at runtime, allowing exploration of what should be possible. However, you have to start somewhere, and the developers have been testing with the codecs and formats they are familiar with. Issues are expected so please raise any problems or requests for additional features as git hub issues. Even better, raise a pull request to fix a problem or add in missing features. Automated testing will be extended in due course, memory management improved, more examples added and hardware accelerated codecs added.

Beam coder will be a cross-platform module for Windows, Mac and Linux. In this early release, only Windows and Linux are available. The Mac platform will follow shortly. The release version of FFmpeg that beam coder links with is currently 4.1.

### Aerostat

Beam coder is the first release of Streampunk Media's [_Aerostat_](https://en.wikipedia.org/wiki/Aerostat) open-source product set, whereby a conceptual fleet of media-oriented _aerostats_ (_blimps_, _air ships_, _zeppelins_ etc.) are launched into the clouds. Media content is beamed between the fleet as if light beams, and beamed to and from locations on the planet surface as required. See also the [_Aerostat Beam Engine_](https://www.npmjs.com/package/beamengine).

## Installation

### Pre-requisites

Native packages require a build stage that needs some setup. This includes Python 2.7 (not 3) and build tools.

1. Install the LTS version of [Node.js](https://nodejs.org/en/) for your platform, currently the latest v10.
2. Enable [node-gyp - the Node.js native addon build tool](https://github.com/nodejs/node-gyp) for your platform by following the [installation instructions](https://github.com/nodejs/node-gyp#installation).

This release of Beam coder targets 64-bit (`x86_64`) architectures.

### Installing

#### Windows

Beam coder is intended to be used as a module/package from other Node.js packages and applications. Install beam coder from the root folder of your package as follows:

    npm install beamcoder

This will install all necessary dependencies, download the FFmpeg shared and dev packages with include and library files, and then compile the native extensions.

Note that if you want to use a local version of FFmpeg then, before the install, symbolic link or copy appropriate folders to:

    ./node_modules/beamcoder/ffmpeg/ffmpeg-4.1-win64-shared
    ./node_modules/beamcoder/ffmpeg/ffmpeg-4.1-win64-dev

To ensure that sufficient threads are available to process several requests in parallel, set the `UV_THREADPOOL_SIZE` environment variable, e.g.:

    set UV_THREADPOOL_SIZE 32

#### Linux

On Linux, use the appropriate package manager to install the FFmpeg 4.1 development dependencies first. An error will be printed if these cannot be found at expected locations. For example, on Ubuntu:

    sudo add-apt-repository ppa:jonathonf/ffmpeg-4
    sudo apt-get install libavcodec-dev libavformat-dev libavdevice-dev libavfilter-dev libavutil-dev libpostproc-dev libswresample-dev libswscale-dev

An example for Redhat / Fedora / CentOs will follow.

Beam coder is intended to be used as a module/package from other applications. Install beam coder from the root folder of your package as follows:

    npm install beamcoder

This will check for all the necessary dependencies and then compile the native extensions.

To ensure that sufficient threads are available to process several requests in parallel, set the `UV_THREADPOOL_SIZE` environment variable, e.g.:

    export UV_THREADPOOL_SIZE=32

#### Mac

To follow.

Note: For MacOSX _Mojave_, install the following package after `xcode-select --install`:

    /Library/Developer/CommandLineTools/Packages/macOS_SDK_headers_for_macOS_10.14.pkg

## Usage

### Introduction

The basic usage pattern is as follows:

1. Use promises to create processing stages for an item of media, for example a demuxer and a decoder. One stage can be used to configure the other (see [codec parameters](#codec-parameters)).
2. In a loop, use sequences of promises to move data through the processing stages, working with the data as required. For example, read some frames from a file and send them to a decoder, receiving uncompressed data for each frame. This data may be suitable for display or analysis.
3. Flush the buffers of each processing stage, close resources and collect any outstanding frames or packets.

Here is a simple example:

```javascript
const beamcoder = require('beamcoder');

async function run() {
  let demuxer = await beamcoder.demuxer('/path/to/file.mp4'); // Create a demuxer for a file
  let decoder = beamcoder.decoder({ name: 'h264' }); // Codec asserted. Can pass in demuxer.
  let packet = {};
  for ( let x = 0 ; x < 1000 && packet != null ; x++ ) {
    packet = await format.read(); // Read next frame. Note: returns null for EOF
    if (packet && packet.stream_index === 0) { // Check demuxer to find index of video stream
      let frames = await decoder.decode(packet);
      // Do something with the frame data
      console.log(x, frames.total_time); // Optional log of time taken to decode each frame
    }
  }
  let frames = await decoder.flush(); // Must tell the decoder when we are done
  console.log('flush', frames.total_time, frames.length);
}

run();
```

For mode examples, see the [`examples`](https://github.com/Streampunk/beamcoder/tree/master/examples) folder.

### Values

In design, each of the main structures (`struct`) of FFmpeg have mappings to Javascript objects. Values are automatically marshalled and unmarshalled between native and Javascript representations. These objects contain an internal instance of the native C structure, have a `type` property providing the type name, and use getters and setters to expose the underlying native properties. Wherever possible, the names of the properties are the same as those in the C library, meaning that [FFmpeg's doxygen documentation](https://www.ffmpeg.org/doxygen/4.1/index.html) can and should be used as a reference.

#### Factory

To create one of the main data types _packet_, _frame_, _demuxer_, _muxer_, _decoder_, _filterer_, or _encoder_, use the  methods of `beamcoder` with the same name e.g. `packet`, `frame`, etc. With no arguments, the value constructed has FFmpeg's default values. To construct a packet:

    let pkt = beamcoder.packet();

To configure the object on initialisation, pass it an options object:

```javascript
let q = beamcoder.packet({ pts: 10, dts: 10, stream_index: 3 });
console.log(q);
{ type: 'Packet',
  pts: 10,
  dts: 10,
  data: null,
  size: 0,
  stream_index: 3,
  /* ... */ }
```

Factory methods and their associated introspection methods that allow discovery of what are available are:

* `...demuxer()` - find [demuxer](#demuxing) input formats with `...demuxers()`.
* `...decoder()` - find [decoder](#decoding) codecs with `...decoders()`.
* `...filterer()` - find details of [filters](#filtering) with `...filters()`.
* `...encoder()` - find [encoder](#encoding) codecs with `encoders()`.
* `...muxer()` - find [muxer](#muxing) output formats with `...muxers()`.
* `...packet()` - create a data packet. See [creating packets section](#creating-packets).
* `...frame()` - construct frames, normally with a `format` property that is one of the `...pix_fmts()` or `...sample_fmts()`. See [creating frames section](#creating-frames).
* `...codecParameters` - create codec parameters for one of the `...codecs()`. See [codec parameters section](#codec-parameters) below.

Note some special cases to this rule, such as creating a _demuxer_ from a URL or filename. In this case, a single string URL parameter can be passed to the constructor rather than an options object. See the [demuxer section](#demuxer) for details.

#### Reading and modifying

After construction, Javascript properties of an object can be used in a natural way, with dot notation and array-style access:

```javascript
let q_pts = q.pts; // Current value of pts for q, q_pts= 10
let same = q['pts'] === q.pts; // true ... the current value of pts for q = 10
q.pts = 12; // Set the value pts for q to 12
q.data = Buffer.from('Text data for this packet.'); // Set packet data
```

This is achieved with [getters and setters](https://www.w3schools.com/js/js_object_accessors.asp). Note that not every openly readable (_enumerable_) property is also writable. This may depend on the current context. For example, some properties are set only by _libav*_ and not by the user. Others can only be updated by _libav*_ during encoding, even when a parameter of the same name could be set by the user when decoding.

The processing work to convert a value from C to Javascript is only done when each separate property is requested. Bear this in mind before being too liberal with, say, `console.log()` that enumerates through every property of an object. Encoders and decoders have approximately 130 properties!

#### Freeing and deleting

Care has been taken to ensure that the reference-counted then garbage collected data structures of Javascript work in tandem with the allocation and free mechanisms of _libav*_. As such, there is no explicit requirement to free or delete objects. As with any Javascript application, if you hold onto references to objects when they are no longer required, garbage collection is prevented and this may have a detrimental impact on performance. This is particularly the case for frames and packets that hold references to large data buffers.

#### Type mappings

Property value mappings from C to Javascript and vice versa are as follows:

* C `int`, `int64_t` and other integer types map to and from Javascript's `number` type. Try to use integer representations in Javascript wherever possible, e.g. use `|0` when dividing. (Note that the full ranges of C `int64_t` and `uint64_t` are not yet supported by beam coder. This feature is waiting on Javascript `BigInt` to move from experimental support to full support in Node.)
* C `float` and `double` types map to and from Javascript's `number` type. Note that as the underlying representation of floating point values is double length in Javascript, causing conversion to and from C `float` to lose precision. It is best practice to test for small value ranges around a value rather than rely on exact values.
* `AVRational` values map to and from a Javascript array containing two integer `number`s. For example, C value `(AVRational){1,25}` maps to Javascript value `[1,25]`.
* C `char *` and `const char *` map to and from Javascript's `string` type.
* FFmpeg enumerations are converted to Javascript strings representing their name. FFmpeg provides its own utility functions for converting many of its enumerations to and from strings, for example [`av_color_space_name`](https://www.ffmpeg.org/doxygen/4.1/pixdesc_8c.html#a7a5b3f4d128f0a0112b4a91f75055339) and [`av_color_space_from_name`](https://www.ffmpeg.org/doxygen/4.1/pixdesc_8c.html#a0c7d0a9d7470c49397a72e1169d2a8e3).
* FFmpeg flags are converted to and from a Javascript object of Boolean-valued properties, where each property is named to match the distinguishing part of the FFmpeg `#define` macro name. For example, the _corrupt_ and _discard_ flags of an `AVFrame` `flags` property become `{ CORRUPT: false, DISCARD: false }` in Javascript. When setting, use an object, e.g. to update the CORRUPT flag, do `frame.flags = { CORRUPT: true };`.
* `AVBufferRef` and other blobs of binary data are represented by [Node.js Buffers](https://nodejs.org/docs/latest-v10.x/api/buffer.html). Wherever possible, the data is not copied. For example, when reading an `AVBuffer` from Javascript, a new reference to the underlying `AVBuffer` is created and a view of the raw data made available via an external buffer. The reference is held until the Javascript buffer is garbage collected. Conversely, when a Javascript buffer is to be read from an FFmpeg structure, the `AVBufferRef` structure holds a V8 reference to the underlying data of the Javascript buffer. The Javascript buffer is kept alive until both of any AV processing functions and Javascript are finished and the buffers are unreferenced.
* Binary data blobs of type `uint8_t *`, such as `extradata` or `AVPacketSideData.data` are assumed to be small and easy to copy. Javascript getters make a copy of the underlying data and return a Buffer. The setters create a copy of the data in a buffer and use it to set the underlying value. Therefore, to modify the data, it must be read via the getter, modified and written back via the setter.
* `AVDictionary` metadata and private data values have a natural mapping to Javascript objects as keys to property names and their value pair. Always set dictionary-based and private data values using an object. Dictionary values are replaced in their entirety. For private data, only the properties contained in the update object will be modified.

### Demuxing

The process of demuxing (de-multiplexing) extracts time-labelled packets of data contained in a media stream or file. FFmpeg provides a diverse range of demuxing capability with support for a wide range of input formats and protocols (`beamcoder.protocols()`).

To see a list and details of all the available demuxer input formats:

    let dms = beamcoder.demuxers();

The output is an object where each property key is the name of a demuxer and each value is an object describing the input format. For example, to find the demuxer for `.mp4` files:

```javascript
Object.values(dms).filter(x => x.extensions.indexOf('mp4') >= 0);
[ { type: 'InputFormat',
    name: 'mov,mp4,m4a,3gp,3g2,mj2',
    long_name: 'QuickTime / MOV',
    mime_type: '', // Generally not available for demuxers - see equivalent muxers
    extensions: 'mov,mp4,m4a,3gp,3g2,mj2',
    flags:
     { NOFILE: true, // Demuxer will manage IO operations
       NEEDNUMBER: false, // Needs '%d' in filename
       SHOW_IDS: true, // Show format stream IDs numbers.
       GENERIC_INDEX: true, // Use generic index building code
       TS_DISCONT: true, // Format allows timestamp discontinuities.
       NOBINSEARCH: true, // Format does not allow to fall back on binary search via read_timestamp
       NOGENSEARCH: true, // Format does not allow to fall back on generic search
       NO_BYTE_SEEK: false, // Format does not allow seeking by byte
       SEEK_TO_PTS: false }, // Seeking is based on PTS
    raw_codec_id: 0,
    priv_data_size: 400,
    priv_class: null } ]
```

The easiest way to create a demuxer is with a filename or URL, for example to open a transport stream containing and HEVC encoding of the [Big Buck Bunny](https://peach.blender.org/) short movie in file [`bbb_1080p_c.ts`](https://www.elecard.com/storage/video/bbb_1080p_c.ts) in a `movie` sub-directory:

    let tsDemuxer = await beamcoder.demuxer('file:media/bbb_1080p_c.ts');

The `demuxer` operation performs file system and/or network access and so is asynchronous. On successful resolution after the contents of the file or stream has been probed, the value is a Javascript object describing the contents of the media input. Here is a summary of the created demuxer:

```javascript
{ type: 'demuxer',
  iformat:
   { type: 'InputFormat',
     name: 'mpegts',
     \* ... *\ },
  ctx_flags: { NOHEADER: false, UNSEEKABLE: false },
  streams:
  [ { type: 'Stream',
      index: 0,
      id: 301,
      time_base: [ 1, 90000 ], /* ... */
      codecpar:
       { type: 'CodecParameters',
         codec_type: 'video',
         codec_id: 173,
         name: 'hevc',
         codec_tag: 'HEVC',
         format: 'yuv420p', /* ... */
         width: 1920,
         height: 1080,
        /* ... */ } },
    { type: 'Stream',
      index: 1,
      id: 302,
      time_base: [ 1, 90000 ], /* ... */
      codecpar:
       { type: 'CodecParameters',
         codec_type: 'audio',
         codec_id: 86018,
         name: 'aac', /* ... */
         format: 'fltp', /* ... */
         channel_layout: 'stereo',
         channels: 2,
         /* ... */ } } ],
  url: '../media/bbb_1080p_c.ts',
  start_time: 80000,
  duration: 596291667,
  bit_rate: 2176799, /* ... */
  stream: [Function],
  read: [Function: readFrame],
  seek: [Function: seekFrame] }
```

From this object dump, it is possible to determine that the file contains two streams:

1. an HD video stream encoded with H.265/HEVC;
2. a stereo audio stream encoded with AAC.

The `duration` property is measured in microseconds (`AV_TIME_BASE`), so is approximately 596 seconds or 9m56s.

For formats that require additional metadata, such as the [`rawvideo` format](https://ffmpeg.org/ffmpeg-formats.html#toc-rawvideo), it may be necessary to pass additional information such as image size or pixel format. To do this, pass in an options object with a `url` property for the filename(s) (may contain `%d` for a sequence of numbered files) and `options` property for the values. For example:

```javascript
let rawDemuxer = await beamcoder.demuxer({
  url: 'file:movie/bbb/raw_pictures_%d.rgb',
  options: {
    video_size: '640x480',
    pixel_format: 'rgb24'
  }
});
```

#### Reading data packets

To read data from the demuxer, use the `read` method of a demuxer-type object, a method that takes no arguments. This reads the next blob of data from the file or stream at the current position, where that data could be from any of the streams. Typically, a packet is one frame of video data or a data blob representing a codec-dependent number of audio samples. Use the `stream_index` property of returned packet to find out which stream it is associated with and dimensions including height, width or audio sample rate. For example:

```javascript
let wavDemuxer = await beamcoder.demuxer('file:my_audio.wav');
let packet = {};
while (packet != null) {
  packet = await wavDemuxer.read();
  if (packet &&
      wavDemuxer.streams[packet.stream_index].codecpar.codec_type === 'video') {
    // Do something with the image data
  }
}
```

The read method is asynchronous and returns a promise. The promise resolves to an object of type `Packet` if it succeeds with data, or a `null` value at the end of the file. If an error occurs, the promise rejects with an error message. An example of a successful read from a WAVE file is shown below:

```javascript
{ type: 'Packet',
  pts: 2792448, // presentation timestamp, measured in stream timebase
  dts: 2792448, // decode timestamp, measured in stream timebase
  data: // the raw data of the packet
   <Buffer fd ff fe ff 02 00 01 00 ff ff 00 00 00 00 ff ff 00 00 02 00 02 00 fe ff fd ff 01 00 04 00 00 00 fb ff ff ff 04 00 01 00 fe ff ff ff 01 00 01 00 00 00 ... >,
  size: 4160, // the size of the raw data
  stream_index: 0, // the stream index of the stream this packet belongs to
  flags:
   { KEY: true, // Packet represents a key frame
     CORRUPT: false, // Corruption detected
     DISCARD: false, // Can be dropped after decoding
     TRUSTED: false, // Packet from a trusted source
     DISPOSABLE: false }, // Frames that can be discarded by the decoder
  duration: 1024, // Wrt the stream timebase
  pos: 11169836 } // Byte offset into the file
```

#### Seeking

Beam coder offers FFmpeg's many options for seeking a particular frame in a file, either by time reference, frame count or file position. To do this, use the `seek` method of a demuxer-type object with an options object to configure the operation.

To seek forward to a keyframe in a given stream or file at a given timestamp:

    await demuxer.seek({ stream_index: 0, timestamp: 2792448 });

The timestamp is the presentation timestamp of the packet measured in the timebase of the stream, which is `[1, 48000]` in this example. To seek based on elapsed time from the beginning of the primary stream (as determined by FFmpeg, normally the first video stream where available), use the `time` property:

    await demuxer.seek({ time: 58.176 });

Another form of seek is to use a byte offset position into the file:

    await demuxer.seek({ pos: 11169836 });

The final form of seeking supported is by number of frames into a given stream:

    await demuxer.seek({ frame: 42, stream_index: 0});

All successful seek calls resolve to a `null` value, or reject if the kind of seek is not supported or an error occurs. To access the seeked-for data, call `read` to get the next frame. Note that if you seek beyond the end of the file or stream, the seek call resolves OK and the next read operation resolves to `null`.

The seek operation has two additional flags that can be specified. The `backward` Boolean-valued property can be used to enable seeking backwards where supported. The `any` Boolean-valued property enables seeking to both key and non-key frames. For example:

    await demuxer.seek({ frame: 31, stream_index: 0, backward: true, any: true});

#### Node.js streams

Beam coder offers a [Node.js Writable stream](https://nodejs.org/docs/latest-v10.x/api/stream.html#stream_writable_streams) interface to a demuxer, allowing source data to be streamed to the demuxer from a file or other stream source.

To create a Writable stream interface with for example a 64kbyte threshold, use:

    let demuxerStream = beamcoder.demuxerStream({ highwaterMark: 65536 });

This stream can then have data written to it or piped from a source stream

    fs.createReadStream('file:media/bbb_1080p_c.ts').pipe(demuxerStream);

Once the source data is connected to the stream, the demuxer can be created:

    let demuxer = await demuxerStream.demuxer();

This function will return a promise that will resolve when it has determined sufficient format details by consuming data from the source. The promise will wait indefinitely until sufficient source data has been provided.

### Decoding

Decoding is the process of taking a stream of compressed data in the form of _packets_ and converting it into uncompressed _frames_. In general, to decode an interleaved (multiplexed) media file, you need a decoder for each of the video and the audio streams. For the purpose of keeping the examples simple in this section, only a single stream is decoded. However, it is possible to set up more than one decoder - say for related video and audio streams - and run them asynchronously and in parallel, i.e. to decode the video and audio required to present a frame.

To see a list of available decoders, use:

    let decs = beamcoder.decoders();

As with the demuxers, the result is an object where the keys are the names of the decoders and the values are objects describing the codec. This includes the codec type (`video`, `audio`, `subtitle`), a _descriptor_ for the family of codecs, some capability flags, supported profiles and more. Here are some examples of querying the available decoders:

```javascript
// Find a decoder that deals with H.264 / AVC
decs['h264'];
{ type: 'Codec',
  name: 'h264',
  long_name: 'H.264 / AVC / MPEG-4 AVC / MPEG-4 part 10',
  codec_type: 'video',
  id: 27, /* ... */ }

// Find all other decoders that can decode H.264
Object.values(decs).filter(x => x.id === 27).map(x => x.name);
[ 'h264', 'h264_qsv', 'h264_cuvid' ]
// Note: h264_qsv and h264_cuvid are wrappers to hardware-accelerated decoders
//       Appearance in the list does not imply that the codec is supported by current hardware.

// Find all decoders claiming support for the MP3 format
Object.values(decs)
  .filter(x => x.long_name.indexOf('MP3') >= 0)
  .map(x => ({ name: x.name, long_name: x.long_name }));
[ { name: 'mp3float', long_name: 'MP3 (MPEG audio layer 3)' },
  { name: 'mp3', long_name: 'MP3 (MPEG audio layer 3)' },
  { name: 'mp3adufloat',
    long_name: 'ADU (Application Data Unit) MP3 (MPEG audio layer 3)' },
  { name: 'mp3adu',
    long_name: 'ADU (Application Data Unit) MP3 (MPEG audio layer 3)' },
  { name: 'mp3on4float', long_name: 'MP3onMP4' },
  { name: 'mp3on4', long_name: 'MP3onMP4' } ]

// List all audio decoders
Object.values(decs).filter(x => x.codec_type === 'audio').map(x => x.name);
[ 'comfortnoise', 'dvaudio', '8svx_exp', '8svx_fib', 's302m', 'sdx2_dpcm',
  'aac', 'aac_fixed', 'aac_latm' /* ... */ ]
```

See the [FFmpeg Codecs Documentation](https://ffmpeg.org/ffmpeg-codecs.html) for more details.

#### Decoder

To create an instance of a decoder, request a `decoder` from beam coder, specifying either the decoder's `name`, a `codec_id`, or by providing a `demuxer` and a `stream_id`. For example:

```javascript
// Create a decoder by name - note this is synchronous
let decoder = beamcoder.decoder({ name: 'h264' });
// or for the first choice codec in the H.264 family
let decoder = beamcoder.decoder({ codec_id: 27 });
// Alternatively, use a demuxer and a stream_index
//   The codec parameters of the streams will be used to set the decoding parameters
let tsDemux = await beamcoder.demuxer('media/bbb_1080p_c.ts');
let decoder = beamcoder.decoder({ demuxer: tsDemux, stream_index: 0 });
```

Other properties of the decoder can be provided on initialisation and may be required in certain cases. For example, the `width` and `height` of the video. These will override the default values.  Note that certain values for the decoder may not be available until a the first few packets have been decoded.

```javascript
let decoder = beamcoder.decoder({ name: 'h264', width: 1920, height: 1080 });
```

A decoder has many properties. These can be set before decoding in the usual way for a Javascript object. Some of the properties are more appropriate for encoding but are present for information. Some properties can only be set by _libav*_, others can only be set by the user, some both. Follow the [AVCodecContext FFmpeg documentation](http://ffmpeg.org/doxygen/4.1/structAVCodecContext.html) for details.

#### Decode

To decode an encoded data _packet_ and create an uncompressed _frame_ (may be a frames-worth of audio), use the asynchronous _decode_ method of a decoder. Decoders may need more than one packet to produce a frame and may subsequently produce more than one frame per packet. This is particularly the case for _long-GOP_ formats.

```javascript
let packet = demuxer.read();
while (packet != null) {
  let dec_result = await decoder.decode(packet);
  // dec_result.frames - possibly empty array of frames for further processing
  // dec_result.total_time - log the microseconds that the operation took to complete
  // Get the next packet, e.g. packet = demuxer.read();
}
```

As long as decoding was successful, the decode operation resolves to an object containing the `total_time` (measured in microseconds) that the operation took to execute, and an array of decoded `frames` that are now available. If the array is empty, the decoder has buffered the packet as part of the process of producing future frames. Frames are delivered in presentation order.

It is possible to pass more than one packet at a time to the decoder, either as an array of packets or a list of arguments:

```javascript
// Array of packets
let dec_result = await decoder.decode([packet1, packet2, packet3 /* ... */ ]);
// List of packets as arguments
let dec_result = await decoder.decode(packet1, packet2, packet3 /* ... */ );
```

#### Creating packets

Packets for decoding can be created without reading them from a demuxer. For example:

    beamcoder.packet({ pts: 43210, dts: 43210, data: Buffer.from(...) });

Packet data buffers are shared between C and Javascript so can be written to and modified without having to write the buffer back into the packet.

#### Flush decoder

Once all packets have been passed to the decoder, it is necessary to call its asynchronous `flush()` method. If any frames are yet to be delivered by the decoder, they will be provided in the resolved value.

```javascript
let flush_result = await decoder.flush();
// flush_result.frames - array of any remaining frames to be decoded
// flush_result.total_time - microseconds taken to execute the flush operation
```

Call the flush operation once and do not use the decoder for further decoding once it has been flushed. The resources held by the decoder will be cleaned up as part of the Javascript garbage collection process, so make sure that the reference to the decoder goes out of scope.

### Filtering

___Simon___

### Encoding

Encoding is the process of taking a stream of uncompressed data in the form of _frames_ and converting them into coded _packets_. Encoding takes place on a single type of stream, for example audio or video. Streams maybe combined downstream by a _muxer_.

To see a list of available encoders, use:

    let encs = beamcoder.encoders();

The encoders are listed in with the same form and structure as the decoders and can be queried in the same way.

```javascript
// Find all encoders that can encode H.264
Object.values(encs).filter(x => x.id === 27).map(x => x.name);
[ 'libx264', 'libx264rgb', 'h264_amf', 'h264_nvenc', 'h264_qsv', 'nvenc', 'nvenc_h264' ]
etc..
```

#### Encoder

To create an instance of an encoder, request an `encoder` from beam coder, specifying either the codec's `name`, or a `codec_id`.

```javascript
// Create an encoder by name of name of a codec family
let v_encoder = beamcoder.encoder({ name: 'h264' });
// In this case, the 'libx264' software codec will be selected.

// Create an encoder by codec identifier - in this case for 'aac' audio
let a_encoder = beamcoder.encoder({ codec_id: 86018 });
```

Additional properties may be set on the encoder object either during construction in the options object or using setters. See the [AVFormatContext documentation from FFmpeg](http://ffmpeg.org/doxygen/4.1/structAVCodecContext.html) for details of the parameters. In general, the user sets up all the configuration parameters prior to starting the encode process. Note hat _libav*_ may set some of the parameters during encoding.

Each codec may have additional private data settings and these are available in the encoder's `priv_data` property. These can be read and set from Javascript but note that the value can only be set by providing a complete value. For example, to set the `libx264` encoder to the `slow` preset:

    v_encoder.priv_data = { preset: 'slow' }; // Only included properties change

The following will not work:

    v_encoder.priv_data.preset = 'slow';

#### Encode

To encode an uncompressed _frame_ and create a compressed _packet_ (may be a frames-worth of audio), use the _encode_ method of an encoder. Encoders may need more than one frame to produce a packet and may subsequently produce more than one packet per frame. This is particularly true for _long-GOP_ formats.

```javascript
// Get or make the first frame of data, store in frame
while (frame != nullptr) {
  let enc_result = await encoder.encode(frame);
  // enc_result.packets - possible empty array of packets for further processing
  // enc_result.total_time - log the microseconds that the operation took to complete
  // Get or make the next frame of data
}
```

#### Creating frames

Frames for encoding or filtering can be created as follows:

    beamcoder.frame({ pts: 43210, width: 1920, height: 1920, format: 'yuv420p',
      data: [ Buffer.from(...plane_1...), Buffer.from(...plane_2...), ... ] });

In general, frame data is planar, e.g. split into separate buffers for _Red_ (`R`), _Green_ (`G`) and _Blue_ (`B`) components or _Chrominance_ (`Y`) and _Luminance_ (`UV`) components. _Alpha_ (`A`) channel, if present, is always the last component. For details of the planes expected for each pixel format, call:

    beamcoder.pix_fmts();

Planar audio representations - those with a `p` in their name - use planes to represent data for each audio channel. For details of sample formats, call:

    beamcoder.sample_fmts();

Beam coder exposes some of FFmpeg's ability to calculate the size of data buffers. If you pass `width`, `height` and `format` properties for video frames, or `channels`/`channel_layout` and `format` frames, as options to the frame constructor, the `linesize` array (number of bytes per line per plane) is computed. For video, multiply each value by the height to get the minimum buffer size for the plane. For audio, the first element of the array is the buffer size for each plane.

To use the linesize numbers to automatically allocate buffers of the correct size, call `alloc()` after the factory method. For example:

    let f = beamcoder.frame({ width: 1920, height: 1080, format: 'yuv422p' }).alloc();

Note that when creating buffers from Javascript, FFmpeg recommends that a small amount of headroom is added to the length. The minimum amount of padding is exposed to Javascript as constant:

    beamcoder.AV_INPUT_BUFFER_MIN_SIZE

#### Flush encoder

Once all frames have been passed to the encoder, it is necessary to call the asynchronous `flush()` method. If any packets are yet to be fully encoded or delivered by the encoder, they will be completed and provided in the resolved value.

```javascript
let flush_result = await encoder.flush();
// flush_result.frames - array of any remaining packets to be encoded
// flush_result.total_time - microseconds taken to execute the flushing process
```

Call the flush method once and do not use the encoder for further encoding once it has been flushed. The resources held by the encoder will be cleaned up as part of the Javascript garbage collection process, so make sure that the reference to the encoder goes out of scope.

### Muxing

Muxing (multiplexing) is the operation of interleaving media data from single streams into a single file or stream, the opposite process to demuxing. In its simplest form, a single stream is written to a file, adding any necessary headers, padding or trailing data according to the file format, for example writing a WAVE file involves writing a header followed by the PCM audio data.

For more complex formats, such as an MP4 file, video and audio data packets with similar timestamps are written to the same part of the output file. When the file is subsequently read, video and audio that needs to be presented together can be read from the same part of the file.

The muxing procedure has five basic stages:

1. Choosing the output format and setting up the output stream metadata.
2. Opening the file or streams for writing.
3. Writing the header metadata.
4. In a loop, writing packets and/or frames.
5. Writing the trailer and closing the file or stream.

Here is an example of reading and writing a WAV file as an illustration:

```javascript
let demuxer = await beamcoder.demuxer('file:input.wav');
let muxer = beamcoder.muxer({ filename: 'file:test.wav' });
let stream = muxer.newStream(demuxer.streams[0]);
await muxer.openIO();
await muxer.writeHeader();
let packet = {};
for ( let x = 0 ; x < 100 && packet !== null ; x++ ) {
  packet = await demuxer.read();
  await muxer.writeFrame(packet);
}
await muxer.writeTrailer();
```

#### Output format

A first stage of creating a muxer is to select an output format. The complete list of output formats can be queried as follows:

    let mxs = beamcoder.muxers();

Each muxer has an `extensions` property that is a comma-separated list commonly used file extensions for the output format. This can be searched, or alternatively beam coder exposes FFmpeg's capability to guess an output format from a filename, format name or mime type. For example:

    of = beamcoder.guessFormat('mpegts'); // Finds the output format named 'mpegts'
    of = beamcoder.guessFormat('image/png'); // Output format for PNG image data
    of = beamcoder.guessFormat('fred.jpeg'); // Output format for '.jpeg' file extension

To create a muxer, user an options object and do one of the following:

* set a `format_name` or `name` property to that of an output format;
* set a `filename` property to a filename with extension to be used to guess the output format;
* set an `oformat` property to an output format object retrieved with, say, `beamcoder.muxers()` or `beamcoder.guessFormat()`.

    muxer = beamcoder.muxer({ format_name: 'mp4' });
    muxer = beamcoder.muxer({ filename: 'example.wav' });
    muxer = beamcoder.muxer({ oformat: of });

For details of the properties of a `muxer`, see the [FFmpeg AVFormatContext documentation](http://ffmpeg.org/doxygen/4.1/structAVFormatContext.html).

The next step is to add the streams for the format. This can only be done using the `newStream()` method of a muxer and must be done in order. The steps are:

1. Create the stream with a codec name.
2. Set the `time_base` for the stream.
3. Update the codec parameters for the stream, ensuring at least `width`, `height` and `format` (pixel format) are set for video streams and `sample_rate`, `channels` and/or `channel_layout` and `format` (sample format) are set of audio streams.

This process is illustrated in the following code:

```javascript
let muxer = beamcoder.muxer({ filename: 'test.wav' }); // Filename set muxer's 'url' property
let stream = muxer.newStream({
  name: 'pcm_s16le',
  time_base: [1, 48000 ],
  interleaved: false }); // Set to false for manual interleaving, true for automatic
Object.assign(stream.codecpar, { // Object.assign copies over all properties
  channels: 2,
  sample_rate: 48000,
  format: 's16',
  channel_layout: 'stereo',
  block_align: 4, // Should be set for WAV
  bits_per_coded_sample: 16,
  bit_rate: 48000*4
});
```

Note that the stream is added to the muxer automatically with the next available stream index. A reference to the stream is returned by `newStream()` for convenience. For details of the properties of stream, see FFmpeg's [AVStream](http://ffmpeg.org/doxygen/4.1/structAVStream.html) and [AVCodecParameters](http://ffmpeg.org/doxygen/4.1/structAVCodecParameters.html) documentation.

An alternative way to create muxers, e.g. for a transmuxing process (direct connection from demuxer to muxer to rewrap content with transcoding), a new muxer stream can be created with a demuxer. An output format will be found to match the input format and the streams and their codec parameters will be copied over. For example:

    let muxAudioStream = muxer.newStream(demuxer.streams[1]);

#### Opening the output

The next step is to open the output file or stream. If a filename or URL has been provided with the creation of the muxer using the `filename` property, the output can be opened with the asynchronous `openIO()` method of the muxer.

Alternatively, pass in an options object containing the `filename` or `url`. The options object can also contain additional private data `options` property, an object, to further configure the protocol. See the [FFmpeg protocol documetation](https://ffmpeg.org/ffmpeg-protocols.html) for information about protocol-specific options.

In the simplest case, where a filename was provided when the muxer was created, simply call the method without any arguments:

    await muxer.openIO();

An example of opening the muxer's output for a WAVE file with options:

```javascript
await muxer.openIO({
  url: 'file:test2.wav', // Update URL
  options: { // Protocol private options
    blocksize: 8192
  }
});
```

On success, the returned promise resolves to `undefined` or, if some of the options could not be set, an object containing an `unset` property detailing which of the properties could not be set. The options can now be viewed and modified through the `priv_data` property of the muxer.

#### Writing the header

The next stage is to write the header to the file. This must be done even for formats that don't have a header as part of the internal process is to initialize the internal data structures for writing. This is as simple as calling the asynchronous `writeHeader()` method of the muxer:

    await muxer.writeHeader();

As with `openIO()`, it is possible to pass in private data options at this stage:

```javascript
await muxer.writeHeader({
  options: { // Private class options
    write_peak: 'on', // Write Peak Envelope chunk - enum of 'off', 'on', 'only'.
    peak_format: 2, // The format of the peak envelope data (1: uint8, 2: uint16).
    write_bext: false // Write BEXT chunk.
  }
});
```

For details of the available options per format, see the associated `OutputFormat` object which is the `oformat` property of the muxer. Specifically:

    console.log(muxer.oformat.priv_class.options);

In some cases, it is necessary to initialize the structures of the muxer before writing the header. In this case, call the asynchronous `initOutput()` method of the muxer. This method can also take `options` to initialise muxer-specific parameters. Further configure the initialized muxer and then call `writeHeader()` before writing any packets or frames.

On success, the `writeHeader()` and `initOutput()` methods both resolve to an object that states where the initialisation of the muxer took place and any of the options that were `unset`. For example:

    { INIT_IN: 'WRITE_HEADER', unset: { write_bext: false } }

#### Writing packets and frames

To write the actual media data to the file, it is necessary to send frames and packets containing the data of the media streams, sub-divided into packets or frames. Packets must contain their stream index, frames must be sent with their stream index and both must have timestamps in the timebase of their stream. (With the exception of streams that don't have a timebase, in which case set `beamcoder.AV_NOPTS_VALUE`.)

The muxer has an `interleaved` property that determines whether beam coder asks FFmpeg does the interleaving of streams or whether the user is taking charge. Set the property to `true` for FFmpeg to do the work, otherwise set it to `false`. Set this property by now and don't change it. From here on in you can only do one or the other.

Send packets or frames from across all the streams with similar presentation timings in the stream in a round robin-like loop, sending more packets per iteration for streams where the packets/frames have shorter durations. In interleaving mode, FFmpeg builds up a buffer of time-related packets and writes them when it has sufficient information to cover a given time segment.

Packets or frames are sent to the muxer with its asynchronous `writeFrame()` method. The method expects a single argument that is either a packet or an options object. If an options object, provide either a single `packet` property, or a `frame` and a `stream_index` property. Here are some examples:

    await muxer.writeFrame(packet);
    await muxer.writeFrame({ packet: packet });
    await muxer.writeFrame({ frame: frame, stream_index: 0 });

The `writeFrame()` promise resolves to `undefined` on success, otherwise the promise rejects with an error.

#### Writing the trailer

The trailer is the end of the file or stream and is written after the muxer has drained its buffers of all remaining packets and frames. Writing the trailer also closes the file or stream. Use the asynchronous `writeTrailer()` method. It takes no arguments:

    await muxer.writeTrailer();

The promise resolves to an `undefined` value. Do not try to write other data to the muxer after calling this method. Any other resources held by the muxer will be released by Javascript garbage collection.

To abandon the muxing process and forcibly close a file or stream without completing it, call the synchronous `forceClose()` method. This assumes that any result of the muxing process is to be left in an incomplete state.

#### Node.js writable streams

___Simon___ ?

### Codec parameters

Another mechanism of passing parameters from demuxers to decoders to encoders and then muxers is to use _codec parameters_. These are a set of parameters that can be used to uniquely identify and represent the kind of codec of a stream and its dimensions and properties.  

## Status, support and further development

Although the architecture of the aerostat beam coder is such that it could be used at scale in production environments, development is not yet complete. In its current state, it is recommended that this software is used in development environments, primarily for building prototypes. Future development will make this more appropriate for production use.

The developers of beam coder aimed to find a balance between being a faithful mapping of FFmpeg to Javascript while creating a Javascript API that is useful and easy to use. This may mean that certain features of FFmpeg are not yet exposed or choices have been made that lead to sub-optimal performance. Areas that are known to need further development and optimisation include:

* imporved shared memory management between Javascript and C, specifically adding support for pools;
* hardware acceleration.

Contributions can be made via pull requests and will be considered by the author on their merits. Enhancement requests and bug reports should be raised as github issues. For support, donations or to sponsor further development, please contact [Streampunk Media](http://www.streampunk.media/).

## License

This project is licensed under the GNU General Public License, version 3 or later. Copyright (C) 2019, Streampunk Media Ltd.

This software links to libraries from the FFmpeg project, including optional parts and optimizations covered by the GPL v2.0 or later. Your attention is drawn to the FFmpeg project's page [FFmpeg License and Legal Considerations](https://www.ffmpeg.org/legal.html) and the Copyright notices of the FFmpeg developers.

This project is in no way part of or affiliated with the FFmpeg project.

### Acknowledgements

A massive thank you to the FFmpeg development team who's tireless and ongoing work make this project - and so many other media software projects - possible.
