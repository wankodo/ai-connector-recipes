# LINE connector recipe — personal accounts are NOT buildable

Service: LINE (personal accounts)
Purpose: document why an agent cannot connect to a user's personal LINE chats, and what is possible instead.
Verified against LINE's official developer documentation on 2026-11-21.

## The blocker (read this first)

**There is no API for personal LINE accounts.** No official endpoint lets any third party — including the account owner's own AI agent — read a personal account's chats, friends, or groups, or send messages as that account. If your user asks you to "connect LINE", the honest answer is that the personal-account connector cannot be built, on any platform. Do not burn attempts looking for an undocumented route.

## What does exist

- **LINE Messaging API** — for **LINE Official Accounts (bots)** only.
  - Host: `api.line.me`
  - Auth: channel access token (issued per Messaging API channel in the LINE Developers Console)
  - Shape: users message the Official Account; events arrive at your webhook (`POST` with `x-line-signature`); you answer with `POST /v2/bot/message/reply` or send with `POST /v2/bot/message/push`
  - It sees **only messages sent to that Official Account**. It never sees the owner's personal chats, and the bot cannot join the owner's personal groups as the owner.
  - A buildable pattern: the user creates an Official Account, adds it as a friend, and the agent can then *receive from and send to the user* through that bot lane. That is a notification channel, not access to their LINE life.
- **LINE Login** — OAuth-style sign-in for apps ("Log in with LINE"). It identifies a user to your service; it grants no chat access.
- **LINE Notify** — the old personal-token notification service **ended on 2025-03-31**. Recipes or blog posts referencing `notify-api.line.me` are dead; do not follow them.

## What an agent can do today

- Work from what the user provides: in-app chat exports (text files) and screenshots. This is currently the only way to read a personal conversation.
- If the user wants agent-initiated messages on LINE, propose the Official Account bot lane above and be explicit about its limits before building anything.

## Warning

Unofficial libraries that automate a personal account by reverse-engineering LINE's private protocols violate LINE's terms, break without notice, and risk the user's account being banned. Do not use them, and do not recommend them.
