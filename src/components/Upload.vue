<template>
    <div id="upload" class="upload">
        <el-container>
            <el-header height="45px"> <p class="title">Perma Bookmark</p> </el-header>
            <el-main border-radius:2px>
                <el-input
                    type="textarea"
                    :autosize="{ minRows: 2, maxRows: 3}"
                    :disabled="operationDisable"
                    placeholder=""
                    v-model="title">
                </el-input>
                <div style="margin: 10px 0;"></div>
                <el-button type="primary" size="small" icon="el-icon-collection" v-show="connected" @click="onDownloadPageV2" :disabled="operationDisable" plain>创建永久书签</el-button>
                <br/>
                <el-link icon="el-icon-link" :disabled="operationDisable">{{arUrl}}</el-link>
            </el-main>
            <el-footer height="30px">
                <i :class="(operationDisable?'el-icon-loading':'el-icon-s-help')"></i>
                {{process}}
            </el-footer>
        </el-container>
    </div>
  </template>
  
  <script>
  import { genNodeAPI } from 'arseeding-js'

  export default {
    name: 'Upload',
    data() {
        return {
            downloadUrl: "url",
            filename: "filepath",
            fileInfo: {},
            title: "",
            arUrl: "永久地址",
            signerAddr: "0x0",
            process: "未开始",
            connected: true,
            operationDisable: false
        }
    },
    mounted () {
      browser.runtime.sendMessage({})
      chrome.tabs.getSelected(null, (tab) => { // 先获取当前页面的tabID
            this.downloadUrl = tab.url
            this.title = tab.title
      })
    },
    methods: {
        async onDownloadPage(){
            // alert(`开始下载网页:\n ${this.downloadUrl}`);
            this.operationDisable = true
            this.process = `开始下载...`
            var cur_id
            // 下载页面
            chrome.downloads.download(
                {url: this.downloadUrl}, 
                function(id) {
                    cur_id = id
                }
            )
            // 下载完成
            chrome.downloads.onChanged.addListener((delta) => {
            if (!delta.state || (delta.state.current != 'complete')) {
                    return;
                }
                if (delta.id != cur_id) {
                    return;
                }
            
                //chrome.downloads.open(delta.id);
                // 获取下载文件路径
                chrome.downloads.search(
                    {id: delta.id}, 
                    (results) => {       
                        this.fileInfo = results[0]
                        this.process = `下载成功, 开始上传...`
                        this.filename = results[0].filename
                        this.uploadToAr(this.downloadUrl)
                    }
                )
            }); 
        },
        async onDownloadPageV2() {
            this.operationDisable = true
            this.process = `开始下载...`

            this.makeRequest((data) => {
                                this.process = "下载完成"
                                this.uploadToAr(data)
                            })
        },
        async makeRequest(callback) {
            var xhr = new XMLHttpRequest();
            xhr.open('GET', this.downloadUrl);
            //xhr.responseType = "arraybuffer"
            xhr.addEventListener('load', function(e) {
                var result = xhr.responseText;
                var result = xhr.response
                callback(result);
            });
            xhr.addEventListener('error', function() {
                alert("get error")
                this.process = "Download failed!"
                this.operationDisable = false
            });
            xhr.send();
        },
        async uploadToAr(fileData) {
            const pk_metamask_ardev = 'your own pk'
            const instance = genNodeAPI(pk_metamask_ardev)

            const arseedUrl = 'https://arseed.web3infra.dev'
            const payCurrency = 'usdc' // everpay supported all tokens

            const ops = {
                tags: [{name: "Content-Type",value:'text/html'}]
            }

            this.process = `上传到Arweave...`
            const res = await instance.sendAndPay(arseedUrl, fileData, payCurrency, ops)
            //alert(JSON.stringify(res))
            this.arUrl = `https://arseed.web3infra.dev/${res.order.itemId}`
            this.process = `上传完毕!`

            this.creatBookmark()

            this.operationDisable = false
        },
        async creatBookmark() {
            chrome.bookmarks.create({
                title: "PermaBookmark: " + this.title,
                url: this.arUrl
            })
            this.process = "永久书签已保存！请在浏览器书签查看"
        },
        async onConnectWallet() {
            this.connected = true
            this.signerAddr = "0x123"
            // if (typeof window.ethereum !== 'undefined') {
            //     await window.ethereum.enable()
            //     this.signerAddr = window.ethereum.selectedAddress
            //     this.connected = true  
            // } else {
            //     alert("Can't find metemask")
            //     const pk_metamask_ardev = 'xxxx'
            //     const instance = await genNodeAPI(pk_metamask_ardev)

            //     const arseedUrl = 'https://arseed.web3infra.dev'
            //     const payCurrency = 'usdc' // everpay supported all tokens

            //     const ops = {
            //         tags: [{name: "Content-Type",value:'data type'}]
            //     }

            //     const res = await instance.sendAndPay(arseedUrl, "Hello arseeding and chrome!", payCurrency, ops)
            //     alert('https://arseed.web3infra.dev/%s', res.order.itemId)
            // }
        }
    }
  }
  </script>
  
  <style scoped>
  p {
    font-size: 20px;
  }
  .el-header {
    background-color: #6fa7f1ea;
    color: #333;
    text-align: center;
    left:0%;
    right:0%;
  }
  .el-footer {
    background-color: #d6d9ddc6;
    color: #333;
    text-align: center;
    line-height: 30px;
  }
  .el-main {
    background-color: #ffffff;
    color: #333;
    text-align: center;
    line-height: 30px;
    padding-top:5px;
    padding-bottom:5px;
    padding-right:5px;
    padding-left:5px;
  }
  .bg-purple-dark {
    background: #72aaf3;
  }
  .bg-purple {
    background: #a9cbf1;
  }
  .bg-purple-light {
    background: #e5e9f2;
  }
  .text {
    font-size: 14px;
  }

  .item {
    margin-bottom: 18px;
  }

  .title {
    font-size: 18px;
    color:#FFFFFF;
    vertical-align:middle;
    margin:10px
  }
  .el-textarea {
    font-size: 10px;
  }

  .el-link {
    font-size: 8px;
    color:#161e31;
    line-height: 100%;
    text-align: left;
  }

  .upload {
    left:0%;
    right:0%;
    height: 235px;
  }

  </style>
  