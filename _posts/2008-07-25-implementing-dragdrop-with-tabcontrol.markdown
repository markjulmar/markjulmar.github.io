---
title: Implementing Drag/Drop with TabControl
date: 2008-07-25T00:00:00.0000000
layout: post
categories: mvvm
tags: mvvm dotnet
---

It's been a long time since I blogged anything specific on WPF -- I've been doing a lot of it lately, along with some Silverlight.  Recently I was experimenting with dragging tabs around on a TabControl at runtime.  My end goal is really to implement it with Silverlight 2, but I've found it's much easier to prototype things in WPF and then port them over because the debugging experience is easier with WPF.  
  
I didn't want to create derived implementations of any classes - I wanted something that was non-intrusive to my code so I decided to use an [attached property](http://msdn.microsoft.com/en-us/library/ms749011.aspx).  Attached properties are basically property values "attached" to a class at runtime - where the property itself isn't defined on the target but instead on some other type.  The cool thing about attached properties is they can register a change notification handler which gives them a reference to the object they are being placed on -- this is how the Spell Checker works with the TextBox in WPF.  All the code for the spell checking lives in the SpellChecker class and when you add the SpellCheck.IsEnabled property onto the TextBox, it adds handlers to the TextBlock's TextChanged property and adds all the nifty spell checking goodness _without changing the code in TextBox_.  
  
Back to my drag/drop prototype.  So with this code, I can add the property to any TabControl and get a nice, simple drag/drop experience.  It's far from complete - it would be cooler if the tabs moved around as you dragged (they don't), but I was just prototyping here.  
  
Here's the code:  

```csharp
using System;
using System.Diagnostics;
using System.Windows;
using System.Windows.Controls;
using System.Windows.Input;
using System.Windows.Media;

namespace WpfApplication1
{
    public class DragDropTabManager
    {
        private static readonly DependencyProperty ManagerProperty =
            DependencyProperty.Register(typeof (DragDropTabManager).ToString(), typeof (DragDropTabManager),
                                        typeof (DragDropTabManager));

        public static readonly DependencyProperty EnabledProperty = 
            DependencyProperty.RegisterAttached("Enabled", typeof(bool), 
                                                typeof(DragDropTabManager),
                                                new PropertyMetadata(false, DDTM\_EnabledChanged));

        private static void DDTM\_EnabledChanged(DependencyObject d, DependencyPropertyChangedEventArgs e)
        {
            var tc = d as TabControl;
            if (tc != null)
            {
                var oldValue = (bool) e.OldValue;
                var newValue = (bool) e.NewValue;

                if (oldValue == true && newValue == false)
                {
                    var ddtm = tc.GetValue(ManagerProperty) as DragDropTabManager;
                    if (ddtm != null)
                    {
                        tc.PreviewMouseDown -= ddtm.TabItem\_PreviewMouseDown;
                        tc.SetValue(ManagerProperty, null);
                    }
                }
                else if (oldValue == false && newValue == true)
                {
                    var ddtm = new DragDropTabManager();
                    tc.SetValue(ManagerProperty, ddtm);
                    tc.PreviewMouseDown += ddtm.TabItem\_PreviewMouseDown;
                }
            }
        }

        public static bool GetEnabled(DependencyObject obj)
        {
            return (bool)obj.GetValue(EnabledProperty);
        }

        public static void SetEnabled(DependencyObject obj, bool value)
        {
            obj.SetValue(EnabledProperty, value);
        }

        private bool isMoving;
        private TabItem movingTabItem;
        private TabItem lastTab;
        private Point ptStart;

        void TabItem\_PreviewMouseDown(object sender, MouseEventArgs e)
        {
            var ti = e.Source as TabItem;
            if (ti != null && e.LeftButton == MouseButtonState.Pressed)
            {
                var tc = ti.Parent as TabControl;
                if (tc != null)
                {
                    tc.MouseMove += tc\_MouseMove;
                    tc.MouseLeftButtonUp += tc\_MouseLeftButtonUp;

                    ptStart = e.GetPosition(tc);
                    movingTabItem = ti;
                }
            }
        }

        void tc\_MouseMove(object sender, MouseEventArgs e)
        {
            var tc = sender as TabControl;
            if (tc == null)
                return;

            Point pt = e.GetPosition(tc);

            if (isMoving == false)
            {
                if (Math.Abs(pt.X - ptStart.X) > 10)
                {
                    movingTabItem.IsHitTestVisible = false;
                    movingTabItem.RenderTransformOrigin = new Point(.5, .5);
                    movingTabItem.RenderTransform = new TranslateTransform(0, 0);
                    tc.Cursor = Cursors.Hand;
                    Panel.SetZIndex(movingTabItem, 1);
                    isMoving = true;
                    tc.CaptureMouse();
                }
                return;
            }

            TabItem newPos = FindTabItem(tc, pt);
            if (newPos == null)
                tc.Cursor = Cursors.No;
            else
            {
                lastTab = newPos;
                var xform = movingTabItem.RenderTransform as TranslateTransform;
                if (xform != null)
                    xform.X = pt.X - ptStart.X;
                tc.Cursor = Cursors.Hand;
            }
        }

        void tc\_MouseLeftButtonUp(object sender, MouseButtonEventArgs e)
        {
            var tc = sender as TabControl;
            Debug.Assert(tc != null);

            tc.ReleaseMouseCapture();
            tc.MouseMove -= tc\_MouseMove;
            tc.MouseLeftButtonUp -= tc\_MouseLeftButtonUp;

            if (isMoving == true)
            {
                isMoving = false;
                tc.Cursor = Cursors.Arrow;
                movingTabItem.RenderTransform = null;
                movingTabItem.IsHitTestVisible = true;

                Panel.SetZIndex(movingTabItem, 0);

                if (lastTab != null)
                {
                    if (lastTab != null && movingTabItem != lastTab)
                    {
                        int targetIndex = tc.Items.IndexOf(lastTab);
                        tc.Items.Remove(movingTabItem);
                        tc.Items.Insert(targetIndex, movingTabItem);

                        movingTabItem.Focus();
                    }
                }
            }
            movingTabItem = lastTab = null;
        }

        private static TabItem FindTabItem(UIElement parent, Point pt)
        {
            var fe = parent.InputHitTest(pt) as FrameworkElement;
            while (fe != null && fe.GetType() != typeof(TabItem))
                fe = VisualTreeHelper.GetParent(fe) as FrameworkElement;
            return fe as TabItem;
        }
    }
}
```
