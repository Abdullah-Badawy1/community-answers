# Hyprland Screen Sharing Solution Documentation

## Problem Description

**Issue**: Screen sharing was not working in Zoom, OBS, and Discord applications on Arch Linux with Hyprland (Wayland compositor).

**Environment**:
- OS: Arch Linux
- Desktop Environment: Hyprland (Wayland compositor)
- Session Type: Wayland
- Affected Applications: Zoom, OBS Studio, Discord

## Root Cause Analysis

The problem occurred because:

1. **Hyprland** is a Wayland compositor that requires specific portal backends for screen sharing
2. Only `xdg-desktop-portal-gtk` was installed, which doesn't properly support screen sharing on Hyprland
3. **Missing Hyprland-specific portal backend** (`xdg-desktop-portal-hyprland`)
4. No proper portal configuration to prioritize the correct backends

## Technical Background

### What are XDG Desktop Portals?

XDG Desktop Portals are a framework that allows sandboxed applications (like Flatpaks) and regular applications to access system resources securely. For screen sharing, they provide:

- **Screen capture capabilities**
- **Application window selection**
- **Permission management**
- **Cross-compositor compatibility**

### Portal Architecture

```
Application (Zoom/Discord/OBS)
         ↓
    xdg-desktop-portal (main service)
         ↓
xdg-desktop-portal-hyprland (backend)
         ↓
    Hyprland compositor
         ↓
    Wayland screen capture
```

## Solution Steps

### 1. Package Installation

Installed the required packages:

```bash
sudo pacman -S xdg-desktop-portal-hyprland
sudo pacman -S xdg-desktop-portal-wlr  # Fallback option
```

**Why these packages?**
- `xdg-desktop-portal-hyprland`: Hyprland-specific implementation for screen sharing
- `xdg-desktop-portal-wlr`: Generic wlroots-based compositor support (fallback)

### 2. Service Management

Restarted portal services:

```bash
killall xdg-desktop-portal-hyprland xdg-desktop-portal-wlr xdg-desktop-portal-gtk xdg-desktop-portal
# Services auto-restart via systemd socket activation
```

### 3. Portal Configuration

Created `/home/abdo/.config/xdg-desktop-portal/portals.conf`:

```ini
[preferred]
default=hyprland;wlr;gtk
org.freedesktop.impl.portal.ScreenCast=hyprland;wlr
org.freedesktop.impl.portal.Screenshot=hyprland;wlr
```

**Configuration Explanation**:
- `default=hyprland;wlr;gtk`: Priority order for general portal operations
- `org.freedesktop.impl.portal.ScreenCast=hyprland;wlr`: Screen sharing backend priority
- `org.freedesktop.impl.portal.Screenshot=hyprland;wlr`: Screenshot backend priority

### 4. Service Verification

Verified running services:

```bash
ps aux | grep xdg-desktop-portal
```

Expected output should show:
- `/usr/lib/xdg-desktop-portal` (main service)
- `/usr/lib/xdg-desktop-portal-hyprland` (Hyprland backend)
- `/usr/lib/xdg-desktop-portal-gtk` (GTK backend)

## How the Solution Works

1. **Applications** (Zoom, Discord, OBS) request screen sharing via D-Bus
2. **xdg-desktop-portal** receives the request
3. **Portal configuration** routes screen sharing to `xdg-desktop-portal-hyprland`
4. **Hyprland backend** interfaces with the Wayland compositor
5. **Screen content** is captured and shared with the application

## Dependencies and Requirements

### Required Packages
- `xdg-desktop-portal` (already installed)
- `xdg-desktop-portal-hyprland` (newly installed)
- `pipewire` (already installed - for audio/video)
- `wireplumber` (already installed - pipewire session manager)

### Optional Packages
- `xdg-desktop-portal-wlr` (fallback support)
- `grim` (screenshot tool - already installed)
- `slurp` (area selection - already installed)

## Testing the Solution

### For Each Application:

**Zoom**:
1. Start a meeting
2. Click "Share Screen"
3. Select screen or window
4. Screen sharing should work

**Discord**:
1. Join a voice channel
2. Click screen share button
3. Select screen or application
4. Screen sharing should work

**OBS**:
1. Add new source
2. Look for "PipeWire Screen Capture" or similar
3. Select screen or window
4. Capture should work

## System Impact Analysis

### Benefits:
- ✅ Screen sharing works in all major applications
- ✅ Native Wayland support (better performance)
- ✅ Secure permission handling
- ✅ No compatibility mode needed

### Package Analysis:
- `xdg-desktop-portal-hyprland`: Essential for Hyprland screen sharing
- `xdg-desktop-portal-wlr`: Provides fallback support, lightweight

---
