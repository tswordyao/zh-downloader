<template>
    <el-row>
        <template v-if="playlist.length===0">
            <p class="error-message nothing">\_(ツ)_/¯</p>
            <p class="error-message">没有嗅探到知乎视频.</p>
        </template>
        <template v-else>
            <el-alert v-if="isDownloading" title="视频下载过程中请勿关闭本页面,若下载格式为MP4,转换过程可能会相当耗时,请耐心等待" type="warning"></el-alert>
            <el-table :data="playlist" style="width:100%" max-height="300">
                <el-table-column type="expand" fixed>
                    <template slot-scope="props">
                        <el-form label-position="left" inline class="table-expand">
                            <el-form-item label="视频ID:">
                                <span>
                                    <el-tooltip class="item" effect="dark" content="点击可直接进入知乎播放" placement="bottom">
                                        <a class="link" :href="'https://www.zhihu.com/video/'+props.row.id" target="_blank">{{props.row.id}}</a>
                                    </el-tooltip>
                                </span>
                            </el-form-item>
                            <el-form-item label="视频名称:">
                                <span>{{ props.row.name }}</span>
                            </el-form-item>
                            <el-form-item label="M3U8地址:">
                                <span>
                                    <a class="link" :href="props.row.playlist[props.row.currentQuality].m3u8" target="_blank">点击查看</a>
                                </span>
                            </el-form-item>
                            <el-form-item label="视频长度:">
                                <span>{{ props.row.playlist[props.row.currentQuality].duration | msToTime}}</span>
                            </el-form-item>
                            <el-form-item label="清晰度:">
                                <span>{{ qualityMap[props.row.playlist[props.row.currentQuality].quality] || '未知'}}</span>
                            </el-form-item>
                            <el-form-item label="视频大小:">
                                <span>{{ props.row.playlist[props.row.currentQuality].size | bytesToSize}}</span>
                            </el-form-item>
                        </el-form>
                    </template>
                </el-table-column>
                <el-table-column label="缩略图">
                    <template slot-scope="scope">
                        <img class="thumbnail" :src="scope.row.thumbnail" />
                    </template>
                </el-table-column>
                <el-table-column prop="name" label="视频名字">
                </el-table-column>
                <el-table-column label="清晰度">
                    <template slot-scope="scope">
                        <el-select v-model="scope.row.currentQuality" size="small">
                            <el-option v-for="(item, key) in scope.row.playlist" :key="key" :label="qualityMap[item.quality] || item.quality" :value="item.quality">
                            </el-option>
                        </el-select>
                    </template>
                </el-table-column>
                <el-table-column label="视频格式">
                    <template slot-scope="scope">
                        <el-select v-model="scope.row.playlist[scope.row.currentQuality].format" size="small">
                            <el-option label="MPEG2-TS" value="ts"></el-option>
                            <el-option label="MP4" value="mp4"></el-option>
                        </el-select>
                    </template>
                </el-table-column>
                <el-table-column label="下载进度" width="80">
                    <template slot-scope="scope">
                        <el-tooltip>
                            <div slot="content">{{isDownloading && progressMessage && scope.row.id === downloadingVedioId ? progressMessage:
                                '点击右边下载按钮即会自动更新此进度'}}
                            </div>
                            <el-progress :percentage="progressValue(scope.row)" type="circle" :width=40 color="#8e71c7"></el-progress>
                        </el-tooltip>
                    </template>
                </el-table-column>
                <el-table-column fixed="right" label="操作" width="100">
                    <template slot-scope="scope">
                        <el-tooltip class="item" effect="dark" content="下载" placement="bottom">
                            <el-button @click="handleDownloadVideo(scope.row)" type="text" :disabled="isDownloading" icon="el-icon-download"></el-button>
                        </el-tooltip>

                        <el-tooltip class="item" effect="dark" content="删除" placement="bottom">
                            <el-button @click="handleDeleteVideo(scope.row)" type="text" :disabled="isDownloading" icon="el-icon-delete"></el-button>
                        </el-tooltip>

                        <el-tooltip v-if="progressValue(scope.row) === 100" class="item" effect="dark" content="打开文件所在位置" placement="bottom">
                            <el-button @click="handleShowFile(scope.row)" type="text" icon="el-icon-view"></el-button>
                        </el-tooltip>
                    </template>
                </el-table-column>
            </el-table>
        </template>
    </el-row>
</template>




<script>
import { mapGetters } from 'vuex';

import { ADD_OR_UPDATE_VIDEO, ADD_OR_UPDATE_DOWNLOAD_INFO } from '../store/mutation-types';

export default {
  name: 'Playlist',
  data() {
    return {
      downloadingVedioId: 0,
      progressMessage: '',
      isDownloading: false,
    };
  },
  props: ['qualityMap'],
  computed: {
    ...mapGetters(['playlist']),
  },
  methods: {
    /**
     * 来自父组件通知的信息,当后台某个视频被采集到的消息
     *
     * @param object progressInfo 下载的进度信息,因下载格式不同而不同,但一定包含msg字段
     */
    onAddVideo(newVideoInfo) {
      this.$store.commit(ADD_OR_UPDATE_VIDEO, newVideoInfo);
    },

    /**
     * 来自父组件通知的信息,当后台某个视频正在下载的通知.
     *
     * @param object progressInfo 下载的进度信息,因下载格式不同而不同,但一定包含msg字段
     */
    onProgressUpdate(progressInfo) {
      this.progressMessage = progressInfo.msg || '';
    },

    /**
     * 来自父组件通知的信息,当后台某个视频下载结束之后通知.
     *
     * @param object downloadInfo 下载信息.
     */
    onFinishedDownload(downloadInfo) {
      this.isDownloading = false;
      this.progressMessage = '';
      if (downloadInfo.error) {
        this.$message({
          showClose: true,
          message: downloadInfo.error,
          type: 'error',
        });
        return;
      }
      this.$store.commit(ADD_OR_UPDATE_DOWNLOAD_INFO, downloadInfo);
      this.downloadFile(downloadInfo.link, downloadInfo.name);
    },

    /**
     * 指定文件名下载资源.
     *
     * @param string link     下载链接
     * @param string filename 下载文件名,如果不给则是当前时间结尾的mp4格式
     */
    downloadFile(link, filename = undefined) {
      var a = document.createElement('a');
      a.href = link;
      a.download = filename || new Date().getTime() + '.mp4';
      a.click();
    },

    /**
     * @param object videoInfo 需要查看进度的视频信息
     */
    progressValue(videoInfo) {
      // 判断是否对应Id的视频
      const selectedVideoInfo = this.$store.state.downloadInfo[videoInfo.id];
      if (!selectedVideoInfo) {
        return 0;
      }
      // 判断是否有对应清晰度的视频
      const selectedQualityInfo = selectedVideoInfo[videoInfo.currentQuality];
      if (!selectedQualityInfo) {
        return 0;
      }
      // 判断是否有对应数据格式的视频
      const selectedFormat = videoInfo.playlist[videoInfo.currentQuality].format;
      if (!selectedQualityInfo[selectedFormat]) {
        return 0;
      }
      // 返回该格式的进度
      return selectedQualityInfo[selectedFormat].progress || 0;
    },

    /**
     * 处理下载按钮事件
     *
     * @param object videoInfo 需要下载的视频信息
     */
    handleDownloadVideo(videoInfo) {
      this.isDownloading = true;
      this.downloadingVedioId = videoInfo.id;
      this.$emit('download', videoInfo);
    },

    /**
     * 处理删除按钮事件
     *
     * @param object videoInfo 需要删除的视频信息
     */
    handleDeleteVideo(videoInfo) {
      this.$store.dispatch('deleteVideo', videoInfo);
      this.$message({
        showClose: true,
        message: ' (✌ﾟ∀ﾟ)☞ 删除成功',
        type: 'success',
      });
      // 通知父组件
      this.$emit('delete', videoInfo);
    },

    /**
     * 获取在Chrome中下载文件的下载信息
     *
     * 由于下载文件用的chrome API无法修改文件名(filename没有用), 因此下载的时候是使用a标签
     * 这是拿不到chrome需要的downloadId的，所以只能通过搜索接口取检索到downloadId,然后才能打开.
     *
     * See: https://developer.chrome.com/extensions/downloads#method-search
     * See: https://developer.chrome.com/extensions/downloads#method-show
     *
     * @param {object} videoInfo 文件信息
     */
    getDownloadInfoByVideoInfo(videoInfo) {
      const selectedVideoInfo = this.$store.state.downloadInfo[videoInfo.id];
      if (!selectedVideoInfo) {
        return;
      }
      // 判断是否有对应清晰度的视频
      const selectedQualityInfo = selectedVideoInfo[videoInfo.currentQuality];
      if (!selectedQualityInfo) {
        return;
      }
      // 判断是否有对应数据格式的视频
      const selectedFormat = videoInfo.playlist[videoInfo.currentQuality].format;
      if (!selectedQualityInfo[selectedFormat]) {
        return;
      }
      // 获取下载链接.
      const link = selectedQualityInfo[selectedFormat].link;

      if (!link) {
        return;
      }

      return new Promise((resolve, reject) => {
        chrome.downloads.search(
          {
            finalUrl: link,
          },
          downloadItems => {
            const downloadInfo = downloadItems[0] && downloadItems[0];
            resolve(downloadInfo);
          }
        );
      });
    },

    /**
     * 查看 Chrome 下载的文件所在位置(不是使用程序直接打开文件)
     * 由于download信息不是实时更新的,因此可能出现downloadInfo.exists误判的情况
     * 这时候是没有响应的
     */
    async handleShowFile(videoInfo) {
      const downloadInfo = await this.getDownloadInfoByVideoInfo(videoInfo);
      if (downloadInfo && downloadInfo.exists) {
        chrome.downloads.show(downloadInfo.id);
      } else {
        this.$message({
          showClose: true,
          message: 'ಥ﹏ಥ 文件不存在,请尝试重新下载!',
          type: 'error',
        });
      }
    },
  },

  filters: {
    /**
     * 毫秒->人类友好的H:M:S格式
     */
    msToTime(s) {
      // https://stackoverflow.com/questions/9763441/milliseconds-to-time-in-javascript
      var pad = (n, z = 2) => ('00' + n).slice(-z);
      return pad((s / 3.6e6) | 0) + ':' + pad(((s % 3.6e6) / 6e4) | 0) + ':' + pad(((s % 6e4) / 1000) | 0);
    },
    /**
     * 比特转化到Mb/TB
     */
    bytesToSize(bytes) {
      var sizes = ['Bytes', 'KB', 'MB', 'GB', 'TB'];
      if (bytes === 0) return 'n/a';
      var i = parseInt(Math.floor(Math.log(bytes) / Math.log(1024)));
      if (i === 0) return bytes + ' ' + sizes[i];
      return (bytes / Math.pow(1024, i)).toFixed(1) + ' ' + sizes[i];
    },
  },
};
</script>

<style>
.nothing {
  font-size: 36px;
}

.error-message {
  text-align: center;
}

.thumbnail {
  width: 128px;
  max-height: 72px;
  border-radius: 2px;
}

.table-expand {
  font-size: 0;
}

.table-expand label {
  width: 90px;
  color: #cbcbcb;
}

.table-expand .el-form-item {
  margin-right: 0;
  margin-bottom: 0;
  width: 50%;
}
</style>