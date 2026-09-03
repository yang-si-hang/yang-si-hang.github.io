---
layout: page
title: Pi0.5 Post-train
description:
img: assets/img/projectis/pi0.5-post-train.png
importance: 1
category: personal
selected: true
---

## Overview

This project focuses on adapting vision-language-action models to real-world robotic manipulation. I built an end-to-end VLA pipeline on a UR10e robot, covering teleoperation-based demonstration collection, multimodal data processing in LeRobot format, π0.5 post-training with LoRA, and real-time policy deployment. I further integrated RTC-based asynchronous action-chunk inference to enable continuous execution on the physical robot.

## Demo

<div class="row">
  <div class="col-sm mt-3 mt-md-0">
    {% include video.liquid
       path="assets/video/projects/pi0.5-pick-place-faststart.mp4"
       class="img-fluid rounded z-depth-1"
       controls=true
    %}
  </div>
</div>