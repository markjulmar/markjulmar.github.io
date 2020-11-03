---
title: Managing Media Streams with TAPI 3.0
date: 2006-03-06T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

I got an email question today asking about how to write a .WAV file out to an active call using our TAPI3 wrapper.  It's actually pretty easy to do and it follows along with the normal SDK method used in C++.  Here's the relevant code (for the full example, [check out the ITAPI3 source code](https://github.com/markjulmar/itapi3) and look at the AutoAttendant sample).

First, when a new offering call shows up, get hold of the file playback terminal for it and set it up to play your .WAV files.  We cannot play the files until the call is answered, but this will essentially "queue" it up.

```csharp
// Method called when TE_CALLSTATE == OFFERING raised  
private void AnswerCall()  
{  
    // Get the playback terminal from the call  
    try  
    {  
        playbackTerminal = activeCall.RequestTerminal(TTerminal.FilePlaybackTerminal, TAPIMEDIATYPES.AUDIO, TERMINAL_DIRECTION.TD_CAPTURE);  
        if (playbackTerminal != null)  
        {  
            playbackTerminal.MediaPlayList = new string[] { "Hello.wav" };  
            activeCall.SelectTerminalOnCall(playbackTerminal);  
            activeCall.Answer();  
        }  
        else  
        {  
            MessageBox.Show("Failed to retrieve playback terminal.");  
            activeCall.Disconnect(DISCONNECT_CODE.DC_REJECTED);  
        }  
    }  
    catch (TapiException ex)  
    {  
    }  
}
```

Next, watch for the call media to change indicating we have an active stream for our terminal.  When that happens, start the playback stream:

```csharp
// Method called when TE_CALLMEDIA is raised  
private void OnCallMedia(object sender, TapiCallMediaEventArgs e)  
{  
    try
    {  
        if (activeCall != null && e.Event == CALL_MEDIA_EVENT.CME_STREAM_ACTIVE &&  
            e.Terminal.Direction == TERMINAL_DIRECTION.TD_CAPTURE &&  
            playbackTerminal != null)  
        {  
            playbackTerminal.Start();  
            SetStatusMessage("File Playback Terminal started ");  
        }  
    }  
    catch (TapiException ex)  
    {  
    }  
}
```

Finally, when the file terminal is finished, close and dispose the stream.  This is done just to cleanup the resources properly:

```csharp
// Method called when TE_FILETERMINAL is raised  
private void OnFileTerminal(object sender, TapiFileTerminalEventArgs e)  
{  
    // We are interested in TMS_IDLE because we will un-select playback and   
    // select recording  
    if (e.State == TERMINAL_MEDIA_STATE.TMS_IDLE)  
    {  
        if (e.Terminal.Direction == TERMINAL_DIRECTION.TD_CAPTURE && playbackTerminal != null)  
        {  
            try  
            {  
                // Remove the playback terminal  
                activeCall.UnselectTerminalOnCall(playbackTerminal);  
                playbackTerminal.Dispose();  
                playbackTerminal = null;  
            }  
            catch (TapiException ex)  
            {  
            }  
      }  
   }  
}
```

That's pretty much it -- there's a full sample of this that should work with voice modems or any streaming-capable TSP.

> **Updated: 3/16/05 --** 
>
> It doesn't appear to completely work with the H.323 provider; the file terminal gets connected but apparently never reports an IDLE state and so never gets disconnected in the above sample. This appears to be true of the Platform SDK samples as well.
