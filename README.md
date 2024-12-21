# Stereo-ADC-soundcards
Stereo ADC soundcards

As far as I am aware there are only 2 freely available USB stereo ADC soundcards available
https://plugable.com/products/usb-audio
https://www.axagon.eu/en/produkty/ada-15
If you know of others please post an issue

A 2 channel beamformer will only focus along the X axis and are often seen in many webcams nowadays. Really they should be front facing as the enclosure it self will attenuate sound from the rear. The beamformer we will use is a simple 'delay sum' beamformer as described in the excellent Invensense application note. https://invensense.tdk.com/wp-content/uploads/2015/02/Microphone-Array-Beamforming.pdf

You can use a Pi Zero2 or above as you can create a wireless satelite microphone array (still needs power, Zero2) , or with a bit of DiY we will create a long mic lead for the USB soundcard and you can hide your bigger Pi away with just mic array showing.

Download and flash a 64bit version of PiOS Lite to your SD card and provide the network and ssh setting you wish.

1st update and upgrade the software and reboot
```
sudo apt update
sudo apt upgrade
sudo reboot
```
Plug in your Pluggable or Axagon USB sound card and `aplay -l`
```
**** List of PLAYBACK Hardware Devices ****
card 0: DEVICE [USB AUDIO DEVICE], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: vc4hdmi [vc4-hdmi], device 0: MAI PCM i2s-hifi-0 [MAI PCM i2s-hifi-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
```
Install the beamformer
```
sudo apt install cmake autotools-dev autoconf libtool pkg-config portaudio19-dev git
git clone https://github.com/StuartIanNaylor/2ch_delay_sum.git
cd 2ch_delay_sum
make clean
make
sudo modprobe snd-aloop
```
`./ds` which will run the delay-sum beamformer but fail without any parameters The last couple of lines should display something like this
```
Devices:
0: () USB AUDIO DEVICE: Audio (hw:0,0) (ALSA)
1: () Loopback: PCM (hw:2,0) (ALSA)
2: () Loopback: PCM (hw:2,1) (ALSA)
3: () sysdefault (ALSA)
4: (output) front (ALSA)
5: (output) surround40 (ALSA)
6: (output) iec958 (ALSA)
7: () spdif (ALSA)
8: () default (ALSA) --default--
9: (output) dmix (ALSA)
```
You can also `./ds --help`
```
./ds --help

Do delay and sum beamforming
Usage: ds [options] input_device[index] output_device[index]
ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 0
Margin (Max(TDOA) = mic distance mm / (343000 / sample_rate)
Set display_levels=0 for silent running

Options:
  --channels                  : mic input channels (int, default = 2)
  --display_levels            : display_levels (int, default = 1)
  --frames                    : frame buffer size (int, default = 4000)
  --margin                    : constraint for tdoa estimation (int, default = 16)
  --sample_rate               : sample rate (int, default = 48000)
```
`./ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 1` `--frames` just keep as is for now, what we need to do is set in the input device and output device. From the above devices ./ds provides the soundcard is device 0 and then we have the 1st Loopback device as output, which is 1, A Alsa loopback device allows you to output audio on device 1 which is an alsa sink. The corresponding device 2 (hw:2,1) is now available as a microphone source for any app which wants the output of the beamformer. We have set the sample rate to 48Khz to get the best resolution we can `--sample_rate=48000` Number of channels is 2 `--channels=2` and now we get to the margin.

The speed of sound is approx 343 m/s so in mm is 343,000 and if we divide that by our sample rate 48000 we get 7.14583mm Which is the distance sound will move for the duration of a single sample. So we can work backwards and measure the distance between the 2 mics and divide by 7.158 gives us the max delay samples (margin)  About 65/70mm is about the max distance of mic spacing or you will start to get an effect called aliasing (https://en.wikipedia.org/wiki/Aliasing) which you don't want. So the sample rate and distance of the mics provides the resolution of the beamformer. The number of microphones sets the maximum attenuation of out of focus audio, but requires another Gccphat calc for every microphone added. The Gccphat calc is nearly all the load of the beamformer so 2 mics is the lowest computational cost, but you could add more mics but reference the invensense doc to how.

So anyway now that we come to try `./ds --frames=4800 --channels=2 --margin=20 --sample_rate=48000 --display_levels=1 0 1`
```
sample max amplitude = 0.000808
sample average = 0.000235
frames = 4800
tdoa=0 tdoa=5 
sample max amplitude = 0.000987
sample average = 0.000214
frames = 4800
tdoa=0 tdoa=0 
sample max amplitude = 0.000944
sample average = 0.000212
frames = 4800
tdoa=0 tdoa=-13 
sample max amplitude = 0.000791
sample average = 0.000194
frames = 4800
tdoa=0 tdoa=7 
```
We should get some repeating info of the current tdoa (Time Difference of Arrival) in samples and the current max amplitude recieved by the mics.
This is because we set the debug `--display_levels=1` as in normal operation set it to `--display_levels=0`
If we `aplay -l` we should see something like
```
**** List of PLAYBACK Hardware Devices ****
card 0: DEVICE [USB AUDIO DEVICE], device 0: USB Audio [USB Audio]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 1: vc4hdmi [vc4-hdmi], device 0: MAI PCM i2s-hifi-0 [MAI PCM i2s-hifi-0]
  Subdevices: 1/1
  Subdevice #0: subdevice #0
card 2: Loopback [Loopback], device 0: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
card 2: Loopback [Loopback], device 1: Loopback PCM [Loopback PCM]
  Subdevices: 8/8
  Subdevice #0: subdevice #0
  Subdevice #1: subdevice #1
  Subdevice #2: subdevice #2
  Subdevice #3: subdevice #3
  Subdevice #4: subdevice #4
  Subdevice #5: subdevice #5
  Subdevice #6: subdevice #6
  Subdevice #7: subdevice #7
```
So we took the input of the usb soundcard did the beamforming and played that back to one of the 8 loopback subdevices which will be the 1st hw:2,0
This will no be available for any app as a standard Alsa source (microphone) on the oppiste side of the loopback hw:2,1
There are 8 subdevices on a loopback so the full format is hw:2,0,0 for the 1st sink, hw:2,1,0 for the 1st paired source... 

Thats a basic delay-sum beamformer and if anyone with a any C experience would like to take over maintenance or even optimise the FFT to use Neon instructions then please do.
All the credit should go to the original https://github.com/robin1001/beamforming which was my 1st ever C hack making it realtime, with a circular buffer including the margin and output to through portaudio.







