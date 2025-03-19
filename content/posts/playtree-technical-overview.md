+++
date = '2025-03-18T12:55:36-07:00'
draft = true
title = 'Playtree Technical Overview'
+++

Playtree lets users create and manage personalized music playlists that branch and loop playback. In this post, I'll explore the technical challenges and solutions I encountered while building Playtree.

## Overview

Playtree is designed to provide a novel way to interact with music. Instead of traditional linear playlists, Playtree uses a graph structure where users create *playnodes* representing musical elements, and *playedges* to define the flow of playback. Multiple playheads can be maintained simultaneously within the same playtree.

Key aspects of the project include:

- **Backend Development**: A RESTful API built with Go handles data persistence.
- **Frontend Development**: A dynamic user interface developed with React allows users to create, edit and play their playtrees.
- **Data Structures and Algorithms**: The application uses a graph data structure to represent playlists and algorithms to manage music playback.

## Technical Details

Here, I'll give a brief outline of Playtree's features, and the tools used to implement Playtree.

### Backend

- I chose Go for the backend due to its efficiency and scalability. The net/http package is used to create RESTful API endpoints.
- Playtree data, including playlist structures, are stored in JSON files. This provides a simple way to persist and manage data.
- A Go [shortuuid](https://github.com/lithammer/shortuuid) library is used to generate unique IDs for each playtree.
- A core feature of Playtree is its integration with the Spotify API. The Go server authenticates certain user requests via Spotify access tokens, which are used as proxies for the user's identity.

### Frontend
- The frontend is built with [React](https://react.dev/), a powerful library for creating interactive user interfaces. React Flow, a library for rendering and interacting with graphs, is used to build the playtree visualization and editing tools, with Dagre for auto-layouting.
- I used [Remix](https://remix.run/) for routing, enabling smooth navigation within the application.
- Reducers are used for state management, helping to cleanly and efficiently update the UI in response to user interactions.
- Custom hooks, like `useClickOutside`, are implemented to handle specific UI behaviors.

### Data Model

- The central component is the Playtree structure, which includes:
    - **Summary**: Contains metadata like ID, name, creator, and access settings.
    - **Playnodes**: Represent individual nodes in the playtree, containing information about musical elements.
        - **Limit**: sets a limit on how many times a playnode can be played.
        - **Repeat**: repeats the same playnode before selecting a playedge to walk.
    - **Playedges**: Define the connections between nodes, establishing the flow of music.
        - **Limit**: sets a limit on how many times a playedge can be walked.
        - **Shares**: weights the likelihood a playedge is selected.
        - **Priority**: forces the highest priority playedges to be selected first.
    - **Playroots**: Define starting points for playback.
    - **Playscopes**: Contextual scopes which restrict play counters to local contexts.
- To ensure data integrity, I've implemented server-side syntax validation using a Go [validator](https://github.com/go-playground/validator/v10) library and client-side semantic validation.

### Key Features
- **Playtree Creation and Editing**: Users can intuitively create, modify, and manage their playtrees, defining the structure and rules for playback.
- **Spotify Integration**: Deep integration with the Spotify API enables music search, user authentication, and playback control.
- **Personalized Playback**: Playtree uses the defined structure and rules to generate a unique and personalized listening experience.

## Challenges and Solutions

### UI for the Playtree editor
I wanted to create an intuitive, fun UI for making and visualizing complex playtrees. Playtrees are graph structures, so I used [React Flow](https://reactflow.dev/), a library for rendering and interacting with graphs. The editor displays highly customized nodes and edges using React Flow. At the same time, I wanted to keep the playtree data structure, as maintained on the server, as the editor's source of truth. Keeping the React Flow UI synced with the playtree data required a nuanced understanding of React's resolution algorithm: if React cannot determine that a node remains the same from one render to another, for example, it will destroy the underlying HTML and recreate it for the next render. This isn't only a performance issue; in some cases, it will lead to critical UI bugs like losing focus on a text input while typing. I carefully syncronized the central playtree state with the React Flow UI state to ensure a clean data flow while avoiding any UI bugs.

Although many of the rules governing valid playtrees are implicitly enforced by the editor UI, there are some features which must be checked before a playtree is saved. I perform client-side validation before a playtree is saved, which issues warnings to a user when their playtree doesn't have a playroot attached (meaning playback won't start anywhere if they try to play). This playtree is considered valid—it doesn't violate any playtree rules—but the editor still issues a warning on save, as the user most likely didn't intend to make a playtree that doesn't start playback anywhere. The editor also checks if a user's playscopes are valid. Playscopes can't partially overlap other playscopes, or be redundant. A playtree with invalid playscopes can't unambiguously apply scope rules, and so it is invalid as a playtree. When the user tries to save a invalid playtree, an error is reported in the editor and saving is prevented.

### Managing playback logic and state
The rules that govern playback on a playtree are rather complicated. The user can play and pause audio, skip forward a song, go back to previously played songs, which must be memoized when playback is randomly selected. Additionally, a user can switch between playheads.

I use a [reducer](https://react.dev/reference/react/useReducer) to handle playback logic. The reducers handle all necessary logic in order to transition from one song to the next. When a song ends or if the user skips forward, a new song is selected either by moving through a playnode or by randomly selecting the next playnode according to the playedge rules. A history stack is maintained on each playhead, which holds a reference to previously visited playnodes and playedges, and a copy of old play counters, if they had changed when visiting that node or edge. Playscopes are preprocessed and placed within maps indexed by playnode so that they can reset play counters when needed.

If a playnode has reached its play limit, or if it has no songs, playback will iteratively skip forward and try to find a song to play. As far as the user is concerned, this happens instantaneously: there should be almost no perceivable time between songs, no matter how many playnodes are skipped over in between. This introduces the possibility of what I've been calling a "zoom condition". In a program where computation is meant to be punctuated by breaks (in this case, waiting for a song to finish), a zoom condition, as I'm using it, is a state which causes the program to do too much intermediary computation without reaching a break point (i.e., finding a new song to play). In Playtree, a zoom condition is easily introduced by a playnode which has no songs and loops infinitely. If unchecked, a zoom condition can cause problems like crashing, freezing, or stalling. In order to prevent zoom conditions, I incorporate a loop counter. If a song is not found within 10,000 iterations of node traversal, playback resets to the beginning of a playnode.

### Integrating with the Spotify API for authentication and seamless music playback.
Playtree uses the [Spotify API](https://developer.spotify.com/documentation/web-api) to search for tracks on Spotify, and to play music within Playtree. I also use the [Spotify Web Playback SDK](https://developer.spotify.com/documentation/web-playback-sdk), which allows for Spotify playback to occur within the app itself, and for certain events pertaining to Spotify playback to be reported real-time to the application.

Spotify API calls must be authenticated, and so I use the Authorization Code flow to obtain an access token from the server side. I need to use the access token on both the client side (because the Web Playback SDK can only be used client-side) and server side (because my server uses Spotify tokens as a proxy for the user's identity within the site). Access tokens are stored as a cookie.

The Playtree client can control Spotify playback, but playback can be controlled in certain ways on other Spotify clients as well. Playback can be transferred from the web player device to another device, a user can change the song that's playing on Spotify manually on the Spotify client, etc. The web player emits events when some of these changes occur, and I can compare the internal Playtree player state to the Spotify state as reported by the web player. With that, I can report when playback has gotten out of sync with Spotify, and offer a user the option to resync.

A subtle challenge has to do with Playtree's preferred playback flow versus Spotify's preferred flow. Spotify prefers for a user to set up a context—a playlist or an album, for example—and to automatically queue up upcoming songs to play next. Spotify even has an autoplay feature which will play related songs once a context is completed. However, Playtree instead prefers for songs to be loaded individually, only selecting the next song to play once the current song is over. A user can see on their Spotify queue what songs are coming up next; because randomness is so central to Playtree, I wanted the song that would be selected to be a surprise to the user when it first started playing. Also, if work on this project continues, a nice feature would be to allow the user to manually override random selection of the next song; thus, the next song should be open throughout the course of the currently playing song. Still, playback should move more or less seamlessly from one song to the next.

To integrate this just-in-time approach with Spotify, I had to be clever. I didn't want to poll the Spotify `/me/player` endpoint, because Spotify API calls are rate-limited. Also, I didn't want to rely on a timeout function, changing the song once the duration of the current song had elapsed, as it could easily get out of sync with the actual state of the Spotify player. So, that left the events emitted by the Spotify web player. Unlike HTML `<audio>` elements, the Spotify web player doesn't come with a dedicated `ended` event; rather, a song's completion must be inferred from other properties of playback. Certain features reliably correlated with a track's completion: the same song would be playing, except its playback would be reset to 0 and playback would be paused. I also store a timestamp on the Playtree playhead of the last time synchronized playback on a song was started. If the time elapsed since starting playback matches the time remaining on the current song, and a playback event with the right features is received, it is inferred that the song is over and that a new song should be selected.

## Conclusion
The Playtree application touched on many key aspects of web development, and offered some specific challenges as well. Now that it's up and running, I'm pleasantly surprised that such a speculatively designed application could be so fun to use. It even seems like it'll have its use cases. I look forward to expanding upon and refining Playtree moving forward.
