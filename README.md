# Nexus DR-X Audio Configuration
Version: 20201210  
Author: Steve Magnuson, AG7GN  

## Introduction

Raspbian's embrace of PulseAudio as of early December 2020 brought significant changes to the way the Nexus DR-X sound is configured.

The Nexus DR-X image has always used PulseAudio to support simultaneous use of left and right radios with a single Fe-Pi audio board. Now that the OS *also* uses PulseAudio, some changes to the default PulseAudio configuration were needed to make things operate as they were before. 

As of early December 2020, by default PulseAudio is configured to automatically detect sound devices attached to the Pi. This works fine for the built in analog (the TRS or headphone jack built in to the Pi) and digital (audio output from the HDMI ports on the Pi). However, PulseAudio fails epically when it comes to setting up the Fe-Pi audio board correctly.

To rectify this, I sought to find a solution that minimally affects the default PulseAudio configuration yet provides all the functionality we need to use the Fe-Pi audio card for ham radio applications.

## PulseAudio Modifications to Support Fe-Pi

Three modifications are needed for PulseAudio to support the Fe-Pi.

### 1. Tell PulseAudio to ignore the Fe-Pi when it's autodetecting audio interfaces

This is accomplished by adding a `/etc/udev/rules.d/89-pulseaudio-fepi.rules` file. This file has one line:

	ATTRS{id}=="Audio", ENV{PULSE_IGNORE}="1"

The `ATTRS{id}` looks for the string __Audio__. This ID is how the cards identify themselves to the OS. The Fe-Pi's ID is simply __Audio__.  When a match is made, it sets an environment variable `PULSE_IGNORE` to 1 ('true'). This tells PulseAudio to ignore cards that identify themselves as __Audio__ during PulseAudio's autodetect process when it starts.

The `89` at the beginning of the file name is used to determine the order in which the rules are executed. PulseAudio's rules are stored in `/usr/lib/udev/rules.d/90-pulseaudio.rules` and `/usr/lib/udev/rules.d/91-pulseaudio-rpi.rules`. Note that rules installed as part of the application are stored in `/usr/lib/udev/rules.d/`. User rules are stored in `/etc/udev/rules.d/`.  Since our file starts with `89`, it is executed before regular PulseAudio rules, ensuring that PulseAudio will ignore the Fe-Pi.

### 2. Manually configure the PulseAudio Fe-Pi settings after PulseAudio finishes autodetect

PulseAudio searches for it's configuration instructions in one of 2 places when PulseAudio is run in "user" mode. It first searches the user's home folder at `$HOME/.config/pulse/default.pa`. If that file is not found, it'll use `/etc/pulse/default.pa`. 

I added `$HOME/.config/pulse/default.pa` to include the Fe-Pi configuration as well additional configuration that allows TX/RX monitoring through the built-in sound interfaces.

The first line in `$HOME/.config/pulse/default.pa` is:

	.include /etc/pulse/default.pa

This tells PulseAudio to load the original `/etc/pulse/default.pa` that was installed as part of PulseAudio *before* we add our own Fe-Pi configuration. This should reduce the impact of future updates to the PulseAudio application on the Fe-Pi configuration.

The remainder of `$HOME/.config/pulse/default.pa` contains:

	# Fe-Pi card setup
	load-module module-alsa-card device_id="Audio" name="fepi" card_name="fepi" \
	namereg_fail=false tsched=yes fixed_latency_range=no ignore_dB=no deferred_volume=yes \
	use_ucm=no rate=96000 format=s16le source_name="fepi-capture" \
	sink_name="fepi-playback" mmap=yes profile_set=default.conf
	
This tells PulseAudio to manually load the Fe-Pi card (`load-module module-alsa-card device_id="Audio"`), assign it the name `fepi`, set up some other audio parameters and define a PulseAudio source of `fepi-capture` and a sink of `fepi-playback`. It also tells PulseAudio to use the Stereo In/Out profile in `/usr/share/pulseaudio/alsa-mixer/profile-sets/default.conf`.

The next lines in `$HOME/.config/pulse/default.pa` are:

	# Create the left source/sink from the Fe-Pi
	load-module module-remap-sink sink_name="fepi-playback-left" master="fepi-playback" \
	channels=1 channel_map="mono" master_channel_map="front-left" remix=no \
	
	load-module module-remap-source source_name="fepi-capture-left" master="fepi-capture" \
	channels=1 channel_map="mono" master_channel_map="front-left" remix=no
	
	# Create the right source/sink from the Fe-Pi
	load-module module-remap-sink sink_name="fepi-playback-right" master="fepi-playback" \
	channels=1 channel_map="mono" master_channel_map="front-right" remix=no \
	
	load-module module-remap-source source_name="fepi-capture-right" \
	master="fepi-capture" channels=1 channel_map="mono" \
	master_channel_map="front-right" remix=no

This tells PulsAudio to create separate left and right channel playback (sink) and capture (source) virtual interfaces that we can tell [ALSA](https://alsa-project.org/wiki/Main_Page) to use for those ham radio applications like Direwolf, which can't use PulseAudio directly.

The next lines in `$HOME/.config/pulse/default.pa` are not related to the Fe-Pi.

	# Create a stereo combined sink for the headphone and HDMI audio interfaces
	# so we don't have to know which is active at any given time
	load-module module-combine-sink sink_name=system-audio-playback \
	sink_properties=device.description='"Combined\ headphone\ and\ HDMI\ sink"' \
	slaves=alsa_output.platform-bcm2835_audio.analog-stereo,\
	alsa_output.platform-bcm2835_audio.digital-stereo channels=2

The above line creates a combined sink (playback) for both the analog (headphone) and digital (HDMI) audio interfaces that are built in to the Pi. The combine sink allows audio to go to BOTH analog and HDMI sound cards so we don't have to figure out which sound card the user has selected for system sounds (like alerts in Fldigi). Note that I named the combined sink `system-audio-playback`. The `slaves` are the IDs of the Pi's built-in analog and digital sound cards.

Finally, the last lines in `$HOME/.config/pulse/default.pa` are:

	# Create a left sink from combined headphone/HDMI built-in sound card sink 
	load-module module-remap-sink sink_name="system-audio-playback-left" \
	master="system-audio-playback" channels=1 channel_map="mono" \
	master_channel_map="front-left" remix=no

	# Create a right sink from combined headphone/HDMI built-in sound card sink
	load-module module-remap-sink sink_name="system-audio-playback-right" \
	master="system-audio-playback" channels=1 channel_map="mono" \
	master_channel_map="front-right" remix=no

The above lines create separate sinks for the left and right radios. These sinks can be selected in Fldigi, for example, to send alerts from the left radio to the left channel of the built-in sound cards and alerts from the right radio to the right channel of the built-in sound cards. 

### 3. Create virtual ALSA interfaces

Virtual ALSA interfaces are needed because some ham radio applications (like Direwolf) cannot use PulseAudio directly. The virtual ALSA interfaces use the PulseAudio sources/sinks we defined in `$HOME/.config/pulse/default.pa`.

Virtual ALSA interfaces are defined in `/etc/asound.conf`:

	pcm.fepi-capture-left {
	  type pulse
	  device "fepi-capture-left"
	}
	pcm.fepi-playback-left {
	  type pulse
	  device "fepi-playback-left"
	}
	pcm.fepi-capture-right {
	  type pulse
	  device "fepi-capture-right"
	}
	pcm.fepi-playback-right {
	  type pulse
	  device "fepi-playback-right"
	}
	pcm.system-audio-playback {
		type pulse
		device "system-audio-playback"
	}
	pcm.system-audio-playback-left {
		type pulse
		device "system-audio-playback-left"
	}
	pcm.system-audio-playback-right {
		type pulse
		device "system-audio-playback-right"
	}

## Adjust Audio Settings  

### 4.1 Start `alsamixer`
Open a terminal and run:

	alsamixer

Press __F1__ for help familiarizing yourself with navigating the `alsamixer` interface.  
Some controls are in stereo.  The up and down arrows change the levels of both left channels (for the left radio) and right (for the right radio).
      
- Pressing __Q__ or __E__ increases the left or right channel (radio) level respectively. 
	
- Pressing __Z__ or __C__ decreases the left or right channel (radio) level respectively.

1. Press __F6__ and select __1 Fe-Pi Audio__ from the list.   For the Fe-Pi, these levels 
are good starting points:
![Fe-Pi Audio Settings](img/fe-pi-settings.png)
Leave the remaining settings as-is.  
	
	You will want to open `alsamixer` while running Fldigi and/or direwolf and adjust
these settings on the __1 Fe-Pi Audio__ device as needed: 

	- __Capture__ (for audio coming from the radio into the Pi - radio RX)  
	- __PCM__ (for audio coming from the Pi to the radio - radio TX)

	Once you're happy with your audio settings, press __Esc__ to exit alsamixer.  

1. Save Audio Settings (not required)
	
	These audio settings should save automatically, but for good measure you can store them again by running:

		sudo alsactl store

	If you want to save different audio level settings for different scenarios, 
you can run this command to save the settings (choose your own file name/location):

		sudo alsactl --file ~/mysoundsettings1.state store

	...and this command to restore those settings:

		sudo alsactl --file ~/mysoundsettings1.state restore

## (Optional) Monitor Radio's TX and/or RX audio on Pi's Speakers

If you have speakers connected to the Pi, you can configure `pulseaudio` to monitor the audio to and/or from the radio

The Pi's built-in sound interface can output audio to the audio jack on the board.  This is the __Analog__ output.  It can also send audio to HDMI-attached monitors that have built-in speakers.  This is the __HDMI__ output.  To toggle between __Analog__ and __HDMI__, right-click on the speaker icon on the menu bar. Select __AV Jack__ for audio out of the headphone jack built in to your Pi or __HDMI__ for audio sent to your audio-equipped monitor. 

To adjust the level of the audio on your Pi's speakers, use the speaker's volume knob if it has one.  The speaker icon in the upper right of the Pi desktop also controls the Pi's speaker volume.  

Another way is to adjust the volume in alsamixer (__0 bcm2835 HDMI__ device) or clicking the Raspberry icon on the desktop, select __Preferences > Audio Device Settings__.  Select the __1 bcm2835 Headphones__ sound card, and adjust the slider to your liking.  You may have to click the __Select Controls...__ button to enable the slider.


### (Optional) Using Fldigi 4.1.09 (and later) Alerts, Notifications and RX Monitoring

Fldigi version 4.1.09 introduced the ability to control where alerts, notifications and monitoring audio is sent.  Obviously, you don't want to send that audio to the radio so having the ability to control where it goes is important.

5.3.1 Fldigi Alerts

Note that Fldigi [Alerts](http://www.w1hkj.com/FldigiHelp/audio_alerts_page.html) won't work for FSQ because it only looks in the signal browser for any text you specify in the alert.

The sound interface used for Alerts and RX Monitoring is set in __Configure > Config Dialog > Soundcard > Devices__.  

1.	In the __Alerts/RX Audio__ section, select `system-audio-playback-left` or `system-audio-playback-right` depending on what radio and instance of Fldigi you're using, and check `Enable Audio Alerts`.  Click __Save__ and __Close__.
1.	In the left pane of the __Fldigi configuration__ window, select __Alerts__ to set up your alerts.  

All alerts will now play through the Pi's built-in sound interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.

5.3.2 Fldigi Notifications

Unlike Alerts, Fldigi [Notifications](http://www.w1hkj.com/FldigiHelp/notifier_page.html) can look for text in Fldigi's Receive Messages pane.  To set up an audio alert in Fldigi, open __Configure > Notifications__ 

1.	Enter your search criteria.
2.	Under __Run Program__, enter the following:

		aplay -D system-audio-playback <path-to-WAV-file>
	For example:

		aplay -D system-audio-playback /home/pi/myalert.wav

This will send audio triggered by a Notification to the built-in audio interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.

You can also use `paplay`, the PulseAudio player, which can play OGG audio files in addition to WAV files.  You can optionally tell `paplay` what audio sink to use.  Example:

		paplay --device=system-audio-playback /home/pi/fsq_ag7gn.ogg
		
Use this command list the audio formats `paplay` supports:

		paplay --list-file-formats

Both `aplay` and `paplay` are installed by default in Raspbian Buster.

5.3.3 Fldigi RX Monitoring

Fldigi has built-in audio monitoring capability.  You can toggle RX monitoring on and off and apply filtering to the received audio by going __View > Rx Audio Dialog__.  The audio will be played through the built-in audio interface.  Don't forget to select __Analog__ or __HDMI__ output as described earlier.
