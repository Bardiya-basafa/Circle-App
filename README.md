# Circle: Social Network Web Application

**Circle** is a full-featured social networking platform built with **ASP.NET Core MVC (.NET 9)**, **Entity Framework Core**, **ASP.NET Core Identity** and **SignalR**. Users can post statuses with images and hashtags, like, comment on and bookmark posts, share 24-hour stories, send and accept friend requests, search for people and content, and get **real-time notifications**. Administrators get a **content moderation dashboard** to review reported posts.

---

## Table of Contents

- [Features](#features)
- [Tech Stack](#tech-stack)
- [Solution Architecture](#solution-architecture)
- [Domain Model](#domain-model)
- [Database Design](#database-design)
- [Service Layer](#service-layer)
- [Real-Time Notifications (SignalR)](#real-time-notifications-signalr)
- [Authentication and Authorization](#authentication-and-authorization)
- [Web Layer](#web-layer)
- [AJAX Interactions](#ajax-interactions)
- [Routes Reference](#routes-reference)
- [Getting Started](#getting-started)
- [Known Issues and Roadmap](#known-issues-and-roadmap)

---

## Features

### Posts and feed
- Create text posts with an optional **image upload**, stored under `wwwroot/postimages/` with GUID file names.
- **Hashtags** are extracted automatically with a regex, normalised to lower case, de-duplicated, stored and counted.
- **Like / unlike** toggle, protected by a transaction and a retry policy for concurrency conflicts.
- **Comments**: add and delete.
- **Bookmarks** (favourites): toggle, plus a dedicated favourites page.
- **Public / private** visibility toggle. Private posts are visible only to their author.
- **Soft delete**: posts are flagged `IsDeleted`, and hashtag counters are decremented.
- **Reporting**: one report per user per post. Posts with 5 or more reports are hidden from the feed automatically.
- Human-friendly timestamps such as "just now", "5 minutes ago" or "2 days ago".

### Stories
- Image stories with public or private visibility.
- The stories page shows only stories from the **last 24 hours**.

### Social graph
- Send, cancel, accept and reject **friend requests**.
- **Friend list** and unfriend.
- **Friend suggestions** (5 users who are not already friends and have no pending request), with each one's friend count.
- Sent requests are grouped by status: *Accepted → Rejected → Cancelled → Pending*.
- Public user profile pages with that user's posts and friends.

### Notifications
- Persistent notifications for **likes, comments, bookmarks, friend requests, accepted requests** and **removal of a post by moderation**.
- The **unread badge updates live** through SignalR.
- Dropdown list that shows unread notifications first, then newest first, with mark-as-read and click-through to the related page.

### Search
- One query searches **posts** (content), **users** (full name) and **your friends** (full name).

### Account
- E-mail/password registration and login.
- **Social login with Google and GitHub** (OAuth).
- Settings: update full name and bio, change password, upload a profile picture.

### Moderation (Admin)
- Admin dashboard listing posts with **more than 3 reports**.
- View every report on a post.
- **Dismiss** reports (resets the counter) or **remove** the post (soft delete and notify the author).

---

## Tech Stack

| Concern | Technology |
|---|---|
| Runtime | .NET 9 |
| Web | ASP.NET Core MVC, Razor views, partial views, **View Components** |
| Data access | Entity Framework Core 9 with SQL Server, code-first migrations |
| Identity | ASP.NET Core Identity with `int` keys (`IdentityUser<int>`) |
| External auth | Google OAuth 2.0, GitHub OAuth (`AspNet.Security.OAuth.GitHub`) |
| Real-time | ASP.NET Core **SignalR** |
| Front-end | Tailwind CSS, UIkit 3, Ionicons, Lucide, SimpleBar, vanilla JS `fetch` |

---

## Solution Architecture

```
CircleApp.sln
├── Domain/            Entities, DTOs (service responses), ViewModels (form inputs)
├── Infrastructure/    AppDbContext, Fluent API config, migrations, seeding, constants, helpers
├── Services/          Business logic: interfaces + implementations, SignalR hub
└── CircleApp.UI/      MVC controllers, view components, Razor views, static assets, Program.cs
```

```
 ┌─────────────────────────── CircleApp.UI ────────────────────────────┐
 │ Controllers ─ ViewComponents ─ Razor Views ─ wwwroot (JS/CSS)        │
 └──────────────┬──────────────────────────────────────────────────────┘
                │ IPostService, IFriendsService, INotificationService ...
 ┌──────────────▼──────────── Services ────────────────────────────────┐
 │ PostService · FriendsService · NotificationService · SearchService   │
 │ UserService · FavoritesService · AdminService · NotificationsHub     │
 └──────────────┬──────────────────────────────────────────────────────┘
                │ AppDbContext
 ┌──────────────▼────────── Infrastructure ────────────────────────────┐
 │ AppDbContext · Migrations · DbInitializer · HashtagHelper · Constants│
 └──────────────┬──────────────────────────────────────────────────────┘
                │
 ┌──────────────▼───────────── Domain ─────────────────────────────────┐
 │ Entities · DTOs · ViewModels                                         │
 └──────────────────────────────────────────────────────────────────────┘
```

- Every service is registered as **scoped** and injected through its interface.
- `BaseController` provides `GetUserId()` (from the `NameIdentifier` claim) and `GetUserFullName()` (from a custom `FullName` claim) to all authenticated controllers.
- **View Components** render sidebar widgets on their own: friend suggestions, trending hashtags and stories.

---

## Domain Model

```
                         ┌──────────────────────┐
                         │ User : IdentityUser<int>
                         │ FullName, Bio,        │
                         │ ProfilePictureUrl,    │
                         │ IsDeleted             │
                         └─┬───┬───┬───┬───┬───┬─┘
             1..*  ┌───────┘   │   │   │   │   └──────────┐ 1..*
             ┌─────▼────┐ ┌────▼─┐ │ ┌─▼────────┐  ┌──────▼─────┐
             │   Post   │ │Story │ │ │ Comment  │  │  Bookmark  │
             └┬──┬──┬──┬┘ └──┬───┘ │ └──────────┘  └────────────┘
              │  │  │  │     │     │
       *..*   │  │  │  └──── Like (PostId? / StoryId?) ◄──┘
   ┌──────────▼┐ │  └─────── Report
   │  Hashtag  │ └────────── Comment, Bookmark
   └───────────┘

   FriendRequest (SenderId, ReceiverId, Status)      FriendShip (SenderId, ReceiverId)
   Notification  (UserId, Type, Content, IsRead)
```

| Entity | Purpose | Key fields |
|---|---|---|
| `User` | Identity user with profile | `FullName`, `Bio`, `ProfilePictureUrl`, `IsDeleted` |
| `Post` | A status update | `Content`, `ImageUrl`, `IsPrivate`, `NrOfReports`, `IsDeleted`, timestamps |
| `Story` | Short-lived image | `ImageUrl`, `IsPrivate`, `IsDeleted`, `DateCreated` |
| `Like` | A like on a post **or** a story | `UserId`, `PostId?`, `StoryId?` |
| `Comment` | A comment on a post | `Content`, timestamps |
| `Bookmark` | A saved post | `UserId`, `PostId`, `DateCreated` |
| `Report` | A user report on a post | `UserId`, `PostId`, `DateCreated` |
| `Hashtag` | A tag and how often it is used | `Name`, `Count`, N:M with Post |
| `FriendRequest` | A pending or closed request | `SenderId`, `ReceiverId`, `Status` (`Pending` / `Accepted` / `Rejected` / `Cancelled`) |
| `FriendShip` | An accepted friendship | `SenderId`, `ReceiverId`, `DateAccepted` |
| `Notification` | An in-app notification | `UserId`, `Type`, `Content`, `IsRead` |

The `Domain` project also holds:
- **DTOs** that services return, for example `FriendResponse`, `SentRequestResponse`, `ReceivedRequestsResponse`, `SuggestedUserResponse`, `SearchResponse` and `NotificationResponse`.
- **ViewModels** bound from forms, for example `LoginVm`, `RegisterVm`, `PostVM` (content plus `IFormFile Image`), `LikePostVm`, `CommentPostVm`, `UpdatePasswordVm`, `UpdateProfileVm` and `CreateStoryVm`.

---

## Database Design

`AppDbContext : IdentityDbContext<User, IdentityRole<int>, int>`

### Composite keys

| Table | Primary key | Effect |
|---|---|---|
| `Likes` | `(PostId, UserId)` | A user can like a post only once |
| `Bookmarks` | `(PostId, UserId)` | A user can bookmark a post only once |
| `Reports` | `(PostId, UserId)` | A user can report a post only once |

### Relationships and delete behaviour

- User → Posts and User → Stories: one-to-many.
- Like, Comment, Bookmark and Report → Post use `NoAction`. → User uses `Restrict`. This avoids SQL Server's *multiple cascade paths* error.
- Post ↔ Hashtag: many-to-many through an explicit `PostHashtags` join table with `NoAction` on both sides.
- FriendRequest and FriendShip each have **two** foreign keys to `User` (`Sender` and `Receiver`), both with `NoAction`.
- Identity tables are renamed to `Users`, `Roles`, `UserRoles`, `UserClaims`, `UserLogins`, `RoleClaims` and `UserTokens`.

### Startup migration and seeding

On startup, `Program.cs`:
1. Applies pending migrations (`Database.MigrateAsync()`).
2. `DbInitializer.SeedAsync`: adds a sample user and two sample posts if the database is empty.
3. `DbInitializer.SeedUserAsync`: creates the `Admin` and `User` roles and a demo **user** account and **admin** account.

---

## Service Layer

### PostService

| Method | Behaviour |
|---|---|
| `GetAllPosts(userId)` | Feed: posts that are public **or** the user's own, with fewer than 5 reports and not deleted. Eager-loads author, likes, bookmarks, comments (with authors) and reports. Newest first. |
| `CreatePost(vm, userId)` | Saves the post. If an image is attached and its MIME type is `image/*`, stores it as `wwwroot/postimages/{guid}{ext}`. Then **extracts hashtags** and creates new tags or increments existing ones. |
| `LikePost(userId, postId)` | **Toggle** inside a **database transaction**. On `DbUpdateConcurrencyException`, rolls back, detaches tracked entities and **retries up to 5 times**. Returns `NotificationResponse { Success, SendNotification }`, so a notification is sent only when a like is *added*. |
| `BookmarkPost` | Toggle. Returns `true` when a bookmark was added. |
| `TogglePostVisibility` | Flips `IsPrivate` (owner only). |
| `ReportPost` | Adds a `Report` (unique per user) and increments `NrOfReports`. |
| `DeletePost` | Soft delete (owner only), decrements the counters of its hashtags and unlinks them. |
| `GetReportedPosts` | Posts with `NrOfReports > 3` for the admin dashboard. |
| `DismissPostReports` | Removes all reports on a post and resets the counter. |
| `RemoveReportedPost` | Soft-deletes the post and **notifies the author**. |

### Hashtag extraction (`HashtagHelper`)

```csharp
Regex.Matches(inputText, @"(?<!\w)#\w{2,}(?!\w)")
     .Select(m => m.Value.ToLower())
     .Distinct()
```

A hashtag must start at a word boundary, have **at least two word characters** after the `#`, and not be followed by another word character.

### FriendsService

| Method | Behaviour |
|---|---|
| `SendFriendshipRequest` | Idempotent: creates a `Pending` request only if none exists for that sender and receiver. |
| `UpdateFriendshipRequestStatus` | On **Accepted**, creates a `FriendShip` and removes the request. On **Rejected**, updates the status. |
| `CancelFriendshipRequest` | Sender withdraws a pending request. |
| `GetFriendships` | Friends regardless of who sent the original request. The *other* user's name and picture are projected. |
| `GetSuggestedFriends` | Excludes yourself, current friends and anyone with a pending request in either direction. Takes 5 and computes each one's friend count in SQL. |
| `GetSentFriendshipRequests` | Sent requests, ordered by status group and then by date. |
| `GetReceivedFriendshipRequests` | Incoming **pending** requests. |
| `RemoveFriendship` | Unfriend. |

### NotificationService

- `AddNewNotification(userId, type, actorName)` stores the notification with a generated message (e.g. *"Jane Doe Liked your post"*), recounts **unread** notifications and pushes the count to that user over SignalR.
- `GetAllNotifications` returns unread first, then newest first.
- `GetNewNotificationsCount` and `SetNotificationAsRead`.

### SearchService

Uses `Contains` (SQL `LIKE '%term%'`) on post content and user full names, plus a friendship query that returns only *your* friends whose names match.

### Other services
- **UserService**: get a profile, update the profile picture, list a user's visible posts.
- **FavoritesService**: posts the user has bookmarked that are not deleted.
- **AdminService**: all reports for a post.

---

## Real-Time Notifications (SignalR)

```
 User A likes User B's post
          │
          ▼
 HomeController.LikePost ──► PostService.LikePost (toggle, transaction)
          │ SendNotification == true
          ▼
 NotificationService.AddNewNotification(B, "like", "A")
          │ 1. INSERT Notification
          │ 2. count unread for B
          ▼
 IHubContext<NotificationsHub>.Clients.User("B").SendAsync("ReceiveNotification", count)
          │  WebSocket
          ▼
 Browser (site.js):  connection.on("ReceiveNotification", n => badge.innerHTML = n)
```

- Hub endpoint: **`/notification-hub`**.
- `Clients.User(id)` relies on SignalR's default `IUserIdProvider`, which reads the authenticated user's `NameIdentifier` claim, so messages go to **every open tab of that user only**.
- On page load, `site.js` gets the starting count from `GET /Notifications/GetNotificationsCount` and then opens the hub connection.
- Opening the bell loads `GET /Notifications/GetNotifications` as a partial view into the dropdown.

---

## Authentication and Authorization

### Local accounts
- **Register** creates `FullName = First + Last` with `UserName = Email`, assigns the `User` role, adds a `FullName` **claim** (so names can be shown without a database query) and signs the user in.
- **Login** looks the user up by e-mail, adds the `FullName` claim if it is missing, then signs in with a password. Admins are sent to the moderation dashboard.
- Password policy: at least 8 characters (Identity defaults otherwise).
- The cookie redirects to `/Authentication/Login` and `/Authentication/AccessDenied`.

### External login (Google / GitHub)
```
/Authentication/ExternalLogin?provider=Google
      └─► OAuth challenge ─► /signin-google ─► /Authentication/ExternalLoginCallback
             ├─ user exists → sign in
             └─ new e-mail  → create user (EmailConfirmed = true), role "User",
                              FullName claim, sign in
```
GitHub requests the `user:email` scope so it can read the e-mail address.

### Roles
| Role | Access |
|---|---|
| `User` | Feed, stories, search, notifications, profiles, settings |
| `Admin` | Moderation dashboard. Admins visiting `/` are redirected to `/Admin` |

---

## Web Layer

### Controllers

| Controller | Responsibilities |
|---|---|
| `HomeController` | Feed, post details, create a post, like / comment / bookmark (return **partial views** for AJAX), visibility, report, delete |
| `StoriesController` | Story list (last 24 hours) and story creation with image upload |
| `FriendsController` | Friends, sent and received requests, send / accept / reject / cancel / remove |
| `NotificationsController` | Unread count (JSON), list (partial), mark as read, redirect based on notification type |
| `SearchController` | Combined search results |
| `UsersController` | Public profile page |
| `SettingsController` | Settings page and profile picture upload to `wwwroot/profile-pictures/` |
| `FavoritesController` | Bookmarked posts, plus the same post actions as the feed |
| `AuthenticationController` | Register, login, logout, external login, update password and profile |
| `AdminController` | Reported posts, report details, dismiss, remove |

### View Components

| Component | Renders |
|---|---|
| `FriendsViewComponent` | "Suggested friends" sidebar with friend counts |
| `HashtagsViewComponent` | Up to 5 **trending hashtags** used by posts currently in the feed, ordered by count and then by creation date |
| `StoriesViewComponent` | Story strip at the top of the feed |

### Partial views
The post card is split into small partials (`_PostImage`, `_PostTextContent`, `_Like`, `_BookmarkIcon`, `_CommentsContent`, `_AddComment`, `_PostOptions`, `_AboutAuthor`), so the server can re-render **just one part** after an AJAX call.

---

## AJAX Interactions

Like, bookmark and comment are sent from the feed **without a page reload**:

```js
const response = await fetch(form.action, {
  method: 'POST',
  headers: {
    'Content-Type': 'application/x-www-form-urlencoded',
    'RequestVerificationToken': form.querySelector('input[name="__RequestVerificationToken"]').value
  },
  body: new URLSearchParams(new FormData(form))
});
container.innerHTML = await response.text();   // server-rendered partial view
```

- The anti-forgery token goes in a header (the like endpoint uses `[ValidateAntiForgeryToken]`).
- The button is disabled while the request is in flight to prevent double submission.
- The server returns HTML (a partial view), so one Razor template drives both the first render and the update.

---

## Routes Reference

| Method | Route | Description |
|---|---|---|
| GET | `/` | Feed (admins are redirected to `/Admin`) |
| GET | `/Home/Details?postId=` | Single post |
| POST | `/Home/CreateStatus` | Create a post (multipart) |
| POST | `/Home/LikePost` | Toggle like → `_Like` partial |
| POST | `/Home/AddComment` | Add a comment → `_CommentsContent` partial |
| POST | `/Home/BookmarkPost` | Toggle bookmark → `_BookmarkIcon` partial |
| POST | `/Home/PostVisibility` · `/Home/ReportPost` · `/Home/DeletePost` · `/Home/DeleteComment` | Post actions |
| GET / POST | `/Stories` · `/Stories/CreateStory` | Stories |
| GET | `/Friends/GetFriendships` · `/Friends/GetSentFriendshipRequests` · `/Friends/GetReceivedFriendshipRequests` | Friend lists |
| GET / POST | `/Friends/SendFriendshipRequest` · `AcceptFriendshipRequest` · `RejectFriendshipRequest` · `CancelFriendshipRequest` · `RemoveFriend` | Friend actions |
| GET | `/Notifications/GetNotificationsCount` | Unread count (JSON) |
| GET | `/Notifications/GetNotifications` | Notification list (partial) |
| POST | `/Notifications/SetNotificationAsRead?notificationId=` | Mark as read |
| GET | `/Search?searchString=` | Search |
| GET | `/Users/Details?userId=` | Profile |
| GET / POST | `/Settings` · `/Settings/UpdateUserProfilePicture` | Settings |
| GET / POST | `/Authentication/Login` · `/Register` · `/Logout` · `/ExternalLogin` · `/UpdateUserPassword` · `/UpdateUserData` | Account |
| GET | `/Admin` · `/Admin/PostReports?postId=` · `/Admin/DismissPostReports` · `/Admin/RemoveReportedPost` | Moderation |
| WS | `/notification-hub` | SignalR hub |

---

## Getting Started

### Prerequisites
- [.NET 9 SDK](https://dotnet.microsoft.com/download)
- SQL Server, SQL Server Express or LocalDB
- (Optional) Google and GitHub OAuth apps for social login

### 1. Clone
```bash
git clone https://github.com/Bardiya-basafa/Circle-App.git
cd Circle-App
```

### 2. Configure the database
Set `ConnectionStrings:DefaultConnection` in `CircleApp.UI/appsettings.json` (or `appsettings.Development.json`):
```json
"ConnectionStrings": {
  "DefaultConnection": "Server=(localdb)\\MSSQLLocalDB;Database=CircleAppDb;Trusted_Connection=True;Encrypt=False;"
}
```

### 3. Configure OAuth (optional)
Keep OAuth credentials **out of source control** by using User Secrets:
```bash
cd CircleApp.UI
dotnet user-secrets init
dotnet user-secrets set "Authentication:Google:ClientId"     "<google-client-id>"
dotnet user-secrets set "Authentication:Google:ClientSecret" "<google-client-secret>"
dotnet user-secrets set "Authentication:GitHub:ClientId"     "<github-client-id>"
dotnet user-secrets set "Authentication:GitHub:ClientSecret" "<github-client-secret>"
```
Callback URLs to register with the providers: `https://localhost:<port>/signin-google` and `https://localhost:<port>/signin-github`.

### 4. Run
```bash
dotnet run --project CircleApp.UI
```
Migrations are applied and seed data is inserted **automatically** on startup. Open the printed URL, **register** a new account, and start posting.

---

## Known Issues and Roadmap

- `FriendsController` and `FavoritesController` are decorated with `[Authorize(Roles = "Admin")]`, but they are user features. They should use the `User` role or plain `[Authorize]`.
- `ReportPost` only finds posts owned by the reporting user. The ownership filter should be removed so users can report other people's posts.
- `StoriesViewComponent` uses a hard-coded `LoggedInUserId = 1` and does not apply the 24-hour window.
- The seeded demo accounts have a `UserName` that differs from their e-mail, while login signs in by e-mail as the username. Seeded accounts therefore cannot log in with their e-mail.
- Search matches deleted and private posts. It should apply the same visibility rules as the feed.
- Feed pagination or infinite scroll, and image size and type validation on upload.
- Real-time updates for new posts and comments, not just notification counts.
- Unit and integration tests for the services.
