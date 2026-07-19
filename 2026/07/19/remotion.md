---
title: remotion
description: code to video with React
tags:
  - react
  - video
---

Remotion is a framework for creating videos programmatically with React[^2]. 🤯

Instead of arranging clips and animations in a conventional video editor, you describe
each frame using components, CSS, JavaScript, images, audio, and other web technologies.

Remotion then renders those frames and encodes them into formats such as MP4, WebM, GIF,
or image sequences.

## Concepts

A Remotion video begins with a **composition**.

A composition defines the React component to render, its dimensions, frame rate,
duration, and optional input properties. For example, a 10-second video running at 30
frames per second contains 300 frames.

Inside a component, `useCurrentFrame()` returns the frame currently being rendered.
Animations are therefore calculations based on frame numbers rather than timers.
A title might move from the bottom of the screen during frames 0–20, remain still, and
fade away during frames 70–90.

Remotion provides helpers for interpolation, spring animation, sequencing, transitions,
audio, and video playback.

During development, Remotion Studio (built into the toolchain) provides a browser-based
preview with a timeline and frame-by-frame scrubbing. Because the composition is
ordinary React code, changes benefit from the usual component model and fast refresh.

When exporting, Remotion renders the composition one frame at a time using a browser
engine, captures each frame as an image, and passes the resulting frames and audio to an
encoder[^1].

Frames may be rendered concurrently, so output should be deterministic and based on the
current frame and input properties rather than mutable state.

[^1]: compresses raw video or audio into a codec (compressed media format)

[^2]: The Fundamentals | Remotion | Make Videos Programmatically. 18 July 2026, <https://www.remotion.dev/docs/the-fundamentals.>
