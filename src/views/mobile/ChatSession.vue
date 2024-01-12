<template>
  <div class="bg"></div>
  <van-config-provider :theme="getMobileTheme()">
    <div class="mobile-chat" v-loading="loading" element-loading-text="正在连接会话...">
      <van-sticky ref="navBarRef" :offset-top="0" position="top">
        <van-nav-bar left-arrow left-text="返回" @click-left="router.back()">
          <template #title>
            <van-dropdown-menu>
              <van-dropdown-item :title="title">
                <van-cell center title="角色"> {{ role.name }}</van-cell>
                <van-cell center title="模型">{{ modelValue }}</van-cell>
              </van-dropdown-item>
            </van-dropdown-menu>
          </template>

          <!--          <template #right>-->
          <!--            <van-icon name="share-o" @click="showShare = true"/>-->
          <!--          </template>-->
        </van-nav-bar>
      </van-sticky>

      <!--      <van-share-sheet-->
      <!--          v-model:show="showShare"-->
      <!--          title="立即分享给好友"-->
      <!--          :options="shareOptions"-->
      <!--          @select="shareChat"-->
      <!--      />-->

      <div id="message-list-box" :style="{height: winHeight+'px'}" class="message-list-box">
        <van-list
            v-model:error="error"
            :finished="finished"
            error-text="请求失败，点击重新加载"
            @load="onLoad"
        >
          <van-cell v-for="item in chatData" :key="item" :border="false" class="message-line">
            <chat-prompt
                v-if="item.type==='prompt'"
                :content="item.content"
                :created-at="dateFormat(item['created_at'])"
                :icon="item.icon"
                :model="model"
                :tokens="item['tokens']"/>
            <chat-reply v-else-if="item.type==='reply'"
                        :content="item.content"
                        :created-at="dateFormat(item['created_at'])"
                        :icon="item.icon"
                        :org-content="item.orgContent"
                        :tokens="item['tokens']"/>
            <chat-mid-journey v-else-if="item.type==='mj'"
                              :content="item.content"
                              :icon="item.icon"
                              :role-id="role"
                              :chat-id="chatId"
                              @disable-input="disableInput(true)"
                              @enable-input="enableInput"
                              :created-at="dateFormat(item['created_at'])"/>
          </van-cell>
        </van-list>
      </div>

      <van-sticky ref="bottomBarRef" :offset-bottom="0" position="bottom">
        <div class="chat-box">
          <van-cell-group>
            <van-field
                v-model="prompt"
                center
                clearable
                placeholder="输入你的问题"
            >
              <template #button>
                <van-button size="small" type="primary" @click="sendMessage">发送</van-button>
              </template>
              <!--              <template #extra>-->
              <!--                <div class="icon-box">-->
              <!--                  <van-icon v-if="showStopGenerate" name="stop-circle-o" @click="stopGenerate"/>-->
              <!--                  <van-icon v-if="showReGenerate" name="play-circle-o" @click="reGenerate"/>-->
              <!--                </div>-->
              <!--              </template>-->
            </van-field>
          </van-cell-group>
        </div>
      </van-sticky>
    </div>
  </van-config-provider>

</template>

<script setup>
import {nextTick, onMounted, ref} from "vue";
import {showToast} from "vant";
import {useRouter} from "vue-router";
import {dateFormat, randString, renderInputText, UUID} from "@/utils/libs";
import {getChatConfig} from "@/store/chat";
import {httpGet} from "@/utils/http";
import hl from "highlight.js";
import 'highlight.js/styles/a11y-dark.css'
import ChatPrompt from "@/components/mobile/ChatPrompt.vue";
import ChatReply from "@/components/mobile/ChatReply.vue";
import {getSessionId, getUserToken} from "@/store/session";
import {checkSession} from "@/action/session";
import {getMobileTheme} from "@/store/system";
import ChatMidJourney from "@/components/mobile/ChatMidJourney.vue";

const winHeight = ref(0)
const navBarRef = ref(null)
const bottomBarRef = ref(null)
const router = useRouter()

const chatConfig = getChatConfig()
const role = chatConfig.role
const model = chatConfig.model
const modelValue = chatConfig.modelValue
const title = chatConfig.title
const chatId = chatConfig.chatId
const loginUser = ref(null)

onMounted(() => {
  winHeight.value = document.body.offsetHeight - navBarRef.value.$el.offsetHeight - bottomBarRef.value.$el.offsetHeight
})

const chatData = ref([])
const loading = ref(false)
const finished = ref(false)
const error = ref(false)

checkSession().then(user => {
  loginUser.value = user
}).catch(() => {
  router.push('/login')
})

const onLoad = () => {
  httpGet('/api/chat/history?chat_id=' + chatId).then(res => {
    // 加载状态结束
    finished.value = true;
    const data = res.data
    if (data && data.length > 0) {
      const md = require('markdown-it')({breaks: true});
      for (let i = 0; i < data.length; i++) {
        if (data[i].type === "prompt") {
          chatData.value.push(data[i]);
          continue;
        } else if (data[i].type === "mj") {
          data[i].content = JSON.parse(data[i].content)
          data[i].content.html = md.render(data[i].content?.content)
          chatData.value.push(data[i]);
          continue;
        }

        data[i].orgContent = data[i].content;
        data[i].content = md.render(data[i].content);
        chatData.value.push(data[i]);
      }

      nextTick(() => {
        hl.configure({ignoreUnescapedHTML: true})
        const blocks = document.querySelector("#message-list-box").querySelectorAll('pre code');
        blocks.forEach((block) => {
          hl.highlightElement(block)
        })

        scrollListBox()
      })
    }
  }).catch(() => {
    error.value = true
  })

};

// 创建 socket 连接
const prompt = ref('');
const showStopGenerate = ref(false); // 停止生成
const showReGenerate = ref(false); // 重新生成
const previousText = ref(''); // 上一次提问
const lineBuffer = ref(''); // 输出缓冲行
const socket = ref(null);
const activelyClose = ref(false); // 主动关闭
const canSend = ref(true);

const disableInput = (force) => {
  canSend.value = false;
  showReGenerate.value = false;
  showStopGenerate.value = !force;
}

const enableInput = () => {
  canSend.value = true;
  showReGenerate.value = previousText.value !== "";
  showStopGenerate.value = false;
}

// 将聊天框的滚动条滑动到最底部
const scrollListBox = () => {
  document.getElementById('message-list-box').scrollTo(0, document.getElementById('message-list-box').scrollHeight + 46)
}

const sendMessage = () => {
  if (canSend.value === false) {
    showToast("AI 正在作答中，请稍后...");
    return
  }

  if (prompt.value.trim().length === 0) {
    return false;
  }

  // 追加消息
  chatData.value.push({
    type: "prompt",
    id: randString(32),
    icon: loginUser.value.avatar,
    content: renderInputText(prompt.value),
    created_at: new Date().getTime(),
  });

  nextTick(() => {
    scrollListBox()
  })

  disableInput(false)
  previousText.value = prompt.value;
  // 创建一个EventSource实例来连接到服务器
  let eventSource = new EventSource(process.env.VUE_APP_API_HOST + "/gpt/streamChatNew?msg=" + prompt.value
      + "&chat_id=" + chatId
      + "&model=" + chatConfig.model
      + "&token=" + getUserToken());
// 监听消息事件
  eventSource.onmessage = function (event) {
    // 处理接收到的数据
    let data = JSON.parse(event.data);
    let message = data.msg; // 提取msg字段
    let msgType = data.type // 提取type字段
    if (msgType === 'stop') {
      eventSource.close(); // 关闭EventSource
      enableInput();
      return;
    }
    const md = require('markdown-it')({breaks: true});
    let reply;

    if (chatData.value.length > 0 && chatData.value[chatData.value.length - 1].type === "reply") {
      // 更新最后一个回复条目
      reply = chatData.value[chatData.value.length - 1];
      reply.orgContent += message;
      reply.content = md.render(reply.orgContent);
    } else {
      // 创建新的回复条目
      reply = {
        type: "reply",
        id: randString(32),
        icon: "/images/avatar/gpt.png",
        orgContent: message,
        content: md.render(message),

      };
      chatData.value.push(reply);
    }

    // 确保UI更新（视你的框架/库而定）
    nextTick(() => {
      scrollListBox();
      enableInput();
    });
  };

// 监听错误事件
  eventSource.onerror = function (error) {
    console.error("EventSource failed:", error);
    eventSource.close(); // 关闭EventSource
    enableInput();
  };

// 记得在合适的时候关闭EventSource
// 例如，在组件销毁时


  prompt.value = '';

  return true;
}

function scrollToBottom() {
  var contentBox = document.querySelector('.content-box');
  contentBox.scrollTop = contentBox.scrollHeight;
}

const stopGenerate = () => {
  showStopGenerate.value = false;
  httpGet("/api/chat/stop?session_id=" + getSessionId()).then(() => {
    enableInput()
  })
}

const reGenerate = () => {
  disableInput(false)
  const text = '重新生成上述问题的答案：' + previousText.value;
  // 追加消息
  chatData.value.push({
    type: "prompt",
    id: randString(32),
    icon: loginUser.value.avatar,
    content: renderInputText(text)
  });
  socket.value.send(text);
}

const showShare = ref(false)
const shareOptions = [
  {name: '微信', icon: 'wechat'},
  {name: '微博', icon: 'weibo'},
  {name: '复制链接', icon: 'link'},
  {name: '分享海报', icon: 'poster'},
]
const shareChat = () => {
  showShare.value = false
  showToast('功能待开发')
}
</script>

<style lang="stylus" scoped>
.bg {
  position fixed
  left 0
  right 0
  top 0
  bottom 0
  background-color #091519
  background-image url("~@/assets/img/back5.jpg")
  background-size cover
  background-position center
  background-repeat no-repeat
  //filter: blur(10px); /* 调整模糊程度，可以根据需要修改值 */
}

.mobile-chat {
  .message-list-box {
    padding-top 50px
    padding-bottom 10px
    overflow-x auto
    background #F5F5F5;

    .van-cell {
      background none
      font-family: 'Microsoft YaHei', '微软雅黑', Arial, sans-serif;
    }
  }

  .chat-box {
    .icon-box {
      .van-icon {
        font-size 24px
        margin-left 10px;
      }
    }
  }
}

.van-theme-dark {
  .mobile-chat {
    .message-list-box {
      background #232425;
    }
  }
}
</style>

<style lang="stylus">
.mobile-chat {
  .van-nav-bar__title {
    .van-dropdown-menu__title {
      margin-right 10px
    }

    .van-cell__title {
      text-align left
    }
  }

  .van-nav-bar__right {
    .van-icon {
      font-size 20px
    }
  }
}
</style>