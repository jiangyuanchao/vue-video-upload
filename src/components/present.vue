<template>
  <div class="hello">
    <div>
      <input type="file" class="upload" @change="doUpload" ref="inputer" multiple />
      <el-button type="primary" size="small" @click="startAll">全部开始</el-button>
      <el-button type="warning" size="small" @click="pauseAll">全部暂停</el-button>
      <el-button type="danger" size="small" @click="clearAll">全部删除</el-button>
    </div>

    <div>
      <el-table :data="tableData" border style="width: 100%">
        <el-table-column prop="id" label="ID" width="180">
        </el-table-column>
        <el-table-column prop="fileName" label="文件名" width="180">
        </el-table-column>
        <el-table-column prop="size" label="文件大小" width="180">
          <template slot-scope="scope">{{ transformSize(scope.row.size) }}</template>
        </el-table-column>
        <el-table-column prop="progress" label="进度" width="180">
          <template slot-scope="scope">
            <el-progress :text-inside="true" :stroke-width="26" :percentage="scope.row.progress"></el-progress>
          </template>

        </el-table-column>
        <el-table-column prop="progress" label="操作" width="180">
          <template slot-scope="scope">
            <el-button type="text" size="small" @click="start(scope.row.id)">开始</el-button>
            <el-button type="text" size="small" @click="stop(scope.row.id)">暂停</el-button>
            <el-button type="text" size="small" @click="remove(scope.row.id)">删除</el-button>
          </template>
        </el-table-column>
      </el-table>
    </div>
  </div>
</template>

<script>
import md5 from 'js-md5'
import PlvVideoUpload from '@polyv/vod-upload-js-sdk';

export default {
  name: 'demo',
  data() {
    return {
      videoUpload: null, // 视频上传实例
      userid: 'b6d4af75ee',//从点播后台查看获取
      secretkey: 'kfwPIfszUx',//从点播后台查看获取
      writeToken: '542f716d-cf17-4f21-8014-114946dbe95a',//从点播后台查看获取
      ptime: '', // 当前时间戳
      tableData: [] //表格数据
    }
  },
  created() {
    this.videoUpload = new PlvVideoUpload({
      region: 'line1', // (可选)上传线路, 默认line1
      events: {
        Error: (err) => { // 错误事件回调
          console.log(err);
          let errMag = `（错误代码：${err.code}）${err.message}`;
          this.$alert(errMag, '标题名称', {
            confirmButtonText: '确定',
            type: 'error',
          });
        },
        UploadComplete: () => { // 全部上传任务完成回调
          console.info('上传结束：', this.videoUpload);
          console.log(this.tableData)
          this.$message({
            message: '全部上传任务完成',
            type: 'success'
          });
          // this.clearAll()
        }
      }
    });
  },
  mounted() {
    this.autoUpdateUserData(null, this.videoUpload);
  },
  methods: {
    start(uploaderid) {// 单个上传
      console.log(uploaderid)
      this.videoUpload.resumeFile(uploaderid);
    },
    stop(uploaderid) {// 单个暂停
      console.log(uploaderid)
      this.videoUpload.stopFile(uploaderid);
    },
    remove(uploaderid) {// 单个删除
      console.log(uploaderid)
      this.videoUpload.removeFile(uploaderid);
      this.tableData = this.tableData.filter((item) => item.id !== uploaderid)
    },
    startAll() {// 全部上传
      if (this.videoUpload) {
        this.videoUpload.startAll();
      }
    },
    pauseAll() {// 全部暂停
      if (this.videoUpload) {
        this.videoUpload.stopAll();
      }
    },
    clearAll() {// 全部删除
      if (this.videoUpload) {
        this.videoUpload.clearAll();
        this.tableData = []
        this.$refs.inputer.value = ''
      }
    },
    doUpload() {// 选择文件
      let inputDOM = this.$refs.inputer; // 通过DOM取文件数据
      console.log(inputDOM.files)
      let list = Array.prototype.slice.call(inputDOM.files);
      if (list.length > 0) {
        list.forEach((file, index, arr) => {
          let fileSetting = { // 文件上传相关信息设置
            title: file.name, // 标题
            desc: 'jssdk插件上传', // 描述
            cataid: '', // 上传分类目录ID
            tag: '', // 标签
            luping: 0, // 是否开启视频课件优化处理，对于上传录屏类视频清晰度有所优化：0为不开启，1为开启
            keepsource: 0, // 是否源文件播放（不对视频进行编码）：0为编码，1为不编码
            state: '' //用户自定义信息，如果提交了该字段，会在服务端上传完成回调时透传返回。
          }
          let uploadManager = this.videoUpload.addFile(
            file, // file 为待上传的文件对象
            {
              FileStarted: this.onFileStarted,// 文件开始上传回调
              FileProgress: this.onFileProgress,// 文件上传中回调
              FileSucceed: this.onFileSucceed,// 文件上传成功回调
              FileFailed: this.onFileFailed,// 文件上传失败回调
              FileStopped: this.onFileStopped,// 文件上传停止回调
            },
            fileSetting
          );

          this.addTableData(uploadManager)
        })
      }
    },
    onFileStarted({ uploaderid, progress }) {
      console.log("文件上传开始: ", uploaderid, progress);
      this.tableData.filter((item) => item.id === uploaderid)[0].progress = 0
    },
    onFileProgress({ uploaderid, progress }) {
      let p = parseInt(progress * 100);// 上传的进度条
      console.log("文件上传中: ", uploaderid, progress);
      this.tableData.filter((item) => item.id === uploaderid)[0].progress = p

    },
    onFileSucceed({ uploaderid, progress }) {
      console.log("文件上传成功: ", uploaderid, progress);
    },
    onFileFailed({ uploaderid, progress }) {
      console.log("文件上传失败: ", uploaderid, progress);
    },
    onFileStopped({ uploaderid, progress }) {
      console.log("文件上传停止: ", uploaderid, progress);
    },
    addTableData(data) { // 增加表格数据
      let obj = {
        id: data.id,
        fileName: data.fileData.title,
        size: data.fileData.size,
        progress: 0
      }
      this.tableData.push(obj)
    },
    autoUpdateUserData(timer, videoUpload) { // 启动获取用户信息
      this.getUserData(videoUpload);
      if (timer) {
        clearTimeout(timer);
        timer = null;
      }
      timer = setTimeout(() => {
        this.autoUpdateUserData(timer, videoUpload);
      }, 3 * 50 * 1000);
    },
    getUserData() { // 获取用户详细信息
      this.ptime = new Date().getTime()
      let userData = {
        userid: this.userid,
        ptime: this.ptime,
        sign: this.getSignData().sign,
        hash: this.getSignData().hash
      };
      this.videoUpload.updateUserData(userData);
    },
    getSignData() { // 加密信息参数
      let hash = md5(this.ptime + this.writeToken)
      let sign = md5(this.secretkey + this.ptime)
      return {
        hash: hash,
        sign: sign,
      }
    },
    transformSize(bytes) {// 文件大小转换
      const bt = parseInt(bytes);
      let result;
      if (bt === 0) {
        result = '0B';
      } else {
        const k = 1024;
        const sizes = ['B', 'KB', 'MB', 'GB', 'TB', 'PB', 'EB', 'ZB', 'YB'];
        const i = Math.floor(Math.log(bt) / Math.log(k));
        if (typeof i !== 'number') {
          result = '-';
        } else {
          result = (bt / Math.pow(k, i)).toFixed(2) + sizes[i];
        }
      }
      return result;
    }
  },
}
</script>

<style scoped>

</style>
