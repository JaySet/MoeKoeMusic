<template>
    <div class="player-container">
        <div class="progress-bar" 
            @mousedown="onProgressDragStart"
            @click="updateProgressFromEvent"
            @mousemove="updateTimeTooltip"
            @mouseleave="hideTimeTooltip">
            <div class="progress" :style="{ width: progressWidth + '%' }"></div>
            <div class="progress-handle" :style="{ left: progressWidth + '%' }"></div>
            <div v-for="(point, index) in climaxPoints" 
                 :key="index"
                 class="climax-point"
                 :style="{ left: point.position + '%' }">
            </div>
            <div v-if="showTimeTooltip" 
                 class="time-tooltip" 
                 :style="{ left: tooltipPosition + 'px' }">
                {{ tooltipTime }}
            </div>
        </div>
        <div class="player-bar">
            <div class="album-art" @click="toggleLyrics">
                <img v-if="currentSong.img" :src="currentSong.img" alt="Album Art" />
                <i v-else class="fas fa-music"></i>
            </div>
            <div class="song-info" @click="toggleLyrics">
                <div class="song-title">{{ currentSong?.name || "MoeKoeMusic" }}</div>
                <div class="artist">{{ currentSong?.author || "MoeJue" }}</div>
            </div>
            <div class="controls">
                <button class="control-btn" @click="playSongFromQueue('previous')">
                    <i class="fas fa-step-backward"></i>
                </button>
                <button class="control-btn" @click="togglePlayPause">
                    <i :class="playing ? 'fas fa-pause' : 'fas fa-play'"></i>
                </button>
                <button class="control-btn" @click="playSongFromQueue('next')">
                    <i class="fas fa-step-forward"></i>
                </button>
            </div>
            <div class="extra-controls">
                <button class="extra-btn" @click="togglePlaybackMode">
                    <i v-if="currentPlaybackModeIndex != '2'" :class="currentPlaybackMode.icon" :title="currentPlaybackMode.title"></i>
                    <span v-else class="loop-icon" :title="currentPlaybackMode.title">
                        <i class="fas fa-repeat"></i>
                        <sup>1</sup>
                    </span>
                </button>
                <button class="extra-btn" @click="toggleQueue"><i class="fas fa-list"></i></button>
                <!-- 音量控制 -->
                <div class="volume-control" @wheel="handleVolumeScroll">
                    <i :class="isMuted ? 'fas fa-volume-mute' : 'fas fa-volume-up'" @click="toggleMute"></i>
                    <div class="volume-slider" @mousedown="onDragStart">
                        <div class="volume-progress" :style="{ width: volume + '%' }"></div>
                        <input type="range" min="0" max="100" v-model="volume" @input="changeVolume" />
                    </div>
                </div>
            </div>
        </div>


        <!-- 播放队列 -->
        <transition name="fade">
            <div v-if="showQueue" class="queue-popup">
                <div class="queue-header">
                    <h3>
                        <span>{{ $t('bo-fang-lie-biao') }}</span> ({{ musicQueueStore.queue.length }})
                        <i class="fas fa-trash-alt close-store" @click="musicQueueStore.clearQueue()" title="close"></i>
                    </h3>
                </div>

                <RecycleScroller :items="musicQueueStore.queue" :item-size="50" key-field="id" :buffer="200"
                    :items-limit="2000" :prerender="Math.min(10, musicQueueStore.queue.length)" ref="queueScroller"
                    class="queue-list">
                    <template #default="{ item, index }">
                        <li class="queue-item" :class="{ 'playing': currentSong.hash == item.hash }" :key="item.id">
                            <div class="queue-song-info">
                                <span class="queue-song-title">{{ item.name }}</span>
                                <span class="queue-artist">{{ $formatMilliseconds(item.timeLength) }}</span>
                            </div>
                            <div class="queue-actions">
                                <button v-if="currentSong.hash == item.hash"
                                    class="queue-play-btn fas fa-music"></button>
                                <template v-else>
                                    <button class="queue-play-btn" @click="addSongToQueue(
                                        item.hash,
                                        item.name,
                                        item.img,
                                        item.author
                                    )"><i class="fas fa-play"></i></button>
                                    <i class="fas fa-times close-store" @click="musicQueueStore.queue.splice(index, 1);$event.target.closest('li').classList.add('marked-as-deleted');"></i>
                                </template>
                            </div>
                        </li>
                    </template>
                </RecycleScroller>
            </div>
        </transition>
    </div>

    <!-- 全屏歌词界面 -->
    <transition name="slide-up">
        <div v-if="showLyrics" class="lyrics-bg"
            :style="(lyricsBackground == 'on' ? ({ backgroundImage: `url(${currentSong?.img || 'https://random.MoeJue.cn/randbg.php'})` }) : ({ background: 'var(--background-color)' }))">
            <div class="lyrics-screen">
                <div class="close-btn">
                    <i class="fas fa-chevron-down" @click="toggleLyrics"></i>
                </div>

                <div class="left-section">
                    <div class="album-art-large">
                        <img v-if="easterEggImage" :src="easterEggImage.src" :class="easterEggClass" alt="Easter Egg" />
                        <img :src="currentSong?.img || 'https://random.MoeJue.cn/randbg.php'" alt="Album Art" />
                    </div>
                    <div class="song-details">
                        <div class="song-title">{{ currentSong?.name }}</div>
                        <div class="artist">{{ currentSong?.author }}</div>
                    </div>

                    <!-- 播放进度条 -->
                    <div class="progress-bar-container">
                        <span class="current-time">{{ formattedCurrentTime }}</span>
                        <div class="progress-bar" 
                            @mousedown="onProgressDragStart"
                            @click="updateProgressFromEvent"
                            @mousemove="updateTimeTooltip"
                            @mouseleave="hideTimeTooltip">
                            <div class="progress" :style="{ width: progressWidth + '%' }"></div>
                            <div class="progress-handle" :style="{ left: progressWidth + '%' }"></div>
                            <div v-for="(point, index) in climaxPoints" 
                                 :key="index"
                                 class="climax-point"
                                 :style="{ left: point.position + '%' }">
                            </div>
                            <div v-if="showTimeTooltip" 
                                 class="time-tooltip" 
                                 :style="{ left: tooltipPosition + 'px' }">
                                {{ tooltipTime }}
                            </div>
                        </div>
                        <span class="duration">{{ formattedDuration }}</span>
                    </div>

                    <div class="player-controls">
                        <button class="control-btn" @click="playSongFromQueue('previous')">
                            <i class="fas fa-step-backward"></i>
                        </button>
                        <button class="control-btn" @click="togglePlayPause">
                            <i :class="playing ? 'fas fa-pause' : 'fas fa-play'"></i>
                        </button>
                        <button class="control-btn" @click="playSongFromQueue('next')">
                            <i class="fas fa-step-forward"></i>
                        </button>
                        <button class="control-btn" @click="togglePlaybackMode">
                            <i v-if="currentPlaybackMode.mode !== 'loop_one'" :class="currentPlaybackMode.icon" :title="currentPlaybackMode.title"></i>
                            <span v-else class="loop-icon" :title="currentPlaybackMode.title">
                                <i class="fas fa-repeat"></i>
                                <sup>1</sup>
                            </span>
                        </button>
                    </div>
                </div>
                <div id="lyrics-container">
                    <div v-if="lyricsData.length > 0" id="lyrics"
                        :style="{ transform: `translateY(${scrollAmount}px)` }">
                        <div v-for="(lineData, lineIndex) in lyricsData" :key="lineIndex" class="line">
                            <span v-for="(charData, charIndex) in lineData.characters" :key="charIndex" class="char"
                                :class="{ highlight: charData.highlighted }">
                                {{ charData.char }}
                            </span>
                        </div>
                    </div>
                    <div v-else class="no-lyrics">{{ SongTips }}</div>
                </div>
            </div>
        </div>
    </transition>
</template>

<script setup>
import { ref, onMounted, computed, onUnmounted, nextTick } from 'vue';
import { RecycleScroller } from 'vue3-virtual-scroller';
import 'vue3-virtual-scroller/dist/vue3-virtual-scroller.css'; 
import { get } from '../utils/request'; 
import { useMusicQueueStore } from '../stores/musicQueue'; 
import { useI18n } from 'vue-i18n';
const { t } = useI18n();
const showLyrics = ref(false); // 是否显示歌词
const isDragging = ref(false);
const showQueue = ref(false);
const currentSong = ref({ name: '', author: '', img: '', url: '', hash: '' }); // 当前播放的音乐信息
const playing = ref(false); // 是否正在播放
const isMuted = ref(false); // 是否静音
const volume = ref(66); // 音量初始值为 100
const audio = new Audio(); // 创建音频对象
const musicQueueStore = useMusicQueueStore(); // 使用 Pinia store
let sliderElement = null; // 用来保存音量滑块的 DOM 引用
const progressWidth = ref(0);
const lyricsData = ref([]);
const scrollAmount = ref(226);
const currentTime = ref(0);
const SongTips = ref(t('zan-wu-ge-ci'));
const lyricsBackground = ref('on');
let currentLineIndex = 0;
const queueScroller = ref(null);
const timeoutId = ref(null);
const playbackModes = ref([
    { icon: 'fas fa-random', title: t('sui-ji-bo-fang') },
    { icon: 'fas fa-refresh', title: t('lie-biao-xun-huan') },
    { icon: '', title: t('dan-qu-xun-huan') } 
]);
const currentPlaybackModeIndex = ref(0);
const currentPlaybackMode = computed(() => playbackModes.value[currentPlaybackModeIndex.value]);
const isProgressDragging = ref(false);
const isDraggingHandle = ref(false);
const climaxPoints = ref([]);
// 切换随机/顺序/单曲播放
const togglePlaybackMode = () => {
    currentPlaybackModeIndex.value = (currentPlaybackModeIndex.value + 1) % playbackModes.value.length;
    audio.loop = currentPlaybackModeIndex.value == 2;
    localStorage.setItem('player_playback_mode', currentPlaybackModeIndex.value);
};
onMounted(() => {
    const savedVolume = localStorage.getItem('player_volume');
    if (savedVolume !== null) volume.value = parseFloat(savedVolume);
    isMuted.value = volume.value === 0;
    audio.volume = volume.value / 100;
    const current_song = localStorage.getItem('current_song');
    if (current_song) currentSong.value = JSON.parse(current_song);
    currentPlaybackModeIndex.value = localStorage.getItem('player_playback_mode') || 1;
    audio.loop = currentPlaybackModeIndex.value == 2;
    if (localStorage.getItem('settings')) {
        lyricsBackground.value = JSON.parse(localStorage.getItem('settings'))['lyricsBackground']
    }
    handleShortcut();
});
const formattedCurrentTime = computed(() => formatTime(currentTime.value));
const formattedDuration = computed(() => formatTime(currentSong.value?.timeLength || 0));
const formatTime = (seconds) => {
    const minutes = Math.floor(seconds / 60);
    const secs = Math.floor(seconds % 60);
    return `${minutes}:${secs.toString().padStart(2, '0')}`;
};
const easterEggImages = [
    { src: './assets/images/miku.png', class: 'miku' },
    { src: './assets/images/miku2.png', class: 'miku2' },
    { src: './assets/images/miku3.png', class: 'miku3' }
];
const easterEggImage = computed(() => {
    const author = currentSong.value?.author || '';
    if (author.includes('初音') || author.includes('Miku')) {
        const randomIndex = Math.floor(Math.random() * easterEggImages.length);
        return easterEggImages[randomIndex];
    }
    return null;
});
const easterEggClass = computed(() => easterEggImage.value?.class || '');
// 播放音乐
const playSong = (song) => {
    currentSong.value = structuredClone(song);
    lyricsData.value = [];
    audio.src = song.url;
    audio.play();
    playing.value = true;
    localStorage.setItem('current_song', JSON.stringify(currentSong.value));
    getLyrics(currentSong.value.hash)
    if (canRequestVip()) {
        get('/youth/vip')
    }
    getMusicHighlights(currentSong.value.hash)
};

// 获取音乐高潮
const getMusicHighlights = async (hash) => {
    const response = await get(`/song/climax?hash=${hash}`);
    if (response.status !== 1) {
        climaxPoints.value = [];
        return;
    }
    climaxPoints.value = response.data.map(point => ({
        position: (parseInt(point.start_time) / 1000 / audio.duration) * 100,
        duration: parseInt(point.timelength) / 1000
    }));
};

// 从队列中播放歌曲
const playSongFromQueue = (direction) => {
    if (musicQueueStore.queue.length === 0) {
        window.$modal.alert(t('ni-huan-mei-you-tian-jia-ge-quo-kuai-qu-tian-jia-ba'));
        return;
    }
    const currentIndex = musicQueueStore.queue.findIndex(song => song.hash === currentSong.value.hash);
    let targetIndex;

    if (currentIndex == -1) {
        targetIndex = 0;
    } else if (currentPlaybackModeIndex.value == 0) {
        targetIndex = Math.floor(Math.random() * musicQueueStore.queue.length);
    } else {
        targetIndex = direction === 'previous' 
            ? (currentIndex === 0 ? musicQueueStore.queue.length - 1 : currentIndex - 1) 
            : (currentIndex + 1) % musicQueueStore.queue.length;
    }
    addSongToQueue(
        musicQueueStore.queue[targetIndex].hash,
        musicQueueStore.queue[targetIndex].name,
        musicQueueStore.queue[targetIndex].img,
        musicQueueStore.queue[targetIndex].author
    );
};

// 切换播放/暂停
const togglePlayPause = () => {
    if (!currentSong.value.hash) {
        playSongFromQueue('next');
        return;
    } else if (!audio.src) {
        addSongToQueue(
            currentSong.value.hash,
            currentSong.value.name,
            currentSong.value.img,
            currentSong.value.author
        );
        return;
    }

    if (playing.value) {
        audio.pause();
        playing.value = false;
    } else {
        audio.play();
        playing.value = true;
    }
};

// 切换静音
const toggleMute = () => {
    isMuted.value = !isMuted.value;
    audio.muted = isMuted.value;
    if (isMuted.value) {
        volume.value = 0;
    } else {
        volume.value = audio.volume * 100;
    }
    localStorage.setItem('player_volume', volume.value);
};
// 更新音量的点击处理函数
const setVolumeOnClick = (event) => {
    const slider = event.target.closest('.volume-slider');
    if (slider) {
        const sliderWidth = slider.offsetWidth;
        const offsetX = event.offsetX;
        volume.value = Math.round((offsetX / sliderWidth) * 100);
        changeVolume(); 
    }
};
// 开始拖拽时触发，初始化拖拽并实时更新音量
const onDragStart = (event) => {
    sliderElement = event.target.closest('.volume-slider');
    if (sliderElement) {
        isDragging.value = true;
        setVolumeOnClick(event); 
        document.addEventListener('mousemove', onDrag);
        document.addEventListener('mouseup', onDragEnd);
    }
};
// 拖拽过程中的音量更新
const onDrag = (event) => {
    if (isDragging.value && sliderElement) {
        const sliderWidth = sliderElement.offsetWidth;
        const rect = sliderElement.getBoundingClientRect();
        const offsetX = event.clientX - rect.left; 
        const newVolume = Math.max(0, Math.min(100, Math.round((offsetX / sliderWidth) * 100))); 
        volume.value = newVolume;
        changeVolume();
    }
};
const onDragEnd = () => {
    isDragging.value = false;
    sliderElement = null; 
    document.removeEventListener('mousemove', onDrag); 
    document.removeEventListener('mouseup', onDragEnd);
};
// 音量调节
const changeVolume = () => {
    audio.volume = volume.value / 100;
    localStorage.setItem('player_volume', volume.value);
    isMuted.value = volume.value === 0;
};

// 获取歌单全部歌曲
const getPlaylistAllSongs = async (id) => {
    try {
        let allSongs = [];
        for (let page = 1; page <= 4; page++) {
            const url = `/playlist/track/all?id=${id}&pagesize=300&page=${page}`;
            const response = await get(url);
            if (response.status !== 1) {
                window.$modal.alert(t('huo-qu-ge-dan-shi-bai'));
                return;
            }
            if (Object.keys(response.data.info).length === 0) break;
            allSongs = allSongs.concat(response.data.info);
            if(response.data.info.length < 300) break;
        }
        addPlaylistToQueue(allSongs);
    } catch (error) {
        console.error(error);
        window.$modal.alert(t('huo-qu-ge-dan-shi-bai'));
    }
}
// 添加歌单到播放列表
const addPlaylistToQueue = async (info) => {
    musicQueueStore.clearQueue();
    const songs = info.map((song, index) => {
        return {
            id: index + 1,
            hash: song.hash,
            name: song.name,
            img: song.cover.replace("{size}", 480),
            author: song.name,
            timeLength: song.timelen
        };
    });
    musicQueueStore.queue = songs;
    addSongToQueue(songs[0].hash, songs[0].name, songs[0].img, songs[0].author);
};

// 添加歌曲到队列并播放的方法
const addSongToQueue = async (hash, name, img, author) => {
    try {
        clearTimeout(timeoutId.value);
        currentSong.value.author = author;
        currentSong.value.name = name;
        currentSong.value.img = img;
        currentSong.value.hash = hash;
        const url = `/song/url?hash=${hash}&free_part=1`;
        const response = await get(url);
        if (response.status !== 1) {
            currentSong.value.author = currentSong.value.name = t('huo-qu-yin-le-shi-bai');
            if(response.status == 3){
                currentSong.value.name = t('gai-ge-qu-zan-wu-ban-quan')
            }
            if (musicQueueStore.queue.length === 0) return;
            currentSong.value.author = t('3-miao-hou-zi-dong-qie-huan-xia-yi-shou');
            timeoutId.value = setTimeout(() => {
                playSongFromQueue('next');
            }, 3000);
            return;
        }
        const existingSongIndex = musicQueueStore.queue.findIndex(song => song.hash === hash);

        if (existingSongIndex === -1) {
            const song = {
                id: musicQueueStore.queue.length + 1,
                hash: hash,
                name: name,
                img: img,
                author: author,
                timeLength: response.timeLength,
                url: response.url[0]
            };
            musicQueueStore.addSong(song);
            playSong(song);
        } else {
            const updatedQueue = [...musicQueueStore.queue];
            const newSong = {
                id: musicQueueStore.queue[existingSongIndex].id,
                hash: hash,
                name: name,
                img: img,
                author: author,
                timeLength: response.timeLength,
                url: response.url[0]
            };
            updatedQueue[existingSongIndex] = newSong;
            musicQueueStore.setQueue(updatedQueue);
            playSong(newSong);
        }
    } catch (error) {
        currentSong.value.author = currentSong.value.name = t('huo-qu-yin-le-di-zhi-shi-bai');
        if (musicQueueStore.queue.length === 0) return;
        currentSong.value.author = t('3-miao-hou-zi-dong-qie-huan-xia-yi-shou');
        timeoutId.value = setTimeout(() => {
            playSongFromQueue('next');
        }, 3000);
    }
};

// 音乐播放结束后自动播放下一首
audio.addEventListener('ended', () => {
    if (currentPlaybackModeIndex.value == 2) return;
    playSongFromQueue('next');
});
audio.addEventListener('pause', () => {
    playing.value = false;
});
audio.addEventListener('play', () => {
    playing.value = true;
});
const toggleQueue = async () => {
    showQueue.value = !showQueue.value;
    if (showQueue.value) {
        await nextTick();
        setTimeout(() => {
            const currentIndex = musicQueueStore.queue.findIndex(song => song.hash === currentSong.value.hash);
            if (currentIndex !== -1 && queueScroller.value) {
                const middleOffset = Math.floor(queueScroller.value.$el.clientHeight / 100);
                const scrollToIndex = Math.max(0, currentIndex - middleOffset);
                queueScroller.value.scrollToItem(scrollToIndex);
            }
        }, 100);
    }
};

const toggleLyrics = async () => {
    showLyrics.value = !showLyrics.value;
    SongTips.value = t('huo-qu-ge-ci-zhong');
    if (!currentSong.value) return;
    try {
        if (lyricsData.value.length != 0) return;
        getLyrics(currentSong.value.hash)
    } catch (error) {
        SongTips.value = t('huo-qu-ge-ci-shi-bai');
    }
};

// 请求歌词
const getLyrics = async (hash) => {
    if (!showLyrics.value) return;
    const lyricSearchResponse = await get(`/search/lyric?hash=${hash}`);
    if (lyricSearchResponse.status !== 200 || lyricSearchResponse.candidates.length === 0) {
        SongTips.value = t('zan-wu-ge-ci');
        return;
    }
    const lyricResponse = await get(`/lyric?id=${lyricSearchResponse.candidates[0].id}&accesskey=${lyricSearchResponse.candidates[0].accesskey}&decode=true`);
    if (lyricResponse.status !== 200) {
        SongTips.value = t('huo-qu-ge-ci-shi-bai');
        return;
    }
    parseLyrics(lyricResponse.decodeContent);
    centerFirstLine();
}

const parseLyrics = (text) => {
    const lines = text.split('\n');
    const parsedLyrics = lines
        .map((line) => {
            const match = line.match(/^\[(\d+),(\d+)\](.*)/);
            if (match) {
                const time = parseInt(match[1]);
                const duration = parseInt(match[2]);
                const lyric = match[3].replace(/<.*?>/g, '');
                const characters = lyric.split('').map((char, index) => ({
                    char,
                    startTime: time + (index * duration) / lyric.length,
                    endTime: time + ((index + 1) * duration) / lyric.length,
                    highlighted: false,
                }));
                return { characters };
            }
            return null;
        })
        .filter((line) => line);
    lyricsData.value = parsedLyrics;
};
const centerFirstLine = () => {
    const lyricsContainer = document.getElementById('lyrics-container');
    if (!lyricsContainer) return;
    const containerHeight = lyricsContainer.offsetHeight;
    const lyricsElement = document.getElementById('lyrics');
    if (!lyricsElement) return;
    const lyricsHeight = lyricsElement.offsetHeight;
    scrollAmount.value = (containerHeight - lyricsHeight) / 2;
};
const throttle = (func, delay) => {
    let lastTime = 0;
    return function (...args) {
        const now = Date.now();
        if (now - lastTime >= delay) {
            lastTime = now;
            func.apply(this, args);
        }
    };
};
// 监听方法
const highlightCurrentChar = (currentTime) => {
    lyricsData.value.forEach((lineData, index) => {
        let isLineHighlighted = false;
        lineData.characters.forEach((charData) => {
            if (currentTime * 1000 >= charData.startTime && !charData.highlighted) {
                charData.highlighted = true;
                isLineHighlighted = true;
            }
        });

        if (isLineHighlighted && currentLineIndex !== index) {
            currentLineIndex = index;
            const lyricsContainer = document.getElementById('lyrics-container');
            if (!lyricsContainer) return;
            const containerHeight = lyricsContainer.offsetHeight;
            const lineElement = document.querySelectorAll('.line')[index];
            if (lineElement) {
                const lineHeight = lineElement.offsetHeight;
                scrollAmount.value = -lineElement.offsetTop + (containerHeight / 2) - (lineHeight / 2); // 计算中心位置
            }
        }
    });
};
// 节流处理
const throttledHighlight = throttle(() => {
    currentTime.value = audio.currentTime; // 实时更新当前时间
    if (!isProgressDragging.value) {  // 只在非拖动状态更新进度条
        progressWidth.value = (currentTime.value / audio.duration) * 100;
    }
    if (audio && showLyrics.value && lyricsData.value) {
        highlightCurrentChar(audio.currentTime);
    }
}, 200);
// 启动监听
audio.addEventListener('timeupdate', throttledHighlight);


defineExpose({
    addSongToQueue,
    getPlaylistAllSongs
});

onMounted(() => {
    document.addEventListener('click', handleClickOutside);
});

onUnmounted(() => {
    document.removeEventListener('click', handleClickOutside);
    if (isElectron()) {
        window.electron.ipcRenderer.removeAllListeners('play-previous-track');
        window.electron.ipcRenderer.removeAllListeners('play-next-track');
        window.electron.ipcRenderer.removeAllListeners('volume-up');
        window.electron.ipcRenderer.removeAllListeners('volume-down');
        window.electron.ipcRenderer.removeAllListeners('toggle-play-pause');
    }
});
const isElectron = () => {
    return typeof window !== 'undefined' && typeof window.electron !== 'undefined';
};
const handleClickOutside = (event) => {
    const queuePopup = document.querySelector('.queue-popup');
    if (queuePopup && !queuePopup.contains(event.target) && !event.target.closest('.extra-btn')) {
        showQueue.value = false;
    }
};
const handleVolumeScroll = (event) => {
    event.preventDefault();
    const delta = Math.sign(event.deltaY) * -1; 
    volume.value = Math.min(Math.max(volume.value + delta * 10, 0), 100);
    changeVolume();
};
const canRequestVip = () => {
    const lastRequestTime = localStorage.getItem('lastVipRequestTime');
    if (lastRequestTime) {
        const now = new Date().getTime();
        const elapsedTime = now - parseInt(lastRequestTime);
        const threeHours = 3 * 60 * 60 * 1000;
        if (elapsedTime < threeHours) {
            return false;
        }
    }
    localStorage.setItem('lastVipRequestTime', new Date().getTime().toString());
    return true;
}

const handleShortcut = (event) => {
    if (isElectron()) {
        window.electron.ipcRenderer.on('play-previous-track', () => playSongFromQueue('previous'));
        window.electron.ipcRenderer.on('play-next-track', () => playSongFromQueue('next'));
        window.electron.ipcRenderer.on('volume-up', () => {
            volume.value = Math.min(volume.value + 10, 100);
            changeVolume();
        });
        window.electron.ipcRenderer.on('volume-down', () => {
            volume.value = Math.max(volume.value - 10, 0);
            changeVolume();
        });
        window.electron.ipcRenderer.on('toggle-play-pause', () => {
            togglePlayPause();
        });
    }
}

// 修改进度条拖动
const onProgressDragStart = (event) => {
    event.preventDefault();
    
    const currentProgressBar = event.target.closest('.progress-bar');
    if (!currentProgressBar) return;
    
    // 检查是否点击在小圆点上
    const handle = event.target.closest('.progress-handle');
    if (!handle) {
        if (!audio.duration) return;
        const rect = currentProgressBar.getBoundingClientRect();
        const offsetX = Math.max(0, Math.min(event.clientX - rect.left, currentProgressBar.offsetWidth));
        const percentage = (offsetX / currentProgressBar.offsetWidth) * 100;
        progressWidth.value = Math.max(0, Math.min(percentage, 100));
    }
    
    isProgressDragging.value = true;
    isDraggingHandle.value = true;
    
    activeProgressBar.value = currentProgressBar;
    
    document.addEventListener('mousemove', onProgressDrag);
    document.addEventListener('mouseup', onProgressDragEnd);
};

// 添加一个 ref 来保存当前活动的进度条元素
const activeProgressBar = ref(null);

const onProgressDrag = (event) => {
    event.preventDefault();
    if (isProgressDragging.value && activeProgressBar.value) {
        const rect = activeProgressBar.value.getBoundingClientRect();
        const offsetX = Math.max(0, Math.min(event.clientX - rect.left, activeProgressBar.value.offsetWidth));
        const percentage = (offsetX / activeProgressBar.value.offsetWidth) * 100;
        progressWidth.value = Math.max(0, Math.min(percentage, 100));
        
        // 更新时间提示
        tooltipPosition.value = offsetX;
        const time = (percentage / 100) * audio.duration;
        tooltipTime.value = formatTime(time);
    }
};

const onProgressDragEnd = (event) => {
    if (isProgressDragging.value && activeProgressBar.value) {
        const rect = activeProgressBar.value.getBoundingClientRect();
        const offsetX = Math.max(0, Math.min(event.clientX - rect.left, activeProgressBar.value.offsetWidth));
        const percentage = (offsetX / activeProgressBar.value.offsetWidth) * 100;
        const newTime = (percentage / 100) * audio.duration;
        
        audio.currentTime = Math.max(0, Math.min(newTime, audio.duration));
    }
    isProgressDragging.value = false;
    isDraggingHandle.value = false;
    showTimeTooltip.value = false;
    activeProgressBar.value = null; 
    document.removeEventListener('mousemove', onProgressDrag);
    document.removeEventListener('mouseup', onProgressDragEnd);
};

// 修改点击进度条的处理方法
const updateProgressFromEvent = (event) => {
    if (isProgressDragging.value) return; // 如果正在拖动则不处理点击
    
    const progressBar = event.target.closest('.progress-bar');
    if (!progressBar || !audio.duration) return;
    
    const rect = progressBar.getBoundingClientRect();
    const offsetX = Math.max(0, Math.min(event.clientX - rect.left, progressBar.offsetWidth));
    const percentage = (offsetX / progressBar.offsetWidth) * 100;
    const newTime = (percentage / 100) * audio.duration;
    
    audio.currentTime = Math.max(0, Math.min(newTime, audio.duration));
    progressWidth.value = percentage;
};

// 在进度条模板中添加时间提示
const showTimeTooltip = ref(false);
const tooltipPosition = ref(0);
const tooltipTime = ref('0:00');

const updateTimeTooltip = (event) => {
    const progressBar = event.target.closest('.progress-bar');
    if (!progressBar || !audio.duration) return;
    
    const rect = progressBar.getBoundingClientRect();
    const offsetX = Math.max(0, Math.min(event.clientX - rect.left, progressBar.offsetWidth));
    
    tooltipPosition.value = offsetX;
    const percentage = (offsetX / progressBar.offsetWidth);
    const time = percentage * audio.duration;
    tooltipTime.value = formatTime(time);
    
    showTimeTooltip.value = true;
};

const hideTimeTooltip = () => {
    if (!isProgressDragging.value) {
        showTimeTooltip.value = false;
    }
};
</script>


<style scoped>
@import '@/assets/style/PlayerControl.css';
</style>