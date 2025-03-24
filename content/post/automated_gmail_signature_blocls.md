---
author: KilPen Tech
title: Automated Gmail Signature Blocks 
date: 2023-07-01
description: Promote your brand with consistent signature blocks across all your employees.
images: ["/images/solutions/signatures.jpg"]
---

## BUSINESS PROBLEM

Our client valued consistency in branding within email. Concerned that employees creating their own signature blocks in email would result in branding conflicts, KilPen was consulted to develop a company-wide signature standard in Gmail.

## OUR MISSION

Using Gmail and Google Cloud Console, KilPen needed to develop a technical solution that would replace user signature blocks domain wide with a HTML template that pulls user details from the Google Workspace company directory.

## OUR SOLUTION

Using Google Apps Scripting and Google Cloud, a tool was developed that iterated through the Workspace domain's users and executed a Google API call, with OAuth2 authentication, to replace their signature blocks. The signature block was dynamically recreated each day based on the HTML template and the person's name, title, address, and phone number stored in the company directory.

## TIMELINE

Three weeks