---
title: "TypeScript Asteroids"
date: "2024-02-04"
lead: "TypeScript Asteroids"
disable_comments: false # Optional, disable Disqus comments if true
authorbox: true # Optional, enable authorbox for specific post
toc: true # Optional, enable Table of Contents for specific post
mathjax: true # Optional, enable MathJax for specific post
categories:
  - "asteroids"
tags:
  - "asteroids"
---

In 2010 Doug McInnes [made](https://github.com/dmcinnes/HTML5-Asteroids)
[this cool JavaScript+HTML5 Asteroids clone](http://www.dougmcinnes.com/2010/05/12/html-5-asteroids).
I've
[converted it to TypeScript](https://github.com/bpcreech/typescript-asteroids);
it's live [here](https://bpcreech.com/asteroids/)!

<!--more-->

## Asteroids

[Asteroids](https://www.arcade-museum.com/Videogame/asteroids) is one of my
favorite video games. Many years ago, the daycare I grew up in (!) had an
original tabletop Asteroids game for us munchkins to play with. I spent hours on
that thing instead of playing sportsball or getting vitamin D. Asteroids is
super cool in both its simplicity, and its use of vector graphics—as in, the
original CRT didn't just do the horizontal scan thing; the tube traced out the
vector pattern on the screen, resulting in things like small shapes showing up
brighter.

So I have spent random minutes playing
[Doug McInnes's JS asteroids clone](https://github.com/dmcinnes/HTML5-Asteroids)
to reset my brain, for years since he first published it back in 2010.

<div style="text-align: center;">
  
![An original Asteroids "Cocktail"-style tabletop game](/img/asteroids-cocktail.jpg) </p>

_An original Asteroids "Cocktail"-style tabletop game;
[source](https://arcadespecialties.com/arcade-games-for-sale/vintage-arcade-games/asteroids-cocktail/)_

</div>

## TypeScriptification

I wanted to play with ML reinforcement learning (more on that below), which
requires me first to do some code refactoring on the game. Notably, to do
training I want to be able to programmatically:

1. Reset the game,
2. Single-step the game loop,
3. Seed the RNG,
4. Extract internal state for observation, and
5. Decouple UI from business logic, so we can run the game logic without a full
   browser (for both simplicity and speed).

I'm not a good enough JS programmer to safely refactor piles of raw JS, so ...
[I converted it to TypeScript](https://bpcreech.com/asteroids)!

This uses:

- [Vite](https://vitejs.dev/) to manage the dev and build toolchain,
- [ESLint](https://eslint.org/) for code linting,
- [Prettier](https://prettier.io/) for formatting, and
- [Github Actions](https://github.com/features/actions) for CI/CD.

## ML reinforcement learning??

**This is still a TODO and will get another post if I manage to operationalize
anything.**

My weird idea is to replace the game's demo screen with a live machine-operated
game, maybe operated by [TensorFlow.js](https://www.tensorflow.org/js). There
has already
[been extensive research in training robots to play Asteroids](https://www.gymlibrary.dev/environments/atari/asteroids/);
what I want to try differently here is specifically train on this
browser-playable game.
