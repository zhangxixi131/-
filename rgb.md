using System;
using System.Drawing;
using System.Drawing.Drawing2D;
using System.Drawing.Text;
using System.Runtime.InteropServices;
using System.Windows.Forms;

class RgbPicker : Form
{
    [DllImport("user32.dll")] static extern bool SetProcessDPIAware();
    [DllImport("user32.dll")] static extern IntPtr GetDC(IntPtr hWnd);
    [DllImport("user32.dll")] static extern int ReleaseDC(IntPtr hWnd, IntPtr hDC);
    [DllImport("gdi32.dll")] static extern IntPtr CreateCompatibleDC(IntPtr hdc);
    [DllImport("gdi32.dll")] static extern IntPtr CreateCompatibleBitmap(IntPtr hdc, int w, int h);
    [DllImport("gdi32.dll")] static extern IntPtr SelectObject(IntPtr hdc, IntPtr hObj);
    [DllImport("gdi32.dll")] static extern bool BitBlt(IntPtr d, int x, int y, int w, int h, IntPtr s, int sx, int sy, int rop);
    [DllImport("gdi32.dll")] static extern uint GetPixel(IntPtr hdc, int x, int y);
    [DllImport("gdi32.dll")] static extern bool DeleteObject(IntPtr hObj);

    const int SRCCOPY = 0x00CC0020;
    const int CAPTUREBLT = 0x40000000;

    IntPtr memDC = IntPtr.Zero;
    IntPtr dib = IntPtr.Zero;

    Timer timer;
    Font fHex, fRgb;
    Color current = Color.Black;
    bool hasSample = false;

    bool dragging, moved;
    Point dragOffset;

    int copiedTicks = 0;
    string copiedText = "";

    readonly Color bg = Color.FromArgb(28, 28, 31);
    readonly Color border = Color.FromArgb(70, 70, 76);

    public RgbPicker()
    {
        FormBorderStyle = FormBorderStyle.None;
        StartPosition = FormStartPosition.Manual;
        TopMost = true;
        ShowInTaskbar = false;
        Text = "RGB 取色器";
        BackColor = bg;
        DoubleBuffered = true;

        fHex = new Font("Segoe UI", 12.5f, FontStyle.Bold, GraphicsUnit.Pixel);
        fRgb = new Font("Segoe UI", 12f, FontStyle.Regular, GraphicsUnit.Pixel);

        // 自适应宽度
        int wHex = TextRenderer.MeasureText("#FFFFFF", fHex).Width;
        int wRgb = TextRenderer.MeasureText("255 255 255", fRgb).Width;
        int need = 52 + Math.Max(wHex, wRgb) + 14;
        Size = new Size(need, 56);

        Rectangle wa = Screen.PrimaryScreen.WorkingArea;
        Location = new Point(wa.Right - Width - 40, wa.Bottom - Height - 40);

        timer = new Timer { Interval = 60 };
        timer.Tick += (s, e) => UpdateSample();
        timer.Start();
    }

    protected override void OnShown(EventArgs e)
    {
        base.OnShown(e);
        UpdateRoundedRegion();
        UpdateSample();
    }

    void UpdateRoundedRegion()
    {
        using (var path = RoundedPath(new Rectangle(0, 0, Width, Height), 9))
        {
            Region old = Region;
            Region = new Region(path);
            if (old != null) old.Dispose();
        }
    }

    static GraphicsPath RoundedPath(Rectangle r, int radius)
    {
        var p = new GraphicsPath();
        int d = radius * 2;
        p.AddArc(r.X, r.Y, d, d, 180, 90);
        p.AddArc(r.Right - d, r.Y, d, d, 270, 90);
        p.AddArc(r.Right - d, r.Bottom - d, d, d, 0, 90);
        p.AddArc(r.X, r.Bottom - d, d, d, 90, 90);
        p.CloseFigure();
        return p;
    }

    Color SamplePixel(int x, int y)
    {
        IntPtr screenDC = GetDC(IntPtr.Zero);
        try
        {
            if (memDC == IntPtr.Zero)
            {
                memDC = CreateCompatibleDC(screenDC);
                dib = CreateCompatibleBitmap(screenDC, 1, 1);
                SelectObject(memDC, dib);
            }
            BitBlt(memDC, 0, 0, 1, 1, screenDC, x, y, SRCCOPY | CAPTUREBLT);
            uint c = GetPixel(memDC, 0, 0);
            if (c == 0xFFFFFFFF) return current;
            return Color.FromArgb((int)(c & 0xFF), (int)((c >> 8) & 0xFF), (int)((c >> 16) & 0xFF));
        }
        finally
        {
            ReleaseDC(IntPtr.Zero, screenDC);
        }
    }

    void UpdateSample()
    {
        Point pt = Cursor.Position;
        // 光标落在本窗口上时不取样，避免读到自己的颜色
        if (Bounds.Contains(pt))
        {
            if (copiedTicks > 0) { copiedTicks--; Invalidate(); }
            return;
        }
        Color c = SamplePixel(pt.X, pt.Y);
        bool changed = !hasSample || c.R != current.R || c.G != current.G || c.B != current.B;
        current = c;
        hasSample = true;

        if (copiedTicks > 0)
        {
            copiedTicks--;
            changed = true;
        }
        if (changed) Invalidate();
    }

    protected override void OnPaint(PaintEventArgs e)
    {
        Graphics g = e.Graphics;
        g.SmoothingMode = SmoothingMode.AntiAlias;
        g.TextRenderingHint = TextRenderingHint.ClearTypeGridFit;

        using (var bgBrush = new SolidBrush(bg))
        using (var path = RoundedPath(new Rectangle(0, 0, Width - 1, Height - 1), 9))
        {
            g.FillPath(bgBrush, path);
            using (var pen = new Pen(border))
                g.DrawPath(pen, path);
        }

        // 色块
        Rectangle sw = new Rectangle(8, 11, 34, 34);
        using (var path = RoundedPath(sw, 5))
        {
            using (var b = new SolidBrush(current))
                g.FillPath(b, path);
            int lum = (current.R * 299 + current.G * 587 + current.B * 114) / 1000;
            using (var pen = new Pen(lum > 150 ? Color.FromArgb(60, 60, 60) : Color.FromArgb(120, 120, 120)))
                g.DrawPath(pen, path);
        }

        string hex = "#" + current.R.ToString("X2") + current.G.ToString("X2") + current.B.ToString("X2");
        using (var brush = new SolidBrush(Color.FromArgb(238, 238, 242)))
            g.DrawString(hex, fHex, brush, 51, 8);

        string rgb = copiedTicks > 0
            ? copiedText
            : current.R + "  " + current.G + "  " + current.B;
        using (var brush = new SolidBrush(copiedTicks > 0 ? Color.FromArgb(120, 200, 255) : Color.FromArgb(150, 152, 160)))
            g.DrawString(rgb, fRgb, brush, 52, 32);
    }

    void CopyText(string kind)
    {
        string hex = "#" + current.R.ToString("X2") + current.G.ToString("X2") + current.B.ToString("X2");
        string rgb = "rgb(" + current.R + ", " + current.G + ", " + current.B + ")";
        try
        {
            Clipboard.SetText(kind == "hex" ? hex : rgb);
            copiedText = "已复制 " + (kind == "hex" ? "HEX" : "RGB");
            copiedTicks = 18;
            Invalidate();
        }
        catch { }
    }

    protected override void OnMouseDown(MouseEventArgs e)
    {
        base.OnMouseDown(e);
        if (e.Button == MouseButtons.Left)
        {
            dragging = true;
            moved = false;
            dragOffset = Cursor.Position;
            dragOffset.Offset(-Left, -Top);
        }
    }

    protected override void OnMouseMove(MouseEventArgs e)
    {
        base.OnMouseMove(e);
        if (dragging)
        {
            Point now = Cursor.Position;
            Point target = new Point(now.X - dragOffset.X, now.Y - dragOffset.Y);
            if (Math.Abs(now.X - Left - dragOffset.X) > 2 || Math.Abs(now.Y - Top - dragOffset.Y) > 2)
                moved = true;
            Location = target;
        }
    }

    protected override void OnMouseUp(MouseEventArgs e)
    {
        base.OnMouseUp(e);
        if (e.Button == MouseButtons.Left)
        {
            bool wasDrag = moved;
            dragging = false;
            moved = false;
            if (!wasDrag) CopyText("hex");
        }
        else if (e.Button == MouseButtons.Right)
        {
            ShowMenu();
        }
    }

    void ShowMenu()
    {
        var menu = new ContextMenuStrip();
        var hex = new ToolStripMenuItem("复制 HEX  #" + current.R.ToString("X2") + current.G.ToString("X2") + current.B.ToString("X2"));
        hex.Click += (s, e) => CopyText("hex");
        var rgb = new ToolStripMenuItem("复制 RGB  " + current.R + ", " + current.G + ", " + current.B);
        rgb.Click += (s, e) => CopyText("rgb");
        var exit = new ToolStripMenuItem("退出");
        exit.Click += (s, e) => Close();
        menu.Items.Add(hex);
        menu.Items.Add(rgb);
        menu.Items.Add(new ToolStripSeparator());
        menu.Items.Add(exit);
        menu.Show(this, PointToClient(Cursor.Position));
    }

    protected override void OnKeyDown(KeyEventArgs e)
    {
        base.OnKeyDown(e);
        if (e.KeyCode == Keys.Escape) Close();
    }

    protected override void OnFormClosed(FormClosedEventArgs e)
    {
        timer.Stop();
        if (dib != IntPtr.Zero) DeleteObject(dib);
        base.OnFormClosed(e);
    }

    [STAThread]
    static void Main()
    {
        try { SetProcessDPIAware(); } catch { }
        Application.EnableVisualStyles();
        Application.Run(new RgbPicker());
    }
}

