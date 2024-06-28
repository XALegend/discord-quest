## Complete New Discord Quest (آپدیت شد)
> [!NOTE]
> برای انجام این کار شما نیاز به نصب برنامه vencord.exe و فعال کردن enable developer react در تنظیمات vencord نیاز هست

1. اول از همه در قسمت [Gift inventory](https://discord.com/quests/1241062724922769408) میشن رو اکسپت کنید
2. وارد یک ویس شوید
3. با یک یوز دیگر وارد همون ویس بشید
4. یک صفحه دلخواه رو استریم کنید
5. <kbd>Ctrl</kbd>+<kbd>Shift</kbd>+<kbd>I</kbd> را فشار دهید
6. وارد کنسول بشد
7. /allow pasting را وارد کنید
8. سپس این کد را در آن پیست کنید

```js
let wpRequire;
window.webpackChunkdiscord_app.push([
   [Math.random()], {}, (req) => {
      wpRequire = req;
   }
]);

let ApplicationStreamingStore, RunningGameStore, QuestsStore, ExperimentStore, FluxDispatcher, api
if (window.GLOBAL_ENV.SENTRY_TAGS.buildId === "366c746173a6ca0a801e9f4a4d7b6745e6de45d4") {
   ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getStreamerActiveStreamMetadata).exports.default;
   RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getRunningGames).exports.default;
   QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getQuest).exports.default;
   ExperimentStore = Object.values(wpRequire.c).find(x => x?.exports?.default?.getGuildExperiments).exports.default;
   FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.default?.flushWaitQueue).exports.default;
   api = Object.values(wpRequire.c).find(x => x?.exports?.getAPIBaseURL).exports.HTTP;
} else {
   ApplicationStreamingStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getStreamerActiveStreamMetadata).exports.Z;
   RunningGameStore = Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP;
   QuestsStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getQuest).exports.Z;
   ExperimentStore = Object.values(wpRequire.c).find(x => x?.exports?.Z?.getGuildExperiments).exports.Z;
   FluxDispatcher = Object.values(wpRequire.c).find(x => x?.exports?.Z?.flushWaitQueue).exports.Z;
   api = Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn;
}

let quest = [...QuestsStore.quests.values()].filter(x => x.userStatus?.enrolledAt && !x.userStatus?.completedAt && new Date(x.config.expiresAt).getTime() > Date.now())[1]
let isApp = navigator.userAgent.includes("Electron/")
if (!isApp) {
   console.log("This no longer works in browser. Use the desktop app!")
} else if (!quest) {
   console.log("You don't have any uncompleted quests!")
} else {
   const pid = Math.floor(Math.random() * 30000) + 1000

   let applicationId, applicationName, secondsNeeded, secondsDone, canPlay
   if (quest.config.configVersion === 1) {
      applicationId = quest.config.applicationId
      applicationName = quest.config.applicationName
      secondsNeeded = quest.config.streamDurationRequirementMinutes * 60
      secondsDone = quest.userStatus?.streamProgressSeconds ?? 0
      canPlay = quest.config.variants.includes(2)
   } else if (quest.config.configVersion === 2) {
      applicationId = quest.config.application.id
      applicationName = quest.config.application.name
      canPlay = ExperimentStore.getUserExperimentBucket("2024-04_quest_playtime_task") > 0 && quest.config.taskConfig.tasks["PLAY_ON_DESKTOP"]
      const taskName = canPlay ? "PLAY_ON_DESKTOP" : "STREAM_ON_DESKTOP"
      secondsNeeded = quest.config.taskConfig.tasks[taskName].target
      secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0
   }

   if (canPlay) {
      api.get({
         url: `/applications/public?application_ids=${applicationId}`
      }).then(res => {
         const appData = res.body[0]
         const exeName = appData.executables.find(x => x.os === "win32").name.replace(">", "")

         const games = RunningGameStore.getRunningGames()
         const fakeGame = {
            cmdLine: `C:\\Program Files\\${appData.name}\\${exeName}`,
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
         }
         games.push(fakeGame)
         FluxDispatcher.dispatch({
            type: "RUNNING_GAMES_CHANGE",
            removed: [],
            added: [fakeGame],
            games: games
         })

         let fn = data => {
            let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value)
            console.log(`Quest progress: ${progress}/${secondsNeeded}`)

            if (progress >= secondsNeeded) {
               console.log("Quest completed!")

               const idx = games.indexOf(fakeGame)
               if (idx > -1) {
                  games.splice(idx, 1)
                  FluxDispatcher.dispatch({
                     type: "RUNNING_GAMES_CHANGE",
                     removed: [fakeGame],
                     added: [],
                     games: []
                  })
               }
               FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
            }
         }
         FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)

         console.log(`Spoofed your game to ${applicationName}. Wait for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
      })
   } else {
      let realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata
      ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
         id: applicationId,
         pid,
         sourceName: null
      })

      let fn = data => {
         let progress = quest.config.configVersion === 1 ? data.userStatus.streamProgressSeconds : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value)
         console.log(`Quest progress: ${progress}/${secondsNeeded}`)

         if (progress >= secondsNeeded) {
            console.log("Quest completed!")

            ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc
            FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)
         }
      }
      FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", fn)

      console.log(`Spoofed your stream to ${applicationName}. Stream any window in vc for ${Math.ceil((secondsNeeded - secondsDone) / 60)} more minutes.`)
      console.log("Remember that you need at least 1 other person to be in the vc!")
   }
}
```
9. به مدت 15 دقیقه این کار رو انجام بدید
## Links
برای خرید نیترو بوست و آیتم های دیسکورد با پایین ترین قیمت در سرور زیر عضو شوید.
(https://discord.com/invite/EZGmusVR7W)
