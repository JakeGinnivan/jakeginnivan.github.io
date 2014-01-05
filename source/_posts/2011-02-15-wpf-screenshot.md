---
layout: post
title: Taking Screenshots of WPF apps
metaTitle: Taking Screenshots of WPF apps
description: During out UI Automation build, when an error occurs it takes a screenshot. We were getting a blank modal window. Here is code that works.
revised: 2011-02-19
date: 2011-02-15
categories: [wpf]
migrated: true
comments: true
sharing: true
footer: true
permalink: /wpf-screenshot/
summary: | 
  

---
In my current project, we have quite an awesome UI automation setup running through every process (happy days path) in our WPF app.

Often we cannot resolve a UI element, and we will get an error like:

    White.Core.AutomationException: Failed to get UIElement ControlType=button,AutomationId=OkButton from DialogWindow

While this is great, often it is hard to figure out why this happens, could be due to a validation error on the previous screen, or a error popup etc. So out infrastructure will automatically take a screenshot whenever it catches a AutomationException.

The problem is the modal window that was often captured (and was blocking a button press on the main window) was coming out as a tiny little square with no contents:
<!-- more -->
![Blank Modal Dialogue][1]

I began digging, then realised that using GDI+ to take a screenshot of a WPF app doesn't seem to always work. It worked fine on my machine, but not our build machine.

After scouring the internet I put a all the solutions i could find into one big LinqPad script and ran it (this was my original code, so it went first):

    var bounds = System.Windows.Forms.Screen.GetBounds(Point.Empty);
    var bitmap = new Bitmap(bounds.Width, bounds.Height);
    using (var g = Graphics.FromImage(bitmap))
    {
        g.CopyFromScreen(Point.Empty, Point.Empty, bounds.Size);
    }

    bitmap.Dump();

.Dump() by the way works awesome with Bitmaps as well as everything else under the sun..

The class that ended up working for me is this:

    public class ScreenCapture
    {
        [DllImport("gdi32.dll")]
        static extern bool BitBlt(IntPtr hdcDest, int xDest, int yDest, int wDest, int hDest, IntPtr hdcSource, int xSrc, int ySrc, CopyPixelOperation rop);
        [DllImport("user32.dll")]
        static extern bool ReleaseDC(IntPtr hWnd, IntPtr hDc);
        [DllImport("gdi32.dll")]
        static extern IntPtr DeleteDC(IntPtr hDc);
        [DllImport("gdi32.dll")]
        static extern IntPtr DeleteObject(IntPtr hDc);
        [DllImport("gdi32.dll")]
        static extern IntPtr CreateCompatibleBitmap(IntPtr hdc, int nWidth, int nHeight);
        [DllImport("gdi32.dll")]
        static extern IntPtr CreateCompatibleDC(IntPtr hdc);
        [DllImport("gdi32.dll")]
        static extern IntPtr SelectObject(IntPtr hdc, IntPtr bmp);
        [DllImport("user32.dll")]
        public static extern IntPtr GetDesktopWindow();
        [DllImport("user32.dll")]
        public static extern IntPtr GetWindowDC(IntPtr ptr);

        public Bitmap CaptureScreenShot()
        {
            var sz = System.Windows.Forms.Screen.PrimaryScreen.Bounds.Size;
            var hDesk = GetDesktopWindow();
            var hSrce = GetWindowDC(hDesk);
            var hDest = CreateCompatibleDC(hSrce);
            var hBmp = CreateCompatibleBitmap(hSrce, sz.Width, sz.Height);
            var hOldBmp = SelectObject(hDest, hBmp);
            BitBlt(hDest, 0, 0, sz.Width, sz.Height, hSrce, 0, 0, CopyPixelOperation.SourceCopy | CopyPixelOperation.CaptureBlt);
            var bmp = Image.FromHbitmap(hBmp);
            SelectObject(hDest, hOldBmp);
            DeleteObject(hBmp);
            DeleteDC(hDest);
            ReleaseDC(hDesk, hSrce);

            return bmp;
        }
    }

And now I have:

![Working modal screenshot][2]


  [1]: /get/screenshots/blankModal.png
  [2]: /get/screenshots/properModal.png