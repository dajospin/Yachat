# Codexgram — Implementation Plan

Instagram-style social app. Auth via Clerk, backend/DB via Convex, Expo (native tabs) + NativeWind for UI. Targets iOS, Android, and Web.

Status: planning complete, no feature code written yet. This doc is the source of truth for scope and sequencing — implement phase by phase, checking items off as they land.

---

## 1. Tech stack

- **Expo SDK 57** (already scaffolded — `expo` ~57.0.23, `expo-router` ~57.0.21, `react-native` 0.86.3, `react` 19.2.3). **Read https://docs.expo.dev/versions/v57.0.0/ before writing any Expo code — do not rely on memorized APIs, this SDK changed significantly.**
- **Clerk** (`@clerk/clerk-expo`) — authentication (Google + Apple sign-in), synced into Convex.
- **Convex** — database, backend functions, real-time subscriptions, file storage (images + video).
- **NativeWind** — styling, Tailwind classes on RN components.
- **expo-router Native Tabs** — iOS/Android tab bar (native `UITabBarController` / `BottomNavigation`). Web gets a separate custom NativeWind tab bar component (Native Tabs has no web equivalent).
- **EAS Build** — iOS/Android builds. Web build via `expo export --platform web` / static hosting. No CI pipeline for v1.

## 2. Scope

### In scope (v1)
- Sign in/up with Google and Apple via Clerk
- Create image or video posts (video up to ~3–5 min), optional caption
- Follow/unfollow; **private accounts require follow-request approval**, public accounts follow instantly
- Like and comment on posts (flat comments, no nested replies; likes on posts only, not comments)
- Delete own posts and own comments; **post owner can also delete any comment on their own post**
- Direct messages: 1:1 **and group** conversations, real-time via Convex subscriptions, text only
- View/edit own profile: username, display name, bio, avatar
- Account settings (via Clerk's own UI/flows)
- Public/private account toggle
- Saved/bookmarked posts
- Basic reporting: report a post or comment (reason + status), admin can review
- Admin role (manually granted): delete any post/comment, view/manage/deactivate users, view reports — all as gated UI inside the same app, no separate dashboard
- 4 tabs: **Home** (feed from followed users), **Explore** (grid of recent public posts, no ranking), **Message**, **Profile**

### Explicitly out of scope (v2+)
- Stories (24h ephemeral posts)
- Push notifications
- Hashtags / keyword search
- Blocking users (reporting exists without blocking in v1 — confirmed asymmetry, not a bug)
- Any monetization/billing
- Automated test suite (manual device/simulator testing is the v1 quality bar)
- CI/CD pipeline

## 3. Data model (Convex schema sketch)

```ts
users: {
  clerkId: string,        // unique, synced from Clerk
  username: string,       // unique
  displayName: string,
  bio?: string,
  avatarStorageId?: Id<"_storage">,
  isPrivate: boolean,
  role: "user" | "admin",
  createdAt: number,
}

posts: {
  authorId: Id<"users">,
  mediaType: "image" | "video",
  mediaStorageId: Id<"_storage">,   // required — no text-only posts
  caption?: string,
  createdAt: number,
}

follows: {
  followerId: Id<"users">,
  followingId: Id<"users">,
  status: "accepted" | "pending",  // "pending" only meaningful when followee isPrivate
  createdAt: number,
}

likes: {
  userId: Id<"users">,
  postId: Id<"posts">,
  createdAt: number,
}

comments: {
  postId: Id<"posts">,
  authorId: Id<"users">,
  text: string,
  createdAt: number,
}

conversations: {
  participantIds: Id<"users">[],   // length 2 = 1:1, length >2 = group
  createdAt: number,
}

messages: {
  conversationId: Id<"conversations">,
  senderId: Id<"users">,
  text: string,
  createdAt: number,
}

savedPosts: {
  userId: Id<"users">,
  postId: Id<"posts">,
  createdAt: number,
}

reports: {
  reporterId: Id<"users">,
  targetType: "post" | "comment",
  targetId: string,
  reason: string,
  status: "open" | "reviewed" | "dismissed",
  createdAt: number,
}
```

Indexes needed at minimum: `users.by_clerkId`, `users.by_username`, `posts.by_author`, `follows.by_follower`, `follows.by_following`, `likes.by_post`, `likes.by_user_post` (uniqueness), `comments.by_post`, `savedPosts.by_user`, `messages.by_conversation`.

## 4. Architecture notes

- Convex stores images/video directly (`ctx.storage`) — no separate media service (Mux/Cloudflare Stream) for v1. **Verify current Convex file-size/storage limits against live docs before building video upload** — this is the biggest technical risk in the plan (see Risks).
- Clerk is the identity source of truth; a Convex mutation/webhook syncs Clerk user → `users` table on first sign-in.
- Private-account visibility and follow-request logic must be enforced in Convex queries (feed, explore, profile), not just in the UI.
- Admin checks (`role === "admin"`) enforced server-side in Convex mutations, not just hidden UI.
- Real-time: Convex's reactive queries drive live feed/likes/comments/messages — no manual polling.

## 5. Assumptions

- Comments are flat; likes apply to posts only.
- Posts require media; caption is optional.
- Username is the unique handle; display name is not unique.
- Admin role is granted manually (direct DB edit), no self-service admin invite flow.
- Accessibility/i18n: standard reasonable defaults only, not a tracked requirement.
- Report exists without a corresponding block feature (confirmed intentional for v1).

## 6. Open risks

- Convex storage behavior/limits for 3–5 min videos (size, bandwidth, no adaptive streaming) — check current docs before committing to this approach at scale.
- expo-router Native Tabs web-compatibility specifics for this exact SDK version — verify against https://docs.expo.dev/versions/v57.0.0/, don't assume from memory.
- Scope is larger than "simple" — private accounts + follow requests, group DMs, saved posts, reporting, and admin role are all real subsystems, not incidental features.

## 7. Implementation phases

Work through these in order; each phase should be usable/testable before moving to the next.

### Phase 0 — Foundation
- [ ] Install & configure Convex (`npx convex dev`, `convex/schema.ts` per section 3)
- [ ] Install & configure Clerk (`@clerk/clerk-expo`), set up Google + Apple sign-in
- [ ] Wire Clerk provider + Convex provider into `src/app/_layout.tsx`
- [ ] Install NativeWind, Tailwind config, `global.css` (re-create — was removed in reset)
- [ ] Convex mutation/webhook to sync Clerk user → `users` table on first sign-in
- [ ] Auth gating: signed-out → sign-in screen; signed-in → tabs

### Phase 1 — Navigation shell
- [ ] 4-tab layout with expo-router **Native Tabs** (iOS/Android): Home, Explore, Message, Profile
- [ ] Custom NativeWind tab bar for web build (same 4 destinations)
- [ ] Empty/placeholder screens for each tab

### Phase 2 — Profile basics
- [ ] View own profile (username, display name, bio, avatar)
- [ ] Edit profile (update fields, upload avatar to Convex storage)
- [ ] Public/private toggle
- [ ] View another user's profile (public info, follow button state)

### Phase 3 — Posts
- [ ] Create post: pick image/video from library or camera, upload to Convex storage, optional caption
- [ ] Enforce video length limit client-side (~3–5 min) before upload
- [ ] Render a post (image or video playback) in a reusable component
- [ ] Delete own post

### Phase 4 — Follow system
- [ ] Follow/unfollow action; instant for public accounts
- [ ] Follow-request flow for private accounts (pending → accepted), accept/reject UI on the requestee side
- [ ] Followers/following lists on profile

### Phase 5 — Home feed & Explore
- [ ] Home: posts from accounts the current user follows (accepted follows only), reverse-chronological
- [ ] Explore: grid of recent posts from public accounts (+ accepted-follow private accounts), no ranking
- [ ] Empty states for both (no follows yet / no posts yet)

### Phase 6 — Likes & comments
- [ ] Like/unlike a post, live count
- [ ] Add comment; list comments on a post
- [ ] Delete comment (author or post owner)

### Phase 7 — Saved posts
- [ ] Save/unsave a post
- [ ] Saved posts view on own profile

### Phase 8 — Direct messages
- [ ] Start a 1:1 conversation from a profile
- [ ] Start a group conversation (select multiple participants)
- [ ] Conversation list (Message tab), real-time updates
- [ ] Message thread view, send/receive in real time

### Phase 9 — Reporting & admin
- [ ] Report action on a post/comment (reason, creates `reports` row)
- [ ] Admin-gated UI: delete any post/comment, view reports list, view/deactivate users
- [ ] Server-side role check on all admin mutations

### Phase 10 — Polish & delivery
- [ ] Empty/error/loading states across all screens
- [ ] Manual test pass on iOS simulator, Android emulator, and web
- [ ] EAS Build config for iOS/Android; web export/hosting
- [ ] Convex prod deployment
