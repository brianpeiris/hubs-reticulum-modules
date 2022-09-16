# Hubs Reticulum Modules

> Note: This document was written in September 2022. It will likely be outdated in a matter of months. It reflects the state of the reticulum codebase at the time of writing.

This is a list of the primary modules in the [reticulum](https://github.com/mozilla/reticulum) code base. This document aims to explain the purpose of each module, why they exist, and how they relate to each other. Some modules include a list of notable fields when they require extra explanation. This document does not attempt to be a comprehensive, detailed description of the entire codebase. Many modules are omitted, and the descriptions here may omit specific details for the sake of brevity, but they should still be useful as a high-level overview of reticulum's design, and the reasoning behind the modules.

## Account

The Account module is the primary representation of a Hubs user. Accounts can be flagged as an admin account, and can be enabled or disabled. Accounts have a [Login](#login) and an [Identity](#identity), and several other resources associated with them.

### Notable fields

- `min_token_issued_at : utc_datetime` - This field enables a basic mechanism for expiring JWT authentication tokens for an account. If set, any existing tokens issued before this date will be considered invalid.

## Login

A Login provides a means of logging into an account. Currently, the only means of logging into a Hubs account is with an email, which is stored as a hashed identifier. We decided to decouple logins from accounts to allow for multiple logins, or for transferring an account from one login to another, though these extended use cases have not been implemented yet. For now, each account has exactly one login.

## Identity

An Identity is an optional name associated with an account. This module was introduced in response to a feature request for large events, where event organizers wanted to be able to strictly control what name was associated with an account. Regular users cannot set their own identity, this can only be done by an administrator through the Hubs admin panel.

## Hub

In reticulum terminology, a "Hub" refers to a single room. "hub" and "room" are used interchangeably in this doc. A Hub consists of fields that are required for basic use of a room, including the room's name, description, SID, entry mode, as well as various other fields that relate to specific room features. The hub.ex file also contains Canada authorization definitions for actions related to rooms and accounts.

### Notable fields

- `host : string` - The WebRTC server host for this hub. A host is assigned when a room is created. It may be reassigned if the existing host becomes unavailable. The host is cleared by a scheduled vaccuum task for rooms that are inactive and older than a certain period.
- `creator_assignment_token : string` - Rooms can be created by anonymous users. In order to allow a user to claim a room after they create an account, we generate a creator_assignment_token that is stored in a user's local storage. They can then use this token to associate a room with their account at a later time.
- `embed_token : string` - Rooms can be embedded in an iframe on any page on the Web. The embed_token is used to ensure that only the room creator is able to embed a room. The token is only shared with a creator of a room.
- `embedded : boolean` - Used for analytics. Set to true if a room has ever been embedded in an iframe.
- `member_permissions : integer` - A bitfield that determines which actions users can perform in a room. The term "member" is a bit of a misnomer here, these permissions apply to all visitors.
- `default_environment_gltf_bundle_url : string` - A URL to a GLTF/GLB file that will be used as the environment for the room. This URL is only used if a room does not have a Scene or Scene Listing associated with it.
- `spawned_object_types : integer` - A bitfield used to track metrics on types of objects that were spawned in a room.
- `entry_mode : EntryMode` - Determines if and how users are allowed to enter a room. In particular, if a room is in "invite" mode, users must have a room URL that has a valid invite code. See [HubInvite](#hubinvite).
- `user_data : map/json` - Arbitrary json data associated with a room when it is created. This was added to rooms to allow a simple layer of extensibility. For example, users managing events can create rooms with an array of arbitrary tags, which can be used to categorize rooms.
- `web_push_subscriptions : WebPushSubscription[]` - A room can have web push subscriptions associated with it. This allows users to listen to events on a room – primarily to get notified when other users enter a room.
- `hub_invites : HubInvite[]` - See [HubInvite](#hubinvite).
- `hub_bindings : HubBinding[] - See [HubBinding](#hubbinding).
- `hub_role_memberships : HubRoleMembership[]` - A room can have multiple "members". Members only have one implicit role at the moment – they are all considered "owners". Owners have the ability to modify a room's setting, and have moderation capabilities inside a room.
- `allow_promotion : boolean` - If set to true, this room could be shown on the front page as a "public" room.
- `room_size : integer` - The maximum number of users allowed in the room. This is not strictly enforced, since it is only used by client-side code, to prevent entry into a room.


