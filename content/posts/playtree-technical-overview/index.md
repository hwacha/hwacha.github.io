+++
date = '2025-03-18T12:55:36-07:00'
draft = true
title = 'Playtree Technical Overview'
tags = ['playtree', 'overview']
menus = 'main'

[params]
    rank = 0
+++

Playtree is an application for making and listening to *playtrees*, which are like playlists, except that playback can branch, loop, and start at multiple entry points. You can try out playtree [here](https://playtree.gdn). In this post, I'll provide a broad overview of Playtree and its development.

Check out the [playtree explainers](/posts/playtree-structure-explained/) for a more in-depth explanation of playtrees.

## At a Glance

![alt](/posts/playtree-technical-overview/example-playtree.png)

Playtree is a web application for creating custom playtrees, which are directed graphs. Playnodes contain songs to be played, and playedges define a path playback can follow.

Key aspects of the project include:

- **Backend Development**: A RESTful API built with Go stores playtrees, authenticates requests via Spotify, and caches user data, such as their currently selected playtree, for future sessions.
- **Frontend Development**: A dynamic user interface developed with React and React Flow allows users to create, edit, share, and play their playtrees. Remix is used for routing and data handling. The Spotify Web Playback SDK is used for an in-app music player.
- **Deployment**: The app is deployed to a Digital Ocean droplet via SSH. An Nginx reverse proxy routes requests made to the Remix and Go servers. Let's Encrypt was used to certify the nginx server to ensure a secure https connection.

## Technical Details

Here, I'll give a brief outline of Playtree and the tools used to implement Playtree.

### Backend

- I chose [Go](https://go.dev/) for the backend due to its efficiency and scalability. The [`net/http`](https://pkg.go.dev/net/http) package is used to create RESTful API endpoints.
- Playtree data, including playlist structures, are stored as JSON files. This provides a simple way to persist and manage data. The Go server parses and validates JSON files before they are created or updated.
- The [shortuuid](https://github.com/lithammer/shortuuid) library is used to generate unique IDs for each playtree.
- The Playtree server authenticates certain user requests via Spotify access tokens, which are used as a proxy for the user's identity in place of a dedicated Playtree account.

### Frontend
- The frontend is built with [React](https://react.dev/).
- [React Flow](https://reactflow.dev/), a library for rendering and interacting with graphs, is used to build the playtree visualization and editing tools. The [Dagre](https://github.com/dagrejs/dagre#readme) library is used for graph auto-layouting.
- [Remix](https://remix.run/) is used for routing, enabling smooth navigation within the application which importantly does not interrupt music playback. Remix also provides a canonical way to manage data flow from server to client.
- React's built-in [reducers](https://react.dev/reference/react/useReducer) are used for state management, helping to cleanly and efficiently update complex UI in response to user interactions.
- Custom hooks, like `useClickOutside`, are implemented to handle specific UI behaviors.

### Data Model
The central data structure is the Playtree. It is created and updated by the playtree editor, stored as JSON on the server, and used by the player to dictate playback. It includes the following:
- **Summary**: Contains metadata like ID, name, creator, and access settings.
- **Playnodes**: Individual nodes in the playtree. When playback enters a playnode, its songs will be played.
- **Playedges**: Possible paths playback can follow from one playnode to another. After playback finishes at a playnode, one of its outgoing playedges is randomly selected.
- **Playroots**: Starting points for playback. If there are multiple playroots, then a user can switch between multiple playheads while listening.
- **Playscopes**: Subregions of the playtree graph that define a local scope for play counters, which are maintained during playback to limit playnodes, playedges, and songs. If playback exits a playscope, all counters within the scope are reset to 0.

To ensure data integrity, I've implemented server-side syntax validation using a Go [validator](https://pkg.go.dev/github.com/go-playground/validator/v10) library as well as client-side semantic validation.

### Key Features
- **Playtree Creation and Editing**: Users can intuitively create, modify, share and manage their playtrees, defining complex playback structures.
- **Spotify Integration**: Integration with the Spotify API enables music search, user authentication, and playback control.
- **Advanced Playback**: The playtree player manages music playback in ways extended beyond normal playlists, offering the user fun, novel, and useful listening experiences.

## Challenges and Solutions

### UI for the Playtree editor
I used React Flow to implement an interactive graph editor. Synchronizing the UI state with the playtree data took a careful approach and a deep understanding of React's reconciliation algorithm.

See [this post](/posts/playtree-editor-ui) for more.

### Managing playback logic and state
Managing a playtree's playback is more involved than it is for an ordinary playlist. Multiple playheads are maintained, playback can loop and randomly branch, and plays on a playnode or playedge can be limited. As a result, implementing the player logic was a complicated task.

See [this post](/posts/playtree-player-logic) for more.

### Integrating the Spotify API
Playtree uses the Spotify API to play music. This introduces the need for an authentication scheme. Also, because playback occurs on a Spotify device, it can be altered by other Spotify clients during Playtree playback. It was important to efficiently and comprehensively synchronize the Playtree playback state with the Spotify playback state in real time.

See [this post](/posts/playtree-spotify-integration) for more.

## Conclusion
The Playtree application touched on many key aspects of web development, and offered some specific challenges as well. Now that it's up and running, I'm pleasantly surprised that such a speculatively designed application could be so fun to use. It even seems like it'll have its use cases. I look forward to expanding upon and refining Playtree moving forward.
