<!--export default class CallState {-->
<!--static STATUS_IDLE = 0;-->
<!--static STATUS_OUTGOING = 1;-->
<!--static STATUS_INCOMING = 2;-->
<!--static STATUS_CONNECTING = 3;-->
<!--static STATUS_CONNECTED = 4;-->
<!--}-->
<template>
  <div class="flex-column flex-align-center flex-justify-center">
    <h1 style="display: none">Voip-Multi 运行在新的window，和主窗口数据是隔离的！！</h1>

    <div v-if="session" class="container">
      <section>
        <!--audio-->
        <div class="content-container">
          <!--self-->
          <div class="participant-container">
            <div v-if="audioOnly || status !== 4 || !selfUserInfo._stream"
                 class="flex-column flex-justify-center flex-align-center">
              <img class="avatar" :src="selfUserInfo.portrait">
              <video v-if="audioOnly && selfUserInfo._stream"
                     class="hidden-video"
                     :srcObject.prop="selfUserInfo._stream"
                     playsInline autoPlay/>
              <p>self</p>
            </div>
            <video v-else
                   class="video"
                   ref="localVideo"
                   :srcObject.prop="selfUserInfo._stream"
                   playsInline
                   autoPlay/>
          </div>

          <!--participants-->
          <div v-for="(participant) in participantUserInfos" :key="participant.uid"
               class="participant-container">
            <div v-if="audioOnly || status !== 4 || !participant._stream"
                 class="flex-column flex-justify-center flex-align-center">
              <img class="avatar" :src="participant.portrait" :alt="participant">
              <video v-if="audioOnly && participant._stream"
                     class="hidden-video"
                     :srcObject.prop="participant._stream"
                     playsInline autoPlay/>
              <p class="single-line">{{ userName(participant) }}</p>
            </div>
            <video v-else
                   class="video"
                   :srcObject.prop="participant._stream"
                   playsInline
                   autoPlay/>
          </div>
          <!--add more-->
          <!--通话建立成功之后，才允许邀请新参与者-->
          <div v-if="status === 4/*connect*/ && participantUserInfos.length < 8" class="participant-container">
            <img @click="invite" class="avatar" src="@/assets/images/add.png">
          </div>
        </div>
      </section>

      <!--actions-->
      <footer>
        <!--incoming-->
        <div v-if="status === 2" class="action-container">
          <div class="action">
            <img @click="hangup" class="action-img" src='@/assets/images/av_hang_up.png'/>
          </div>
          <div class="action">
            <img @click="answer" class="action-img" src='@/assets/images/av_video_answer.png'/>
          </div>
          <div v-if="!audioOnly" class="action">
            <img @click="down2voice" class="action-img" src='@/assets/images/av_float_audio.png'/>
            <p>切换到语音聊天</p>
          </div>
        </div>
        <!--outgoing-->
        <div v-if="status === 1 || status === 3" class="action-container">
          <div class="action">
            <img @click="hangup" class="action-img" src='@/assets/images/av_hang_up.png'/>
          </div>
        </div>

        <!--connected-->
        <div v-if="status === 4" class="duration-action-container">
          <p>{{ duration }}</p>
          <div class="action-container">

            <div class="action">
              <img @click="hangup" class="action-img" src='@/assets/images/av_hang_up.png'/>
            </div>
            <div v-if="!audioOnly" class="action">
              <img @click="screenShare" class="action-img" src='@/assets/images/av_share.png'/>
            </div>
          </div>
        </div>
      </footer>
    </div>
  </div>
</template>

<script>
import avenginekit from "../../wfc/av/internal/engine.min";
import CallSessionCallback from "../../wfc/av/engine/CallSessionCallback";
import PickUserView from "@/ui/main/pick/PickUserView";
import CallState from "@/wfc/av/engine/callState";

export default {
  name: 'Multi',
  data() {
    return {
      session: null,
      audioOnly: false,
      muted: false,
      status: 1,
      selfUserInfo: null,
      initiatorUserInfo: null,
      participantUserInfos: [],
      groupMemberUserInfos: [],

      startTimestamp: 0,
      currentTimestamp: 0,
    }
  },
  methods: {
    setupSessionCallback() {
      let sessionCallback = new CallSessionCallback();

      sessionCallback.didChangeState = (state) => {
        this.status = state;
        if (state === CallState.STATUS_CONNECTED) {
          if (this.startTimestamp === 0) {
            this.startTimestamp = new Date().getTime();
            this.timer = setInterval(() => {
              this.currentTimestamp = new Date().getTime();
            }, 1000)
          }
        } else if (state === CallState.STATUS_IDLE) {
          if (this.timer) {
            clearInterval(this.timer);
          }
        }
      };

      sessionCallback.onInitial = (session, selfUserInfo, initiatorUserInfo, participantUserInfos, groupMemberUserInfos) => {
        this.session = session;


        this.audioOnly = session.audioOnly;
        this.selfUserInfo = selfUserInfo;
        this.initiatorUserInfo = initiatorUserInfo;
        // 为了逻辑更清晰，参数引用传递，参数中传入的participantUserInfos会变化，如果直接使用的话，didParticipantJoined里面，可啥都不做
        this.participantUserInfos = [...participantUserInfos];
        this.groupMemberUserInfos = groupMemberUserInfos;

        // pls refer to: https://vuejs.org/v2/guide/reactivity.html
        this.$set(this.selfUserInfo, '_stream', null)
        participantUserInfos.forEach(p => this.$set(p, "_stream", null))
        groupMemberUserInfos.forEach(m => this.$set(m, "_stream", null))

      };

      sessionCallback.didChangeMode = (audioOnly) => {
        this.audioOnly = audioOnly;
      };

      sessionCallback.didCreateLocalVideoTrack = (stream) => {
        this.selfUserInfo._stream = stream;
      };

      sessionCallback.didReceiveRemoteVideoTrack = (userId, stream) => {
        let p;
        for (let i = 0; i < this.participantUserInfos.length; i++) {
          p = this.participantUserInfos[i];
          if (p.uid === userId) {
            p._stream = stream;
            break;
          }
        }
      };

      sessionCallback.didParticipantJoined = (userId, userInfo) => {
        console.log('didParticipantJoined', userId)
        userInfo._stream = null;
        this.participantUserInfos.push(userInfo);
      }

      sessionCallback.didParticipantLeft = (userId) => {
        console.log('didParticipantLeft', userId, this.participantUserInfos.length)
        this.participantUserInfos = this.participantUserInfos.filter(p => p.uid !== userId);
        console.log('didParticipantLeft d', userId, this.participantUserInfos.length)
      }

      sessionCallback.didCallEndWithReason = (reason) => {
        console.log('callEndWithReason', reason)
      }

      sessionCallback.didVideoMuted = (userId, muted) => {
        this.muted = muted;
      };

      avenginekit.sessionCallback = sessionCallback;
    },

    answer() {
      this.session.call();
    },

    hangup() {
      this.session.hangup();
    },
    down2voice() {
      this.session.downgrade2Voice();
    },
    screenShare() {
      this.session.isScreenSharing() ? this.session.stopScreenShare() : this.session.startScreenShare();
    },

    invite() {
      let beforeClose = (event) => {
        let users = event.params.users;
        let userIds = users.map(u => u.uid);
        this.session.inviteNewParticipants(userIds);
      };
      this.$modal.show(
          PickUserView,
          {
            users: this.session.groupMemberUserInfos,
            initialCheckedUsers: [...this.session.participantUserInfos, this.session.selfUserInfo],
            uncheckableUsers: [...this.session.participantUserInfos, this.session.selfUserInfo],
            showCategoryLabel: false,
            confirmTitle: '确定',
          }, {
            name: 'pick-user-modal',
            width: 600,
            height: 480,
            clickToClose: false,
          }, {
            // 'before-open': this.beforeOpen,
            'before-close': beforeClose,
            'closed': this.closed,
          })
    },

    userName(user) {
      if (user.groupAlias) {
        name = user.groupAlias;
      } else if (user.friendAlias) {
        name = user.friendAlias;
      } else if (user.displayName) {
        name = user.displayName;
      } else {
        name = user.name;
      }
      return name;
    },

    timestampFormat(timestamp) {
      timestamp = ~~(timestamp / 1000);
      let str = ''
      let hour = ~~(timestamp / 3600);
      str = hour > 0 ? ((hour < 10 ? "0" : "") + hour + ':') : '';
      let min = ~~((timestamp % 3600) / 60);
      str += (min < 10 ? "0" : "") + min + ':'
      let sec = ~~((timestamp % 60));
      str += (sec < 10 ? "0" : "") + sec
      return str;
    }
  },

  computed: {
    duration() {
      if (this.currentTimestamp <= 0) {
        return '00:00'
      }
      let escapeMillis = this.currentTimestamp - this.startTimestamp;
      return this.timestampFormat(escapeMillis)
    }
  },

  mounted() {
    avenginekit.setup();
    this.setupSessionCallback();
  },

  destroyed() {
    // reset
    this.$set(this.selfUserInfo, '_stream', null)
    groupMemberUserInfos.forEach(m => this.$set(m, "_stream", null))
  }
}
</script>

<style lang="css" scoped>

.container {
  width: 100vw;
  height: 100vh;
  display: flex;
  flex-direction: column;
  justify-content: space-between;
}

.content-container {
  width: 100%;
  position: relative;
  display: flex;
  flex-wrap: wrap;
  justify-content: center;
  align-content: flex-start;
}

.participant-container {
  display: flex;
  width: 200px;
  height: 220px;
  /*background-color: rebeccapurple;*/

  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.participant-container > video {
  max-width: 100%;
  max-height: 100%;
  width: 100%;
  height: 100%;
  object-fit: cover;
}

.hidden-video {
  height: 0;
}

.participant-container p {
  max-height: 20px;
  color: white;
}

.duration-action-container {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
}

.duration-action-container p{
  color: white;
  padding: 10px 0;
}

.action-container {
  width: 100%;
  bottom: 0;
  left: 0;
  display: flex;
  justify-content: space-around;
  padding-bottom: 20px;
}

.action-container .action {
  flex: 1;
  display: flex;
  flex-direction: column;
  align-items: center;
  font-size: 12px;
}

.avatar {
  width: 200px;
  height: 200px;
}

.action-img {
  width: 60px;
  height: 60px;
}
</style>
