---
layout: post
title: "Learning MusicToolbox in OS X for MIDI file(s) playback"
description: ""
category: "Using Mac"
tags: [C,"entertainment","OS X","sample code","MIDI","study"]
---
{% include JB/setup %}

*(I assume you, the reader has some related background and C language knowledge.)*

For a graduate with \*nix background, a console tool is very handy for kinds of situations. This time I need a command-line 
application for MIDI files playback on OS X (at least useful for larger MIDI system testing). Mac OS X provides MusicToolbox API
for facilitate MIDI file playback applications. Some readers may already know the low level CoreMIDI framework, which is good.

When I opened the '.midi' file at first time, the system told me QuickTime 7 should be downloaded and installed. I was shocked 
since QuickTime X has been installed and can already playback some common media formats. Later I found that QT7 is an app in i386
architecture, and furthermore GarageBand (the guitar study and playing app bundled with OS X) is 32-bit too. Interestingly the 
CoreMIDI midi server and the CoreAudio audio server are all native 64-bit (x86-64) processes. 

The MIDI system parses the standard midi format (SMF) or other computer music formats into a MIDI sequence including several 
MIDI tracks. Each track consists of one or more events. Most common kind of event is the note event, which represents a music note.
The system sends notes and other events to an endpoint, which may be an MIDI output (actually a MIDI output port, multiple ports 
can sound together), effecting chain or other MIDI events consumers. What QT7 actually provides us is a software MIDI output 
device, which tranlates MIDI events into sampled audio and send the audio to various audio consumer (via CoreAudio Framework, 
the CoreAudio server), including physical audio output, virtual audio output, AudioUnit audio effects, etc. One important 
thing need to be introduced is (MIDI) instruments. Just like computer fonts, which convert text file into 'points on screen', 
MIDI instruments convert MIDI note events into 'sound'. Instruments are in DLS, SoundFont, etc formats.

The MusicToolbox provides us relatively easy to use API to play MIDI files. Just create a MusicSequence (yeah, a MIDI sequence 
really, but it doesn't call the in-core representation as MIDI sequence) from a file (others are 
not covered here), then create a music player, set the player use the created music sequence, and start player. Here I don't need 
to open MIDI output device, audio device, connecting them or other procedures after preparing the music sequence.

Next, I found that the music player did not stop even after all the notes had been played. At first glance, I thought that design
decision was bad. Some time later I heard the MusicToolbox enabled the user editing the MIDI sequence, esp. appending notes at the 
end of the sequence. This behavior is better for music: use serveral simple music player(s) to provide chorus, receive MIDI events
from external device, etc. Of course, my problem is solvable: the API allows me to query the length of a specified track (recall
a sequence includes several tracks!), and allows me to query current player position. 

###Code Listing 1
{% highlight c linenos %}
//  playmidi
//
//  2012 PrZhu.
//  This code is for sample purpose, it does not match any level of quality.
//  Please write your own version, thank you.
//  If you really have found my code is inspirable, please leave your name 
//  and your message here. I will be very appreciated.
//  cc -o playmidi playmidi.c -framework CoreFoundation -framework AudioToolbox 
//  (and -framework CoreMIDI if code listing 2 is included) 

#include <CoreFoundation/CoreFoundation.h>
#include <AudioToolbox/AudioToolbox.h>
#include <stdlib.h>
#include <stdio.h>

int main (int argc, const char * argv[])
{
    CFShow(CFSTR("Hello, World!\n"));
    const char *instr = "/dev/stdin";
    CFURLRef url = CFURLCreateFromFileSystemRepresentation(NULL, (const UInt8 *)instr, strlen(instr), false);
    MusicSequence ms;
    NewMusicSequence(&ms);
    MusicSequenceFileLoad(ms, url, kMusicSequenceFile_MIDIType, 0);
    MusicTrack track;
 //   MusicSequenceGetTempoTrack(ms, &track); // ignore this
    MusicSequenceGetIndTrack(ms, 0, &track);

    MusicTimeStamp trackLength;
    UInt32 ioLength = sizeof(MusicTimeStamp); 
    MusicTrackGetProperty(track, kSequenceTrackProperty_TrackLength, &trackLength, &ioLength);
    printf("Length: %lf\n", trackLength);
            
    MusicPlayer mp;
    NewMusicPlayer(&mp);
    MusicPlayerSetSequence(mp, ms);
    MusicPlayerStart(mp);
    MusicTimeStamp t;
    do {
        usleep(250000);
        MusicPlayerGetTime(mp, &t);
        printf("\r%lf", t);
        fflush(stdout);
    } while (t < trackLength);
    MusicPlayerStop(mp);
    puts("");
    return 0;
}
{% endhighlight %}

The sample program reads MIDI file from standard input and create a music sequence from MIDI file
(<tt>kMusicSequenceFile_MIDIType</tt>). In order to stop the player and exit the application, I get the index 0 track (
it is normal a music sequence contains no track! so the code is **not** safe) and query the 
<tt>kSequenceTrackProperty_TrackLength</tt> property though a music sequence may contains serveral ordinary
tracks, the code is not really the correct way to get the sequence length. After that, I create a music player, set the 
sequence and start.
Now I only know using a 'sleep, query, compare' loop to know whether the play ends. Since the loop also prints current 
position, it is acceptable.

Next part is a little bit low level. It changes the destination of the music sequence. In many cases I do not want 
my sequence go to the default output device to play, e.g., I want to send the sequence to *fluidsynth* open source
synthesizer, send the sequence to an effect chain, make chorus...The following code provides a sample for setting
a music sequence's destination to destination 0. In most case, there are **no** destinations for you to set. What
really happens behind a music player is that the player create a destination with apple's default parameter when 
no destinations are set in the music sequence.

###Code Listing 2
{% highlight c linenos %}
    {
        MIDIEntityRef midiDestination;
        ItemCount numOfDestinations = MIDIGetNumberOfDestinations();
        ItemCount i ;
        for(i = 0; i < numOfDestinations; i++ ){            
            midiDestination = MIDIGetDestination(i);
            CFPropertyListRef midiDestinationProperties;
            MIDIObjectGetProperties(midiDestination, &midiDestinationProperties, true);
            CFShow(midiDestinationProperties);
        }
        if (0 < numOfDestinations) {
            MusicSequenceSetMIDIEndpoint(ms, MIDIGetDestination(0));
        }
    }
{% endhighlight %}

Insert these code at line <tt>32</tt> of previous code listing(Code Listing 1). Before starting the application, 
I started *fluidsynth* with:

	fluidsynth -g 3 -m coremidi -a coreaudio /Library/Audio/Sounds/Banks/FluidR3_GM.sf2

The command set the '-g' param which makes the sound lots of larger and use a soundfont distributed with fluidsynth.
In my current system, this is the first MIDI destination created.

Thus sample running like this:

	PrZhumatoMacBook-Pro:playmidi przhu$ ./playmidi < /Users/przhu/Music/sound/national/National_Banner_Song.mid 
	Hello, World!
	Length: 79.999023
	<CFBasicHash 0x10171a060 [0x7fff77007fa0]>{type = mutable dict, count = 2,
	entries =>
		3 : <CFString 0x10171a790 [0x7fff77007fa0]>{contents = "uniqueID"} = <CFNumber 0xffffffe9e449e2c3 [0x7fff77007fa0]>{value = -370914846, type = kCFNumberSInt64Type}
		11 : <CFString 0x7fff76ffc500 [0x7fff77007fa0]>{contents = "name"} = <CFString 0x10171a030 [0x7fff77007fa0]>{contents = "FluidSynth virtual port 23860"}
	}
	80.134320

It means that the destination name is "FluidSynth virtual port 23860"(you'd better ignore the 'uniqueID')

Sorry for my long article. Feel free to comment. 
For programming detail, you can refer to Apple Developer's *MusicPlayer Reference* and its accompanied references, 
*CoreMIDI*'s *MIDIEntityRef*, *MIDIGetDestination* and others.

Apple, (Mac )OS X, CoreMIDI, CoreAudio, MusicToolbox, fluidsynth, DLS, SoundFont (tell me if I forgot) are names or trademarks 
owned by coresponding author(s)/orgnazition(s). Use of these names with best regards and wishes.
