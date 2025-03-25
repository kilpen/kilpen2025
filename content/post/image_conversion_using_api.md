---
author: KilPen Tech
title: Image Conversion Using API
date: 2023-06-01
description: Image Conversion Using API
images: ["/images/solutions/pngtotiff.png"]
---

## BUSINESS PROBLEM

Our client needed to produce 1,000+ images using an image creation tool. The images exported from the tool in PNG format, but a TIFF format was needed. Doing this conversion by hand would take considerable resources.

## OUR MISSION

Develop an automation tool that would integrate into the existing product development workflow. The tool needs to seamlessly convert images into the TIFF format and sort output files into appropriate folders according to an information management scheme.

## OUR SOLUTION

Powered by Google Workplace and an API-based image converter, KilPen developed a solution that ran hourly checking for newly created images. A call was made to the API with the new image resulting in conversion to the desired format. Google Apps Scripting was used to retrieve the new image and save into Google Drive where a separate sorting algorithm placed files in the appropriate place.

## TIMELINE

One week
