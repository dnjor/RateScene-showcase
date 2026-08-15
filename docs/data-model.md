# RateScene — Data Model

This document focuses on the most important data relationships and database design decisions in RateScene. It is intentionally selective rather than a complete reference for every model, field, or internal implementation detail.

The goal is to highlight the relationships that best explain how the platform's core data is structured and how important integrity rules are enforced.

---

## 1. User & Profile

RateScene uses a custom `User` model built on Django's `AbstractUser` for authentication and account identity.

Profile-specific information is stored separately in a `Profile` model connected to the user through a one-to-one relationship.

```mermaid
flowchart LR
    A[Django AbstractUser] --> B[RateScene User]
    B -->|One-to-One| C[Profile]
```

The `User` model owns account identity such as username and email, while the `Profile` model stores user-facing profile data such as display name, biography, and avatar.

```python
class Profile(models.Model):
    user = models.OneToOneField(
        settings.AUTH_USER_MODEL,
        on_delete=models.CASCADE,
        related_name="profile",
    )
```

Keeping these responsibilities separate avoids mixing authentication data with profile presentation data.

Because the relationship uses `on_delete=models.CASCADE`, removing a user also removes the associated profile.

---

## 2. User & Title Ecosystem

A `Title` is the central content entity used for movies and TV shows.

Users interact with titles through dedicated relationship models rather than through a single generic user-title record.

### Relationship Overview

```mermaid
erDiagram
    USER ||--o{ TITLE_RATING : rates
    TITLE ||--o{ TITLE_RATING : receives

    USER ||--o{ TITLE_REVIEW : writes
    TITLE ||--o{ TITLE_REVIEW : receives

    USER ||--o{ USER_TITLE_INTERACTION : maintains
    TITLE ||--o{ USER_TITLE_INTERACTION : tracks
```

This means the platform supports many users interacting with many titles, while each relationship type has its own responsibility.

- `TitleRating` stores the user's score for a title.
- `TitleReview` stores the user's written review.
- `UserTitleInteraction` stores the user's personal state for a title, including:
  - `watch_status`: `watchlist`, `watched`, or no watch state.
  - `is_favorite`: `true` or `false`.

### One Record per User and Title

Each relationship model allows only one record for the same user/title pair.

For example, a user cannot have two separate rating records for the same title:

```python
models.UniqueConstraint(
    fields=["user", "title"],
    name="unique_user_title_rating",
)
```

Equivalent constraints are used for reviews and user-title interactions.

```text
User + Title
│
├── One Rating
├── One Review
└── One UserTitleInteraction
    ├── watch_status → watchlist / watched / none
    └── is_favorite → true / false
```

> `UserTitleInteraction` is one consolidated record for the user-title relationship, not a single interaction type.

This allows many users to interact with the same title while preventing duplicate state for the same user.

### Title Identity

Titles imported from TMDB are uniquely identified using both the external TMDB identifier and media type:

```python
models.UniqueConstraint(
    fields=["tmdb_id", "media_type"],
    name="unique_tmdb_title",
)
```

This prevents duplicate local title records for the same external media entity.

### Rating Implies Watched

Rating a title also updates the user's personal title state.

The rating and watched-state update are performed together:

```text
User rates Title
      ↓
Create or Update Rating
      ↓
Mark Title as WATCHED
```

The service performs both operations inside a database transaction:

```python
@transaction.atomic
def save_rating_and_mark_watched(*, user, title, score):
    rating, created = create_or_update_rating(
        user=user,
        title=title,
        score=score,
    )

    interaction, _ = update_user_title_interaction(
        user=user,
        title=title,
        data={
            "watch_status":
            UserTitleInteraction.WatchStatus.WATCHED
        },
    )
```

Once a title has been rated, its watch status cannot be changed back to a non-watched state.

This keeps the user's rating and viewing state logically consistent.

### Account Deletion Behavior

Different user-title relationships have different lifecycle rules when an account is deleted.

```text
User deleted
│
├── UserTitleInteraction → removed
├── TitleRating          → preserved, user = NULL
└── TitleReview          → preserved, user = NULL
```

Personal state such as favorites and watch status is tied directly to the user's account and is removed with it.

Ratings and reviews are treated as historical contributions to the title and remain available after account deletion, while their relationship to the deleted user is removed.

The models implement this difference through their deletion behavior:

```python
# Personal user-title state
user = models.ForeignKey(
    User,
    on_delete=models.CASCADE,
)

# Rating / Review
user = models.ForeignKey(
    User,
    on_delete=models.SET_NULL,
    null=True,
    blank=True,
)
```

This preserves the title's historical rating and review data without retaining the deleted account relationship.

### Engineering Principle

The user-title model separates different kinds of state instead of combining them into one large relationship.

Ratings, reviews, and personal collection state each have their own lifecycle and integrity rules, while database constraints keep each user/title relationship unique and consistent.

---

## 3. Discussion & Thread Relationships

RateScene organizes discussions around `Space` objects. A space defines the context of the discussion, while posts, comments, replies, and threads follow the same structure regardless of that context.

### Space Context

A `Space` can be `GLOBAL`, `CUSTOM`, or `TITLE`.

```mermaid
flowchart TD
    S[Space]

    S --> G[GLOBAL]
    S --> C[CUSTOM]
    S --> T[TITLE]

    T -. One-to-One .-> TITLE[Title]

    G --> P[Post]
    C --> P
    T --> P

    P --> CM[Comment]
    CM --> R[Comment Reply]
    R --> NR[Nested Reply]

    CM -. anchors .-> TH[Thread]
    TH --> TM[ThreadMessage]
```

All three space types can contain posts.

A `TITLE` space has one additional relationship: it can be connected to a specific `Title` through a one-to-one relationship.

### Discussion Hierarchy

The discussion structure remains the same regardless of the space type:

```text
Space
└── Post
    └── Comment
        └── Comment (Reply)
            └── Comment (Nested Reply)
```

Replies do not use a separate model. They reuse the `Comment` model through self-referencing relationships.

`parent` identifies the comment being directly replied to, while `root` keeps replies associated with the same comment tree.

Threads form a separate focused conversation layer connected to the comment structure:

```text
Comment
└── Thread
    ├── Participant 1
    ├── Participant 2
    └── ThreadMessages
```

A thread therefore keeps its relationship to the discussion it originated from while storing its messages separately from normal comments.

> For thread creation rules, participant permissions, public visibility, transactions, and notifications, see [Discussion Thread Lifecycle](feature-flows.md#3-discussion-thread-lifecycle).

---

## 4. Shared Interaction & Notification Models

RateScene uses shared models for behavior that can apply across different parts of the platform instead of creating feature-specific models for each content type.

### Generic Reactions

A single `Reaction` model handles likes and dislikes across supported content objects.

```mermaid
flowchart LR
    U[User] --> R[Reaction]

    R --> CT[content_type]
    R --> ID[object_id]

    CT --> O[content_object]
    ID --> O

    R --> V{value}
    V --> L[LIKE +1]
    V --> D[DISLIKE -1]
```

A reaction target is identified using two values:

- `content_type` identifies the type of model.
- `object_id` identifies the specific record within that model.

Django's `GenericForeignKey` combines them into `content_object`:

```python
content_object = GenericForeignKey(
    "content_type",
    "object_id",
)
```

This allows one reaction model to be reused across different supported content types rather than introducing separate like/dislike models for each feature.

Each user can have only one reaction for the same target:

```mermaid
flowchart LR
    U[User] --> K[content_type + object_id]
    K --> R[One Reaction]
    R --> L[LIKE]
    R --> D[DISLIKE]
```

Database constraints also restrict the stored reaction value to `+1` for Like or `-1` for Dislike.

### Post View Identity

Post views support both authenticated users and anonymous visitors while keeping one clear visitor identity per view.

```mermaid
flowchart TD
    PV[PostView]
    PV --> P[Post]
    PV --> I{Visitor Identity}
    I --> U[Logged-in User]
    I --> G[Guest visitor_key]
```

A view uses either `user` or `visitor_key`, never both.

Separate uniqueness rules prevent the same logged-in user or the same guest session from producing duplicate view records for the same post.

### Notifications

Notifications connect a recipient with the user or event that caused the notification and can reference related platform content through the same generic-target pattern.

```mermaid
flowchart LR
    A[Actor] --> N[Notification]
    N --> R[Recipient]
    N --> T[type]
    N --> CT[content_type]
    N --> ID[object_id]
    CT --> O[content_object]
    ID --> O
```

This allows one notification model to reference different supported content types without requiring a separate notification model for each feature.

`SYSTEM` notifications may exist without a content target, while other notification types require both `content_type` and `object_id`.

### Engineering Principle

Shared interaction models keep cross-cutting features reusable while database constraints preserve valid identities, unique reactions, and consistent generic content relationships.
