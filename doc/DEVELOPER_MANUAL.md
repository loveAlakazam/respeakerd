# Developer Manual

## 1. Overview

`respeakerd` is the server application for the microphone array solutions of SEEED, based on `librespeaker` which combines the audio front-end processing algorithms.

It's also a good example on how to utilize the `librespeaker`. Users can implement their own server application / daemon to invoke `librespeaker`.

This manual shows how to compile and run this project `respeakerd`, and then introduces the protocol used in the communication between `respeakerd` and a Python client implementation of AVS (https://github.com/respeaker/avs).

## 2. How to compile

### 2.1 Dependencies

- json: https://github.com/nlohmann/json, header only
- base64: https://github.com/tplgy/cppcodec, header only
- gflags: https://gflags.github.io/gflags/, precompiled for ARM platform
- libsndfile1-dev libasound2-dev: save PCM to wav file
- cmake
- librespeaker

```shell
$ sudo apt install -y librespeaker cmake
```

### 2.2 Compile

```shell
$ cd PROJECT-ROOT/build
$ cmake ..
$ make
$ cp src/respeakerd . && chmod a+x ./respeakerd
```

## 3. Command line parameters

The command line parameters may change during the development of this project. To get the updated information of the command line paramenters, please

```shell
$ cd PROJECT-ROOT/build
$ ./respeakerd -help
```

```shell
-agc_level (dBFS for AGC, the range is [-31, 0]) type: int32 default: -10
-debug (print more message) type: bool default: false
-enable_wav_log (enable logging audio streams into wav files for VEP and respeakerd) type: bool default: false
-fifo_file (the path of the fifo file when enable pulse mode) type: string default: "/tmp/music.input"
-mode (the mode of respeakerd, can be standard, pulse, manual_with_kws, manual_without_kws) type: string default: "standard"
-ref_channel (the channel index of the AEC reference, 6 or 7) type: int32 default: 6
-snowboy_model_path (the path to snowboay's model file) type: string default: "./resources/alexa.umdl"
-snowboy_res_path (the path to snowboay's resource file) type: string default: "./resources/common.res"
-snowboy_sensitivity (the sensitivity of snowboay) type: string default: "0.5"
-source (the source of pulseaudio) type: string default: "default"
```

## 4. PulseAudio mode

### 4.1 PulseAudio configuratin

```shell
$ sudo vim /etc/pulse/default.pa
```

Add the following line to the end of the file:

```text
load-module module-pipe-source source_name="respeakerd_output" rate=16000 channels=1
set-default-source respeakerd_output
```

### 4.2 Start `respeakerd` in PulseAudio mode

```
$ cd PROJECT-ROOT/build
$ ./respeakerd -mode=pulse -source="alsa_input.platform-sound_0.seeed-8ch" -debug -snowboy_model_path="./resources/snowboy.umdl" -snowboy_res_path="./resources/common.res" -snowboy_sensitivity="0.4"
```

Specify other options if you need. Please note that if no application's consuming the audio stream from `respeakerd_output` source, respeakerd will get blocked. But this is not a big deal. Now let's move on to the setup of Alexa C++ SDK example App [TODO].

## 5. Manual doa mode

Manual doa mode is designed for the user who want to detect the speaker direction with other methods(e.g. with camera) and pick the voice audio in that direction.


And there are 2 manual doa modes: one is `manual_with_kws` and another is `manual_without_kws`. Literally, the first mode will output a `hotword event` when the keyword is detected. And `manual_without_kws` mode only outputs audio data.

### 5.1 Start respeakerd in manual doa mode

```
$ cd PROJECT-ROOT/build
$ ./respeakerd -debug -snowboy_model_path="./resources/snowboy.umdl" -snowboy_res_path="./resources/common.res" -snowboy_sensitivity="0.5" -mode=manual_with_kws
```

### 5.2 Start test_manual_doa python client

`test_manual_doa.py` is an example to show how to set direction to respeakerd. In this example, direction will be set to next 60 degree after a `hotword event`. Please refer to Appendix A for more details.

```
$ cd PROJECT-ROOT/clients/Python
$ python test_manual_doa.py
```



## Appendix A. Socket protocol

`respeakerd` exposes unix domain socket at `/tmp/respeakerd.sock`, this socket is a duplex stream socket, including `input channel` and `output channel`.

Output channel: `respeakerd` outputs audio data and events to clients.

Input channel: clients report messages to `respeakerd`, e.g. cloud_ready status message.

Please note that for now the `respeakerd` only accepts one client connection.

The messages are wrapped in json format, splited by "\r\n", like:

```json
{json-packet}\r\n{json-packet}\r\n{json-packet}\r\n
```

### A.1 The json structure of output channel

```json
{"type": "audio", "data": "audio data encoded with base64", "direction": float number in degree unit}
```

```json
{"type": "event", "data": "hotword", "direction": float number in degree unit}
```

### A.2 The json structure of input channel

For now the following messages are supported:

```json
{"type": "status", "data": "ready"}
```

This is a status message which indicates that the client application has just connected to the cloud (here this client is both a client of respeakerd and a client of ASR cloud, e.g. Alexa Voice Service), respeakerd can now accept voice commands. In the following of this muanual, we illustrate all the mentions of cloud with Alexa.

```json
{"type": "status", "data": "connecting"}
```

This is a status message which indicates that Alexa client has just lost connection to the cloud, respeakerd can't accept voice commands until `ready` state.

```json
{"type": "cmd", "data": "stop_capture"}
```

This is a command message issued from the client. Generally the client gets this message from Alexa cloud, as Alexa has detected the end of a sentence. `respeakerd` hasn't utilize this message for now, it just keeps posting data to the client, becuase the base library the client is using - `voice-engine` - does drop packets when Alexa isn't available to receive inputs.

```json
{"type": "status", "data": "on_speak"}
```

This is a status message which indicates that the client has just received the speech synthesis from Alexa and will begin to play. `respeakerd` utilizes this status to enhence the algorithms. It's recommended that the client should capture this event and pass it down to `respeakerd` if you're doing your own client application.

```json
{"type": "cmd", "data": "set_direction", "direction": int number of degree[0, 359]}
```

This is a command message issued from the client.
It only works at `manual doa mode` in respeakerd. Generally, respeakerd produces 6 audio beams from 6 microphones with Beamforming algorithm, and one of beams will be selected as output beam by the Direction Of Arrial(DOA) algorithm. At `manual doa mode`, respeakerd will not do DOA algorithm as `standard mode ` or `pulse mode`, and the output beam is fixed(default: beam0). 
The client should send this message, when it needs to pick audio data from other direction. The following Table shows the range of degree of each beam and microphone:

beam: | beam1 | beam2 | beam3 | beam4 | beam5 | beam6
----|----|----|----|----|----|----
microphone: | mic1 | mic2 | mic3 | mic4 | mic5  | mic6 
degree: | 330-359, 0-29 | 30-89  | 90-149 | 150-209 | 210-269 | 270-329


## Appendix B. D-Bus protocol

respeakerd uses System Bus to deliver signals.

Object name: "/io/respeaker/respeakerd"
Interface: "respeakerd.signal"

respeakerd outputs:
- `trigger` signal
- `respeakerd_ready` signal

respeakerd listens for:
- `ready` signal
- `connecting` signal
- `on_speak` signal

These signals will be listened by pixel_ring_server
- `on_idle` signal
- `on_listen` signal
- `on_think` signal
- `on_speak` signal

Except `trigger` and `respeakerd_ready`, all other signals are generated by the AVS C++ SDK sample application.

