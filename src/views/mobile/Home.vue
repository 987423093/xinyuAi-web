<template>
  <van-config-provider :theme="getMobileTheme()">
    <div class="mobile-home">
      <router-view/>

      <van-tabbar route v-model="active" @change="onChange">
<!--        <van-tabbar-item to="/mobile/chat/list" name="home" icon="chat-o"></van-tabbar-item>-->
<!--        <van-tabbar-item to="/mobile/setting" name="setting" icon="setting-o"></van-tabbar-item>-->
<!--        <van-tabbar-item to="/mobile/profile" name="profile" icon="user-o"></van-tabbar-item>-->
      </van-tabbar>

    </div>
  </van-config-provider>

</template>

<script setup>
import {ref} from "vue";
import {getMobileTheme} from "@/store/system";
import {useRouter} from "vue-router";
import {isMobile} from "@/utils/libs";
import {checkSession} from "@/action/session";

const router = useRouter()
checkSession().then(() => {
  // if (!isMobile()) {
  //   router.replace('/chat')
  // }
  router.push("/mobile/chat/list")
}).catch(() => {
  router.push('/login')
})

const active = ref('home')
const onChange = (index) => {
  console.log(index)
  // showToast(`标签 ${index}`);
}

</script>

<style lang="stylus">
.mobile-home {
  .container {
    .van-nav-bar {
      position fixed
      width 100%
    }

    .content {
      padding 46px 10px 0 10px;
    }
  }

}

// 黑色主题
.van-theme-dark body {
  background #1c1c1e
}

.van-toast--fail {
  background #fef0f0
  color #f56c6c
}

.van-nav-bar {
  position fixed
  width 100%
}
</style>