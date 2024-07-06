---
extends: post.html
title: Rewiring a Leslie Switch
category: music
date: 2024-06-21 20:10:00
tags: ['music', 'electronics']
external_links:
    - title: Neo Ventilator II
      link: https://neo-instruments.com/ventilator-2
      description: Information on the Ventilator II. I got this one over the Mini-Vent for the ability to connect a
                   half-moon switch.
      
    - title: Hammond CU-1 switch
      link: https://hammondorganco.com/cu1-leslie-switch
      description: The Hammond CU-1 Leslie switch, probably the closest thing to a standard in this case. 
---

The Hammond B3 organ plays sound through an external speaker, usually a Leslie rotary cabinet. Far from being a
simple keyboard amp, a Leslie speaker consists of a compression driver at the top speaking into a rotating horn, and a
bass speaker underneath speaking into a baffle that rotates in the opposite direction, creating a combined Doppler and
tremolo effect. The rotating components can spin at a slow (40-50rpm) or fast (300-400rpm) speed, controlled by a
half-moon switch mounted onto the organ console.

Modern reproductions of the original instrument usually have an internal Leslie emulation that can be controlled by
a half-moon switch via a 6.35mm TRS connection. I acquired one of these a while ago and found that when I connected it
into my keyboard, slow and fast speeds worked fine but the stop setting didn't. Rather than buying another one, I
decided to see if I could mod it to work properly.

## Connections

Upon opening the switch up, I found that it had eight pins:

<figure>
  <img src="/img/leslie_switch.svg" alt="Leslie switch connections" />
  <p>Leslie switch connections</p>
</figure>

Several of these are connected together, presumably to make it easier to solder multiple connections, and only 1 and 8
are singular.

Investigating with a multimeter, I tested the connections between each pin when the switch was at different positions.
First with the switch at Stop, pins 2-7 were all connected together:

<figure>
  <img src="/img/leslie_switch_stop.svg" alt="Leslie switch connections - stop" />
  <p>Leslie switch connections - 2-3 -> 4-5 -> 6-7</p>
</figure>

When the switch was at slow, pins 2-5 and 8 were connected:

<figure>
  <img src="/img/leslie_switch_slow.svg" alt="Leslie switch connections - slow" />
  <p>Leslie switch connections - 2-3 -> 4-5 -> 8</p>
</figure>

Finally, when the switch was at fast, pins 1 and 4-7 were connected:

<figure>
  <img src="/img/leslie_switch_fast.svg" alt="Leslie switch connections - fast" />
  <p>Leslie switch connections - 1 -> 4-5 -> 6-7</p>
</figure>

Next, I connected an old TRS cable into a [Neo Ventilator II](https://neo-instruments.com/ventilator-2) and made
connections between tip, ring and sleeve in an attempt to figure out which ones gave me which Leslie speed. With the
Ventilator set to [CU-1](https://hammondorganco.com/cu1-leslie-switch) mode, I found:

- Connecting tip to sleeve sets the speed to slow
- Connecting ring to sleeve sets the speed to fast
- When none are connected, the speed will be set to stop

Equipped with this knowledge, I was able to get the soldering iron out and re-wire it as such:

<figure>
  <img src="/img/leslie_switch_trs.svg" alt="Leslie switch wiring" />
  <p>Tip in this case is on pin 8, ring is pin 1 and sleeve is pin 4 or 5.</p>
</figure>

I finally reassembled it and connected it up, and slow, stop and fast all worked beautifully!

## Footnote

Although disassembling things is all very well, be careful with components inside that help the thing roll, rotate or
click. You may accidentally remove a cover that you probably shouldn't and end up chasing ball bearings and springs
across the floor. Don't ask me how I know this.
