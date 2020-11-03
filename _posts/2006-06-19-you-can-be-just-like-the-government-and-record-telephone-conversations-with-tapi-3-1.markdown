---
title: You can be just like the government and record telephone conversations.. with TAPI 3.1
date: 2006-06-19T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

I've gotten several questions on this topic so I thought it might be a good thing to show in a blog.  Recording with TAPI 3.1 is actually pretty easy if you are running on Windows XP or better.  TAPI 3.1 provides some simple filename-based methods to dump the conversation into a .WAV file.  If you want to control the file, or stream it somewhere else, it's a bit more difficult and Microsoft provides a decent sample with the platform SDK that does it.

Here's a simple example -- in this scenario we will create a new outgoing call, connect it and then setup an outgoing stream to play a welcome message:

```csharp
TapiCall currCall = selectedAddress.CreateCall(numberToDial, LINEADDRESSTYPES.PhoneNumber, TAPIMEDIATYPES.AUDIO);  
if (currCall != null)  
{
    currCall.Connect(false);  
    playbackTerminal = currCall.RequestTerminal(TTerminal.FilePlaybackTerminal, TAPIMEDIATYPES.AUDIO, TERMINAL_DIRECTION.TD_CAPTURE);  
    if (playbackTerminal != null)  
    {  
        playbackTerminal.MediaPlayList = new string[] { MESSAGE_PROMPT };  
        currCall.SelectTerminalOnCall(playbackTerminal);  
    }  
}
```

When the call is actually connected, then we will start the playbackTerminal stream and begin recording the conversation - this would typically be done in the TE_CALLSTATE handler:

```csharp
if (e.State == CALL_STATE.CS_CONNECTED && playbackTerminal != null)  
{  
    // Start the playback message..  
    playbackTerminal.Start(); // Begin recording the conversation - may be half-duplex..  
    RecordConversation("RecordedMessage.wav");  
}  
else if (e.State == CALL_STATE.CS_DISCONNECTED)  
{  
    // Stop recording when the call terminates.  
    if (recordTerminal != null)  
        recordTerminal.Stop();  
  
   recordTerminal = null;  
   playbackTerminal = null;  
}  
```

The code for recording the conversation is pretty simple as well, given a filename, just get a recording terminal and assign it.  TAPI will take care of creating the file and writing contents into it.

```csharp
private void RecordConversation(string fileName)  
{  
    // This code only works on XP or better (TAPI 3.1).  
    if (currCall != null)  
    {  
        recordTerminal = currCall.RequestTerminal(TTerminal.FileRecordingTerminal, TAPIMEDIATYPES.MULTITRACK, TERMINAL_DIRECTION.TD_RENDER);  
        if (recordTerminal != null)  
        {  
            recordTerminal.RecordFileName = fileName;  
            currCall.SelectTerminalOnCall(recordTerminal);  
            recordTerminal.Start();  
        }  
    }  
}  
```

Finis.
