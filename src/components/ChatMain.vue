<template>
  <v-app>
    <!-- 사이드 바 -->
    <v-navigation-drawer v-model="drawer" style="background: #F9F9F9 ">
      <div style="display: flex; justify-content: flex-end;">
        <button @click="addChatTab()">
          <v-icon icon="mdi-chat-plus" class="chat-plus"></v-icon>
        </button>
      </div>
      <div>
        <div
          v-for="(chatList, index) in data.chatLists"
          :key="index"
          class="ma-2"
        >
          <div class="chat-list-container" :class='index == activeTab ? "active-tab" : ""' @click="changeChatTab(index)">
            <v-icon icon="mdi-chat-processing" class="ma-3"></v-icon>
            <span class="chat-title">{{ chatList.name }}</span>
            <button @click="removeChatTab(index)" class="remove-button mr-3" style="margin-left: auto;">
              <v-icon icon="mdi-chat-minus" class="chat-minus"></v-icon>
            </button>
          </div>
        </div>
      </div>
    </v-navigation-drawer>

    <!-- 헤더 -->
    <v-app-bar flat style="background: #517970;">
      <v-app-bar-nav-icon @click="drawer = !drawer" style="color: white"></v-app-bar-nav-icon>
      <span class="header-title">Rest In MInd</span>
    </v-app-bar>

    <!-- 메인 콘텐츠 -->
    <v-main class="d-flex flex-column" style="height: 100vh; width: 100vw;">
      <v-container class="py-8 px-6 d-flex flex-column flex-grow-1" style="width: 100vmin" fluid>
        <!-- 채팅 창 -->
        <div v-if='data.chatLists.length == 0' class="chat-empty-container flex-grow-1 mb-4">
          <div style="text-align: center">
            <img src="./images/speech-bubble.png" style="width: 70px;">
            <br><br>
            새로운 채팅 시작하기
          </div>
        </div>
        <div v-else class="chat-container flex-grow-1 overflow-auto mb-4" ref="chatContainer">
          <div
            v-for="chatMessage in data.chatMessageList[activeTab]"
            :key="chatMessage.chat"
            class="ma-2"
            :class="chatMessage.talker === 'user' ? 'justify-end' : 'justify-start'"
            style="display: flex;"
          >
            <div 
              class="pa-4 chat-bubble"
              :class="chatMessage.talker === 'user' ? 'chat-user' : 'chat-bot'"
            >
              {{ chatMessage.chat }}
            </div>
          </div>
            <div v-if="isbusy"
              class="pa-4 chat-bubble chat-bot"
              style="width: 70px"
            >
              <v-progress-circular
                indeterminate
                color="#517970"
              ></v-progress-circular>
            </div>

        </div>
        
        <!-- 채팅 입력 창 -->
        <div xs12>
          <v-text-field
            v-model="param.chat"
            clear-icon="mdi-close-circle"
            outline
            clearable
            label="메세지를 입력하세요."
            type="text"
            @click:clear="clearMessage"
            @keyup.enter="sendMessage"
          >
            <template #append>
              <v-icon
                :class="{'active-send': param.chat !== '', 'inactive-send': param.chat === ''}"
                :disabled='param.chat == ""'
                @click="sendMessage"
              >
                mdi-send
              </v-icon>
            </template>
          </v-text-field>
        </div>
      </v-container>
    </v-main>
  </v-app>
</template>

<script setup>
import { reactive, ref, onMounted, watch, nextTick } from 'vue'
import axios from 'axios';

axios.defaults.headers.common['Access-Control-Allow-Origin'] = '*';
axios.defaults.headers.get['Content-Type'] = 'application/json;charset=utf-8';

// chatContainer를 ref로 선언
const chatContainer = ref(null);
const drawer = ref(true)
const isbusy = ref(false)
const activeTab = ref(0)
const data = reactive({
  chatLists: [], //{name:'새로운 채팅 '+ (data.chatLists.length+1), idx: data.chatLists.length, thread_id}
  chatMessageList: [
    [], 
    
  ],
  links: [
    ['mdi-inbox-arrow-down', 'Inbox'],
    ['mdi-send', 'Send'],
    ['mdi-delete', 'Trash'],
    ['mdi-alert-octagon', 'Spam'],
  ],
})

const param = reactive({
  chat: '',       // 채팅 내용
  thread_id: '',   // 채팅방 idx
})

// 모바일 기기 체크
function isMobile() {
    var userAgent = navigator.userAgent;
    var mobile = /(iPhone|iPad|Android|BlackBerry|Windows Phone)/i.test(userAgent);
    return mobile;
  }

// 스크롤을 하단으로 이동시키는 함수
const scrollToBottom = () => {
  const container = chatContainer.value;
  if (container) {
    container.scrollTop = container.scrollHeight;
  }
};

// 컴포넌트가 마운트되었을 때 스크롤을 하단으로 이동
onMounted(() => {
  scrollToBottom();
  if (isMobile()) {
      drawer.value = false;
    } else {
      drawer.value = true;
    }
});

// 데이터가 변경될 때마다 스크롤을 하단으로 이동
watch(data.chatMessageList, () => {
  nextTick(() => {
    scrollToBottom();
  });
});


// 채팅 탭 변경
const changeChatTab = (index) => {
  // activeTab=index;
  activeTab.value = index;
}

// 채팅 탭 추가
const addChatTab = () => {
  data.chatLists.push({name:'새로운 채팅 '+ (data.chatLists.length+1), idx: data.chatLists.length, thread_id: '' })
  data.chatMessageList.push (
    [],
  )
  activeTab.value = data.chatLists.length -1;
}

// 채팅 탭 삭제
const removeChatTab = (index) => {
  data.chatLists.splice(index, 1);
  data.chatMessageList.splice(index, 1);
}

// 메세지 보내기
const sendMessage = () => {
  if(param.chat != ''){
    // 채팅방이 하나도 없을 경우 새로운 채팅방 생성
    if(data.chatLists.length == 0) {
      addChatTab();
    }
    param.thread_id = data.chatLists[activeTab.value].thread_id
    data.chatMessageList[activeTab.value].push(
      {chatIdx:data.chatMessageList+1, talker:'user', chat: param.chat}
    )

    // api호출
    isbusy.value = true;
    sendChatAPI();
  
    clearMessage();
    scrollToBottom()
  }
}

// post 호출
const sendChatAPI = async() => {
  let responseMsg = '';
  let _thread_id = '';
  try {
    const response = await axios.post('http://150.230.255.221:3031/sendChat', param);
    responseMsg = response.data[0]
    _thread_id = response.data[1]
    // if(responseMsg == 'ERROR : Connection refused') {
    //   responseMsg = '서버와의 연결이 끊겼습니다. 잠시후 다시 시도해주세요.'
    // }
  } catch (error) {
    console.error('Error:', error);
    responseMsg = '서버와의 연결이 끊겼습니다. 잠시후 다시 시도해주세요.'
  }

  data.chatLists[activeTab.value].thread_id = _thread_id;
  data.chatMessageList[activeTab.value].push(
    {chatIdx:data.chatMessageList+1, talker:'chatbot', chat: responseMsg}
  );
  isbusy.value = false;
}

// 메세지 초기화
const clearMessage = () => {
  param.chat = '';
}

</script>

<style scoped>
  .chat-container {
    max-height: calc(100vh - 200px); /* 화면 높이에서 채팅 입력 창 높이와 다른 여백을 뺀 값 */
    overflow-y: auto;
  }
  .chat-empty-container {
    max-height: calc(100vh - 200px); /* 화면 높이에서 채팅 입력 창 높이와 다른 여백을 뺀 값 */
    overflow-y: auto;
    display: flex; 
    justify-content: center; 
    align-items: center;
  }
  .chat-list-container {
    display: flex;
    align-items: center;
    height: 45px;
    border-radius: 5px;
    cursor: pointer;
  }
  .chat-title {
    font-size: medium;
  }

  .justify-end {
    justify-content: flex-end;
  }

  .justify-start {
    justify-content: flex-start;
  }
  .chat-plus{
    width: 65px; 
    height: 65px; 
    color: #517970
  }
  .chat-minus {
    color: white; /* 원하는 색상으로 변경 */
  }
  .remove-button:hover .chat-minus {
    color: #2F3E46; /* 원하는 색상으로 변경 */
  }
  .active-tab {
    background: #517970; 
    color: white
  }
  .chat-bubble {
    max-width: 60%;
    /* border-radius: 30px; */
  }
  .chat-user {
    background: #517970;
    /* opacity : 0.9; */
    color: white;
    border-top-left-radius: 30px;
    border-top-right-radius: 30px;
    border-bottom-left-radius: 30px;
  }
  .chat-bot {
    background: #F0F0F0;
    border-top-right-radius: 30px;
    border-bottom-left-radius: 30px;
    border-bottom-right-radius: 30px;
  }
  .active-send {
    color: #517970; /* 원하는 색상으로 변경 */
    transform: rotate(-45deg);
    transition: transform 0.3s ease; /* 애니메이션을 추가하여 부드럽게 회전 */
  }

  .inactive-send {
    color: gray; /* 기본 색상 */
    transition: transform 0.3s ease; /* 상태 전환 시 애니메이션 */
  }

  @font-face {
    font-family: 'HancomMalangMalang-Regular';
    src: url('https://fastly.jsdelivr.net/gh/projectnoonnu/2406-1@1.0/HancomMalangMalang-Regular.woff2') format('woff2');
    font-weight: 400;
    font-style: normal;
  }

  .header-title{
    font-family: 'HancomMalangMalang-Regular';
    color: white;
    font-weight: bold;
    right: 20px;
    position: absolute;
  }
</style>