hi, too lazy to do discord quests? this is for you, just copy and paste it in ur devtools and let it do the stuff, code is all yours (this script is from someone else but it doesn't work anymore, so I updated it)

---

### what this script actually does
- **detects active quests:** scans ur account for uncompleted tasks
- **spoof video stuff:** sends video progress timestamps so you don't have to sit through videos
- **spoofs play tasks:** mocks discord's internal running game store (`RunningGameStore`) so discord thinks ur playing the quest game normally
- **auto loop:** finishes one quest and jumps to the next one until there isn't any left

---

### How to use

1. **accept and select game stuff as desktop** in the quests you want to do
2. open discord DevTools (`ctrl + shift + I`), if it doesnt work watch a tut on how to enable dev tools
3. switch to console tab
4. Copy the code block below, paste it, and press `enter`, if it's the first time you're pasting it, discord won't let you paste the script, so just type `allow pasting` and it should let you paste

```javascript
(async () => {
    delete window.$;

    function handleApiError(err, context) {
        console.error(`api error in ${context}:`, err);
        
        if (err?.status === 503) {
            console.warn("discords servers are overloaded right now (503). wait 1-2 minutes and try again.");
        } else if (err?.status === 429) {
            console.warn("you are being rate limited by discord. wait 5 minutes before re-running.");
        } else if (err?.status === 401 || err?.status === 403) {
            console.warn("authorization failed. make sure you are logged in and accepted the quest first.");
        } else {
            console.warn(" network error. check your connection or retry in a bit.");
        }
    }

    let wpRequire;
    try {
        wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
        webpackChunkdiscord_app.pop();
    } catch (err) {
        console.error("failed to hook into discords webpack chunk listener:", err);
        console.warn("reload discord (ctrl + r) and try pasting again.");
        return;
    }

    let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getStreamerActiveStreamMetadata)?.exports?.A ?? Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata)?.exports?.Z;
    let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getRunningGames)?.exports?.Ay ?? Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames)?.exports?.ZP;
    let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getQuest)?.exports?.A ?? Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest)?.exports?.Z;
    let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getAllThreadsForParent)?.exports?.A;
    let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getSFWDefaultChannel)?.exports?.Ay;
    let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.h?.__proto__?.flushWaitQueue)?.exports?.h ?? Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue)?.exports?.Z;
    let api = Object.values(wpRequire.c).find(x => x?.exports?.Bo?.get)?.exports?.Bo ?? Object.values(wpRequire.c).find(x => x?.exports?.tn?.get)?.exports?.tn;

    let missingStores = [];
    if (!QuestsStore) missingStores.push("questsstore");
    if (!RunningGameStore) missingStores.push("runninggamestore");
    if (!FluxDispatcher) missingStores.push("fluxdispatcher");
    if (!api) missingStores.push("api");

    if (missingStores.length > 0) {
        console.error(`failed to locate required internal modules: ${missingStores.join(", ")}`);
        console.warn("discord might have updated its internal modules. try reloading discord (ctrl+r).");
        return;
    }

    const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];

    let quests = [...QuestsStore.quests.values()].filter(x => 
        x.userStatus?.enrolledAt && 
        !x.userStatus?.completedAt && 
        new Date(x.config.expiresAt).getTime() > Date.now() && 
        supportedTasks.find(y => Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y))
    );
    let isApp = typeof DiscordNative !== "undefined";

    if (quests.length === 0) {
        console.warn("you dont have any active, uncompleted quests!");
        console.warn("open settings -> quests and click accept quest first.");
        return;
    }

    let doJob = async function() {
        const quest = quests.pop();
        if (!quest) return;

        try {
            const pid = Math.floor(Math.random() * 30000) + 1000;
            const questName = quest.config?.messages?.questName ?? "unknown quest";
            const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
            const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null);
            const taskData = taskConfig.tasks[taskName];

            const applicationId = 
                taskData?.applications?.[0]?.id ?? 
                taskData?.applicationId ?? 
                quest.config?.application?.id ?? 
                quest.config?.applicationId ?? 
                quest.config?.taskConfig?.application?.id ??
                quest.config?.taskConfigV2?.application?.id;

            if (!applicationId && taskName !== "WATCH_VIDEO" && taskName !== "WATCH_VIDEO_ON_MOBILE") {
                console.error(`could not resolve application id for quest: "${questName}"`);
                console.warn("discord updated this quests schema structure.");
                return doJob();
            }

            const secondsNeeded = taskData.target;
            let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;

            if (taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
                const speed = 7;
                let completed = false;
                console.log(`spoofing video task for: "${questName}"`);
                
                while (true) {
                    try {
                        const remaining = Math.min(speed, secondsNeeded - secondsDone);
                        await new Promise(resolve => setTimeout(resolve, remaining * 1000));

                        const timestamp = secondsDone + speed;
                        const res = await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}});
                        completed = res?.body?.completed_at != null;
                        secondsDone = Math.min(secondsNeeded, timestamp);

                        if (timestamp >= secondsNeeded) break;
                    } catch (err) {
                        handleApiError(err, `video progress heartbeat failed for "${questName}"`);
                        console.warn("retrying step in 10 seconds...");
                        await new Promise(r => setTimeout(r, 10000));
                    }
                }
                if (!completed) {
                    try {
                        await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: secondsNeeded}});
                    } catch (err) {
                        handleApiError(err, `final completion heartbeat failed for "${questName}"`);
                    }
                }
                console.log(`quest completed: "${questName}"`);
                doJob();

            } else if (taskName === "PLAY_ON_DESKTOP") {
                if (!isApp) {
                    console.error(`play_on_desktop cannot run in standard browser windows for "${questName}".`);
                    console.warn("run this script inside the official discord desktop app, (not smth like vesktop)");
                    return doJob();
                }

                api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
                    const appData = res.body?.[0];
                    const exeName = appData?.executables?.find(x => x.os === "win32")?.name?.replace(">", "") ?? "game.exe";

                    const fakeGame = {
                        cmdLine: `C:\\Program Files\\${appData?.name ?? "Game"}\\${exeName}`,
                        exeName,
                        exePath: `c:/program files/${(appData?.name ?? "game").toLowerCase()}/${exeName}`,
                        hidden: false,
                        isLauncher: false,
                        id: applicationId,
                        name: appData?.name ?? questName,
                        pid: pid,
                        pidPath: [pid],
                        processName: appData?.name ?? questName,
                        start: Date.now(),
                    };
                    const realGames = RunningGameStore.getRunningGames();
                    const fakeGames = [fakeGame];
                    const realGetRunningGames = RunningGameStore.getRunningGames;
                    const realGetGameForPID = RunningGameStore.getGameForPID;

                    RunningGameStore.getRunningGames = () => fakeGames;
                    RunningGameStore.getGameForPID = (p) => fakeGames.find(x => x.pid === p);
                    FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames});

                    let fn = data => {
                        try {
                            let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
                            console.log(`progress on "${questName}": ${progress}/${secondsNeeded} seconds`);

                            if (progress >= secondsNeeded) {
                                console.log(`quest completed: "${questName}"`);
                                RunningGameStore.getRunningGames = realGetRunningGames;
                                RunningGameStore.getGameForPID = realGetGameForPID;
                                FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []});
                                FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                                doJob();
                            }
                        } catch (err) {
                            console.error("failed processing game heartbeat update:", err);
                        }
                    };
                    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                    console.log(`game process spoofed to "${appData?.name ?? questName}". leave client running for ~${Math.ceil((secondsNeeded - secondsDone) / 60)} minutes.`);
                }).catch(err => {
                    handleApiError(err, `failed to query application metadata for application id ${applicationId}`);
                    console.warn("retrying quest execution in 10 seconds...");
                    setTimeout(doJob, 10000);
                });

            } else if (taskName === "STREAM_ON_DESKTOP") {
                if (!isApp) {
                    console.error(`stream_on_desktop requires native client bindings for "${questName}".`);
                    console.warn("run this script inside the official discord desktop app, (not smth like vesktop)");
                    return doJob();
                }

                if (!ApplicationStreamingStore) {
                    console.error("streaming metadata store is unavailable.");
                    console.warn("restart discord client and try again.");
                    return doJob();
                }

                let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
                ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
                    id: applicationId,
                    pid,
                    sourceName: null
                });

                let fn = data => {
                    try {
                        let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);
                        console.log(`progress on "${questName}": ${progress}/${secondsNeeded} seconds`);

                        if (progress >= secondsNeeded) {
                            console.log(`quest completed: "${questName}"`);
                            ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
                            FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                            doJob();
                        }
                    } catch (err) {
                        console.error("failed processing stream heartbeat update:", err);
                    }
                };
                FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                console.log(`stream metadata spoofed for "${questName}". join any voice channel and stream a window for ~${Math.ceil((secondsNeeded - secondsDone) / 60)} minutes.`);

            } else if (taskName === "PLAY_ACTIVITY") {
                const channelId = ChannelStore?.getChannelIds?.()?.[0]?.id ?? Object.values(GuildChannelStore?.getSFWDefaultChannel?.() ?? {}).find(x => x != null && x.length > 0)?.[0]?.id;
                
                if (!channelId) {
                    console.error("could not resolve a text/voice channel id for activity execution.");
                    console.warn("click into any text or voice channel in discord first, then re-run the script.");
                    return doJob();
                }

                const streamKey = `call:${channelId}:1`;
                console.log(`spoofing activity progress for: "${questName}"`);

                while (true) {
                    try {
                        const res = await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: false}});
                        const progress = res.body.progress.PLAY_ACTIVITY.value;
                        console.log(`progress on "${questName}": ${progress}/${secondsNeeded} seconds`);

                        if (progress >= secondsNeeded) {
                            await api.post({url: `/quests/${quest.id}/heartbeat`, body: {stream_key: streamKey, terminal: true}});
                            break;
                        }
                        await new Promise(resolve => setTimeout(resolve, 30000));
                    } catch (err) {
                        handleApiError(err, `activity heartbeat ping failed for "${questName}"`);
                        console.warn("retrying activity update in 15 seconds...");
                        await new Promise(r => setTimeout(r, 15000));
                    }
                }
                console.log(`quest completed: "${questName}"`);
                doJob();
            }
        } catch (err) {
            console.error("an unexpected runtime error occurred:", err);
            console.warn("attempting to recover and proceed in 10 seconds...");
            setTimeout(doJob, 10000);
        }
    };

    doJob();
})();
```

---

### license
this script is with MIT license, so, u can take the script but uhh it's not my fault if you got your access of doing quests removed, u can't really get banned for using this
