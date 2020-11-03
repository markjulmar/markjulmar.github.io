---
title: Watermark TextBox in Windows Store apps
date: 2013-05-24T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm uwp dotnet
---

A common request for WSAs is to add a "watermark" to TextBox entries so users get a hint as to what is expected in the TextBox. You can see this in many **Search Charm** implementations as it allows a search hint to be provided via the [SearchPane.PlaceHolderText](http://msdn.microsoft.com/en-us/library/windows/apps/windows.applicationmodel.search.searchpane.placeholdertext.aspx) property.  However, the built-in TextBox in XAML doesn't have this feature (HTML does!).

There are a few custom controls out there which allow for this, however, I'm a big fan of behaviors over custom controls - so when I needed this, I created a custom behavior to apply to any existing XAML TextBox to provide a watermark:

![](/images/WatermarkText.jpg "WatermarkText")

The behavior sets the text to the watermark and then watches for focus changes to add/remove the text when there is no existing text entered.  It has two forms of activation - either you can add it to the normal **Interaction.Behaviors** collection:

```xml
<TextBox ...>
    <si:Interaction.Behaviors>
        <i:WatermarkTextBehavior WatermarkColor="Red" WatermarkText="Search Keyword..." />
    </si:Interaction.Behaviors>
</TextBox>
```

Here, the main advantage is you can set the watermark text color. The default is a gray color. Alternatively, you can use an attached property (Text) which will add a new behavior to the TextBox without needing the full syntax:

```xml
<TextBox ... i:WatermarkTextBehavior.Text="Search Keyword...">
</TextBox>
```

The code is part of the latest MVVMHelpers library and is checked in at: [WatermarkTextBoxBehavior.cs](https://github.com/markjulmar/mvvmhelpers/blob/04e100cebc75b9b2c4c72af26bfe8dc7b683dd83/MvvmHelpers.Portable/PlatformLibs/JulMar.Desktop/Interactivity/WatermarkTextBehavior.cs "WatermarkTextBoxBehavior.cs").

Enjoy!
