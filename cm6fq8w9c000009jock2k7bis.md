---
title: "How Do We Export Slack Messages Without Admin Permissions?"
datePublished: Tue Jan 28 2025 00:16:21 GMT+0000 (Coordinated Universal Time)
cuid: cm6fq8w9c000009jock2k7bis
slug: how-do-we-export-slack-messages-without-admin-permissions
cover: https://cdn.hashnode.com/res/hashnode/image/upload/v1738020092258/62256b85-d60a-4425-ad07-44fc000a66db.png
tags: slack, migrations, exporter

---

As a regular Slack user without admin rights, we can export Slack messages thanks to a great open-source project, [slackdump](https://github.com/rusq/slackdump).

> Save or export your private and public Slack messages, threads, files, and users locally without admin privileges.

Download a release [here](https://github.com/rusq/slackdump/releases/) and unzip the file. Open a terminal, navigate to the unzipped folder, and run:

```powershell
./slackdump workspace new <workspace name>
```

The workspace name is part of our Slack URL: `<workspace name>.slack.com`. After running the command above, a menu will appear. Choose the `Interactive` option:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1738009689000/9edad2ab-c241-4a2b-abed-d3028e654543.png align="center")

A browser window will open, asking us to enter our email address and password:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1738009898604/8804bd54-9919-4c42-a06a-f76b67f25d39.png align="center")

Before continuing, get the ID of the channel we want to export. We can find this by selecting the `open conversation details` option in each channel.

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1738011490311/73716ab9-1c5d-4600-9972-f18932331281.png align="center")

After signing in successfully, run the following command to begin the export:

```powershell
./slackdump export -workspace <workspace name> <channel ID>
```

As a result of the command, a ZIP file will be created in the following format: `slackdump_yyyymmdd_hhmmss.zip`. Use the following command to view the data:

```powershell
./slackdump view <file name>
```

As a result, a browser will display your exported messages in an easy-to-read format:

![](https://cdn.hashnode.com/res/hashnode/image/upload/v1738012992639/1f7295ab-60a8-490c-a182-456d055290ea.png align="center")

Thank you, and happy coding.