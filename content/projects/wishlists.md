+++
title="Wish Lists Web App"
date="2024-12-12"
[taxonomies]
tags = ["FastAPI", "React"]
+++

[Wish Lists](https://github.com/DavidRambo/wishlists) is a web app for small groups of friends and family to exchange gift ideas.
The key feature is that users may indicate which gift they intend to give others without the recipients knowing.
A user has a different view of their own lift than they do of others'.

It's a cleaner interface and more secure than using email chains.
(This is how my wife's family handles wish lists. The recipient sends their list out, and then everyone replies to all minus the original sender with their pick.)
Compared to sharing a spreadsheet, it allows the recipient to add more ideas without getting spoiled.
Of course, it does take a bit more legwork to use compared to email or a spreadsheet.

To help with ease of use, I've added plaintext parsing, so a bunch of gift ideas can be entered in one go.
It had also been using a csv parsing microservice a fellow student wrote in JS.
I'd like to reimplement that as a built-in feature.

It uses [FastAPI](fastapi.tiangolo.com) and [sqlite](https://sqlite.org/) on the backend, and [React](react.dev) plus [React Router v6](https://reactrouter.com) for the frontend.
The frontend's code structure and visual design takes as its starting point the React Router [tutorial](https://reactrouter.com/en/main/start/tutorial).
Indeed, the CSS is heavily indebted to that tutorial's CSS.
