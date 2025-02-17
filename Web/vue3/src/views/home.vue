<template>
  <div class="home-container">
    <div class="header">
      <user-info
        class="header-item user-info"
        :user-id="userId"
        :user-name="userName"
        :avatar-url="avatarUrl"
        @log-out="handleLogOut"
      ></user-info>
      <language-icon class="header-item language"></language-icon>
    </div>
    <stream-preview ref="streamPreviewRef"></stream-preview>
    <room-control
      :given-room-id="givenRoomId"
      @create-room="handleCreateRoom"
      @enter-room="handleEnterRoom"
    ></room-control>
  </div>
</template>

<script setup lang="ts">
import UserInfo from '@TUIRoom/components/RoomHeader/UserInfo.vue';
import StreamPreview from '@TUIRoom/components/RoomHome/StreamPreview.vue';
import RoomControl from '@TUIRoom/components/RoomHome/RoomControl.vue';
import LanguageIcon from '@/TUIRoom/components/RoomHeader/Language.vue';
import { checkNumber } from '@/TUIRoom/utils/common';
import router from '@/router';
import { useRoute } from 'vue-router';
import { onMounted, Ref, ref } from 'vue';
import { getBasicInfo } from '@/config/basic-info-config';
import { useI18n } from 'vue-i18n';
import TUIRoomEngine from '@tencentcloud/tuiroom-engine-js';
import useGetRoomEngine from '../TUIRoom/hooks/useRoomEngine';

const route = useRoute();
const streamPreviewRef = ref();
const userName = ref();
const avatarUrl = ref();
const userId = ref();
const { t } = useI18n();
const roomEngine = useGetRoomEngine();

const roomId = checkNumber((route.query.roomId) as string) ? route.query.roomId : '';
const givenRoomId: Ref<string> = ref((roomId) as string);

const basicInfo = getBasicInfo();
userName.value = basicInfo?.userName;
avatarUrl.value = basicInfo?.avatarUrl;
userId.value = basicInfo?.userId;

function setTUIRoomData(action: string, mode?: string) {
  const roomParam = streamPreviewRef.value.getRoomParam();
  const roomData = {
    action,
    roomMode: mode || 'FreeSpeech',
    roomParam,
  };
  sessionStorage.setItem('tuiRoom-roomInfo', JSON.stringify(roomData));
}

async function checkRoomExist(roomId: string) {
  let isRoomExist = false;
  const tim = roomEngine.instance?.getTIM();
  try {
    await tim.searchGroupByID(roomId);
    isRoomExist = true;
  } catch (error: any) {
    // 房间不存在
  }
  return isRoomExist;
}

/**
 * Generate room number when creating a room
 *
 * 创建房间时生成房间号
**/
async function generateRoomId(): Promise<number> {
  const roomId = Math.ceil(Math.random() * 1000000);
  const isRoomExist = await checkRoomExist(String(roomId));
  if (isRoomExist) {
    return await generateRoomId();
  }
  return roomId;
}

/**
 * Processing Click [Create Room]
 *
 * 处理点击【创建房间】
**/
async function handleCreateRoom(mode: string) {
  setTUIRoomData('createRoom', mode);
  const roomId = await generateRoomId();
  router.replace({
    path: 'room',
    query: {
      roomId,
    },
  });
}

/**
 * Processing Click [Enter Room]
 *
 * 处理点击【进入房间】
**/
async function handleEnterRoom(roomId: number) {
  const isRoomExist = await checkRoomExist(String(roomId));
  if (!isRoomExist) {
    alert(t('The room does not exist, please confirm the room number or create a room!'));
    return;
  }
  setTUIRoomData('enterRoom');
  router.replace({
    path: 'room',
    query: {
      roomId,
    },
  });
}

/**
 * Processing users click [Logout Login] in the upper left corner of the page
 *
 * 处理用户点击页面左上角【退出登录】
**/
async function handleLogOut() {
/**
 * The accessor handles the logout method
 *
 * 接入方处理 logout 方法
**/
}

onMounted(async () => {
  const currentUserInfo = await getBasicInfo();
  if (currentUserInfo) {
    sessionStorage.setItem('tuiRoom-userInfo', JSON.stringify(currentUserInfo));
    const { sdkAppId, userId, userSig } = currentUserInfo;
    /**
     * TUIRoomCore.checkRoomExistence method can only be used after logging into TUIRoomCore.
     *
     * 登录 TUIRoomCore, 只有登录 TUIRoomCore 之后，才可以使用 TUIRoomCore.checkRoomExistence 方法
    **/
    await TUIRoomEngine.init({ sdkAppId, userId, userSig });
    streamPreviewRef.value.startStreamPreview();
  }
});

</script>

<style lang="scss" scoped>
.home-container {
  width: 100%;
  height: 100%;
  background-color: #010101;
  color: #B3B8C8;
  display: flex;
  justify-content: center;
  align-items: center;
  font-family: PingFangSC-Medium;
  .header {
    width: 100%;
    position: absolute;
    top: 0;
    padding: 22px 24px;
    display: flex;
    align-items: center;
    .header-item {
      &:not(:first-child) {
        margin-left: 16px;
      }
      .language{
        cursor: pointer;
      }
    }
  }
}
</style>
