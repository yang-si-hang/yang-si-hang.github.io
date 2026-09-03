---
layout: page
title: Soft Tissue Cutting
description:
img: assets/img/projectis/tissue-cut.png
importance: 1
category: work
selected: true
---

## Overview

This project develops an autonomous robotic cutting framework for heterogeneous soft tissue by combining active physical perception, pre-tensioning control, and imitation learning. Tissue stiffness is estimated through an EKF-based identification method, while an information-driven exploration strategy selects informative interaction regions. The inferred physical-semantic information is then incorporated into a Diffusion Policy for real-world robotic cutting.

## Demo

<div class="row">
  <div class="col-sm mt-3 mt-md-0">
    {% include video.liquid
       path="assets/video/projects/tissue-cut-faststart.mp4"
       class="img-fluid rounded z-depth-1"
       controls=true
    %}
  </div>
</div>