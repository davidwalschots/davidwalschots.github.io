---
layout: post
title:  "Simulating an aircraft in MSFS"
date:   2024-01-12 17:33:28 +0100
categories: rust msfs
---

I've had a fascination for aviation for a very long time. Back when I was 18 years old I attempted to become a commercial pilot. Sadly a form of child epilepsy with seizures that happened when I was between the age of 7 and 10 put a stop to that. Luckily flight simulators such as Microsoft's Flight Simulator were able to cover the hole that was left.

Microsoft released its latest Flight Simulator (MSFS) back in August 2020. In the years prior to its release I remember sometimes fiddling around with simulating some aircraft systems in a simple .NET WinForms application. Nothing major. Just the buttons on the overhead panel with some reads of voltages, pressure, and the like. After the release of MSFS various open source groups got started on creating open source aircraft for the simulator. I joined one of them: [FlyByWire Simulations](https://flybywiresim.com/) (FBW). Their goal was to create a free and open source Airbus A320neo for MSFS. Initially we based our work on the default A320neo that ships with the simulator. We then started to replace the default systems with our own.

## The default aircraft system architecture

The Airbus A320 cockpit contains various displays showing the pilot information about the aircraft's systems, its attitude, velocity, and the like. Within the default MSFS aircraft these displays would not only run the actual display code, but also the code for the systems that they were displaying. Thus, the display would determine e.g. the position of a button on the overhead panel and then process that information to make some visual change on the display. All this code was written in JavaScript, and the displays themselves were written in HTML with SVG for the graphics.

Such an approach is okay for a simple aircraft, but we wanted to create something grand: a fully functional and fully simulated aircraft. So we needed something better.

## The FBW aircraft system architecture

Microsoft lets developers provide WebAssembly modules to the simulator. These modules can be written in any language that compiles to WebAssembly. Microsoft provides an SDK library for interacting with the simulator. Initially some system programming was done in C++, but progress quickly stalled. Then one of FBW's team members ([devsnek](https://github.com/devsnek)) created [a crate](https://github.com/flybywiresim/msfs-rs) which provides safe bindings to the MSFS SDK and advised us to use Rust. By providing the safe bindings they made it easier for anyone, including those without prior non-garbage collected language experience, to get started on the project. This is when I got to work.

### Design motivation

Good software design makes implementing new features easier. While we cannot foresee every feature beforehand, going through the major features ensures a consistent design which is easy to work with and understand. In my opinion, good software design should primarily focus on defining the structural concepts that exist in the software. The amount of concepts should be limited, as to not overburden those who develop within it with the continuous question of: "should I use concept _x_ or _y_ to achieve _z_?".


### Requirements

I set the following requirements:

1. The majority of the software should be executable outside of the simulator, as this allows for testing and debugging without having to start the simulator.


https://github.com/davidwalschots/rfcs/blob/systems-design/text/000-systems-design.md
