---
layout: post
title: Remembering WPF Window Positions
metaTitle: Simple attached property to remember WPF window positions!
description: Want a WPF window to open up where it was last closed? It is as simple at WindowsSettings.Save="True"
revised: 2011-02-20
date: 2009-08-04
categories: [wpf]
migrated: true
comments: true
sharing: true
footer: true
permalink: /remembering-wpf-window-positions/
summary: | 
  

---
A few of my apps require opening the same window for a lot of different items and I wanted each WPF window to remember its size, position and state all the time. Nothing annoys me more than closing a window, opening the same window and its back on my second monitor.

So I went in search of a solution that someone else had written first, I found a example on the [MSDN website][1].What makes this sample good is it makes use of the Win32 API’s, specifically GetWindowPlacement and SetWindowPlacement in User32.dll. Because of this our code doesn't have to worry about missing second monitors, reduced resolution and other edge cases that can cause the normal headaches.

The [second example][2] I found has the idea of using attached properties, which I liked a lot.

The problem with the MSDN sample is you need to set up each window manually by overriding some of the Windows methods, the second solution did not cater for the edge cases.
<!-- more -->
<h1>How do we do it?</h1>
The first problem is persisting the data, I wanted to be able to store the position across sessions for multiple windows AND store the class in a reusable library. So I decided to create a internal ApplicationSettings class within my class to store each window, passing the Type.FullName of the window to the ApplicationSettings constructor to make sure settings names are not doubled up.

    internal class WindowApplicationSettings : ApplicationSettingsBase
    {
        public WindowApplicationSettings(WindowSettings windowSettings)
            : base(windowSettings._window.GetType().FullName) 
        {
        }

        [UserScopedSetting]
        public WINDOWPLACEMENT? Placement
        {
            get
            {
                if (this["Placement"] != null)
                {
                    return ((WINDOWPLACEMENT)this["Placement"]);
                }
                return null;
            }
            set
            {
                this["Placement"] = value;
            }
        }
    }

Next we can create the attached property which when set to True hooks into the window it is attaching to and subscribes to the needed events.

    /// <summary>
    /// Register the "Save" attached property and the "OnSaveInvalidated" callback 
    /// </summary>
    public static readonly DependencyProperty SaveProperty
        = DependencyProperty.RegisterAttached("Save", typeof(bool), typeof(WindowSettings),
                                              new FrameworkPropertyMetadata(new PropertyChangedCallback(OnSaveInvalidated)));

    public static void SetSave(DependencyObject dependencyObject, bool enabled)
    {
        dependencyObject.SetValue(SaveProperty, enabled);
    }

    /// <summary>
    /// Called when Save is changed on an object.
    /// </summary>
    private static void OnSaveInvalidated(DependencyObject dependencyObject, DependencyPropertyChangedEventArgs e)
    {
        var window = dependencyObject as Window;
        if (window == null || !((bool) e.NewValue)) return;
        var settings = new WindowSettings(window);
        settings.Attach();
    }

    /// <summary>
    /// Save the Window Size, Location and State to the settings object
    /// </summary>
    protected virtual void SaveWindowState()
    {
        WINDOWPLACEMENT wp;
        var hwnd = new WindowInteropHelper(_window).Handle;
        GetWindowPlacement(hwnd, out wp);
        Settings.Placement = wp;
        Settings.Save();
    }

    private void Attach()
    {
        if (_window == null) return;
        _window.Closing += WindowClosing;
        _window.SourceInitialized += WindowSourceInitialized;
    }

    void WindowSourceInitialized(object sender, EventArgs e)
    {
        LoadWindowState();
    }

    private void WindowClosing(object sender, CancelEventArgs e)
    {
        SaveWindowState();
        _window.Closing -= WindowClosing;
        _window.SourceInitialized -= WindowSourceInitialized;
        _window = null;
    }

The key is the SourceInitialized event, it lets us know when the underlying Win32 Window is ready and we can make a call to SetWindowPlacement.

    /// <summary>
    /// Load the Window Size Location and State from the settings object
    /// </summary>
    protected virtual void LoadWindowState()
    {
        Settings.Reload();

        if (Settings.Placement == null) return;
        try
        {
            // Load window placement details for previous application session from application settings
            // if window was closed on a monitor that is now disconnected from the computer,
            // SetWindowPlacement will place the window onto a visible monitor.
            var wp = Settings.Placement.Value;

            wp.length = Marshal.SizeOf(typeof (WINDOWPLACEMENT));
            wp.flags = 0;
            wp.showCmd = (wp.showCmd == SW_SHOWMINIMIZED ? SW_SHOWNORMAL : wp.showCmd);
            var hwnd = new WindowInteropHelper(_window).Handle;
            SetWindowPlacement(hwnd, ref wp);
        }
        catch (Exception ex)
        {
            Debug.WriteLine(string.Format("Failed to load window state:\r\n{0}", ex));
        }
    }

After we close the window we get this in our user.config file.

    <userSettings> 
        <ioWpf.WindowSettings_x002B_WindowApplicationSettings.ParameterPopulator.Views.MainView>
            <setting name="Placement" serializeAs="Xml"> 
                <value> 
                    <WINDOWPLACEMENT xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
                        xmlns:xsd="http://www.w3.org/2001/XMLSchema"> 
                        <length>44</length> 
                        <flags>0</flags> 
                        <showCmd>1</showCmd> 
                        <minPosition> 
                            <X>-1</X> 
                            <Y>-1</Y> 
                        </minPosition> 
                        <maxPosition> 
                            <X>-1</X> 
                            <Y>-1</Y> 
                        </maxPosition> 
                        <normalPosition> 
                            <Left>93</Left> 
                            <Top>371</Top> 
                            <Right>893</Right> 
                            <Bottom>771</Bottom> 
                        </normalPosition> 
                    </WINDOWPLACEMENT> 
                </value> 
            </setting> 
        </ioWpf.WindowSettings_x002B_WindowApplicationSettings.ParameterPopulator.Views.MainView>
    </userSettings>

There you have it, both a simple and very robust way to remember WPF windows positions.

<h1>Complete Source</h1>

    /// <summary>
    /// Persists a Window's Size, Location and WindowState to UserScopeSettings 
    /// </summary>
    public class WindowSettings
    {
        [DllImport("user32.dll")]
        static extern bool SetWindowPlacement(IntPtr hWnd, [In] ref WINDOWPLACEMENT lpwndpl);

        [DllImport("user32.dll")]
        static extern bool GetWindowPlacement(IntPtr hWnd, out WINDOWPLACEMENT lpwndpl);

        // ReSharper disable InconsistentNaming
        const int SW_SHOWNORMAL = 1;
        const int SW_SHOWMINIMIZED = 2;
        // ReSharper restore InconsistentNaming

        internal class WindowApplicationSettings : ApplicationSettingsBase
        {
            public WindowApplicationSettings(WindowSettings windowSettings)
                : base(windowSettings._window.GetType().FullName) 
            {
            }

            [UserScopedSetting]
            public WINDOWPLACEMENT? Placement
            {
                get
                {
                    if (this["Placement"] != null)
                    {
                        return ((WINDOWPLACEMENT)this["Placement"]);
                    }
                    return null;
                }
                set
                {
                    this["Placement"] = value;
                }
            }
        }
        private Window _window;

        public WindowSettings(Window window)
        {
            _window = window;
        }

        /// <summary>
        /// Register the "Save" attached property and the "OnSaveInvalidated" callback 
        /// </summary>
        public static readonly DependencyProperty SaveProperty
            = DependencyProperty.RegisterAttached("Save", typeof(bool), typeof(WindowSettings),
                                                  new FrameworkPropertyMetadata(new PropertyChangedCallback(OnSaveInvalidated)));

        public static void SetSave(DependencyObject dependencyObject, bool enabled)
        {
            dependencyObject.SetValue(SaveProperty, enabled);
        }

        /// <summary>
        /// Called when Save is changed on an object.
        /// </summary>
        private static void OnSaveInvalidated(DependencyObject dependencyObject, DependencyPropertyChangedEventArgs e)
        {
            var window = dependencyObject as Window;
            if (window == null || !((bool) e.NewValue)) return;
            var settings = new WindowSettings(window);
            settings.Attach();
        }

        /// <summary>
        /// Load the Window Size Location and State from the settings object
        /// </summary>
        protected virtual void LoadWindowState()
        {
            Settings.Reload();

            if (Settings.Placement == null) return;
            try
            {
                // Load window placement details for previous application session from application settings
                // if window was closed on a monitor that is now disconnected from the computer,
                // SetWindowPlacement will place the window onto a visible monitor.
                var wp = Settings.Placement.Value;

                wp.length = Marshal.SizeOf(typeof (WINDOWPLACEMENT));
                wp.flags = 0;
                wp.showCmd = (wp.showCmd == SW_SHOWMINIMIZED ? SW_SHOWNORMAL : wp.showCmd);
                var hwnd = new WindowInteropHelper(_window).Handle;
                SetWindowPlacement(hwnd, ref wp);
            }
            catch (Exception ex)
            {
                Debug.WriteLine(string.Format("Failed to load window state:\r\n{0}", ex));
            }
        }

        /// <summary>
        /// Save the Window Size, Location and State to the settings object
        /// </summary>
        protected virtual void SaveWindowState()
        {
            WINDOWPLACEMENT wp;
            var hwnd = new WindowInteropHelper(_window).Handle;
            GetWindowPlacement(hwnd, out wp);
            Settings.Placement = wp;
            Settings.Save();
        }

        private void Attach()
        {
            if (_window == null) return;
            _window.Closing += WindowClosing;
            _window.SourceInitialized += WindowSourceInitialized;
        }

        void WindowSourceInitialized(object sender, EventArgs e)
        {
            LoadWindowState();
        }

        private void WindowClosing(object sender, CancelEventArgs e)
        {
            SaveWindowState();
            _window.Closing -= WindowClosing;
            _window.SourceInitialized -= WindowSourceInitialized;
            _window = null;
        }

        private WindowApplicationSettings _windowApplicationSettings;

        internal virtual WindowApplicationSettings CreateWindowApplicationSettingsInstance()
        {
            return new WindowApplicationSettings(this);
        }

        [Browsable(false)]
        internal WindowApplicationSettings Settings
        {
            get
            {
                if (_windowApplicationSettings == null)
                {
                    _windowApplicationSettings = CreateWindowApplicationSettingsInstance();
                }
                return _windowApplicationSettings;
            }
        }
    }


    [Serializable]
    [StructLayout(LayoutKind.Sequential)]
    public struct RECT
    {
        private int _left;
        private int _top;
        private int _right;
        private int _bottom;

        public RECT(int left, int top, int right, int bottom)
        {
            _left = left;
            _top = top;
            _right = right;
            _bottom = bottom;
        }

        public override bool Equals(object obj)
        {
            if (obj is RECT)
            {
                var rect = (RECT)obj;

                return rect._bottom == _bottom &&
                       rect._left == _left &&
                       rect._right == _right &&
                       rect._top == _top;
            }
            return base.Equals(obj);
        }

        public override int GetHashCode()
        {
            return _bottom.GetHashCode() ^
                   _left.GetHashCode() ^
                   _right.GetHashCode() ^
                   _top.GetHashCode();
        }

        public static bool operator ==(RECT a, RECT b)
        {
            return a._bottom == b._bottom &&
                   a._left == b._left &&
                   a._right == b._right &&
                   a._top == b._top;
        }

        public static bool operator !=(RECT a, RECT b)
        {
            return !(a == b);
        }

        public int Left
        {
            get { return _left; }
            set { _left = value; }
        }

        public int Top
        {
            get { return _top; }
            set { _top = value; }
        }

        public int Right
        {
            get { return _right; }
            set { _right = value; }
        }

        public int Bottom
        {
            get { return _bottom; }
            set { _bottom = value; }
        }
    }

    [Serializable]
    [StructLayout(LayoutKind.Sequential)]
    public struct POINT
    {
        private int _x;
        private int _y;

        public POINT(int x, int y)
        {
            _x = x;
            _y = y;
        }

        public int X
        {
            get { return _x; }
            set { _x = value; }
        }

        public int Y
        {
            get { return _y; }
            set { _y = value; }
        }

        public override bool Equals(object obj)
        {
            if (obj is POINT)
            {
                var point = (POINT) obj;

                return point._x == _x && point._y == _y;
            }
            return base.Equals(obj);
        }
        public override int GetHashCode()
        {
            return _x.GetHashCode() ^ _y.GetHashCode();
        }

        public static bool operator ==(POINT a, POINT b)
        {
            return a._x == b._x && a._y == b._y;
        }

        public static bool operator !=(POINT a, POINT b)
        {
            return !(a == b);
        }
    }

    [Serializable]
    [StructLayout(LayoutKind.Sequential)]
    public struct WINDOWPLACEMENT
    {
        public int length;
        public int flags;
        public int showCmd;
        public POINT minPosition;
        public POINT maxPosition;
        public RECT normalPosition;
    }



  [1]: http://msdn.microsoft.com/en-us/library/aa972163.aspx
  [2]: http://bloggingabout.net/blogs/erwyn/archive/2007/02/01/remembering-window-positions-in-wpf.aspx
