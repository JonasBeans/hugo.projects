---
date: '2026-04-06T17:51:18+02:00'
draft: false
title: 'Scrap Rover webinterface update'
---

## Description of the update

This update covers the addition of camera movement controls to the Scrap Rover web interface — both as clickable HTML buttons and keyboard shortcuts, with keypresses visualised live using Keyviz. The current problem of dropped inputs (caused by in-flight HTTP requests being ignored) is addressed by introducing a request queue, ensuring every keypress is handled in order. The next milestone is gamepad controller support via the Web Gamepad API.

## Showcase video 

{{< youtube gsEMHnSeeK4 >}}

