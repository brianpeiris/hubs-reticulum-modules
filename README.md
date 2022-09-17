# Hubs Reticulum Modules

> Note: This document was written in September 2022. It will likely be outdated in a matter of months. It reflects the state of the reticulum codebase at the time of writing.

This is a list of the primary modules in the [reticulum](https://github.com/mozilla/reticulum) codebase. This document aims to explain the purpose of each module, why they exist, and how they relate to each other. Some modules include a list of notable fields when they require extra explanation. This document does not attempt to be a comprehensive, detailed description of the entire codebase. Many modules are omitted, and the descriptions here may omit specific details for the sake of brevity, but they should still be useful as a high-level overview of reticulum's design, and the historical reasoning behind the modules.

## Account

The Account module is the primary representation of a Hubs user. Accounts can be flagged as an admin account, and can be enabled or disabled. Accounts have a [Login](#login) and an [Identity](#identity), and several other resources associated with them.

### Notable fields

- `min_token_issued_at : utc_datetime` - This field enables a basic mechanism for expiring JWT authentication tokens for an account. If set, any existing tokens issued before this date will be considered invalid.

## Login

A Login provides a means of logging into an account. Currently, the only means of logging into a Hubs account is with an email, which is stored as a hashed identifier. Logins are decoupled from accounts to allow for multiple logins, or for transferring an account from one login to another, though these extended use cases have not been implemented yet. For now, each account has exactly one login.

## Identity

An Identity is an optional name associated with an account. This module was introduced in response to a feature request for large events, where event organizers wanted to be able to strictly control what name was associated with an account. Regular users cannot set their own identity, this can only be done by an administrator through the Hubs admin panel.

## Hub

In reticulum terminology, a "Hub" refers to a single room. "hub" and "room" are used interchangeably in this doc. A Hub consists of fields that are required for basic use of a room, including the room's name, description, SID, entry mode, as well as various other fields that relate to specific room features. The hub.ex file also contains [Canada](https://hex.pm/packages/canada) authorization definitions for actions related to rooms and accounts.

### Notable fields

- `host : string` - The WebRTC server host for this hub. A host is assigned when a room is created. It may be reassigned if the existing host becomes unavailable. The host is cleared by a scheduled vacuum task for rooms that are inactive and older than a certain period.
- `creator_assignment_token : string` - Rooms can be created by anonymous users. In order to allow a user to claim a room after they create an account, a `creator_assignment_token` is generated and stored in a user's local storage. They can then use this token to associate a room with their account at a later time.
- `embed_token : string` - Rooms can be embedded in an iframe on any page on the Web. The `embed_token` is used to ensure that only the room creator is able to embed a room. The token is only shared with a creator of a room.
- `embedded : boolean` - Used for analytics. Set to true if a room has ever been embedded in an iframe.
- `member_permissions : integer` - A bitfield that determines which actions users can perform in a room. The term "member" is a bit of a misnomer here, these permissions apply to all visitors.
- `default_environment_gltf_bundle_url : string` - A URL to a GLTF/GLB file that will be used as the environment for the room. This URL is only used if a room does not have a Scene or Scene Listing associated with it.
- `spawned_object_types : integer` - A bitfield used to track metrics on types of objects that were spawned in a room.
- `entry_mode : EntryMode` - Determines if and how users are allowed to enter a room. In particular, if a room is in "invite" mode, users must have a room URL that has a valid invite code. See [HubInvite](#hubinvite).
- `user_data : map/json` - Arbitrary json data associated with a room when it is created. This was added to rooms to allow a simple layer of extensibility. For example, users managing events can create rooms with an array of arbitrary tags, which can be used to categorize rooms.
- `web_push_subscriptions : WebPushSubscription[]` - A room can have web push subscriptions associated with it. This allows users to listen to events on a room – primarily to get notified when other users enter a room.
- `hub_invites : HubInvite[]` - See [HubInvite](#hubinvite).
- `hub_bindings : HubBinding[]` - See [HubBinding](#hubbinding).
- `hub_role_memberships : HubRoleMembership[]` - A room can have multiple "members". Members only have one implicit role at the moment – they are all considered "owners". Owners have the ability to modify a room's setting, and have moderation capabilities inside a room.
- `allow_promotion : boolean` - If set to true, this room could be shown on the front page as a "public" room.
- `room_size : integer` - The maximum number of users allowed in the room. This is not strictly enforced, since it is only used by client-side code, to prevent entry into a room.

## HubChannel

The HubChannel module is a Phoenix Channel that handles and broadcasts real-time messages for avatar movement and other actions and activity in a Hubs room. Some of the messages sent through the hub channel are subject to authorization rules based on user permissions. It is also used to gate entry into a room, based on permissions and room state. The module also handles presence tracking, which is used to track the existence and metadata about users in a room, and broadcast that information to all other users in the room.

The references to "naf" in this channel are related to the [networked-aframe](https://github.com/mozillareality/networked-aframe) library used by the Hubs client.

## HubRoleMembership

The HubRoleMembership module is used to establish a one-to-many relationship between Hubs rooms and Accounts. If a user has a membership in a room, they are given moderation capabilities in that room. There is only one implicit role defined as of this writing.

## AccountFavorite

The AccountFavorite module is used to store rooms that are favorited by a user.

## Sids

SIDs are "string identifiers". SIDs are user-facing identifiers used for multiple resources including Hubs, Scenes, Avatars, and Projects, instead of their actual database ids. SIDs are meant to be relatively short and readable, and decouple public identifiers from the database identifier implementation.

## RoomObject

RoomObjects represent media objects that are "pinned" in rooms. An object might be an image, video, audio, 3d model, or anything described by the contents of the `gltf_node` field. Only logged-in users can pin objects to a room, which is why they are associated with an account.

### Notable fields

- `gltf_node : string (encrypted)` - An encrypted string containing JSON that describes an object. Usually this just contains the object's transform (position, rotation, scale), and the URL to the media that it represents.

## HubInvite

A HubInvite provides an access control mechanism for rooms. If a room has a HubInvite associated with it, users must have that HubInvite SID in the room URL in order to view and enter that room. Rooms can only have one active HubInvite at any given time. Only room owners and room moderators can issue and revoke HubInvites.

## HubBinding

The HubBinding mechanism is used to bind a room's access and authorization to an external community service, such as Discord or Slack. This mechanism is used by the Hubs Discord Bot to bind Hubs rooms to a Discord channel. When a room is bound to a community channel, access to that Hub room is gated with an account associated with that community, via OAuth, and the permissions in a room are determined by a user's level of access in that community channel. For example, moderators of a Discord server would also be moderators of Hubs rooms bound to channels in that Discord server. Users' Display names and  identifier information are also pulled from the external community API, in order to display them on avatars in the Hubs room.

## OAuthProvider

The OAuthProvider module is used to store OAuth information for a Hubs account, to enable the HubBinding mechanism.

## Project

Projects represent scenes that are created in the [Spoke](https://github.com/mozilla/spoke/) scene creator tool. Projects have a `project_owned_file` JSON file that represents the contents of the scene. Scenes are exported and published from a Project. Projects can also have Assets, which are individual media files that can be included in a Project's scene.

## Asset

Assets are individual media files that are uploaded and listed in the Spoke scene creator tool. Assets can be images, audio, video or 3D models. Assets have thumbnail images associated with them, for display in the UI. Assets are related to projects though the ProjectAsset relationship.

## ProjectAsset

The ProjectAsset module is used to maintain the many-to-many relationship between Projects and Assets.

## Scene

Scenes are 3D environments generated by the Spoke scene creator tool, or by the [Blender Exporter Plugin](https://github.com/MozillaReality/hubs-blender-exporter/). Scenes consist of various meta data fields that are provided by scene creators, as well as the content of the scene itself, in the form of a GLTF file, and a Scene file, which is a JSON description of a scene. Spoke Scenes belong to a Spoke Project. Scenes can also be imported from an existing Hubs server instance. 

Scenes can be "remixed", using an existing scene as a base for modification. This is represented by the `parent_scene` or `parent_scene_listing` fields.

### Notable fields

- `allow_remixing : boolean` - Users can opt-in to allowing other users the ability to use their scene as a base for modification.
- `allow_promotion : boolean` - When users create Scenes, they can opt-in to allowing promotion, which gives Hubs the consent to publicize and feature the scene in Hubs' public listings.
- `attribution : string` - The primary attribution for the creator of the scene.
- `attributions : map/json` - Metadata that provides additional attributions for art that the Scene might contain.

## SceneListing

SceneListings are copies of published scenes, created from Scenes that are marked as promotable. They are used to populate the featured and searchable scenes in Hubs' scene browser and media browser. SceneListings are created by an approval action in the Hubs Admin Panel. 

### Notable fields

- `tags : map/json` - A simple structure containing a list of string tags. Typically this used to tag a scene as "featured" for display in Hubs' media browser.
- `order : integer` - The order of scenes can be manually specified, so that they appear in a particular order determined by a Hubs administrator.

## Avatar

Avatars are 3D models used to represent a user in a Hubs room. Users can create avatars, or choose them from public AvatarListings. Avatars can have a GLTF model file, as well as separate texture image files, which are applied on top of the GLTF model. Avatars can also be remixed by other users, and imported from other Hubs servers.

## AvatarListing

AvatarListings are copies of Avatars, created from Avatars that are marked as promotable. They are used to populate the featured and searchable avatars in Hubs' avatar browser and media browser. AvatarListings are created by an approval action in the Hubs Admin Panel.

## Storage

The Storage module is a set of utility functions related to storing uploaded files on disk. The module also takes care of encrypting and decrypting these files via the Crypto module. Each file also has a metadata file alongside it. A file is either an expiring file, which is deleted after a specified time period and does not have an associated record in the database; an OwnedFile, which is permanent  associated with a user and represented in the database; or a CachedFile, which is temporary represented in the database, but not associated with a user.

When users access expiring files, they must provide the decryption key in the URL. OwnedFiles have their decryption key stored in the database.

## OwnedFile

An OwnedFile represents a file on disk that belongs to a particular user. OwnedFiles are usually promoted from expiring files via the Storage module. Inactive OwnedFiles are demoted to expiring files, which are eventually vacuumed by a scheduled job.

## CachedFile

A CachedFile represents a file on disk that is used to cache media pulled from the web. Typically these are [Sketchfab](https://sketchfab.com/) models, or screenshots of websites captured by Hubs' screenshot service, [photomnemonic](https://github.com/MozillaReality/photomnemonic). CachedFiles are created by the MediaResolver module. CachedFiles have a `cache_key` which is used to lookup the file, and are vacuumed by a scheduled job when they have fallen out of use.

## StorageUsed

The StorageUsed module makes system calls to determine the amount of file storage used by a Hubs server. It is called periodically via a configured [CacheX](https://hex.pm/packages/cachex) cache-warmer worker job.

## Application

The Application module notably contains caching configurations for various data that is produced by other modules.

## Router

As with all [Phoenix](https://www.phoenixframework.org/) applications, the Router module defines the user-facing and API entry points into the application. It also defines various pipelines to guard access to endpoints, inject various headers, and add additional behaviors to endpoints. Some endpoints use Plugs that are configured directly in their respective controller modules.

## PageController

The PageController module is notable in that it acts as a fallback to endpoints that are not explicitly defined in the Route module. It handles the rendering of all of Hubs' HTML pages, and injects relevant headers, meta data, configuration data, and extra content into those pages. The PageController also includes a basic CORS proxy for proxying content from the web, when importing it into a Hubs room, and a proxy to an image resizing service. The module also handles static assets, and configurable assets.

## PageOriginWarmer

The PageOriginWarmer module is used to retrieve and cache a list of well-known static assets, mostly HTML pages, for the Hubs client. The pages are also split into two chunks, at the point where meta tags should be inserted.

## MediaResolver

The MediaResolver module is used to retrieve metadata about media from across the web. MediaResolver will attempt to resolve media for a set of well-known media sources, or fallback to [Open Graph](https://ogp.me/) image meta data, or finally a screenshot of the given URL. In most cases, MediaResolver does not return the media itself, just metadata about media that is available at the URL.

## MediaSearch

The MediaSearch module is used to search and list media from Hubs' own content (avatars, scenes, rooms, assets), as well as media from popular media sites, if their API integrations are configured on the Hubs server.

## AppConfig

The AppConfig module is used to store runtime configurable values that are configured by a Hubs administrator. Configs are defined in the [schema.toml](https://github.com/mozilla/hubs/blob/master/src/schema.toml) in the Hubs client repo. Most config values are primitive types, but configs can also be files or json structures. App configs only contain values that are suitable for public use.

## Meta

The Meta module is used to collect various meta information about a Hubs server. It is used by the PageController module to inject that meta data into HTML pages, and by the MetaController module to make the meta data available via an API. This meta information is used by the client code to connect to phoenix channels, to determine if various third-party integrations have been configured, and by the admin panel to determine if a Hubs server has been setup with content, or is over its storage quota.

## Email

The Email module is used to produce emails with magic links, used to sign into Hubs accounts. The emails are sent via the [Bamboo](https://hex.pm/packages/bamboo) email library.

## LoginToken

The LoginToken module is used to generate and persist short-lived tokens for magic link emails.

## AuthChannel

The AuthChannel module handles authentication requests used in the magic link login mechanism.

## LinkChannel

The LinkChannel module is a short-lived channel that is used to transfer Hubs account information from one device to another while redirecting a user to a Hubs room using a short link code. This is primarily used to make it easier to join a Hubs room from a standalone VR headset. The payload sent through the channel is produced and consumed by the Hubs client code.

## Locking

The Locking module is used to capture and release global locks across reticulum servers, using the common PostgreSQL database, and [advisory locks](https://www.postgresql.org/docs/9.1/functions-admin.html#FUNCTIONS-ADVISORY-LOCKS). Locks are used for global operations such as database migrations, and vacuum jobs.

## Crypto

The Crypto module contains various cryptographic functions for encrypting, decrypting and hashing data.

## SessionStat

The SessionStat module is used to store analytics about Hubs room usage. It records anonymous information about when users entered rooms, and meta data about them.

## NodeStat

The NodeStat module is used to store analytics about a server's usage. In particular it periodically records how many users were simultaneously present in rooms at any given time, via the StatsJob module.

## RoomAssigner

The RoomAssigner module is used to assign a WebRTC server to a Hubs room as needed. The module picks a server based on its current load, using a weighted sample.

## JanusLoadStatus

The JanusLoadStatus module is use to query and cache the current load and meta information about WebRTC servers that are available to a Hubs server. "Janus" is a misnomer, it refers to the previous WebRTC server that Hubs used, but it now uses [dialog](https://github.com/mozilla/dialog).

## Speelycaptor

The Speelycaptor module is used to upload, convert and retrieve video files via Hubs' [speelycaptor](https://github.com/mozilla/speelycaptor) video conversion service.

## Coturn

Used to generate TURN info for the [coturn](https://github.com/coturn/coturn) server, when TURN is enabled. TURN info is sent to the client via `generate_turn_info` in the Hub module.

## PeerageProvider

The PeerageProvider module is used to enable Erlang node discovery via the [Peerage](https://hex.pm/packages/peerage) library. PeerageProvider uses the Habitat module to query the [habitat](https://docs.chef.io/habitat) census, used in Hubs Cloud.

## WebPushSubscription

The WebPushSubscription is used to register [Web Push](https://developer.mozilla.org/en-US/docs/Web/API/Push_API) subscriptions. This was used to subscribe to Hubs room events, such as when a user has entered a room you've created, but this feature is currently missing from the Hubs client UI.

## SupportSubscription

The SupportSubscription was used in a feature that allowed users to request support from a Hubs team member, but this capability is no longer available.

## GraphQL Modules

There are a number of modules not documented here, that are related to Hubs' unreleased GraphQL API. The GraphQL API is meant to be used by the public, to be able to automate common actions related to Hubs rooms. Most of these modules live under the RetWeb.Schema, Ret.Api and RetWeb.Api.V2 namespaces.
