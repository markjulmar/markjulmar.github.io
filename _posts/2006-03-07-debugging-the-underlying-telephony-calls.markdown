---
title: Debugging the underlying Telephony calls
date: 2006-03-07T00:00:00.0000000
layout: post
categories: tapi
tags: tapi
---

So, a question was asked "How do I determine what's happening in the TAPI3 wrapper"?  The answer is you turn on the internal trace source -- ITapi3 was built with a build in tracing facility to tell you when it had any underlying interface or COM failures and it's easy to activate.  First, add an **Application Configuration File** to your project.  Open that file and add the following lines:

```xml
<?xml version="1.0" encoding="utf-8" ?>  
<configuration>  
   <system.diagnostics>  
      <sources>  
         <source name="ITapiTrace" switchName="tapiSwitch" switchType="System.Diagnostics.SourceSwitch">  
            <listeners>
                <add name="MyTraceLog" type="System.Diagnostics.TextWriterTraceListener" initializeData="MyTrace.txt" />  
            </listeners>  
         </source>  
      </sources>  
      <switches>  
         <add name="tapiSwitch" value="All" />  
      </switches>  
   </system.diagnostics>  
</configuration>
```

This will create a file called "MyTrace.txt" in your working directory.  The important line is the **source** tag which identifies the internal **TraceSource** object used by the ITapi3 library.  Inside this file will be the internal TAPI3 calls being made for your application.  As an example, the following trace shows me that several underlying COM errors occurred in the running of a simple TAPI3 application -- it was unable to retrieve the ITTerminal interface from the **ITAddressEvent** interface (which actually isn't really an error), failed to open the line (because Unimodem won't allow the media type VOICE to be passed for my modem), and failed to set the play list for this MSP -- **[0x80040216]** is actually a DirectShow error **[VFW_E_NOT_FOUND]**.

```console
ITapiTrace Verbose: 0 : Creating ITTAPI instance  
ITapiTrace Verbose: 0 : Hooking up connection sink to ITTAPI interface  
ITapiTrace Information: 0 : ITTapi::put_EventFilter(0x8001F) hr=0x0  
ITapiTrace Error: 0 : COM Hresult 0x80040004 "The MEDIATYPE passed in to this method was invalid." generated   
   at JulMar.Tapi3.TapiException.ThrowExceptionForHR(Int32 hr)  
   at JulMar.Tapi3.TTapi.RegisterCallNotifications(ITAddress* pitf, Int16 vbMonitor, Int16 vbOwner, Int32 supportedMediaTypes)  
   at JulMar.Tapi3.TAddress.Open(TAPIMEDIATYPES supportedMediaTypes)  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 5, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 4, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 3, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 2, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 1, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 0, Terminal=  
ITapiTrace Verbose: 0 : Processing TapiCallNotificationEventArgs: Event=CNE_OWNER, Call=TCall: 171360625 CS_OFFERING  
ITapiTrace Verbose: 0 : Processing TapiCallStateEventArgs: Call=TCall: 171360625 CS_OFFERING, State=CS_OFFERING, Cause=CEC_NONE  
ITapiTrace Error: 0 : COM Hresult 0x80040216 "" generated   
   at JulMar.Tapi3.TapiException.ThrowExceptionForHR(Int32 hr)  
   at JulMar.Tapi3.TTerminal.set_MediaPlayList(String[] fileList)  
   at AnsMachine.AutoAttendantForm.AnswerCall()  
   at AnsMachine.AutoAttendantForm.OnCallState(Object sender, TapiCallStateEventArgs e)  
ITapiTrace Verbose: 0 : Processing TapiCallStateEventArgs: Call=TCall: 171360625 CS_DISCONNECTED, State=CS_DISCONNECTED, Cause=CEC_DISCONNECT_NORMAL  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 5, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 4, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 3, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 2, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 1, Terminal=  
ITapiTrace Error: 0 : ITAddressEvent::get_Terminal failed hr=0x80040055  
ITapiTrace Verbose: 0 : Processing TapiAddressChangedEventArgs: Evt=AE_RINGING, Address=DSSP Line #1 - Address 0, Terminal=  
ITapiTrace Error: 0 : ITTapi::Shutdown hr=0x0
```
