# wecom-app-push

Two directions for WeCom (企业微信) custom apps:

1. Push app messages to employees (approval to-dos, business alerts, receipt
   reminders) via the WeCom bot API.
2. Receive user messages in your own system: API callback setup, callback
   crypto (AES/WXBizMsgCrypt), letting colleagues query system data directly
   from WeCom chat.

Includes the non-obvious steps: trusted-domain verification file deployment,
the two userid syntaxes, message encryption/decryption, and nginx static
pass-through with /v3/ prefix isolation for debugging.

Ships as an agent SKILL.md: install by copying `SKILL.md` into your agent's
skills directory.

MIT License.
