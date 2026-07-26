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
let wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
webpackChunkdiscord_app.pop();

let ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getStreamerActiveStreamMetadata)?.exports?.A;
let RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getRunningGames)?.exports?.Ay;
let QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getQuest)?.exports?.A;
let ChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.A?.__proto__?.getAllThreadsForParent)?.exports?.A;
let GuildChannelStore = Object.values(wpRequire.c).find(x => x?.exports?.Ay?.getSFWDefaultChannel)?.exports?.Ay;
let FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.h?.__proto__?.flushWaitQueue)?.exports?.h;
let api = Object.values(wpRequire.c).find(x => x?.exports?.Bo?.get)?.exports?.Bo;

const supportedTasks = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"];

let quests = [...QuestsStore.quests.values()].filter(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now() && supportedTasks.find(y => Object.keys((x.config.taskConfig ?? x.config.taskConfigV2).tasks).includes(y)));
let isApp = typeof DiscordNative !== "undefined";

if(quests.length === 0) {
    console.log("You don't have any uncompleted quests!");
} else {
    let doJob = function() {
        const quest = quests.pop();
        if(!quest) return;

        const pid = Math.floor(Math.random() * 30000) + 1000;

        const questName = quest.config.messages.questName;
        const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
        const taskName = supportedTasks.find(x => taskConfig.tasks[x] != null);
        const taskData = taskConfig.tasks[taskName];
        const applicationId = taskData?.applications?.[0]?.id ?? taskData?.applicationId ?? quest.config.application?.id;

        if (!applicationId && taskName !== "WATCH_VIDEO") {
            console.error(`Could not resolve Application ID for quest: ${questName}`);
            return doJob();
        }

        const secondsNeeded = taskData.target;
        let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;

        if(taskName === "WATCH_VIDEO" || taskName === "WATCH_VIDEO_ON_MOBILE") {
            const speed = 7;
            let completed = false;
            let fn = async () => {
                while(true) {
                    const remaining = Math.min(speed, secondsNeeded - secondsDone);
                    await new Promise(resolve => setTimeout(resolve, remaining * 1000));

                    const timestamp = secondsDone + speed;
                    const res = await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: Math.min(secondsNeeded, timestamp + Math.random())}});
                    completed = res.body.completed_at != null;
                    secondsDone = Math.min(secondsNeeded, timestamp);

                    if(timestamp >= secondsNeeded) break;
                }
                if(!completed) {
                    await api.post({url: `/quests/${quest.id}/video-progress`, body: {timestamp: secondsNeeded}});
                }
                console.log("Quest completed!");
                doJob();
            };
            fn();
            console.log(`Spoofing video for ${questName}.`);
        } else if(taskName === "PLAY_ON_DESKTOP") {
            if(!isApp) {
                console.log("This no longer works in browser for non-video quests. Use the discord desktop app to complete the", questName, "quest!");
            } else {
                api.get({url: `/applications/public?application_ids=${applicationId}`}).then(res => {
                    const appData = res.body[0];
                    const exeName = appData.executables?.find(x => x.os === "win32")?.name?.replace(">","") ?? appData.name.replace(/[\/\:*?"<>|]/g, "");

                    const fakeGame = {
                        cmdLine: `C:\Program Files\${appData.name}\${exeName}`,
                        exeName,
                        exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
                        hidden: false,
                        isLauncher: false,
                        id: applicationId,
                        name: appData.name,
                        pid: pid,
                        pidPath: [pid],
                        processName: appData.name,
                        start: Date.now(),
                    };
                    const realGames = RunningGameStore.getRunningGames();
                    const fakeGames = [fakeGame];
                    const realGetRunningGames = RunningGameStore.getRunningGames;
                    const realGetGameForPID = RunningGameStore.getGameForPID;
                    RunningGameStore.getRunningGames = () => fakeGames;
                    RunningGameStore.getGameForPID = (pid) => fakeGames.find(x => x.pid === pid);
                    FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: realGames, added: [fakeGame], games: fakeGames});

                    let fn = data => {
                        let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
                        console.log(`Quest progress: ${progress}/${secondsNeeded}`);

                        if(progress >= secondsNeeded) {
                            console.log("Quest completed!");
                            RunningGameStore.getRunningGames = realGetRunningGames;
                            RunningGameStore.getGameForPID = realGetGameForPID;
                            FluxDispatcher.dispatch({type: "RUNNING_GAMES_CHANGE", removed: [fakeGame], added: [], games: []});
                            FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                            doJob();
                        }
                    };
                    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn);
                    console.log(`Spoofed your game to ${appData.name}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`);
                });
            }
        }
    };
    doJob();
}
```

---

### license
this script is with MIT license, so, u can take the script but uhh it's not my fault if you got your access of doing quests removed, u can't really get banned for using this
