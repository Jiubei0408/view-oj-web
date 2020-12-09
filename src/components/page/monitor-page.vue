<template>
  <div>
    <edit-problemset-dialog
        :user_list="user_list"
        :contest_oj_list="contest_oj_list"
        :oj_list="oj_list" ref="editDialog"/>
    <div class="title">
      <b>The monitor of contests and problem sets</b>
    </div>
    <el-button-group class="horizontal-center">
      <el-tooltip effect="dark" content="Edit(todo)" placement="top">
        <el-button round><i class="el-icon-edit"></i></el-button>
      </el-tooltip>
      <el-tooltip effect="dark" content="Refresh All" placement="top">
        <el-button round><i class="el-icon-refresh"></i></el-button>
      </el-tooltip>
    </el-button-group>
    <el-container :v-loading="true" style="margin-top: 20px">
      <p v-if="category_list.length === 0" class="horizontal-center">暂无数据</p>
      <el-aside>
        <el-menu :default-active="activeIndex" @select="menuSelected">
          <el-submenu v-for="category in category_list" :key="category.id" :index="category.id.toString()">
            <template slot="title">
              <span>{{ category.name }}</span>
            </template>
            <el-menu-item :index="category.id +'-total'">总览</el-menu-item>
            <el-menu-item v-for="series in category.series" :key="series.id"
                          :index="category.id + '-' + series.id">
              {{ series.name }}
            </el-menu-item>
          </el-submenu>
        </el-menu>
      </el-aside>
      <el-main v-if="!isRefresh">
        <el-tabs type="card">
          <el-tab-pane label="榜单">
            todo
          </el-tab-pane>
        </el-tabs>
      </el-main>
    </el-container>
  </div>
</template>

<script>
// import MonitorTable from '@/components/pageitem/monitor-table'
import EditProblemsetDialog from "@/components/pageitem/edit-problemset-dialog";

export default {
  name: 'monitor-page',
  components: {
    EditProblemsetDialog,
  },
  data() {
    return {
      api: this.$store.state.api,
      oj_list: [],
      contest_oj_list: [],
      user_list: [],
      category_list: [],
      loading: false,
      current: {
        category_id: '',
        series_id: ''
      },
      isRefresh: true
    }
  },
  computed: {
    isTotal() {
      return this.current.series === 'total'
    },
    activeIndex() {
      if (this.current.category_id === '') return null
      return this.current.category_id + '-' + this.current.series_id
    }
  },
  methods: {
    menuSelected(index) {
      if (this.activeIndex === index) return
      let sep = this.activeIndex.split('-')
      this.current.category_id = sep[0]
      this.current.series_id = sep[1]
      this.isRefresh = true
      this.refreshData()
    },
    refreshData() {

    },
    getcategoryList() {
      this.loading = true
      let that = this
      this.$http.get(this.api + '/v2/monitor/summary')
          .then(resp => {
            that.loading = false
            that.category_list = resp.data.data
            if (that.category_list.length > 0) {
              this.current.category_id = that.category_list[0].id
              this.current.series_id = 'total'
            }
          })
          .catch(err => {
            that.loading = false
            that.$message.error(err)
          })
    },
    getOJList() {
      let that = this
      this.$http.get(this.api + '/v2/monitor/valid_oj')
          .then(data => {
            this.oj_list = []
            for (var item of data.data.data) {
              if (item.name === 'vjudge') continue
              this.oj_list.push({
                id: item.id,
                name: item.name
              })
            }
          })
          .catch(function (error) {
            if (error.response) {
              that.$message.error(error.response.data.msg)
            }
          })
      this.$http.get(this.api + '/v2/monitor/valid_contest_oj')
          .then(data => {
            this.contest_oj_list = []
            for (let item of data.data.data) {
              this.contest_oj_list.push({
                id: item.id,
                name: item.name
              })
            }
          })
          .catch(error => {
            if (error.response) that.$message.error(error.response.data.msg)
          })
    },
    getUserList() {
      if (!this.$store.state.Userlist) this.$store.commit('updateUserlist')
      this.initUserList()
    },
    initUserList() {
      if (this.$store.state.Userlist) {
        let usermap = new Map()
        for (let item of this.$store.state.Userlist) {
          let user = item[1]
          if (user.status === 0) continue
          if (!usermap.has(user.group)) usermap.set(user.group, [])
          user.label = user.nickname
          usermap.get(user.group).push(user)
        }
        let user_list = []
        for (let item of usermap) {
          user_list.push({
            label: item[0],
            children: item[1]
          })
        }
        this.user_list = [{
          label: '全部',
          children: user_list
        }]
      } else {
        setTimeout(() => {
          this.initUserList()
        }, 500)
      }
    }
  },
  created() {
    document.title = "Monitor - viewOJ"
    this.getcategoryList()
    this.getOJList()
    this.getUserList()
  }
}
</script>

<style scoped>
.title {
  text-align: center;
  font-size: 25px;
  margin-top: 30px;
  margin-bottom: 30px;
}

.el-menu-item:focus:not(:hover) {
  background-color: transparent;
}
</style>
