---
layout: post
title: "Reaper class"
date: 2025-02-23
categories: [music, podcasting]
tags: []
description: "Un cours sur l'édition de podcasts avec reaper"
toc: true
---

- [Preparation](#preparation)
- [Session setup: preferences, and saving the session](#session-setup-preferences-and-saving-the-session)
  - [Importing a configuration](#importing-a-configuration)
  - [Peaks files for viewing waveforms](#peaks-files-for-viewing-waveforms)
- [The actions list](#the-actions-list)
- [Adding tracks, adding items](#adding-tracks-adding-items)
- [Cut/copy/paste](#cutcopypaste)
- [Previewing Sounds](#previewing-sounds)
- [Stretching samples (for music only)](#stretching-samples-for-music-only)
- [Removing a Time selection from a session](#removing-a-time-selection-from-a-session)
- [Removing background noise from a recording](#removing-background-noise-from-a-recording)
  - [Adding ReaFIR to Your Track](#adding-reafir-to-your-track)
  - [Basic Noise Reduction](#basic-noise-reduction)
  - [Fine-Tuning](#fine-tuning)
  - [Tips](#tips)

## Preparation

Things you'll need:


- [Reaper](https://www.reaper.fm/download.php) installed on your computer

- The [extensions sws](https://www.sws-extension.org/). Don't install yet.

- [My config](https://github.com/AntoineBalaine/reaper_config/raw/refs/heads/main/ab_config.ReaperConfigZip?download=) to be installed - instructions to follow. [Here's an in-depth tutorial if things don't work](https://www.youtube.com/watch?v=HB4C3njFxRg) - we'll setup together.

- The course sample pack or a folder containing some audio files.

### Session setup: preferences, and saving the session

* A tutorial about [session setup](https://www.youtube.com/watch?v=K4WhkBzK3PA)

- Before opening reaper, please place your recordings folder on your desktop, we'll need it later.

#### Importing a configuration

- Open reaper. Once Reaper is installed, you'll have to import the **200415 - AB Config.ReaperConfigZip** file into reaper.

_Here's a [download link](https://github.com/AntoineBalaine/reaper_config/raw/refs/heads/main/ab_config.ReaperConfigZip?download=) to a more recent version of the config. I'm not very confident it'll have everything you need, but it's worth a shot._

- Open up the preferences from the menu bar (`REAPER` - `settings`)

- In the «`general`» tab of the prefs, click this button and find the config file in your computer.

![2.jpeg](/assets/images/2.jpeg)

- At the bottom of that page, there is a search bar, which you'll use for everything.

![1.jpeg](/assets/images/1.jpeg)

#### Peaks files for viewing waveforms

- type «`audio device setting`» in the search bar, press find, and see what comes up. Type 44100 in the «`request sample rate`» block. Right below that, the block size can be set to 128 for now.

#### Where are the [re-peaks stored](https://www.youtube.com/watch?time_continue=115&v=tAHiFoj6ra0&feature=emb_logo) ?
_why keep all your peak files in a single folder_

type «`reapeaks`» in the search bar. The search points to a folder. Select a different folder in your computer to store your peak files from there - the current preset one only exists in my computer, so you'll have to setup your own. After that, you can close the preferences.

#### Session folder structure
_The subject is worth knowing about, though you have everything pre-set here._

Hit `cmd+s` to save the session. Name it, and make sure to check the «`create subdirectory`» button - just to make sure your folder structure doesn't go up the wazoo

![7.jpeg](/assets/images/7.jpeg)

### The actions list
_your entry point to do anything, which you will use all the time._

- The [action list's tutorial here](https://www.youtube.com/watch?v=CZ1IliW_0p4).
Press `cmd + a` (or `?` if not using my config, or open from the menu bar's `Actions` => `show actions list`). A window pops up at the bottom of the screen. At its top, there is a «`filter`» search bar. Type «`toggle transport`». The first option is the one you want to select. At the top of the screen, the transport section of the session should disappear, you can make it reappear by clicking a second time.

![3.jpeg](/assets/images/3.jpeg)

- now type «`toggle master track visible`» in the list and select the first option popping up - see what happens.

- now find «`Track : insert new track`» in the list and see what happens. On the left of the action's name, there is a part describing the current shortcuts for the actions. If it's empty, feel free to add one.

![8.jpeg](/assets/images/8.jpeg)

### Adding tracks, adding items


_Putting together midi and audio items on the same track - how this software allows you to use both interchangeably._

- At the top right of the page, make sure **snap to grid** is enabled by clicking on the magnet icon. If you can't see the magnet icon, don't panic, you can still proceed to the rest of the tutorial.

![13.jpeg](/assets/images/13.jpeg)

- In the track lane of the arrangement view, place your mouse cursor on the line that says 9.1.0. Click and drag while holding cmd and see what happens

![10.jpeg](/assets/images/10.jpeg)

_A baby has appeared! This is called an item_

- Now from the recordings pack that you have on the desktop, grab any audio file and drag it into the session up to bar 17.1.0, right next to the grey item you just created.

### Cut/copy/paste

_Cut, copy, and paste can be done from the [mouse cursor](https://github.com/X-Raym/REAPER-ReaScripts/blob/master/Items%20Editing/X-Raym_Copy%20selected%20items%20and%20paste%20at%20mouse%20position.lua), or edit cursor, or [time selection](https://www.youtube.com/watch?v=RKp8C7MS63A) - you'll see soon enough «why» pasting at mouse cursor rather than edit cursor, its time gain and its downsides_

- Look up the shortcut for «`Adjust grid (mousewheel)`» in the action list, and try it. Grid subdivision should be reducing itself.

- Look up the shortcut for «`fast horizontal zoom`» in the action list, and try it.

- Now, with those shortcuts, set up a Grid size that is inferior to your items' size, and zoom in to get them in sight.

![11.jpeg](/assets/images/11.jpeg)

_something like this_

#### editing items using the custom shortcuts
_if the following shortcuts don't work upon trying them, don't panic: you can set them in the actions list_
- Place your mouse at bar 10, over your empty grey item, and press "a".
  - actions list: `CUSTOM - Trim Clip Right Side (mouse)`

- Place your mouse at bar 18, over your audio item, and press "b".
  - actions list: `Custom: AB -  PLUS Trim Clip Left Side (mouse)`

- select the audio item, press c, place your mouse at bar 22 and press "v".
  - actions list: `Custom: AB - paste at mouse cursor`

- press "`z`" or `ctrl+z` or `actions: undo` to undo anything at any point.
  - You might have to press z several times in a row for some of these actions

- In the action list, look up the name of the actions corresponding to these three shortcuts:
![12.jpeg](/assets/images/12.jpeg)

#### The thing to note:
these shortcuts apply only to whatever item is underneath the mouse. You can still run these actions at the edit cursor position instead (which is more traditional) by pressing ctrl + shortcut (a/s/b/v).

- Run the «`paste`» action, but this time with a twist: press shift + v instead of just v. Look up the name of this shortcut in the action list. This is useful for beatmaking.

## Previewing Sounds

- In the action list, search «`show media explorer`». In the list of shortcuts that just appeared, find the desktop, double click it. In the now appearing desktop list, find the sample pack folder, and double click it. The list of sounds appears. Inside this list, any audio samples that you click will be played back from the bottom of the page.

![14.jpeg](/assets/images/14.jpeg)


## Stretching samples (for music only)

- Drag a sample - any sample - into the session. Set the grid subdivision to be smaller than the length of the sample. With snap to grid enabled, place your mouse on one of the grid lines above the sample, and press shift + w. A little square appears on the item: that's called a stretch marker.

![15.gif](/assets/images/15.gif)

- Click and drag the stretch marker to the right or left in one grid increment. Reaper automatically adds other stretch markers at the extremities of the sample, stretches the sound, and maintains the same length with different play speeds - according to where the center marker is.

- Click all three of the stretch markers while holding alt/option to remove them.

- Place your mouse above the right or left extremity of the sample, and press alt/option. You should see your mouse cursor turning into a little hand. While holding alt, click the sample's edge, and drag to the right.

![16.gif](/assets/images/16.gif)

- You can always reset the length of the item to its original speed by following these steps:
  - click the bottom half of the item
  - In the window that pops up, find the box that says «`playrate`»
  - Type the value `1`
  - press the «`apply button`» at the bottom right of the window

![18.jpeg](/assets/images/18.jpeg)

- **FIRST BEAT** Once you are done previewing the samples, choose the ones you like, drag them into the session, and make a beat with them - a short one. Move, cut, copy, paste, or stretch them in any way you please.

## Removing a Time selection from a session

1. Place your edit cursor where you want the cut to begin
2. `Ctrl-Click` and drag to create a time selection (highlighted area)
3. Open the actions list and search for: `Time selection: Remove contents of time selection (moving later items)`.
4. Reaper will remove that section and automatically close the gap
![rpr_cuttime.gif](/assets/images/rpr_cut_time.gif)
### Alternative Methods
- **Select an area and right-click and select `Cut selected area of items`**
  1. this doesn't cut off time in the session, though.
- **Ripple Edit**:
  1. Enable "Ripple Edit" by clicking the ripple edit button (or search in actions list)
  2. Make your time selection
  3. Press Delete
  4. All content to the right will shift left to fill the gap

## Removing background noise from a recording

Use the audio plugin `ReaFIR` to remove background noise.

### Adding ReaFIR to Your Track

1. Select the track you want to apply noise reduction to
2. Click the FX button or press F
![rpr_reafir1.png](/assets/images/rpr_reafir1.png)
3. In the FX browser, search for "ReaFIR"
![rpr_reafir2.png](/assets/images/rpr_reafir2.png)
4. Double-click on ReaFIR to add it to your track

### Basic Noise Reduction

1. Find a section of your recording that contains only the noise you want to remove
2. Play this section and click the "Automatically build noise profile" button in ReaFIR
![rpr_reafir3.png](/assets/images/rpr_reafir3.png)
3. Let it play for a few seconds to capture the noise pattern
4. Click the button again to stop capturing
5. Set the "`Mode`" dropdown to "`Subtract`"
6. Adjust the "FFT size" slider if needed (higher values for more precision, lower for less latency)
![rpr_reafir.gif](/assets/images/rpr_reafir.gif)
_Make sure to watch til the end, please_
### Fine-Tuning

- Use the "Reduction amount" slider to control how aggressively the noise is removed
- If you hear artifacts or "underwater" effects, reduce the reduction amount
- To manually adjust the noise profile, disable "Automatically build noise profile" and drag points on the graph
- For more subtle noise reduction, try working with the lower frequencies first

### Tips

- Always create a backup of your audio before processing
- Preview the audio before and after by toggling the FX bypass button
- For very specific noise issues, you can manually edit the noise profile by manipulating the curve on the graph
- Save your settings as a preset if you need to process multiple similar recordings
