module.exports = () => ({
  name: "Quest Autocompleter",
  description: "Automatically completes Discord quests and claims rewards",
  author: "Anonymous",
  version: "1.1.0",
  color: "#5865F2",
  icon: "medal",
  requiresRestart: false,

  onStart: async ({ webpack }) => {
    const {
      ApplicationStreamingStore,
      RunningGameStore,
      QuestsStore,
      ChannelStore,
      GuildChannelStore,
      FluxDispatcher,
      api
    } = await getStores(webpack);

    const isApp = typeof DiscordNative !== "undefined";

    async function completeQuest(quest) {
      try {
        const pid = Math.floor(Math.random() * 30000) + 1000;
        const applicationId = quest.config.application.id;
        const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
        const taskName = ["WATCH_VIDEO", "PLAY_ON_DESKTOP", "STREAM_ON_DESKTOP", "PLAY_ACTIVITY", "WATCH_VIDEO_ON_MOBILE"].find(x => taskConfig.tasks[x] != null);
        
        // Video watching tasks
        if (taskName.includes("WATCH_VIDEO")) {
          await completeVideoTask(quest, taskName);
        } 
        // Game playing tasks
        else if (taskName === "PLAY_ON_DESKTOP") {
          if (!isApp) return;
          await completePlayTask(quest, applicationId, pid, RunningGameStore, FluxDispatcher);
        } 
        // Streaming tasks
        else if (taskName === "STREAM_ON_DESKTOP") {
          if (!isApp) return;
          await completeStreamTask(quest, applicationId, pid, ApplicationStreamingStore, FluxDispatcher);
        } 
        // Voice activity tasks
        else if (taskName === "PLAY_ACTIVITY") {
          await completeVoiceTask(quest, ChannelStore, GuildChannelStore);
        }
      } catch (e) {
        console.error(`Failed to complete quest: ${e}`);
      }
    }

    async function run() {
      const activeQuests = [...QuestsStore.quests.values()].filter(quest =>
        quest.id !== "1412491570820812933" &&  // Nitro quest ID
        quest.userStatus?.enrolledAt &&
        !quest.userStatus?.completedAt &&
        new Date(quest.config.expiresAt).getTime() > Date.now()
      );

      if (!activeQuests.length) return;
      
      await Promise.all(activeQuests.map(completeQuest));
      showToast("Completed all available quests!");
    }

    // Run immediately and then every 6 hours
    await run();
    setInterval(run, 1000 * 60 * 60 * 6);
  }
});

async function getStores(webpack) {
  const wpRequire = webpackChunkdiscord_app.push([[Symbol()], {}, r => r]);
  webpackChunkdiscord_app.pop();

  return {
    ApplicationStreamingStore: Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getStreamerActiveStreamMetadata).exports.Z,
    RunningGameStore: Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getRunningGames).exports.ZP,
    QuestsStore: Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getQuest).exports.Z,
    ChannelStore: Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.getAllThreadsForParent).exports.Z,
    GuildChannelStore: Object.values(wpRequire.c).find(x => x?.exports?.ZP?.getSFWDefaultChannel).exports.ZP,
    FluxDispatcher: Object.values(wpRequire.c).find(x => x?.exports?.Z?.__proto__?.flushWaitQueue).exports.Z,
    api: Object.values(wpRequire.c).find(x => x?.exports?.tn?.get).exports.tn
  };
}

async function completeVideoTask(quest, taskName) {
  const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
  const secondsNeeded = taskConfig.tasks[taskName].target;
  let secondsDone = quest.userStatus?.progress?.[taskName]?.value ?? 0;
  
  const maxFuture = 15, speed = 30, interval = 200;
  const enrolledAt = new Date(quest.userStatus.enrolledAt).getTime();
  let completed = false;
  
  while (true) {
    const maxAllowed = Math.floor((Date.now() - enrolledAt) / 1000) + maxFuture;
    const diff = maxAllowed - secondsDone;
    const timestamp = secondsDone + speed;
    
    if (diff >= speed) {
      const res = await api.post({ 
        url: `/quests/${quest.id}/video-progress`, 
        body: { timestamp: Math.min(secondsNeeded, timestamp + Math.random() * 5) } 
      });
      completed = res.body.completed_at != null;
      secondsDone = Math.min(secondsNeeded, timestamp);
    }
    
    if (timestamp >= secondsNeeded || completed) break;
    await sleep(interval);
  }
  
  if (!completed) {
    await api.post({ 
      url: `/quests/${quest.id}/video-progress`, 
      body: { timestamp: secondsNeeded } 
    });
  }
}

async function completePlayTask(quest, appId, pid, RunningGameStore, FluxDispatcher) {
  const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
  const secondsNeeded = taskConfig.tasks.PLAY_ON_DESKTOP.target;
  
  const appDataRes = await api.get({ url: `/applications/public?application_ids=${appId}` });
  const appData = appDataRes.body[0];
  const exeName = appData.executables.find(x => x.os === "win32").name.replace(">", "");
  
  const fakeGame = {
    cmdLine: `C:\Program Files\${appData.name}\${exeName}`,
    exeName,
    exePath: `c:/program files/${appData.name.toLowerCase()}/${exeName}`,
    hidden: false,
    isLauncher: false,
    id: appId,
    name: appData.name,
    pid: pid,
    pidPath: [pid],
    processName: appData.name,
    start: Date.now(),
  };
  
  const realGames = RunningGameStore.getRunningGames();
  const realGetRunningGames = RunningGameStore.getRunningGames;
  const realGetGameForPID = RunningGameStore.getGameForPID;
  
  RunningGameStore.getRunningGames = () => [fakeGame];
  RunningGameStore.getGameForPID = (pid) => [fakeGame].find(x => x.pid === pid);
  FluxDispatcher.dispatch({ 
    type: "RUNNING_GAMES_CHANGE", 
    removed: realGames, 
    added: [fakeGame], 
    games: [fakeGame] 
  });
  
  await new Promise(resolve => {
    const completeHandler = data => {
      const progress = quest.config.configVersion === 1 
        ? data.userStatus.streamProgressSeconds 
        : Math.floor(data.userStatus.progress.PLAY_ON_DESKTOP.value);
      
      if (progress >= secondsNeeded) {
        RunningGameStore.getRunningGames = realGetRunningGames;
        RunningGameStore.getGameForPID = realGetGameForPID;
        FluxDispatcher.dispatch({ 
          type: "RUNNING_GAMES_CHANGE", 
          removed: [fakeGame], 
          added: [], 
          games: [] 
        });
        FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", completeHandler);
        resolve();
      }
    };
    
    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", completeHandler);
  });
}

async function completeStreamTask(quest, appId, pid, ApplicationStreamingStore, FluxDispatcher) {
  const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
  const secondsNeeded = taskConfig.tasks.STREAM_ON_DESKTOP.target;
  
  const realFunc = ApplicationStreamingStore.getStreamerActiveStreamMetadata;
  ApplicationStreamingStore.getStreamerActiveStreamMetadata = () => ({
    id: appId, pid, sourceName: null
  });
  
  await new Promise(resolve => {
    const completeHandler = data => {
      const progress = quest.config.configVersion === 1 
        ? data.userStatus.streamProgressSeconds 
        : Math.floor(data.userStatus.progress.STREAM_ON_DESKTOP.value);
      
      if (progress >= secondsNeeded) {
        ApplicationStreamingStore.getStreamerActiveStreamMetadata = realFunc;
        FluxDispatcher.unsubscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", completeHandler);
        resolve();
      }
    };
    
    FluxDispatcher.subscribe("QUESTS_SEND_HEARTBEAT_SUCCESS", completeHandler);
  });
}

async function completeVoiceTask(quest, ChannelStore, GuildChannelStore) {
  const taskConfig = quest.config.taskConfig ?? quest.config.taskConfigV2;
  const secondsNeeded = taskConfig.tasks.PLAY_ACTIVITY.target;
  
  const channelId = ChannelStore.getSortedPrivateChannels()[0]?.id ?? 
    Object.values(GuildChannelStore.getAllGuilds())
      .find(x => x != null && x.VOCAL.length > 0).VOCAL[0].channel.id;
  
  const streamKey = `call:${channelId}:1`;
  
  while (true) {
    try {
      const res = await api.post({ 
        url: `/quests/${quest.id}/heartbeat`, 
        body: { stream_key: streamKey, terminal: false } 
      });
      
      const progress = res.body.progress.PLAY_ACTIVITY.value;
      if (progress >= secondsNeeded) {
        await api.post({ 
          url: `/quests/${quest.id}/heartbeat`, 
          body: { stream_key: streamKey, terminal: true } 
        });
        break;
      }
    } catch {
      break;
    }
    await sleep(3000);
  }
}

function sleep(ms) {
  return new Promise(resolve => setTimeout(resolve, ms));
}

function showToast(message) {
  FluxDispatcher.dispatch({
    type: "SHOW_TOAST",
    toast: {
      id: `quest-complete-${Date.now()}`,
      message,
      type: "SUCCESS",
      options: { duration: 3000 }
    }
  });
}
