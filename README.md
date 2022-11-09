# eznve
Easy C++ API for creating compressed video with NVENC from CUDA generated frames. It is designed for this use case:
1. You're generating frames in CUDA and writing them to an RGBA image in device memory.
2. You want to take each frame and feed it into a H264/HEVC encoder and do something with the output.
3. You don't want to fuss around too much with configuration or import giant libraries.

## Basic usage
```cpp
#include <eznve.hpp>
/* ... */
CUcontext ctx = example::make_cuda_context();
auto out_file = std::ofstream{"hello.h264", std::ios::binary};
auto encoder = 
    eznve::encoder(
        {1280, 720}, // {width, height}
        {30, 1}, // frames per second in {numerator, denominator}
        eznve::codec::h264, // h264 or hevc 
        ctx, // a cuda context for the encoder

        // pass in a callback that will be invoked when a submitted
        // frame causes stream data to be generated.
        [&out_file](const eznve::encode_info& info) {
            out_file.write(info.data.data(), info.data.size());
        }    
    );

while(should_make_frames()) {
    generate_cool_frame(encoder.buffer()); // write to encoder's input buffer
    if(encoder.submit_frame()) { // encoder returns true if it invoked the write callback
        notify_something_data_was_written();
    }
}
encoder.flush(); // close out the encoder
out_file.close();
```

## Obtaining
`eznve` is designed for CMake FetchContent. Just add this in your CMakeLists:
```cmake
FetchContent_Declare(
  eznve
  GIT_REPOSITORY https://github.com/untrioctium/eznve.git
)
FetchContent_MakeAvailable(eznve)
```
Then link against `eznve` in your application and include `eznve.hpp`

## IDR frames
`submit_frame()` can accept `frame_flag::idr`, which forces the frame to be an IDR frame. This essentially puts the frame in as is, and the encoder will track future changes based on that frame. The presence of IDR frames allows efficient skipping through the generated video, as all changes since the last IDR frame must be calculated when seeking. It also increases quality and can reduce visual artifacts. On the flip side, this means that the generated stream will be larger as it will contain fewer compressed frames. A good balance is to submit a frame as IDR for every 1-5 seconds of video depending on your needs. The first frame in a stream is always IDR.

## Using the generated stream
Out of the box, the generated stream is not very useful. You could dump it to disk and play it with VLC, but it will likely be choppy. The stream must be muxed, which is currently outside the scope of this project. You can mux the stream using software or libraries like [FFmpeg](https://ffmpeg.org/) or [Bento4](https://www.bento4.com/). Both of these provide both standalone applications and APIs that could be used to mux inside or outside of your application. If you are streaming to the web, [jMuxer](https://github.com/samirkumardas/jmuxer) can directly consume the output  data, perhaps sent to it via a WebSocket, and mux it on the fly in an HTML5 video element, which is the current use case of this library.

## eznve API
```cpp
// struct used in data write callback to provide information
// on the encoded chunk
struct chunk_info {
    // a span containing the encoded chunk. do not use this
    // pointer outside of the write callback. copy or immediately
    // use the pointed to data.
    std::span<const char> data;

    // information about the encoded chunk
    uint32_t index;
    uint64_t timestamp;
    uint64_t duration;
}

class encoder {
public:
    // Construct a new encoder
    // dims - {width, height} of encoded video
    // fps - {numerator, denominator} of FPS of encoded video. {60, 1} is 60fps, {100, 3} would be 33.33...fps
    // codec - eznve::codec::h264 or eznve::codec::hevc
    // ctx - a CUDA context to use 
    // cb - a function or function object of signature void(const chunk_info&).
    //      called when submit_frame has enough frames to generate a chunk.
    encoder(uint2 dims, uint2 fps, codec c, CUcontext ctx, data_callback_t&& cb);

    // destroys the encoder. if you have not called flush() and there
    // is still state in the encoder, it will be called for you. this
    // may result in the write callback being invoked. 
    ~encoder();

    // encoder cannot be copied
	encoder(const encoder&) = delete;
	encoder& operator=(const encoder&) = delete;

    // encoder can be moved
    encoder(encoder&&) noexcept = default;
	encoder& operator=(encoder&&) = default;

    // submit a frame stored in the device memory of buffer()
    // frame_flag - you may pass frame_flag::idr to force an IDR frame
    //
    // returns true if a chunk was generated and the write callback was invoked.
    // a value of false may be possible if the encoder needs more frames
    bool submit_frame(frame_flag = frame_flag::none);

    // closes out the encoder, effectively resetting it.
    // call this before ending the session to ensure that anything
    // left gets encoded and emitted. this may call the write callback.
    // like submit_frame(), it will return true if a chunk was emitted
    //
    // after this function is called. it's as if you've constructed a new
    // encoder, and you may reuse it for another encoding task.
    void flush();

    // changes the write callback
    void set_callback(data_callback_t&& cb) noexcept;

    // returns the device pointer where the encoder expects to find a frame
    // after you have called submit_frame(). this memory is managed by the
    // encoder, so do not free or otherwise use it other than writing to it.
    //
    // the format is normal 32 bits per pixel interleaved RGBA
    CUdeviceptr buffer() const noexcept;

    // these functions return the configured video dimensions
    unsigned int width() const noexcept;
    unsigned int height() const noexcept;

    // returns the FPS in decimal form. less precise than using the actual
    // numerator/denominator format, but fine for quick math.
    double fps() const noexcept;

    // provides the FPS in {numerator, denominator} form
    uint2 fps_exact() const noexcept;

    // returns the number of bytes produced by the encoder so far
    std::size_t total_bytes() const noexcept;

    // returns the number of frames processed by the encoder so far
    std::size_t total_frames() const noexcept;

    // gives the current time in seconds that the encoder is on
    double time() const noexcept;
}