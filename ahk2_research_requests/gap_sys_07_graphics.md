# AHK2 Research Request: Graphics, Drawing & GDI+

## What I Need

I'm creating window overlays with semi-transparent GUIs, but I don't understand GDI+ or how to draw custom graphics. I want to create visual feedback - borders, highlights, annotations.

## Current Understanding (Challenge!)

- GDI+ is for advanced graphics
- I can only draw via GUI controls
- Transparency is handled by WinSetTransparent
- Drawing is slow (should be avoided)

**What's possible that I don't know about?**

## What I'm Building

Better window highlighting:
```ahk
// Current: Create GUI with solid color
gui := Gui("+AlwaysOnTop -Caption +ToolWindow")
gui.BackColor := "Blue"
WinSetTransparent 100
gui.Show("x" x " y" y " w" w " h" h " NoActivate")

// Want: Draw custom border, rounded corners, gradients
// Want: Draw arrows pointing to windows
// Want: Animated effects (fade in/out)
```

## Questions

1. How do I initialize GDI+ in AHK2?
2. How do I draw lines, rectangles, circles?
3. How do I apply transparency and gradients?
4. How do I draw on screen without creating windows?
5. How do I animate smoothly (60 FPS)?

## Guide Should Cover

- GDI+ initialization and cleanup
- Drawing primitives (lines, shapes, text)
- Color and alpha blending
- Creating custom window highlights
- Performance (when drawing becomes too slow)
- Complete example: Animated window border

## Success

Draw custom visual feedback that looks professional, not basic colored rectangles.
