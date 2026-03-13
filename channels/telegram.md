---
title: "Telegram"
description: "Interact with your agent through Telegram"
icon: telegram
---

In this guide, we will show you how to set up a Telegram channel to interact with your IronClaw agent. We assume that you already have an agent up and running.

<Note>
If you haven't set up your agent yet, follow our [Quickstart guide](../quickstart)
</Note>

---

## Set up a Telegram channel

<Steps>

<Step title="Create a new bot with BotFather">

In order to create a new Telegram bot, you need to talk to [BotFather](https://t.me/botfather), the official Telegram bot that helps you create and manage your bots.

<Steps>
    <Step title="Start a conversation with BotFather">
    Search for "BotFather" in the Telegram app and start a conversation with it. You can also use this link: [https://t.me/botfather](https://t.me/botfather)
    </Step>
    <Step title="Create a new bot">
    Send the command `/newbot` to BotFather and follow the instructions to create a new bot. You will need to choose a name and a username for your bot. The username must end with "bot". For example, "my_agent_bot".
    </Step>
    <Step title="Get your bot token">
    After creating your bot, BotFather will give you a token that looks like this: `123456789:ABCdefGhIJKlmNoPQRsTUVwxyZ`. This token is used to authenticate your bot and allow it to access the Telegram API. Keep this token safe and do not share it with anyone. You will need it later to configure the Telegram channel in IronClaw.
    </Step>
</Steps>

</Step>
<Step title="Configure the Telegram channel in IronClaw">

    Invoke the IronClaw CLI onboard wizard using the `--channels-only` flag to configure only the channels without going through the entire onboarding process again:

    ```
    ironclaw onboard --channels-only
    ```

    <Steps>
        <Step title="Config the Tunnel">
            If you have not setup a `Tunnel` yet, the wizard will ask you to choose a tunnel provider and set it up. We recommend using [ngrok](https://dashboard.ngrok.com/) for its ease of use and reliability.

            ![ngrok setup](/images/channels/tunnel.png)
        </Step>
        <Step title="Install the Telegram channel">
        Select the Telegram channel from the list of available channels to install it.
        ![select channel](/images/channels/telegram-channel.png)
        </Step>
        <Step title="Add your bot token">
            Enter the bot token you got from BotFather in the previous step.
        </Step>
    </Steps>
</Step>

<Step title="Test your Telegram channel">
    Now that you have configured your Telegram channel, it is time to test it out. Start the `ironclaw` agent if it is not already running:

    ```
    ironclaw
    ```

    Send a message to your bot in Telegram. It will respond with a command that you will need to execute in the terminal to complete the channel setup:

    ```
    ironclaw pairing approve telegram <PAIRING_CODE>
    ```
</Step>

</Steps>

---

## Telegram side settings

<Accordion title="Privacy mode and group visibility">
Telegram bots default to Privacy Mode, which limits what group messages they receive.If the bot must see all group messages, either:

    - disable privacy mode via `/setprivacy`, or
    - make the bot a group admin.

When toggling privacy mode, remove and re-add the bot in each group so Telegram applies the change.
</Accordion>
<Accordion title="Group permissions">
    Admin status is controlled in Telegram group settings.Admin bots receive all group messages, which is useful for always-on group behavior.
</Accordion>
<Accordion title="Helpful BotFather toggles">
    - `/setjoingroups` to allow/deny group adds
    - `/setprivacy` for group visibility behavior

</Accordion>

---

## Access Control and Security

By default, the Telegram channel is configured to require a manual pairing approval for each new user that interacts with the bot. This means that whenever a new user sends a message to the bot, you will receive a pairing request in the terminal, and you will need to approve it by running the command `ironclaw pairing approve telegram <PAIRING_CODE>`.

