<template>
    <div class="main-content"
    @mouseleave="handleMouseLeave"
    @keydown.enter="keydownEnterFn">
        <div class="header">
            <el-row class="header-row-one">
                <div class="left">
                    <h1 class="text">Daily</h1>
                </div>
                <div class="right">
                    <i class="iconfont icon-wenjianjia cursor-pointer" @click.stop="openDownloadFile"></i>
                    <div class="header-set">
                        <i class="iconfont icon-shezhi cursor-pointer" @click.stop="setterShow=!setterShow"></i>
                    </div>
                    <div>
                        <i class="iconfont icon-tuichu cursor-pointer" @click.stop="quit"></i>
                    </div>
                </div>
            </el-row>

            <div class="header-search" v-if="imageSource==='paper'">
                <el-select
                    class="header-search-input"
                    v-model="searchKey"
                    size="small"
                    @change="searchKeyFn"
                    @focus="searchKeyFocus=true"
                    @blur="searchKeyFocus=false">
                    <el-option v-for="item in paperClass" :key="item.value" :label="item.name" :value="item.value"></el-option>
                </el-select>
            </div>

            <div class="header-search" v-else>
                <el-input
                    class="header-search-input"
                    v-model="searchKey"
                    placeholder="请输入关键词"
                    size="small"
                    @focus="searchKeyFocus=true"
                    @blur="searchKeyFocus=false"></el-input>
                <i class="iconfont icon-sousuo" @click.stop="searchKeyFn"></i>
            </div>

            <sw-progress v-if="progressValue>0" :value="progressValue" :color="currentImageBacColor"></sw-progress>
        </div>

        <div class="content" :class="{'content-win':osType=='win'}" @scroll="contentScroll">
            <div class="content-main" v-if="images.length>0" :style="{'margin-top': '110px'}">
                <div class="sign">
                    <p >
                        {{sign}}
                    </p>
                    <p style="text-align: right">
                        {{date}}
                    </p>
                </div>
                <waterfall :line-gap="400" :watch="images" :align="'center'" :auto-resize="true">
                    <waterfall-slot v-for="(img,index) in images" :key="index" :width="400" :height="(400/img.width)*img.height">
                        <div
                                class="image-item "
                                :ref="'image_item_'+index"
                                :style="{'backgroundColor':img.backgroundColor}"
                                @mousemove.stop="currentMouseOverIndex=index,setterShow=false"
                                @mouseleave.stop="currentMouseOverIndex=-1">
                            <div  :style="{'width':400+'px','height':(400/img.width)*img.height+'px'}">
                                <img class="image-item-img" :src="img.url" alt="">
                            </div>
<!--                            <div class="sign" v-if="index==0">-->
<!--                                <p>{{sign}}</p>-->
<!--                                <p style="text-align: right">&#45;&#45;{{date}}</p>-->
<!--                            </div>-->
                            <div class="image-set-wallpaper-box" v-show="currentMouseOverIndex==index&&isSetting==false">
                                <div class="image-set-wallpaper cursor-pointer"  @click.stop="setWallpaper(img,index)">
                                    <i class="iconfont icon-xianshiqi"></i>
                                    <span class="cursor-pointer">设置壁纸</span>
                                </div>
                                <div class="image-set-wallpaper cursor-pointer" @click.stop="setWallpaper(img,index)">
                                    <i class="iconfont icon-xiazai"></i>
                                    <span class="cursor-pointer">下<span style="visibility: hidden">下载</span>载</span>
                                </div>
                            </div>


                            <div class="image-item-flag" v-show="currentMouseOverIndex==index&&isSetting==false">
                                <div class="image-item-flag-direction" v-show="img.directionColumn">
                                    <i class="iconfont icon-xiaoqing-tubiao-hengping"></i>
                                </div>
                                <div class="image-item-tip" :style="{'color':img.tip=='5k'?'#e0620d':img.tip=='4k'?'17abe3':'d3217b'}">{{img.tip}}</div>
                            </div>
                        </div>
                    </waterfall-slot>
                </waterfall>
                 <div class="is-loading" v-if="infoShow!==''">
                     <i v-if="getDataFlag" class="el-icon-loading"></i>
                     <span v-html="infoShow"></span>
                </div>
            </div>

            <div class="content-main-no" v-else @mousemove.stop="setterShow=false">
                <span v-html="infoShow"></span>
            </div>
        </div>
        <div class="refresh-btn" :class="{'refresh-btn-ing':refreshBtnIng}">
            <i class="iconfont icon-shuaxin cursor-pointer" @click="refreshFn"></i>
        </div>
        <div class="to-top cursor-pointer">
            <i class="iconfont" title="返回顶部" @click.stop="scroll">&#8593;</i>
        </div>
        <setter
            :class="['setter-content',osType=='mac'?'setter-content-mac':'']"
            :show.sync="setterShow"
            @imageSourceChange="imageSourceChange"
            :getDataFlag="getDataFlag">
        </setter>

    </div>
</template>

<script>
// 在渲染器进程 (网页) 中。
import { mapState, mapActions } from 'vuex'
import Waterfall from 'vue-waterfall/lib/waterfall'
import WaterfallSlot from 'vue-waterfall/lib/waterfall-slot'
import * as axios from 'axios'

import { osType } from '../../utils/utils'
import { version } from '../../../package'
import setter from './setter'
import swProgress from './progress'

const { shell } = require('electron')
const os = require('os')
const osu = require('node-os-utils')
const macaddress = require('macaddress')
const md5 = require('../../utils/md5').md5_32

const { postRegister, apiStatisticActive } = require('../../api/api')
const { mkdirSync } = require('../../file/file')

const INFOSHOW = {
    loading: '念念不忘，必有回响...',
    noData: '暂时没有得到想要的内容...',
    netError: '&nbsp &nbsp &nbsp &nbsp网络暂时发生了错误，请求不到数据了。可能是网络没有正常连接，请确保网络已连接。'
    + '也可能是所选择图库相关接口已修改，在设置中换个图库试试',
    null: ''
}

export default {
    name: 'mainContent',
    components: {
        setter,
        swProgress,
        Waterfall,
        WaterfallSlot,
    },
    data() {
        return {
            currentMouseOverIndex: -1, // 当前鼠标在那个图片上
            currentWallpaperIndex: 0, // 当前壁纸的索引
            page: 0, // 请求数据的页数
            progressValue: 0, // 进度值
            searchKey: '', // 搜索关键字
            setterShow: false, // 是否显示设置
            isSetting: false, // 是否正在设置壁纸
            havaDataFlag: true, // 标记是否还有数据
            getDataFlag: false, // 标记页面是否正在请求数据
            searchKeyFocus: false, // 标记当前搜索框是否正在焦点中
            refreshBtnIng: false, // 是否正在刷新
            images: [], // 图片列表
            osType, // 系统类型
            imageSource: 'pexels', // 图片来源
            currentImageBacColor: '#fff', // 进度条的颜色
            infoShow: INFOSHOW.loading, // 相关提示信息
            paperClass: [], // paper的分类,
            sign: '',
            date: ''
        }
    },

    computed: mapState({
        config: state => state.main.config
    }),

    mounted() {
        // 安装量的统计
        this.firstInstall()
        this.imageSource = this.$localStorage.getStore('userConfig').imageSource || 'pexels'
        this.searchKey = this.$localStorage.getStore('searchKey')
        this.images = []
        this.cleartLocalStorage()
        if (this.imageSource === 'paper'){
            this.paperInit()
        }
        this.getDailySign();
        this.getData()
        this.eventInit()
        this.date = new Date().toDateString();
    },

    methods: {
        ...mapActions([
            'changeOsInfoStore',
        ]),
        quit() {
            this.$ipcRenderer.send('btn', {
                type: 'quit',
                data: ''
            })
        },
        scroll() {
            const el = document.querySelectorAll('.content')[0]
            console.log(el)
            el.scrollTo(0, 0)
        },
        /**
         * 清除因版本更新后不再使用字段
         */
        cleartLocalStorage(params) {
            this.$localStorage.removeStore('first_install_flag')
            this.$localStorage.removeStore('first_install_flag_v1.1')
        },

        /**
         * 主进程过来的消息事件注册
         * @function eventInit
         */
        eventInit() {
            /**
             * 设置壁纸完成事件
             */
            this.$ipcRenderer.on('dataWallpaper', (event, arg) => {
                // 设置一个时间记录最后更新的时间
                if (arg === 'success') {
                    this.currentWallpaperIndex = this.currentWallpaperIndex + 1
                }
                this.$localStorage.setStore('lastUpdataTime', parseInt((new Date()).getTime() / 1000, 10))
                this.isSetting = false
                this.progressValue = 0
                this.$swLoading.close()
            })
            /**
             * 设置下载完成事件
             */
            this.$ipcRenderer.on('downloadImg', (event, arg) => {
                // 设置一个时间记录最后更新的时间
                this.isSetting = false
                this.progressValue = 0
                this.$swLoading.close()
            })
            /**
             * 定时器
             */
            this.$ipcRenderer.on('intervalTime', (event, arg) => {
                this.wallpaperAuto()
                this.firstInstall()

                const nowDate = parseInt((new Date()).getTime() / 1000, 10)
                const statisticTimeFlag = this.$localStorage.getStore('statisticTimeFlag')
                if (!statisticTimeFlag) {
                    this.$localStorage.setStore('statisticTimeFlag', nowDate)
                }
                const timingWipeDataFlag = this.$localStorage.getStore('timingWipeDataFlag')
                if (!timingWipeDataFlag) {
                    this.$localStorage.setStore('timingWipeDataFlag', nowDate)
                }

                // 2小时 统计日活
                if (nowDate - statisticTimeFlag >= 2 * 60 * 60) {
                    apiStatisticActive({
                        uid: this.$localStorage.getStore('osInfoUid'),
                        version
                    })
                    this.$localStorage.setStore('statisticTimeFlag', nowDate)
                }
                // 7天 自动清除已下载
                // if (nowDate - timingWipeDataFlag >= 7 * 24 * 60 * 60) {
                //     this.$localStorage.setStore('timingWipeDataFlag', nowDate)
                //     // 删除默认文件下的所有内容
                //     this.$ipcRenderer.send('btn', {
                //         type: 'deleteFile',
                //         data: this.config.downloadImagePath
                //     })
                // }
            })

            /**
             * 数据相关事件
             */
            this.$ipcRenderer.on('datainfo', (event, { type = '', data }) => {
                // 获得了接口列表
                if (type === 'urls') {
                    this.getDataFlag = false
                    this.infoShow = INFOSHOW.null
                    this.refreshBtnIng = false
                    if (data.length === 0) {
                        this.havaDataFlag = false
                        this.infoShow = INFOSHOW.noData
                        return
                    }
                    if (this.page === 0) {
                        this.images = []
                    }
                    this.urlsDeal(data)
                }
                else if (type === 'urlsError'){
                    this.refreshBtnIng = false
                    this.getDataFlag = false
                    this.infoShow = INFOSHOW.netError
                }
                // 主窗口显示|隐藏
                else if (type === 'windowShow') {
                    if (data) {
                        this.setterShow = false
                    } else {
                        this.setterShow = false
                    }
                }
                // 更新进度条
                else if (type === 'updaterProgress') {
                    this.progressValue = data
                    if (this.progressValue >= 100) {
                        const time = setTimeout(() => {
                            clearTimeout(time)
                            this.progressValue = 0
                        }, 1000)
                    }
                }
            })
            this.$ipcRenderer.on('sign', (event, { type = '', data }) => {
                this.sign = data
            })
        },

        /**
         * enter按键 搜索
         * @function keydownEnterFn
         */
        keydownEnterFn() {
            if (this.searchKeyFocus) {
                this.searchKeyFn()
            }
        },
        /**
         * 检测首次安装
         * @function firstInstall
         */
        firstInstall() {
            // 第一次注册
            function getMacAddress() {
                return new Promise((resolve, reject) => {
                    macaddress.one((err, mac) => {
                        if (err) {
                            reject()
                        } else {
                            resolve(mac)
                        }
                    });
                })
            }
            if (this.$localStorage.getStore('first_install_flag_v1.1.1') !== 'dailywallpaper') {
                Promise.all([osu.osCmd.whoami(), osu.os.oos(), osu.os.arch(), getMacAddress()]).then((result) => {
                    const [userName, oss, arch, mac] = result
                    const time = (new Date()).getTime()
                    const data = {
                        username: userName.replace('\n', '').replace('\r', ''), // 用户名
                        version, // 软件版本
                        uid: md5(`${userName}${oss}${arch}${mac}`), // 软件唯一ID,
                    }
                    postRegister(data).then((res) => {
                        this.changeOsInfoStore(data)
                        this.$localStorage.setStore('osInfo', data)
                        this.$localStorage.setStore('osInfoUid', data.uid)
                        this.$localStorage.setStore('first_install_flag_v1.1.1', 'dailywallpaper')
                    })
                })
            }
        },

        /**
         * 返回随机颜色值
         * @function randomColor
         */
        randomColor() {
            return `#${Math.floor(Math.random() * 0xffffff).toString(16)}`
        },

        /**
         * 设置壁纸
         * @function setWallpaper
         * @param {Object} img 当前图片数据
         * @param {Number} index 数组索引
         */
        setWallpaper(img, index) {
            this.isSetting = true
            this.setterShow = false
            if (!this.$refs[`image_item_${index}`][0]) {
                return
            }
            this.$swLoading.open(this.$refs[`image_item_${index}`][0])
            this.$ipcRenderer.send('downloadImg', img)
            this.currentImageBacColor = this.images[index].backgroundColor
            this.currentWallpaperIndex = index
        },
        /**
         * 打开图片存的文件夹
         * @function openDownloadFile
         */
        openDownloadFile() {
            // 判断是否有文件夹
            mkdirSync(this.config.downloadImagePath)
            shell.openItem(this.config.downloadImagePath)
        },

        /**
         * 判断图片质量
         * @function imageTip
         * @param {Number} width 宽
         * @param {Number} height 高
         */
        imageTip(width, height) {
            const val = parseInt(width, 10) * parseInt(height, 10)
            if (val > 5120 * 2880) {
                return '5k'
            }
            if (val > 4096 * 2160) {
                return '4k'
            }
            if (val > 1920 * 1080) {
                return '2k'
            }
            if (val === 1920 * 1080) {
                return '1080'
            }
            return '高清'
        },

        /**
         * 判断图片是横图还是竖图
         * @function imageDirection
         */
        imageDirection(width, height) {
            return !(width >= height)
        },

        /**
         * 刷新
         * @function refreshFn
         */
        refreshFn() {
            if (this.refreshBtnIng === false) {
                this.destroyAll()
                this.page = 0
                this.images = []
                this.getData()
            }
            this.isSetting = false
            this.refreshBtnIng = true
            // 设置一个时间记录最后更新的时间
            this.$localStorage.setStore('lastUpdataTime', parseInt(new Date().getTime() / 1000, 10))
            this.isSetting = false
            this.$swLoading.close()
        },

        /**
         * 中断所有的请求
         * @function destroyAll
         */
        destroyAll() {
            this.progressValue = 0
            this.isSetting = false
            this.setterShow = false
            this.$swLoading.close()
            this.$ipcRenderer.send('cancelAllRequest', true) // 取消所有请求
        },

        /** 壁纸自动更新 */
        wallpaperAuto() {
            const userConfig = this.$localStorage.getStore('userConfig')
            // 如果正在设置,则弹出去
            if (this.isSetting === true || this.images.length === 0) {
                return
            }
            // 自动设置壁纸的时候还剩下5张就请求下一页
            if (this.havaDataFlag && this.currentWallpaperIndex === this.images.length - 5) {
                this.page = this.page + 1
                this.getData()
            }
            if (userConfig.wallpaperAutoUp === true) {
                if (this.$localStorage.getStore('lastUpdataTime')) {
                    const currentTime = parseInt(new Date().getTime() / 1000, 10)
                    // eslint-disable-next-line no-restricted-globals
                    if (!isNaN(parseInt(userConfig.updataTime, 10))) {
                        let time = this.$localStorage.getStore('lastUpdataTime')
                        time = parseInt(time, 10)
                        const updataTime = parseInt(userConfig.updataTime, 10)
                        if (Math.abs(currentTime - time) > updataTime) {
                            const index = this.images[this.currentWallpaperIndex] ? this.currentWallpaperIndex : 0
                            this.setWallpaper(this.images[index], index)
                        }
                    }
                }
            }
        },

        /**
         * 搜索按钮
         * @function searchKeyFn
         */
        searchKeyFn() {
            if (this.searchKey === this.$localStorage.getStore('searchKey')) {
                return
            }
            this.$localStorage.setStore('searchKey', this.searchKey)
            this.destroyAll()
            this.images = []
            this.getData()
        },

        /**
         * 改变图片来源
         * @function imageSourceChange
         */
        imageSourceChange(val) {
            this.destroyAll()
            this.imageSource = val
            this.page = 0
            this.images = []
            window.setTimeout(() => {
                this.getData()
            }, 100)
        },

        /**
         * 对获取到的地址进行处理
         * @function urlsDeal
         */
        urlsDeal(urls) {
            urls.forEach((e) => {
                const obj = {
                    url: e.url,
                    name: '',
                    tip: this.imageTip(e.width, e.height),
                    directionColumn: this.imageDirection(e.width, e.height),
                    downloadUrl: e.downloadUrl,
                    width: Number(e.width),
                    height: Number(e.height),
                    backgroundColor: this.randomColor()
                }
                this.images.push(obj)
            })
        },

        /**
         * 滚动条事件,请求下一页
         * @function contentScroll
         * @param {Object} event 事件
         */
        contentScroll(event) {
            const el = event.srcElement || event.target
            if (this.havaDataFlag === true && this.getDataFlag === false) {
                if (el.scrollTop + 1800 > el.querySelector('.content-main').clientHeight) {
                    this.page = this.page + 1
                    this.getData()
                }
            }
        },

        /**
         * 获取数据接口
         * @function getData
         */
        async getData() {
            this.getDataFlag = true
            this.infoShow = INFOSHOW.loading
            const obj = {
                searchKey: this.searchKey,
                page: this.page,
                imageSource: this.imageSource
            }
            this.$ipcRenderer.send('getImageUrls', obj)
        },
        async getDailySign() {
            this.$ipcRenderer.send('getSign', null)
        },

        handleMouseLeave(){
            if (this.setterShow){
                this.setterShow = false
            }
        },

        paperInit(){
            if (this.paperClass.length === 0){
                // 发送同步消息，主进程通过returnValue返回 [注意：同步消息会阻塞渲染进程，会阻塞。也就是在此期间渲染进程什么都干不了！！干不了！！]
                this.paperClass = this.$ipcRenderer.sendSync('runFunc', 'getPaperSetting')
                if (this.paperClass.length){
                    this.searchKey = this.paperClass[0].value
                }
                else {
                    window.setTimeout(() => {
                        this.searchKey = ''
                        this.refreshBtnIng = false
                        this.getDataFlag = false
                        this.infoShow = INFOSHOW.netError
                    }, 100)
                }
            }
        }
    },
    watch: {
        imageSource(val) {
            if (val === 'paper') {
                this.paperInit()
            } else {
                this.searchKey = ''
                this.$localStorage.setStore('searchKey', this.searchKey)
            }
        },
    }
}
</script>

<style lang="less" scoped>
.setter-content {
    top: 45px;

    &.setter-content-mac {
        top: 55px;
    }
}

.main-content {
    // padding-top: 30px;
    width: 100%;
    height: 100%;
    overflow: hidden;
    box-sizing: border-box;
    position: relative;

    background-color: transparent;

    // background-color: red;
    .cursor-pointer{
        cursor: pointer;
    }
    .header {
        position: fixed;
        width: 100%;
        height: 96px;
        z-index: 3000;
        padding-left: 20px;
        padding-right: 20px;
        background-color: rgba(34, 34, 34, 0.9);
        overflow: hidden;

        .header-row-one {
            display: flex;
            width: 100%;
            height: 56px;
            justify-content: space-between;

            .left {
                position: relative;
                width: 100%;
                height: 100%;

                .text {
                    color: rgba(255, 255, 255, 0.9);
                    cursor: default;
                    user-select: none;
                    // z-index: 2;
                }
            }
        }

        .iconfont {
            color: #fff;
            margin-left: 10px;
        }

        .left {
            flex: auto;
        }

        .right {
            flex: none;
            display: flex;
            align-items: center;
        }

        .header-set {
            position: relative;
        }

        .header-search {
            width: 100%;
            display: flex;
            align-items: center;
            position: relative;

            .header-search-input {
                width: 100%;
                flex: none;
            }

            .iconfont {
                position: absolute;
                right: 5px;
            }
        }
    }

    .header::before {
        content: "";
        display: block;
        position: absolute;
        top: 0px;
        left: 0px;
        width: 100%;
        height: 100%;
        filter: blur(50px);
        /* opacity: 0.9; */
        background-color: rgba(37, 31, 30, 0.9);
    }

    .image-item-img-first {
        margin-top: 95px;
    }

    .content {
        width: calc(~"100% + 15px");
        height: 100%;
        overflow-x: hidden;
        overflow-y: scroll;
        padding: 1px;
        background-color: #222222;
        border-radius: 5px;
        .sign{
            width: 100%;
            display: flex;
            padding: 20px 0;
            /*position: absolute;*/
            /*z-index: 10;*/
            /*left: 0;*/
            /*top: 0;*/
            align-items: center;
            flex-direction: column;
            justify-content: center;
            p{
                padding:0 20px;
                width: 1200px;
                font-size: 24px;
                font-weight: 500;
                color: white;
                line-height: 34px;
            }
        }

        .image-item {
            width: 100%;
            /*height: 180px;*/
            height: 100%;
            position: relative;
            display: flex;
            align-items: center;
            justify-content: center;
            border-bottom: 1px solid #bbb;
            .image-item-img {
                width: 100%;
                height: 100%;
                overflow: hidden;
            }
            .image-set-wallpaper-box{
                position: absolute;
            }
            .image-set-wallpaper {
                background-color: rgba(0, 0, 0, 0.4);
                width: auto;
                height: 33px;
                margin-bottom: 15px;
                width: 120px;
                line-height: 33px;
                cursor: default;
                color: #fff;
                text-align: center;
                border-radius: 15px;

                .iconfont {
                    margin-right: 8px;
                }
            }

            .image-set-wallpaper:hover {
                background-color: rgba(0, 0, 0, 0.7);
            }

            .image-item-flag {
                position: absolute;
                top: 10px;
                right: 14px;
                display: flex;
                justify-content: flex-end;
                width: auto;
                height: 26px;

                .image-item-flag-direction {
                    width: 26px;
                    height: 26px;
                    border-radius: 4px;
                    color: #fff;
                    background-color: rgba(0, 0, 0, 0.6);
                    line-height: 26px;
                    text-align: center;
                    cursor: default;
                    font-size: 12px;
                    margin-right: 10px;
                }

                .image-item-tip {
                    width: 26px;
                    height: 26px;
                    border-radius: 4px;
                    color: #52b7fc;
                    background-color: rgba(0, 0, 0, 0.6);
                    line-height: 26px;
                    text-align: center;
                    cursor: default;
                    font-size: 12px;
                }
            }

            .image-item-tip {
                width: 26px;
                height: 26px;
                border-radius: 4px;
                color: #52b7fc;
                background-color: rgba(0, 0, 0, 0.6);
                line-height: 26px;
                text-align: center;
                cursor: default;
                font-size: 12px;
            }
        }
        .is-loading{
            width: 100%;
            height: 40px;
            color: #fff;
            font-size: 12px;
            display: flex;
            align-items: center;
            justify-content: center;
            i{
                font-size: 16px;
                margin-right: 5px;
            }
        }
    }

    .content-win {
        width: calc(~"100% + 17px");
    }

    .content-main-no {
        display: flex;
        align-items: center;
        justify-content: center;
        width: 100%;
        height: 100%;
        padding: 20px;
        line-height: 20px;
        color: #ccc;
        font-size: 12px;
    }
}

.refresh-btn {
    position: fixed;
    z-index: 999;
    left: 14px;
    bottom: 8px;
    color: #fff;

    .iconfont {
        font-size: 24px;
    }
}
.to-top {
    position: fixed;
    z-index: 999;
    right: 14px;
    color: white;
    bottom: 8px;
    .iconfont {
        font-size: 24px;
    }
}
.refresh-btn-ing {
    animation: refreshbtning 1.5s linear infinite;
    transform-origin: center center;
    transform: rotate(360deg);
}

@keyframes refreshbtning {
    0% {
        transform: rotate(360deg);
    }

    100% {
        transform: rotate(0);
    }
}
</style>
