<template>
  <el-container>
    <div class="side-bar">
      <SideBar></SideBar>
    </div>
    <el-container>
      <el-header>
        <MyHeader></MyHeader>
      </el-header>
      <el-main>
        <v-app>
          <v-tabs
              background-color="white"
              color="deep-purple accent-4"
              left
          >
            <v-tab>概览</v-tab>
            <v-tab>成员</v-tab>
            <v-tab>活动</v-tab>
            <v-tab>公告</v-tab>
            <v-tab @click="getPosts($router.history.current.params.id)">论坛</v-tab>
            <v-tab v-show="hasJoined">创建新讨论</v-tab>
            <v-tab-item>
              <ClubList :clubs="curClub" :leave-club="true"
                        :check-info="false"
                        :has-joined="hasJoined"
                        style="margin-left: 350px;max-height: 250px"
              ></ClubList>
              <div
                  style="margin-left: 350px;width: 600px;"
              >
                <v-img :src="curClub[0].welcome_image">
                  <v-row
                      align="center"
                      justify="center"
                      style="margin-top: 50%"
                  >
                    <v-col class="text-center" cols="12">
                      <h4 style="font-style: italic;color: orange">{{ curClub[0].welcome }}</h4>
                    </v-col>
                  </v-row>
                </v-img>
              </div>
            </v-tab-item>
            <v-tab-item>
              <v-row style="margin-left: 10px;margin-top: 10px">
                <v-icon color="blue">mdi-account-group</v-icon>
                <h1 style="margin-left: 10px;margin-top: 5px">查看社团内的成员</h1>
              </v-row>
              <MemberList :members="members" follow="true" text="您可以关注同社团的小伙伴"></MemberList>
              <v-btn @click="showCharts" color="blue lighten-3">查看成员男女比</v-btn>
              <v-btn @click="getPdf('#'+'mychart', '男女比')" v-show="charts" color="blue lighten-3"
                     style="margin-left: 10px">导出男女比PDF
              </v-btn>
              <v-btn @click="showChartsBar" style="margin-left: 20px" color="blue lighten-3">查看成员分布</v-btn>
              <v-btn @click="getPdf('#'+'mychart2', '成员分布')" v-show="chartsBar" color="blue lighten-3"
                     style="margin-left: 10px">导出成员分布PDF
              </v-btn>
              <v-btn @click="toExcel" style="margin-left: 20px" color="blue lighten-3" v-show="isMaster">导出成员数据</v-btn>
              <v-btn style="margin-left: 20px" color="blue lighten-3" v-show="isMaster">
                Excel上传至数据库
                <input
                    type="file"
                    accept=".xls,.xlsx"
                    class="upload-file"
                    @change="Excel($event)"/>
              </v-btn>
              <v-row style="margin-top: 20px;margin-left: 20px">
                <div v-show="charts" style="width:500px;height:500px" id="mychart"></div>
                <div v-show="chartsBar" style="width:500px;height:500px;float:right;" id="mychart2"></div>
              </v-row>
            </v-tab-item>
            <v-tab-item>
              <ActivityList :activities="activities" :has-joined="hasJoined" text="查看社团的活动"></ActivityList>
            </v-tab-item>
            <v-tab-item>
              <v-row style="margin-left: 10px;margin-top: 10px">
                <v-icon color="blue">mdi-clipboard-check-multiple-outline</v-icon>
                <h1 style="margin-left: 10px;margin-top: 5px">查看社团公告</h1>
              </v-row>
              <NoticeList :notices="notices" :is-master="isMaster"></NoticeList>
            </v-tab-item>
            <v-tab-item>
              <v-row style="margin-left: 10px;margin-top: 10px">
                <v-icon color="blue">mdi-chat-plus</v-icon>
                <h1 style="margin-left: 10px;margin-top: 10px">和社团的小伙伴一起尽情讨论吧！</h1>
              </v-row>
              <PostList :posts="posts" :is-master="isMaster"></PostList>
            </v-tab-item>
            <v-tab-item>
              <PublishPost :club-id="this.$router.history.current.params.id"></PublishPost>
            </v-tab-item>
          </v-tabs>
        </v-app>
      </el-main>
    </el-container>
  </el-container>
</template>

<script>
import MyHeader from "@/components/MyHeader";
import SideBar from "@/components/SideBar";
import ClubList from "@/components/ClubList";
import MemberList from "@/components/MemberList";
import ActivityList from "@/components/ActivityList";
import NoticeList from "@/components/NoticeList";
import * as echarts from "echarts";
import PostList from "@/components/PostList";
import PublishPost from "@/components/PublishPost";
import Qs from "qs";
import transform from "@/components/exportToExcel";
import * as XLSX from "xlsx";

export default {
  name: "MyClub",
  components: {PostList, PublishPost, NoticeList, ActivityList, MemberList, ClubList, MyHeader, SideBar},
  data() {
    /*
    DO: 前端容器，curClub是当前的社团【挂载和路由更新时获取】，注意是数组格式，但只含一个元素
     members是当前社团的所有成员，activities是当前社团的所有活动，notices是当前社团的所有公告
     */
    return {
      hasJoined:false,
      isMaster: false,
      club: {},
      myClubList: [],
      clubList: [],
      curClub: [{
        id: 1,
        name: "凌峰社",
        time: '2010.11.11',
        type: "体育",
        description: "是一个以攀岩、暑期登山、科考，以及其他户外活动为特色的北航“明星社团”",
        num: 200,
        level: 5,
        cover: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        welcome: "你好",
        welcome_image: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
      }],
      members: [{
        user_id: "20373021",
        real_name: "陈俊杰",
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        label: "社长",
        following: true,
      }, {
        user_id: "123",
        real_name: "蒋博文",
        avatar: 'https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg',
        label: "副社长",
        following: false,
      }, {
        user_id: "124",
        real_name: "陈楚岩",
        avatar: 'https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg',
        label: "办公室部长",
        following: true,
      }],
      activities: [{
        event_id: 1,
        club_name: "凌峰社",
        club_face: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        title: "黄花梁——龙门涧探险",
        intro:
            `黄花梁位于北京市门头沟区清水镇梨园岭村北，呈南—北走向，全长约5千米，南接梨园岭，北达东灵山前往椴木沟的垭口，由数座缓坡山峰组成，最高海拔1850米左右。山梁之上，亚高山草甸漫山遍野，郁郁葱葱。每到初夏，忘忧草盛开，金黄色的花朵装点着翠绿的青草，铺满5公里的山梁，故得名“黄花梁”。

龙门涧分为东西两涧，各绵延十余里。由于这里聚集了我国几类著名风景区的景色，诸如“三峡之气势”、“桂林之秀美”、“匡庐之飞瀑”、“黄山之叠泉”，都可以在这里看到缩影，因此，龙门涧得到了许多如“燕京小三峡”、“京西小桂林”、“京西小黄山”等美誉。

路线亮点：可以看到梨园岭古村落，梨园岭长城遗址，保存比较完整的无编号敌楼，登顶有大片开阔的亚高山草甸观景。

车程三小时左右可从学院路到达龙门涧

路线为：燕家台村——龙门涧——黄花梁——梨园岭

全长18km
累计爬升1500m
最高海拔（黄花梁山顶）1850m

总结
风景：★★★★☆（在我走过的路线里仅次于京门铁路）

难度：★★★★☆（排除掉下雨路滑的因素，最多三星，难度主要在第一天）

强度：★★★☆☆（18km能走一天线，爬升有点急，对大多数人来说强度大了，对山队来说还不太够）

体验：★★★★☆（遇上雨天和寒潮还算不错，高海拔扎营还是一如既往的痛苦，但是风景好，路线也挺有趣，人更有趣）

没想到第一次领队就是探线，出发前看着路线极其自信，但是实际上对难度的估计有很大的偏差，这根本不是一条休闲线（还好没让萌叔带妹子来）！好在所有人都比我更有经验，更加强大，以及一学期以来磨合出的默契，所以没有出什么大的问题，让我能顺利的把这条线带下来，也感谢大家一路对菜鸡领队的帮助和包容。原以为能给社团探一条新的休闲线，现在看来只能留给明年的山队走了。山顶的海拔较高，可以体验被大风吹的折磨，强度适中，也有一定难度，适合山队前期出两天线。

最后感谢耀姐、阿传、李昀、冠阳学长、郭哥、文慧、语诚、帅帅对本次探线的大力支持`,
        time: "2022-09-25",
        real_name: "陈俊杰",
        apply_time: "2022-10-01~2022-12-18",
        begin_time: "2022-12-10~2022-12-18",
        member_count: 12,
        limit: 24,
        score: 1,
        avg_score: 2,
        like: 12,
        dislike: 10,
        show: false,
      }, {
        event_id: 2,
        club_name: "凌峰社",
        club_face: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        title: "hhhh",
        intro: "xxx",
        time: "2022-09-25 10:01:06",
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        real_name: "陈俊杰",
        apply_time: "2022-10-01 12:00:00~2022-12-18 13:01:20",
        begin_time: "2022-12-10 12:00:00~2022-12-18 13:01:20",
        member_count: 12,
        limit: 24,
        score: 1,
        like: 12,
        dislike: 10,
        show: false,
      }, {
        event_id: 3,
        club_name: "凌峰社",
        club_face: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        title: "hhhh",
        intro: "xxx",
        time: "2022-09-25",
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        real_name: "陈俊杰",
        apply_time: "2022-10-01~2022-12-18",
        begin_time: "2022-12-10~2022-12-18",
        member_count: 12,
        limit: 24,
        score: 1,
        like: 12,
        dislike: 10,
        show: false,
      }, {
        event_id: 4,
        club_name: "凌峰社",
        club_face: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        title: "hhhh",
        intro: "xxx",
        time: "2022-09-25",
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        real_name: "陈俊杰",
        apply_time: "2022-10-01~2022-12-18",
        begin_time: "2022-12-10~2022-12-18",
        member_count: 12,
        limit: 24,
        score: 1,
        like: 12,
        dislike: 10,
        show: false,
      }, {
        event_id: 5,
        club_name: "凌峰社",
        club_face: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        title: "hhhh",
        intro: "xxx",
        time: "2022-09-25",
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        real_name: "陈俊杰",
        apply_time: "2022-10-01~2022-12-18",
        begin_time: "2022-12-10~2022-12-18",
        member_count: 12,
        limit: 24,
        score: 1,
        like: 12,
        dislike: 10,
        show: false,
      }],
      notices: [
        {
          notice_id: 1,
          title: "社长换届1",
          content: "卸职：张三，任命：李四",
          user_id: "20373021",
        }, {
          notice_id: 2,
          title: "社长换届2",
          content: "卸职：张三，任命：李四",
          user_id: "20373021",
        }, {
          notice_id: 3,
          title: "社长换届3",
          content: "卸职：张三，任命：李四",
          user_id: "20373021",
        }, {
          notice_id: 4,
          title: "社长换届4",
          content: `卸职：
经理事会商定，同意原北京航空航天大学凌峰社社长郭伟恒辞去社长一职，感谢他一年来为凌峰社所做的贡献，并祝愿他在以后的学习工作中发扬凌峰精神，向上攀登，仰望最辽阔的天空。
任命：
经理事会商定，任命李书实担任凌峰社社长一职，希望他在新的一年中带领凌峰社向高攀怀抱希望，让凌峰的旗帜在顶峰飘扬！`,
          user_id: "20373021",
        }
      ],
      posts: [{
        post_id: 1,
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        user_name: "陈俊杰",
        title: "自动化测试",
        like: 22,
        dislike: 0,
        time: "2022-12-20 12:40:20",
        content: "省流不看版 ..."
      }, {
        post_id: 2,
        avatar: "https://fuss10.elemecdn.com/e/5d/4a731a90594a4af544c0c25941171jpeg.jpeg",
        user_name: "陈俊杰",
        title: "自动化测试",
        like: 22,
        dislike: 0,
        time: "2022-10-15 10:04:20",
        content: "66666666666666666666666666666666666666666666666666666666666666666666666"
      }],
      charts: false,
      chartsBar: false,
      firstClick: true,
      firstClickBar: true,
      htmlTitle: '男女比文件名'
    }
  },
  methods: {
    getOneClub(id) {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_one_club",
          Qs.stringify({
            club_id: id,
            user_id: localStorage.getItem('user_id')
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.club = res.data.club;
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getAllClubs() {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_all_clubs",
      ).then((res) => {
        if (res.data.code === 0) {
          this.clubList = res.data.club_list;
          let id = Number(this.$router.history.current.params.id);
          console.log("mounted id", id);
          this.curClub = this.clubList.filter((club) => {
            return club.club_id === id
          })
          this.isMaster = this.curClub[0].master_id === localStorage.getItem('user_id')
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getClubList(club_id) {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_club_list",
          Qs.stringify({
            jwt: {
              'code': localStorage.getItem('code'),
              'user_id': localStorage.getItem('user_id'),
              'time': localStorage.getItem('time')
            }
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.myClubList = res.data.club_list;
          let id = Number(club_id);
          console.log("mounted id", id);
          console.log(this.myClubList)
          this.curClub = this.myClubList.filter((club) => {
            return club.club_id === id
          })
          this.hasJoined = this.curClub.length > 0
          console.log("curClub is ", this.curClub);
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getMembers(club_id) {
      // console.log(this.curClub[0].club_id)
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_club_members",
          Qs.stringify({
            club_id: club_id,
            user_id: localStorage.getItem('user_id')
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.members = res.data.member_list;
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getActivities(club_id) {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_club_events",
          Qs.stringify({
            club_id: club_id,
            user_id: localStorage.getItem('user_id')
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.activities = res.data.event_list;
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getNotices(club_id) {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_club_notices",
          Qs.stringify({
            club_id: club_id
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.notices = res.data.notice_list;
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    getPosts(club_id) {
      this.$axios.post(
          "http://127.0.0.1:8000/api/get_club_posts",
          Qs.stringify({
            club_id: club_id,
            user_id: localStorage.getItem('user_id'),
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.posts = res.data.post_list;
          console.log(this.posts)
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
    },
    showChartsBar() {
      if (this.firstClickBar) {
        this.initEchartsBar();
        this.firstClickBar = false;
      }
      this.chartsBar = !this.chartsBar;
    },
    showCharts() {
      if (this.firstClick) {
        this.initEcharts()
        this.firstClick = false
      }
      this.charts = !this.charts
    },
    initEchartsBar() {
      const chartDom = document.getElementById('mychart2');
      const myChart = echarts.init(chartDom);
      console.log(this.members[0].institute)
      let dataName = ["交通学院", "自动化学院", "经济管理学院", "数学学院", "计算机学院", "可靠性与系统工程学院", "软件学院"];
      let dataAxis = ["交通", "自动化", "经管", "数学", "计算机", "可靠性", "软件"];
      let data = [0, 0, 0, 0, 0, 0, 0];
      for (let i = 0;i < this.members.length;i++){
        for (let j = 0;j < 7;j++) {
          if (this.members[i].institute === dataName[j]) {
            data[j]++
            break
          }
        }
      }
      let yMax = 500;

      let dataShadow = [];
      for (let i = 0; i < data.length; i++) {
        dataShadow.push(yMax);
      }
      const option = {
        title: {
          text: '成员分布柱状图（人数-院系）',
        },
        xAxis: {
          type: 'category',
          splitLine: {show: false},
          data: dataAxis,
          axisTick: {
            show: false
          },
          axisLine: {
            show: false
          },
          z: 10
        },
        yAxis: {
          axisLine: {
            show: false
          },
          axisTick: {
            show: false
          },
          axisLabel: {
            color: '#999'
          }
        },
        dataZoom: [
          {
            type: 'inside'
          }
        ],
        series: [
          {
            name: "总人数",
            type: 'bar',
            showBackground: true,
            itemStyle: {
              color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                {offset: 0, color: '#83bff6'},
                {offset: 0.5, color: '#188df0'},
                {offset: 1, color: '#188df0'}
              ])
            },
            emphasis: {
              itemStyle: {
                color: new echarts.graphic.LinearGradient(0, 0, 0, 1, [
                  {offset: 0, color: '#2378f7'},
                  {offset: 0.7, color: '#2378f7'},
                  {offset: 1, color: '#83bff6'}
                ])
              }
            },
            data: data,
            label: {
              show: true,
              position: 'inside'
            }
          }
        ]
      };
      const zoomSize = 6;
      myChart.on('click', function (params) {
        console.log(dataAxis[Math.max(params.dataIndex - zoomSize / 2, 0)]);
        myChart.dispatchAction({
          type: 'dataZoom',
          startValue: dataAxis[Math.max(params.dataIndex - zoomSize / 2, 0)],
          endValue:
              dataAxis[Math.min(params.dataIndex + zoomSize / 2, data.length - 1)]
        });
      });

      option && myChart.setOption(option);
    },
    initEcharts() {
      let male = this.members.filter((member) => {
        return member.sex === '男'
      })
      const option = {
        tooltip: {
          trigger: 'item'
        },
        legend: {
          top: '5%',
          left: 'center'
        },
        series: [
          {
            name: 'Access From',
            type: 'pie',
            radius: ['40%', '70%'],
            avoidLabelOverlap: false,
            label: {
              show: true,
              position: 'inside'
            },
            emphasis: {
              label: {
                show: true,
                fontSize: '40',
                fontWeight: 'bold'
              }
            },
            labelLine: {
              show: false
            },
            data: [
              {value: male.length, name: '男'},
              {value: this.members.length - male.length, name: '女'},
            ]
          }
        ]
      };
      const myChart = echarts.init(document.getElementById("mychart"));// 图标初始化
      myChart.setOption(option);// 渲染页面
      //随着屏幕大小调节图表
      window.addEventListener("resize", () => {
        myChart.resize();
      });
    },
    toExcel() {
      /*
      DO: 输出的数据来源
       */
      transform(this.members, "成员列表", this.callback);
    },
    callback(info) {
      console.log(info)
    },
    Excel(e) {
      let that = this
      // 错误情况判断
      const files = e.target.files
      if (files.length <= 0) {
        return false;
      } else if (!/\.(xls|xlsx)$/.test(files[0].name.toLowerCase())) {
        this.$message({
          message: "上传格式不正确，请上传xls或者xlsx格式",
          type: "warning"
        });
        return false
      } else {
        that.upload_file = files[0].name
      }
      // 读取表格
      const fileReader = new FileReader()
      fileReader.onload = ev => {
        try {
          const data = ev.target.result;
          const workbook = XLSX.read(data, {
            type: "binary"
          })
          // 读取第一张表
          const wsname = workbook.SheetNames[0]
          const ws = XLSX.utils.sheet_to_json(workbook.Sheets[wsname])
          // 打印 ws 就可以看到读取出的表格数据
          // 定义一个新数组，存放处理后的表格数据
          that.lists = []
          ws.forEach(item => {
            console.log(item);
            that.lists.push({
              // 对ws进行处理后放进lists内
              /*
              DO: 这里可以对原表格进行一定的处理
               */
              item
            })
          })
          // 调用方法将lists数组发送给后端
          this.submit_form(that.lists)
        } catch (e) {
          return false
        }
      }
      fileReader.readAsBinaryString(files[0])
    },
    submit_form(data) {
      // 在这里发送数据
      /*
      DO:这里会获取data，（格式为json），将其写入数据库
       */
      this.$axios.post(
          "http://127.0.0.1:8000/api/join_club_bulk",
          Qs.stringify({
            data: data,
            length: data.length,
            club_id: this.$router.history.current.params.id,
          })
      ).then((res) => {
        if (res.data.code === 0) {
          this.$message.success("批量添加社员成功");
          this.getMembers(this.$router.history.current.params.id);
        } else this.$notify.error(res.data.message)
      }).catch((error) => {
        console.log(error)
      })
      console.log(data[0].item.user_id)
    },
  },
  mounted() {
    // this.getOneClub(this.$router.history.current.params.id);
    let club_id = this.$router.history.current.params.id;
    console.log("here mounted");
    this.getClubList(club_id);
    this.getMembers(club_id);
    this.getActivities(club_id);
    this.getNotices(club_id);
    this.getPosts(club_id);
    this.getAllClubs(club_id);
  },
  beforeRouteUpdate(to, from, next) {
    console.log("update now");
    let id = Number(to.params.id);
    /*
    DO: curClub应该是一个数组
     */
    console.log("here!!!!");
    console.log(id);
    console.log(this.clubList);
    this.curClub = this.clubList.filter((club) => {
      return club.club_id === id
    })
    //this.getClubList();
    this.getMembers(id);
    this.getActivities(id);
    this.getNotices(id);
    this.getPosts(id);
    this.getAllClubs(id);

    console.log(this.curClub);
    next();
  },

}
</script>

<style scoped>
.side-bar {
  width: 65px;
  transition: width 0.5s;
  -moz-transition: width 0.5s;
  -webkit-transition: width 0.5s;
  -o-transition: width 0.5s;
}

.side-bar:hover {
  width: 150px;
}

.upload-file {
  font-size: 20px;
  opacity: 0;
  position: absolute;
  left: 0;
  top: 0;
  filter: alpha(opacity=0);
  width: 100%;
}
</style>
