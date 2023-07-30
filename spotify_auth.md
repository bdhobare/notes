# Integrating with the Spotify API from Rust
The Spotify [Web API](https://developer.spotify.com) allows you, among other things, to:
- Retrieve data about an artist, album or track;
- Control and interact with playback(e.g. play, stop, seek etc.);
- Search music on spotify;
- Fetch and manage personal library(e.g., creating a new playlist; adding tracks to it, etc.).

Here we are going to implement a Rust web service that authenticates with the API and fetches the tracks on your Discover Weekly playlist.

## How Spotify API Auth works
![auth.png](auth.png)
- When the user wants to authorize access to their Spotify Account, they are taken to https://accounts.spotify.com/en/login where they will login and then authorize access to their account.
- The URL calls the Spotify Authorization API that returns an `authorisation_code` to the callback URL registered by the developer.
- In the callback URL, the developer will then make a POST call to the Spotify Token API to retrieve the `access_token` and `refresh_token`.
- The developer will then use the `access_token` to call the Spotify Web API to fetch data.

This flow is a classic OAuth 2.0 authorization flow, so if you are familiar with [OAuth 2.0](https://datatracker.ietf.org/doc/html/rfc6749) it should be quite familiar.
We are going to implement this authorization flow in Rust using the [Actix-Web](https://actix.rs/) crate.

## Bootstraping a new Actix-Web project
