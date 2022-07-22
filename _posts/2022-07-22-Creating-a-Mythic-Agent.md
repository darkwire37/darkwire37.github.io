layout: post
title: "Creating a Mythic Agent (in Python)"
date: 07-22-22 09:36:00 -0000
categories: Red-Team C2 Development Python

## Introduction
During my time learning cyber, I have encountered a couple Mythic Agents running in lab environments.  Mythic seemed like an effective C2 framework, so I figured I'd make an agent myself.  
Is there a better way to learn a system than develop it yourself?  I think not!

## Environment Setup
My environment was a pretty easy setup.  I am running Ubuntu 22.04 on VMWare Workstation 16 Pro. 
As for Mythic itself, I followed the setup instructions available [here.](https://docs.mythic-c2.net/installation)  This included even the Docker installation!
If all you wanted was a working C2 server with off the shelf payloads, you can stop here and you'll have a working C2 framework!  Pretty cool huh?!

