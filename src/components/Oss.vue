<template>
  <div class="page">
    <h2>货主承运商 oss直传</h2>
    <br>
    <form name="theform">
      <input type="radio" name="myradio" value="local_name" checked="true"> 上传文件名字保持本地文件名字
      <input type="radio" name="myradio" value="random_name"> 上传文件名字是随机文件名
      <br>上传到指定目录:
      <input type="text" id="dirname" placeholder="如果不填，默认是上传到根目录" size="50">
    </form>

    <h4>您所选择的文件列表：</h4>
    <div id="ossfile">你的浏览器不支持flash,Silverlight或者HTML5！</div>

    <br>

    <div id="container">
      <a id="selectfiles" href="javascript:void(0);" class="btn">选择文件</a>
      <a id="postfiles" href="javascript:void(0);" class="btn">开始上传</a>
    </div>

    <pre id="console"></pre>

    <p>&nbsp;</p>
  </div>
</template>
<script>
export default {
  name: 'elOss',
  data () {
    return {
      region: 'oss-cn-beijing',
      bucket: 'https://keven520.oss-cn-beijing.aliyuncs.com', // 这里需要填入自己的bucket，申请完阿里云之后获得
      id: 'upload',
      baseurl: '',
      percentage: 0,
      url: '',
      urls: []
    }
  },
  mounted () {
    const accessid = 'LTAIviSWChYCOhKi'
    const accesskey = 'anLuQmxRsS5uV7oIzx4wE4dCLWvzP1'
    const host = 'https://keven520.oss-cn-beijing.aliyuncs.com'

    let g_dirname = ''
    let g_object_name = ''
    let g_object_name_type = ''
    // let now = timestamp = Date.parse(new Date()) / 1000;

    var policyText = {
      expiration: '2020-01-01T12:00:00.000Z', // 设置该Policy的失效时间，超过这个失效时间之后，就没有办法通过这个policy上传文件了
      conditions: [
        ['content-length-range', 0, 1048576000] // 设置上传文件的大小限制
      ]
    }
    // console.log('Base64: ', Base64)
    var policyBase64 = Base64.encode(JSON.stringify(policyText))
    // console.log('policyBase64: ', policyBase64)
    var message = policyBase64
    var bytes = Crypto.HMAC(Crypto.SHA1, message, accesskey, { asBytes: true })
    var signature = Crypto.util.bytesToBase64(bytes)

    // 单选
    function check_object_radio () {
      var tt = document.getElementsByName('myradio')
      for (var i = 0; i < tt.length; i++) {
        if (tt[i].checked) {
          g_object_name_type = tt[i].value
          break
        }
      }
    }

    // 上传到指定目录
    function get_dirname () {
      let dir = document.getElementById('dirname').value
      if (dir !== '' && dir.indexOf('/') !== dir.length - 1) {
        dir = dir + '/'
      }
      alert(dir)
      g_dirname = dir
      console.log('g_dirname: ', g_dirname)
    }

    // 随机数
    function random_string (len) {
      len = len || 32
      var chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678'
      var maxPos = chars.length
      var pwd = ''
      for (let i = 0; i < len; i++) {
        pwd += chars.charAt(Math.floor(Math.random() * maxPos))
      }
      return pwd
    }

    function get_suffix (filename) {
      let pos = filename.lastIndexOf('.')
      let suffix = ''
      if (pos !== -1) {
        suffix = filename.substring(pos)
      }
      return suffix
    }

    function calculate_object_name (filename) {
      if (g_object_name_type === 'local_name') {
        g_object_name += '${filename}'
      } else if (g_object_name_type === 'random_name') {
        let suffix = get_suffix(filename)
        g_object_name = g_dirname + random_string(10) + suffix
      }
      return ''
    }

    function get_uploaded_object_name (filename) {
      if (g_object_name_type === 'local_name') {
        let tmp_name = g_object_name
        tmp_name = tmp_name.replace('${filename}', filename)
        return tmp_name
      } else if (g_object_name_type === 'random_name') {
        return g_object_name
      }
    }

    function set_upload_param (up, filename, ret) {
      g_object_name = g_dirname
      if (filename !== '') {
        let suffix = get_suffix(filename)
        calculate_object_name(filename)
      }
      let new_multipart_params = {
        key: g_object_name,
        policy: policyBase64,
        OSSAccessKeyId: accessid,
        success_action_status: '200', // 让服务端返回200,不然，默认会返回204
        signature: signature
      }
      console.log('new_multipart_params: ', new_multipart_params)

      up.setOption({
        url: host,
        multipart_params: new_multipart_params
      })

      up.start()
    }

    var uploader = new plupload.Uploader({
      runtimes: 'html5,flash,silverlight,html4',
      browse_button: 'selectfiles',
      // multi_selection: false,
      container: document.getElementById('container'),
      flash_swf_url: 'lib/plupload-2.1.2/js/Moxie.swf',
      silverlight_xap_url: 'lib/plupload-2.1.2/js/Moxie.xap',
      url: 'http://oss.aliyuncs.com',

      init: {
        PostInit: function () {
          document.getElementById('ossfile').innerHTML = ''
          document.getElementById('postfiles').onclick = function () {
            set_upload_param(uploader, '', false)
            return false
          }
        },

        FilesAdded: function (up, files) {
          plupload.each(files, function (file) {
            document.getElementById('ossfile').innerHTML +=
              '<div id="' +
              file.id +
              '">' +
              file.name +
              ' (' +
              plupload.formatSize(file.size) +
              ')<b></b>' +
              '<div class="progress"><div class="progress-bar" style="width: 0%"></div></div>' +
              '</div>'
          })
        },

        BeforeUpload: function (up, file) {
          check_object_radio()
          get_dirname()
          set_upload_param(up, file.name, true)
        },

        UploadProgress: function (up, file) {
          var d = document.getElementById(file.id)
          d.getElementsByTagName('b')[0].innerHTML =
            '<span>' + file.percent + '%</span>'
          var prog = d.getElementsByTagName('div')[0]
          var progBar = prog.getElementsByTagName('div')[0]
          progBar.style.width = 2 * file.percent + 'px'
          progBar.setAttribute('aria-valuenow', file.percent)
        },

        FileUploaded: function (up, file, info) {
          if (info.status === 200) {
            // 上传成功
            document
              .getElementById(file.id)
              .getElementsByTagName('b')[0].innerHTML =
              'upload to oss success, object name:' +
              get_uploaded_object_name(file.name)
          } else {
            document
              .getElementById(file.id)
              .getElementsByTagName('b')[0].innerHTML = info.response
          }
        },

        Error: function (up, err) {
          document
            .getElementById('console')
            .appendChild(
              document.createTextNode('\nError xml:' + err.response)
            )
        }
      }
    })
    console.log('uploader.init: ', uploader.init)
    uploader.init()
  },
  methods: {
    getbaseurl () {
      // 文件夹命名
      var date = new Date()
      var year = date.getFullYear()
      var month = date.getMonth() + 1
      month = month < 10 ? '0' + month : month
      var mydate = date.getDate()
      mydate = mydate < 10 ? '0' + mydate : mydate
      this.baseurl =
        'img/' + year + '/' + year + month + '/' + year + month + mydate + '/'
    },
    btnclick () {
      var btn = document.getElementById(this.id)
      btn.click()
    },
    doUpload () {
      const _this = this
      // const urls = []
      const client = new OSS({
        region: _this.region,
        accessKeyId: 'LTAIviSWChYCOhKi', // 填入自己的id
        accessKeySecret: 'anLuQmxRsS5uV7oIzx4wE4dCLWvzP1', // 填入自己的id
        bucket: _this.bucket
      })
      _this.percentage = 0
      console.log('client: ', client)
    },
    // 随机生成文件名
    random_string (len) {
      len = len || 32
      var chars = 'ABCDEFGHJKMNPQRSTWXYZabcdefhijkmnprstwxyz2345678'
      var maxPos = chars.length
      var pwd = ''
      for (let i = 0; i < len; i++) {
        pwd += chars.charAt(Math.floor(Math.random() * maxPos))
      }
      return pwd
    }
  },
  watch: {
    url (val) {
      if (val) {
        this.urls.push(val)
      }
    }
  }
}
</script>
<style>
.btn{
        color: #fff;
	    background-color: #337ab7;
	    border-color: #2e6da4; 
	    display: inline-block;
	    padding: 6px 12px;
	    margin-bottom: 0;
	    font-size: 14px;
	    font-weight: 400;
	    line-height: 1.42857143;
	    text-align: center;
	    white-space: nowrap;
	    text-decoration: none;
	    vertical-align: middle;
	    -ms-touch-action: manipulation;
	    touch-action: manipulation;
	    cursor: pointer;
	    -webkit-user-select: none;
	    -moz-user-select: none;
	    -ms-user-select: none;
	    user-select: none;
	    background-image: none;
	    border: 1px solid transparent;
	    border-radius: 4px;
	}
	a.btn:hover{
      background-color: #3366b7;
	}
	.progress{
		margin-top:2px;
	    width: 200px;
	    height: 14px;
	    margin-bottom: 10px;
	    overflow: hidden;
	    background-color: #f5f5f5;
	    border-radius: 4px;
	    -webkit-box-shadow: inset 0 1px 2px rgba(0,0,0,.1);
	    box-shadow: inset 0 1px 2px rgba(0,0,0,.1);
	}
	.progress-bar{ 
		background-color: rgb(92, 184, 92);
		background-image: linear-gradient(45deg, rgba(255, 255, 255, 0.14902) 25%, transparent 25%, transparent 50%, rgba(255, 255, 255, 0.14902) 50%, rgba(255, 255, 255, 0.14902) 75%, transparent 75%, transparent);
		background-size: 40px 40px;
		box-shadow: rgba(0, 0, 0, 0.14902) 0px -1px 0px 0px inset;
		box-sizing: border-box;
		color: rgb(255, 255, 255);
		display: block;
		float: left; 
		font-size: 12px;
		height: 20px;
		line-height: 20px;
		text-align: center;
		transition-delay: 0s;
		transition-duration: 0.6s;
		transition-property: width;
		transition-timing-function: ease;
		width: 266.188px;
	}
</style>

