<template>
    <div :class="['play-bar', lockName]" v-if="curSongInfo" @mouseenter="enterBar" @mouseleave="leaveBar">
        <div class="fold">
            <div class="fold-btn" @click="lockBar">
                <i class="iconfont icon-lock" :class="[locked ? 'active': '']"></i>
            </div>
        </div>
        <progress-line class="audioProgress"
            :progressWidth="audioProgressWidth"
            @setProgressLine="setAudioProgress"></progress-line>
        <div class="w1200">
            <audio
            ref="audio"
            preload="auto"
            @canplay="canplaySong"
            @playing="playSong"
            @ended="endedSong"
            @error="errorSong"
            @timeupdate="updateSongTime"
            :src="curSongInfo.url"></audio>
            <el-row class="play-bar-inside">
                <el-col :span="7" class="bar-l">
                    <router-link :to="{ path: '/song', query: { id: curSongInfo.id }}">
                        <el-image :src="curSongInfo.album.picUrl" class="bar-cover-img">
                            <div slot="placeholder" class="image-slot">
                                <i class="iconfont icon-placeholder"></i>
                            </div>
                        </el-image>
                    </router-link>
                    <div class="bar-name">
                        <router-link :to="{ path: '/song', query: { id: curSongInfo.id }}" class="song_name">{{curSongInfo.name}}</router-link>
                        <p><router-link :to="{ path: '/singer', query: { id: author.id }}" class="song_author" v-for="(author, k) in curSongInfo.singer" :key="author.name">{{ k !== 0 ? ' / ' + author.name : author.name }}</router-link></p>
                    </div>
                    <div class="bar-time">
                        <span>{{$utils.formatSongTime(currentTime * 1000)}}</span> / {{curSongInfo.duration}}
                    </div>
                </el-col>
                <el-col :span="9" class="bar-m">
                    <div class="bar-control">
                        <i class="iconfont icon-audio-prev" title="?????????" @click.stop="audioHandler('prev')"></i>
                        <i :class="['iconfont', playIcon]" @click.stop="audioHandler('play')"></i>
                        <i class="iconfont icon-audio-next" title="?????????" @click.stop="audioHandler('next')"></i>
                    </div>
                </el-col>
                <el-col :span="8" class="bar-r">
                    <div class="bar-oper">
                        <div class="volume-main">
                            <i :class="['iconfont', mutedIcon]" title="??????" @click.stop="volumeHandler"></i>
                            <progress-line
                                class="volumeLine"
                                :progressWidth="volumeProgressWidth"
                                @setProgressLine="setvolumeProgress"></progress-line>
                        </div>
                        <i class="iconfont" :class="modeIcon.className" :title="modeIcon.title" @click.stop="changePlayMode"></i>
                        <div class="popver" v-clickoutside="popverClose">
                            <div class="lyric">
                                <span class="lyric-btn" title="??????" @click="lyricsHandle">???</span>
                            </div>
                            <div class="playlist" @click="playlistHandle">
                                <i class="iconfont icon-playlist" title="????????????"></i>
                                <div class="playlist-num">{{ 99 > playList.length ? playList.length : '99+'}}</div>
                            </div>
                            <div class="lyrics-container" v-show="lyricsVisible">
                                <h3 class="lyrics-header">
                                    <span>??????</span>
                                    <i class="iconfont icon-closed" @click="lyricsHandle"></i>
                                </h3>
                                <lyrics :sId="curSongInfo.id" :currentTime="currentTime"></lyrics>
                            </div>
                            <div class="playlist-container" v-show="playlistVisible">
                                <h3 class="playlist-header">
                                    <span>????????????<em>(???{{playList.length}}???)</em></span>
                                    <div class="delSonglist" @click="clearSonglist"><i class="iconfont icon-del" title="????????????"></i>????????????</div>
                                </h3>
                                <song-list :songList="playList" :isScroll="true" :height="400" :typeSize="'mini'" :isShowTips="false"></song-list>
                            </div>
                        </div>
                    </div>
                </el-col>
            </el-row>
        </div>
    </div>
</template>

<script>
import { mapGetters, mapMutations } from 'vuex'
import ProgressLine from '@components/common/progress'
import songList from '@components/common/song-list'
import Lyrics from '@components/common/lyrics.vue'
export default {
    components: {
        ProgressLine,
        songList,
        Lyrics
    },
    data () {
        // ??????????????????
        return {
            initAudioReady: false, // ?????????????????????
            isMuted: false, // ????????????
            currentTime: 0, // ??????????????????
            totalTime: 0, // ???????????????
            volumeNum: 0.5, // ?????????(0~1)
            oldVolume: 0, // ????????????????????????????????????????????????????????????
            playMode: 0, // ????????????  0???????????????1???????????????2????????????
            timer: null,
            lyricsVisible: false,
            playlistVisible: false,
            // ?????????????????????????????????
            isLock: false,
            locked: false,
            lockName: 'active'
        }
    },
    created () {
        this.setPlayList(this.playList)
    },
    mounted () {
        this.leaveBar(this)
    },
    // ???????????? ?????????data??????
    computed: {
        ...mapGetters(['playIndex', 'playList', 'isPlayed', 'playListTips']),
        // ??????????????????
        playIcon () {
            return !this.isPlayed ? 'icon-audio-play' : 'icon-audio-pause'
        },
        // ????????????
        mutedIcon () {
            return this.isMuted ? 'icon-volume-active' : 'icon-volume'
        },
        // ????????????
        modeIcon () {
            const params = [{
                className: 'icon-loop',
                title: '????????????'
            }, {
                className: 'icon-single-cycle',
                title: '????????????'
            }, {
                className: 'icon-shuffle',
                title: '????????????'
            }]
            return params[this.playMode]
        },
        audioProgressWidth () { // ?????????????????????
            return this.currentTime / this.totalTime * 100 + '%'
        },
        volumeProgressWidth () {
            return this.volumeNum / 1 * 100 + '%'
        },
        curSongInfo () {
            return this.playList[this.playIndex]
        }
    },
    // ????????????
    methods: {
        ...mapMutations({
            setPlayStatus: 'SET_PLAYSTATUS',
            setPlayIndex: 'SET_PLAYINDEX',
            setPlayList: 'SET_PLAYLIST'
        }),
        // ????????????/??????/?????????/???????????????
        audioHandler (type) {
            if (type === 'play') {
                this.setPlayStatus(!this.isPlayed)
                this.setPlayIndex(this.playIndex)
            } else {
                this.changeSong(type)
            }
        },
        // ????????????????????? ??????????????????????????????
        updateSongTime (e) {
            if (!this.initAudioReady) {
                return
            }
            this.currentTime = e.target.currentTime
        },
        // ?????????????????????????????????????????????
        canplaySong (e) {
            this.initAudioReady = true
        },
        // ?????????????????? ???????????????????????????????????????
        playSong (e) {
            this.initAudioReady = true
            this.setPlayStatus(true)
            this.totalTime = e.target.duration
        },
        // ?????????????????? ?????????????????????
        endedSong (e) {
            if (this.playMode === 1) {
                this.loopSong()
            } else {
                this.changeSong('next')
            }
        },
        // ??????????????????
        errorSong (e) {
            this.initAudioReady = false
            this.setPlayStatus(false)
        },
        // ???????????????????????????
        volumeHandler () {
            this.isMuted = this.$refs.audio.muted = this.isMuted ? 0 : 1
            this.isMuted && (this.oldVolume = this.volumeNum)
            this.volumeNum = this.isMuted ? 0 : this.oldVolume
        },
        // ??????????????????????????????????????????
        setAudioProgress (params) {
            this.initAudioReady = false
            this.currentTime = params.val * this.totalTime

            // ????????????????????????????????????????????????
            if (params.flag) {
                this.$refs.audio.currentTime = params.val * this.totalTime
            }
        },
        // ??????????????????????????????????????????
        setvolumeProgress (params) {
            this.volumeNum = params.val
            this.oldVolume = params.val
            this.$refs.audio.volume = params.val
            this.isMuted = this.$refs.audio.muted = params.val ? 0 : 1
        },
        // ??????????????????
        changePlayMode () {
            this.playMode = this.playMode >= 2 ? 0 : this.playMode + 1
        },
        // ??????????????????
        clearSonglist () {
            window.localStorage.removeItem('playList')
            this.setPlayList([])
            this.setPlayStatus(false)
            this.setPlayIndex(0)
        },
        // ??????????????????
        changeSong (type) { // type: prev/next  ?????????/?????????
            if (this.playList.length !== 1) { // ?????????????????????????????????????????????
                let index = this.playIndex
                if (this.playMode === 2) { // ????????????
                    index = Math.floor(Math.random() * this.playList.length - 1) + 1
                } else {
                    if (type === 'prev') {
                        index = index === 0 ? this.playList.length - 1 : --index
                    } else {
                        index = index >= this.playList.length - 1 ? 0 : ++index
                    }
                }

                this.initAudioReady = false
                this.setPlayStatus(false)
                this.setPlayIndex(index)
            } else {
                this.loopSong()
            }
        },
        // ??????????????????
        loopSong () {
            this.$refs.audio.currentTime = 0
            this.$refs.audio.play()
            this.setPlayStatus(true)
        },
        enterBar () {
            clearTimeout(this.timer)
            this.lockName = 'active'
        },
        leaveBar (e) {
            // ??????????????????????????????mouseleave ?????? ?????????e????????? undefined  ???????????????????????????????????? e????????????
            // if (!e) return
            const self = this
            if (!self.isLock && !self.locked) {
                clearTimeout(self.timer)
                self.timer = setTimeout(() => {
                    self.lockName = self.isLock ? 'active' : ''
                }, 3000)
            }
        },
        lockBar () {
            this.locked = !this.locked
            this.isLock = this.locked
            this.leaveBar()
        },
        lyricsHandle () {
            this.lyricsVisible = !this.lyricsVisible
            this.playlistVisible = false
            this.isLock = this.lyricsVisible
        },
        playlistHandle () {
            this.playlistVisible = !this.playlistVisible
            this.lyricsVisible = false
            this.isLock = this.playlistVisible
        },
        popverClose () {
            this.lyricsVisible = this.playlistVisible = this.isLock = false
            this.leaveBar()
        }
    },
    watch: {
        curSongInfo (newSong, oldSong) {
            if (!newSong || (oldSong && newSong.id === oldSong.id)) {
                return
            }
            // ?????????????????????????????????  ????????????????????????????????????
            this.initAudioReady = false
            this.currentTime = 0
            this.$nextTick(() => {
                const audio = this.$refs.audio
                if (audio) {
                    audio.play()
                }
            })
        },
        isPlayed (playing) {
            // ???????????????????????????????????????
            if (!this.initAudioReady) {
                return
            }
            this.$nextTick(() => {
                const audio = this.$refs.audio
                if (audio) {
                    playing ? audio.play() : audio.pause()
                }
            })
        }
    },
    destroyed () {
        clearTimeout(this.timer)
    }
}
</script>
<style scoped lang="less">
.play-bar {
    position: fixed;
    bottom: 0;
    left: 0;
    z-index: 5;
    width: 100%;
    height: 70px;
    background: #fff;
    box-shadow: 0 5px 40px -1px rgba(2,10,18,.1);
    transition: all .4s ease-out;
    transform: translateY(100%);

    &.active {
        transform: translateY(0);
    }
}
.fold {
    position: absolute;
    top: -30px;
    left: 0;
    z-index: 3;
    width: 100%;
    height: 30px;

    .fold-btn {
        position: absolute;
        right: 200px;
        display: inline-block;
        width: 60px;
        height: 30px;
        line-height: 30px;
        text-align: center;
        border-radius: 50px 50px 0px 0px;
        background: #fff;
        box-shadow: 0 5px 40px -1px rgba(2,10,18,.1);
        cursor: pointer;

        .active {
            color: @color-theme
        }
    }
}
.audioProgress {
    position: absolute;
    top: -6px;
    z-index: 9;
}
.play-bar-inside {
    font-size: 0;
    line-height: 0;

    .bar-l {
        display: flex;
        padding: 10px 0;
        align-items: center;
        justify-content: center;
    }

    .bar-cover-img {
        width: 50px;
        height: 50px;
        border-radius: 4px;
        box-shadow: 0 0 15px 5px rgba(@color-theme,.15);
    }

    .bar-name {
        flex: 1;
        font-size: 0;
        padding: 0 10px 6px 30px;
        overflow: hidden;

        .song_name {
            display: block;
            padding-bottom: 20px;
            line-height: 12px;
            font-size: 12px;
            font-weight: bold;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }

        p {
            line-height: 0;
            overflow: hidden;
            text-overflow: ellipsis;
            white-space: nowrap;
        }

        .song_author {
            line-height: 12px;
            font-size: 12px;
            color: #999;
        }
    }

    .icon-collect, .icon-collect-active {
        font-size: 24px;
        line-height: 24px;
        margin: 0 15px;
        cursor: pointer;
    }

    .bar-time {
        width: 115px;
        line-height: 50px;
        font-size: 14px;
        color: #606266;
        text-align: center;
    }

    .bar-control {
        display: flex;
        padding: 10px 0;
        line-height: 1;
        align-items: center;
        justify-content: center;

        .iconfont {
            margin: 0 20px;
            font-size: 30px;
            cursor: pointer;
            color: #666;
        }

        .icon-audio-play, .icon-audio-pause {
            font-size: 50px;
            font-weight: bold;
            color: @color-theme;
        }
    }

    .volumeLine {
        width: 150px;
        margin-left: 10px;
    }

    .bar-oper {
        position: relative;
        display: flex;
        font-size: 0;
        line-height: 50px;
        padding-left: 30px;
        align-items: center;
        justify-content: center;

        .iconfont {
            margin-left: 20px;
            font-size: 24px;
            cursor: pointer;
        }
    }

    .popver {
        position: relative;
    }

    .lyric {
        display: inline-block;
        padding: 26px 3px;
        margin-left: 20px;
        font-size: 18px;
        line-height: 18px;
        vertical-align: top;
        color: #999;
        cursor: pointer;
    }

    .playlist {
        position: relative;
        display: inline-block;
        line-height: 70px;
        color: #999;
        cursor: pointer;

        .playlist-num {
            position: absolute;
            top: 25%;
            left: 90%;
            font-size: 12px;
            line-height: 12px;
            color: #999;
        }
    }

    .volume-main {
        flex: 1;
        padding: 10px 0;
        display: flex;
        align-items: center;
    }
}

.playlist-header {
    display: flex;
    line-height: 40px;

    span {
        display: inline-block;
        flex: 1;
    }

    em {
        display: inline-block;
        padding-left: 10px;
        font-size: 12px;
        line-height: 14px;
        font-style: normal;
        font-weight: normal;
        color: #666;
        vertical-align: baseline;
    }

    .delSonglist {
        font-size: 14px;
        font-weight: normal;
        cursor: pointer;
    }

    .icon-del {
        margin-right: 5px;
        font-size: 16px;
        color: #333;
    }
}
// ----------------------------------?????????--------------------------
.lyrics-container {
    position: absolute;
    left: -200px;
    bottom: 75px;
    width: 400px;
    padding: 20px;
    height: 430px;
    border: 1px solid #EBEEF5;
    border-radius: 4px 4px 0 0;
    box-shadow: 0 2px 12px 0 rgba(0,0,0,.1);
    background: #fff;

    .lyrics-header {
        display: flex;
        padding: 0 0 24px;
        line-height: 16px;
        font-weight: 500;
        font-size: 16px;

        span {
            display: block;
            flex: 1;
        }

        .icon-closed {
            font-size: 20px;
        }
    }
}
.playlist-container {
    position: absolute;
    left: -250px;
    bottom: 75px;
    width: 500px;
    padding: 20px;
    border: 1px solid #EBEEF5;
    border-radius: 4px 4px 0 0;
    font-size: 14px;
    box-shadow: 0 2px 12px 0 rgba(0,0,0,.1);
    background: #fff;
}
</style>
