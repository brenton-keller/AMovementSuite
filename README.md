# Windows Scaling Suite

Windows Scaling Suite is an AutoHotkey v2.0-based collection of scripts that gives you flexible window management at your fingertips. It lets you dynamically resize, move, snap, and even set windows as always-on-top using simple mouse and keyboard combinations.

---

## Features

- **Width Scaling**  
  Toggle width scaling with *Left Ctrl + Right Mouse Button*. Adjust window width on the fly.

- **Height Scaling**  
  Toggle height scaling with *Left Shift + Right Mouse Button*. Adjust window height interactively.

- **XY (Proportional) Scaling**  
  Enable proportional scaling with *Left Ctrl + Left Shift + Right Mouse Button*. A preview GUI lets you see the changes before applying them.

- **Window Movement with Snapping**  
  Move windows using *Left Alt + Right Mouse Button*. The module supports:
  - Edge snapping *s* to other open windows.
  - Screen (monitor) edge snapping.
  - A settings GUI to customize the “snap distance.”
  - Built-in safety checks to re-enable normal behavior if keys are released.

- **Window Positioning Grid**  
  Press the `Z` key while moving to bring up a grid-based positioning menu. Snap windows to common layouts (e.g., left/right halves, thirds, or quarters) using on-screen buttons.

- **Always-on-Top Toggle**  
  Use *Left Control + Middle Mouse Button* to quickly toggle the always-on-top state of a window, with visual notifications confirming the change.

- **Modular Design**  
  The suite is split into several include files:
  - **WindowScaleColors.ahk2** – Contains color and minimum dimension definitions.
  - **WindowScaleWidth.ahk2** and **WindowScaleHeight.ahk2** – Manage directional scaling.
  - **WindowScaleProportional.ahk2** and **WindowScaleXY.ahk2** – Provide proportional scaling with a preview.
  - **WindowMove.ahk2** – Handles window moving, snapping, and positioning.
  - **WindowAlwaysOnTop.ahk2** – Implements the always-on-top functionality.

---

## Getting Started

1. **Install AutoHotkey v2.0**  
   Download and install AutoHotkey version 2.0 from the [official website](https://www.autohotkey.com/).

2. **Clone/Download the Repository**  
   Clone or download the Windows Scaling Suite repository into your local machine.

3. **Run the Main Script**  
   Launch the main script (e.g., `WindowScaling.ahk2`). A system tray icon will appear—this is your central hub for toggling features.

---

## Usage

- **Tray Menu Options:**  
  Right-click the tray icon to toggle individual features:
  - *Width Scaling (Ctrl+RButton)*
  - *Height Scaling (Shift+RButton)*
  - *XY Scaling (Ctrl+Shift+RButton)*
  - *Window Move (LAlt+RButton)*
  - Open settings for window move (to adjust snap distance).

- **Resizing Windows:**
  - To scale, hold the appropriate modifier keys while right-clicking (using the corresponding combination for width, height, or proportional scaling).
  - A preview window appears (if proportional scaling is enabled) to show you the new size and position before the change is applied.

- **Moving Windows:**
  - Hold *Left Alt* and the right mouse button.  
  - If you hold the `S` key during movement, snapping (to either screen or window edges) will be enabled.
  - Press the `Z` key during movement to bring up an on-screen grid for quick positioning.

- **Always-on-Top:**
  - Hover over a window and press *Left Control + Middle Mouse Button* to toggle its topmost state.
  - The script validates whether the window is eligible (for example, system windows are excluded).

---

## Customization

- **Snap Distance:**  
  Adjust the snap distance (in pixels) via the *Window Move Settings* tray option.

- **Modifier Keys:**  
  The default hotkeys use common modifiers (Ctrl, Shift, Alt). Feel free to change these assignments in the code if needed.

- **Excluded Windows:**  
  The scripts include helper functions (e.g., `IsValidWindowForResize` and `IsValidWindowForMove`) to ignore specific window types. Modify the excluded classes or styles to better suit your environment.

- **Visual Appearance:**  
  The preview GUI uses color constants defined in the `WindowScaleColors.ahk2` include. Update these values to customize the look and feel.

---

## Contributing

Contributions, issues, and feature requests are welcome!  
Feel free to fork the repository and submit pull requests.

---

## License

Distributed under the MIT License. See `LICENSE` for more information.

---

Happy scaling and window managing! Enjoy the power of precise (and fun) window control on your Windows system. 