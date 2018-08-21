---
layout: post
title: In-Memory Image Processing with Pipes
---

At RubyConfAU 2018 presenter Michael Morris demonstrated [Kartalytics](https://github.com/Ferocia/kartalytics), which captured in-race data of their games of Mario Kart 8 in real-time.

It's both an amusing and clever project, and after the conference I was eager to look into the detail of how they were capturing the images.

In short, they use an HDMI to Ethernet device which sends a video stream as UDP packets to a Raspberry Pi. They use [ffmpeg](https://ffmpeg.org/) to capture screenshots from the video stream at a rate of 5 frames per second, writing these images to the SD card on the Raspberry Pi. Another process iterates over these images, reading them off the SD card and processing them to extract in-game information.

What they did, writing the files to the SD and then reading them again with a separate process, is exactly what I would have done with a quick side project. However that block device I/O can have a penalty.

So I decided to try some experiments to bypass the SD card as a little side project.

#### Experiment 1 - extracting images from a video stream

You can get ffmpeg to extract images from a video file using a command like

    ffmpeg -i video.mpg thumb%04d.jpg

This creates files like thumb0001.jpg, thumb0002.jpg etc.

The first thing I needed to find out was if there was a way to get `ffmpeg` to write the images to a pipe.

Apparently you can, using:

    ffmpeg -i video.mpg -f image2pipe pipe:1

This forces the output format to be images streamed to a pipe, and specifies we want it sent to pipe:1, which is [stdout](https://en.wikipedia.org/wiki/Standard_streams#Standard_output_(stdout)).

That means we can use [IO.popen](https://ruby-doc.org/core-2.5.0/IO.html#method-c-popen) in Ruby to read this stream. 

```ruby
IO.popen(ffmpeg_command) do |data|
  # process data stream
end
```

To distinguish one image from another in the stream, I had to do a little research on the JPEG format. From that I discovered that the JPEG/JFIF format starts with the bytes `0xFF 0xD8` and ends with the bytes `0xFF 0xD9`.

With all that, I could write a quick script to read the stream of images and distinguish them from each other:

<script src="https://gist.github.com/seandmccarthy/bf50ec1fb6fc445ee1f9c143623b6bcd.js"></script>

#### Experiment 2 - processing the images from a video stream

In Kartalytics they use the rmagick library, a Ruby wrapper for ImageMagick, to process the images. I wanted to see if I could process straight from the stream.

This is possible with the [Image.from_blob](https://rmagick.github.io/image1.html#from_blob) function. So with a small change we can take the raw image data and create an ImageMagick `Image` object ready for processing.

```ruby
image = Magick::Image.from_blob(img_bytes.pack('C*')).first
```

The full script is:

<script src="https://gist.github.com/seandmccarthy/83850aefc9e60535af19cee128143d68.js"></script>

#### Conclusion

So the experiment worked, and turned out to be not nearly as difficult as I thought. The fun part being to learn a little more about the JPEG/JFIF format and to play with pipes,  made possible because ffmpeg is so versatile with its formats.
