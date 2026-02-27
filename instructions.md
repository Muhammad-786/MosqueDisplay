# Faizan-E-Madinah Display (MosqueDisplay Cloud)

## Overview
**Faizan-E-Madinah Display** is a rich, dynamic, and responsive digital signage clock and display system designed for mosques. It provides an elegant UI for showcasing prayer times, specifically Jummah schedules, accompanied by an animated media slideshow and real-time countdowns. 

It is a cloud-based application that stores configurations, prayer times, and slideshow images via **Firebase (Firestore + Authentication)**, allowing administrators to make real-time updates through a hidden Admin Panel without modifying code.

## Key Features
- **Dynamic Clock & Date:** Shows the exact time with an elegant typography style and the current date (with ordinal suffixes).
- **Jummah Times Schedule:** Displays Speech, Khutbah, and Jamaat timings configured safely via the cloud.
- **Auto Countdown System:** A smart countdown system that automatically detects Fridays and changes status based on timing:
  - Shows exact "Time Remaining" before Jamaat.
  - Switches to a "Jummah Mubarak" overlay during prayer and post-prayer timing.
  - Automatically resets to "Next Jummah" and the upcoming Friday's date for the rest of the week.
- **Live Media Slideshow:**
  - Dynamic loading of slideshow images from Firebase.
  - Subtle *Ken Burns* zoom effects for a premium feel.
  - Progress bar auto-syncing with the slideshow intervals.
  - Image size compression built-in to handle large uploads gracefully.
- **Customizable Appearance:** Real-time theme and layout switching:
  - **Themes:** Gold (Default), Emerald, Royal, Rose, and Custom (Pick your own hex colors).
  - **Layouts:** Classic, Media Focus, Fullscreen, and Vertical (great for portrait TVs).

---

## ⚙️ Setup and Configuration

The entire application runs from a single comprehensive file (`index.html`), making it exceptionally versatile and easy to deploy on any simple web host, GitHub Pages, or locally.

### 1. Firebase Configuration Overview
This app depends completely on **Google Firebase** for its backend functionality. 
If setting this up from scratch or migrating to a new Firebase project:
1. Create a Firebase Project at [Firebase Console](https://console.firebase.google.com/).
2. Enable **Firestore Database** (Create collections: `mosque_data` with a doc named `settings` and a `slides` collection).
3. Enable **Authentication** (Email/Password).
4. Add a Web App in settings, and replace the `const firebaseConfig = { ... }` object near the top of the `<script>` tag in `index.html` with your new Firebase credentials.
5. Add an admin user in the Authentication tab to use for logging into the admin panel.

### 2. Hosting & Deployment
No build step is required! 
You can run this display on:
- Any Smart TV Browser
- A Raspberry Pi connected to a TV (in Kiosk mode)
- Standard Web Hosting
- Netlify, Vercel, or GitHub pages.

To run locally, you can simply open `index.html` in your web browser, but running it via a local server (like `Live Server` in VSCode) is recommended to prevent any CORS issues with Firebase or local images.

---

## 🛡️ Admin Panel Guide

To provide non-technical users a way to change slides, colors, layouts, and timings, a built-in admin panel is included directly within the display interface.

### How to Access the Admin Panel
1. Open the Display application (`index.html`) in a browser.
2. Move your cursor over the **Bottom-Left corner** of the screen.
3. A subtle, glowing circular "trigger" button will fade into view. 
4. **Click the button** to open the Admin Modal overlay.
5. Sign in using your Firebase administrator Email and Password.

### Admin Features

#### 1. Timings Tab
- Update the **Speech**, **Khutbah**, and **Jamaat** hours and minutes.
- **Display Start Time:** Controls what time on Friday the system should kick in to start showing the active Jummah Countdown.

#### 2. Slideshow Tab
- **Upload Image File:** Pick an image from your computer to upload. The app includes automatic client-side compression to prevent exceeding Firestore document size limits.
- **Add image via URL:** Provide a direct link to an image.
- **Slide Manager:** 
  - Allows you to toggle the visibility of individual slides (Temporarily hide without deleting).
  - Use Up/Down arrows to logically order the slides in the rotation loop.
  - Use the Trash icon to permanently remove an image from the database.

#### 3. Appearance Tab
- **Themes:** Select the primary color accent for the application (Gold, Emerald, Royal, Rose).
- **Custom Color:** Alternatively, specify an exact hex color to match your mosque's branding.
- **Layouts:** Switch the interface formatting based on your screen setup without dipping into the CSS code.

> **Note:** Any changes made in the Admin Panel are instantly saved to Firebase and will propagate to any active displays simultaneously in real-time.

---

## 🛠️ Code Structure Overview

For developers intending to modify the raw source:
- **CSS Variables:** Top configuration definitions inside `<style>` (`:root`). All themes dynamically modify these CSS variables.
- **Responsiveness Layout Logic:** Implemented through `data-theme` and `data-layout` attributes on the `<body>` element. (E.g., `body[data-layout="vertical"]`).
- **State Management:** Controlled by `APP_CONFIG` within the script. Modifying this config allows for synchronous testing before database commits.
- **Authentication:** `firebase.auth().signInWithEmailAndPassword` guards the Admin Panel saving functionalities.
- **Real-time Sync:** The app actively listens to Firebase changes using `.onSnapshot()`. If an admin saves settings, the screen instantly reflects those changes without requiring a browser refresh.

---
*Created for Faizan-E-Madinah & Education Center.*
